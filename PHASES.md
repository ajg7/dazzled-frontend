# Dazzled — MVP Implementation Plan
### Incident Alerting & On-Call Platform

---

## Repo Strategy

| Repo | Name | Purpose |
|---|---|---|
| Backend | `dazzled-backend` | .NET Web API, MassTransit/RabbitMQ consumers, Hangfire, Twilio, SMTP |
| Frontend | `dazzled-frontend` | TanStack Start, React, TypeScript, Vite, Zustand, TanStack Suite |
| Infrastructure | `dazzled-infra` | Docker Compose, Grafana provisioning, env templates, CI/CD configs |

These repos are independently deployable and independently versioned. The infra repo is the **glue** — it holds the compose files that wire all three services together for local development and production. Neither the frontend nor backend repo should contain infrastructure concerns.

---

## The Core Loop (keep this sacred)

```
Ingest alert → Deduplicate into Incident → Resolve on-call → Notify →
Escalate on timeout → Ack/Resolve → Close escalation chain
```

Everything in the MVP serves this loop. Nothing is built that doesn't.

---

## Architecture Overview

```
[Grafana Alerting / generic webhook / Dazzled manual]
         │  POST /api/v1/ingest/{integrationKey}
         ▼
   dazzled-backend (ASP.NET Core API)
         │ persist Alert → MSSQL
         │ publish AlertReceived
         ▼
      RabbitMQ (MassTransit)
         │
         ▼
   AlertProcessor Consumer ── dedupe/create Incident ──► MSSQL
         │ publish IncidentTriggered
         ▼
   EscalationEngine Consumer ── resolve on-call ──► MSSQL
         │ schedule Hangfire delayed job (MSSQL-backed)
         │ publish NotificationRequested (per user per channel)
         ▼
   NotificationDispatcher Consumer
         ├──► Twilio SMS / Voice
         └──► SMTP (MailKit)
         │
         ▼ (Twilio webhook / email link / DTMF)
   API ── AckIncident / ResolveIncident ──► MSSQL
         │ cancel Hangfire escalation job

   dazzled-frontend (TanStack Start / React / TS)
         │ REST + optional SignalR ◄──► dazzled-backend
         │ Zustand (client/auth state)
         │ TanStack Query (server state)
         │ TanStack Router (file-based, type-safe)
         │ TanStack Table (incident lists)
         │ TanStack Form (policy/schedule builders)
         │ Grafana iframe embed (dashboard page)

   Grafana ◄── MSSQL data source
```

---

## Phase 0 — `dazzled-infra`: Foundation

**Goal:** Every developer can `docker compose up` and have the full backing service stack running locally with zero manual setup.

### 0.1 — Repo structure

```
dazzled-infra/
├── docker-compose.yml          # full local stack
├── docker-compose.override.yml # dev overrides (ports, volumes)
├── .env.example                # committed template, never the real .env
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── mssql.yml       # auto-provisions MSSQL data source
│   │   └── dashboards/
│   │       ├── dashboard.yml   # dashboard provider config
│   │       └── incidents.json  # pre-built incident dashboard
├── mssql/
│   └── init.sql                # initial DB + login creation (if needed)
├── scripts/
│   └── wait-for-it.sh          # health-check helper for startup ordering
└── README.md                   # local dev setup instructions
```

### 0.2 — `docker-compose.yml` services

```yaml
services:
  mssql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      ACCEPT_EULA: "Y"
      SA_PASSWORD: "${MSSQL_SA_PASSWORD}"
    ports: ["1433:1433"]
    volumes: ["mssql_data:/var/opt/mssql"]

  rabbitmq:
    image: rabbitmq:3-management
    environment:
      RABBITMQ_DEFAULT_USER: "${RABBITMQ_USER}"
      RABBITMQ_DEFAULT_PASS: "${RABBITMQ_PASS}"
    ports:
      - "5672:5672"    # AMQP
      - "15672:15672"  # Management UI (http://localhost:15672)

  grafana:
    image: grafana/grafana:latest
    environment:
      GF_SECURITY_ALLOW_EMBEDDING: "true"
      GF_AUTH_ANONYMOUS_ENABLED: "true"        # MVP only — lock down later
      GF_AUTH_ANONYMOUS_ORG_ROLE: "Viewer"
    ports: ["3001:3000"]
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning

  mailpit:
    image: axllent/mailpit
    ports:
      - "1025:1025"   # SMTP
      - "8025:8025"   # Web UI (view sent emails)

  ngrok:
    image: ngrok/ngrok:latest
    command: http host.docker.internal:5000  # points at the backend API
    environment:
      NGROK_AUTHTOKEN: "${NGROK_AUTHTOKEN}"
    ports: ["4040:4040"]  # ngrok inspector UI

volumes:
  mssql_data:
  grafana_data:
```

**Why Mailpit over Mailhog:** Mailpit is actively maintained, Mailhog is effectively abandoned.

**Why ngrok in compose:** Twilio webhooks (status callbacks, inbound SMS, TwiML voice gather) need a public HTTPS URL even in dev. Having ngrok in compose means it starts automatically and you can grab the tunnel URL from `http://localhost:4040`.

### 0.3 — `.env.example`

```env
# MSSQL
MSSQL_SA_PASSWORD=YourStrong!Password

# RabbitMQ
RABBITMQ_USER=dazzled
RABBITMQ_PASS=dazzled_secret

# Ngrok
NGROK_AUTHTOKEN=your_ngrok_token

# Twilio (set in backend env too)
TWILIO_ACCOUNT_SID=ACxxxx
TWILIO_AUTH_TOKEN=xxxx
TWILIO_FROM_PHONE=+1xxxxxxxxxx

# SMTP (Mailpit in dev)
SMTP_HOST=localhost
SMTP_PORT=1025
SMTP_FROM=alerts@dazzled.dev
```

---

## Phase 1 — `dazzled-backend`: Skeleton & Foundations

**Goal:** A running .NET solution with auth, EF Core, MassTransit, and Hangfire wired but no real business logic yet.

### 1.1 — Solution structure

```
dazzled-backend/
├── Dazzled.sln
├── src/
│   ├── Dazzled.Api/                  # ASP.NET Core Web API — controllers, middleware, DI root
│   ├── Dazzled.Application/          # Use cases, command/query handlers, IServices
│   ├── Dazzled.Domain/               # Entities, enums, domain events, value objects
│   ├── Dazzled.Infrastructure/       # EF Core, MassTransit consumers, Twilio, SMTP, Hangfire
│   └── Dazzled.Contracts/            # MassTransit message contracts (shared between producers/consumers)
├── tests/
│   ├── Dazzled.UnitTests/
│   └── Dazzled.IntegrationTests/
├── Dockerfile
├── .env.example
└── README.md
```

Keep Application free of infrastructure references. Domain has zero external dependencies. This isn't full Clean Architecture ceremony — it's just enough separation to keep the dedup logic and escalation resolution unit-testable without a running database.

### 1.2 — Key NuGet packages

```xml
<!-- Dazzled.Api -->
Microsoft.AspNetCore.Authentication.JwtBearer
Swashbuckle.AspNetCore
Hangfire.AspNetCore
Hangfire.SqlServer
Serilog.AspNetCore

<!-- Dazzled.Infrastructure -->
Microsoft.EntityFrameworkCore.SqlServer
Microsoft.EntityFrameworkCore.Design
MassTransit
MassTransit.RabbitMQ
MassTransit.EntityFrameworkCore       <!-- outbox pattern -->
Twilio
MailKit
Polly
BCrypt.Net-Next

<!-- Dazzled.UnitTests / IntegrationTests -->
xunit
Microsoft.AspNetCore.Mvc.Testing      <!-- WebApplicationFactory -->
FluentAssertions
NSubstitute
```

### 1.3 — `Program.cs` wiring order

```
1. Add Serilog
2. Add DbContext (MSSQL connection string from env)
3. Add ASP.NET Identity-lite (just JWT, no Identity framework overhead)
4. Add MassTransit → UseRabbitMq → scan consumers from Infrastructure assembly
5. Add Hangfire → UseSqlServerStorage (same DB, separate schema)
6. Add Polly HttpClientFactory policies (for Twilio HTTP client)
7. Add SignalR (for live incident updates)
8. Add Health Checks (MSSQL, RabbitMQ)
9. Map controllers, Swagger, Hangfire dashboard, health endpoints
10. On startup: run pending EF migrations (env-gated)
```

### 1.4 — JWT auth (hand-rolled, no Identity overhead)

Two roles for MVP: **Admin** (configures everything) and **Responder** (acks/resolves incidents).

```csharp
// In Dazzled.Application
public interface ITokenService
{
    string GenerateToken(User user);
    ClaimsPrincipal? ValidateToken(string token);
}
// Signed with HMAC-SHA256 key from env, 8h expiry
// Claims: sub (userId), email, role
```

`POST /api/v1/auth/login` → validates BCrypt hash → returns `{ token, user }`.

### 1.5 — EF Core DB Schema (migrations from Domain entities)

```
Users               — Id, Name, Email, PhoneE164, PasswordHash, Role, CreatedAtUtc
Teams               — Id, Name
TeamMembers         — TeamId, UserId

Services            — Id, Name, IntegrationKey (Guid), EscalationPolicyId, TeamId
EscalationPolicies  — Id, Name, TeamId
EscalationSteps     — Id, PolicyId, StepOrder, TimeoutMinutes
EscalationTargets   — Id, StepId, TargetType (User|Schedule), TargetId

OnCallSchedules     — Id, Name, TeamId, RotationType (Weekly)
ScheduleMembers     — Id, ScheduleId, UserId, RotationOrder
ScheduleOverrides   — Id, ScheduleId, UserId, StartsAtUtc, EndsAtUtc

Alerts              — Id, ServiceId, Fingerprint, RawPayload (nvarchar(max)), ReceivedAtUtc
Incidents           — Id, ServiceId, Title, Severity, Status, DedupKey,
                       CurrentStepOrder, HangfireJobId,
                       TriggeredAtUtc, AckedAtUtc, ResolvedAtUtc, AckedByUserId
IncidentEvents      — Id, IncidentId, EventType, ActorUserId, Note, OccurredAtUtc
Notifications       — Id, IncidentId, UserId, Channel (SMS|Voice|Email),
                       Status (Pending|Sent|Delivered|Failed|Acked),
                       ProviderMessageId, SentAtUtc, AckedAtUtc
```

**Fingerprint / DedupKey is the most critical design decision.** Grafana sends a `fingerprint` field — use it verbatim as `DedupKey`. For generic webhooks, hash `ServiceId + payload.dedupKey` (caller-supplied) or fall back to `ServiceId + payload.title`. Never create two open incidents from the same `ServiceId + DedupKey`.

### 1.6 — MassTransit message contracts (`Dazzled.Contracts`)

```csharp
public record AlertReceived(Guid AlertId, Guid ServiceId, string DedupKey, string Fingerprint);
public record IncidentTriggered(Guid IncidentId, Guid ServiceId, Guid PolicyId, int StepOrder);
public record NotificationRequested(Guid IncidentId, Guid UserId, NotificationChannel Channel);
// Channel: SMS | Voice | Email
```

These live in `Dazzled.Contracts` so both producer (API) and consumer (Infrastructure) reference the same types without circular dependencies.

### 1.7 — MassTransit outbox

Configure the EF Core outbox from day one:

```csharp
mt.AddEntityFrameworkOutbox<DazzledDbContext>(o =>
{
    o.UseSqlServer();
    o.UseBusOutbox();
});
```

This ensures "write Incident + publish event" is one atomic DB transaction. A crash between the two cannot silently drop a page — which is the most dangerous single failure mode in this system.

---

## Phase 2 — `dazzled-backend`: CRUD Foundations

**Goal:** All configuration entities are readable and writable via REST API. Frontend can configure the entire system before the alert pipeline is built.

### 2.1 — Endpoints to build

```
POST   /api/v1/auth/login
GET    /api/v1/auth/me

GET    /api/v1/users
POST   /api/v1/users
PUT    /api/v1/users/{id}

GET    /api/v1/teams
POST   /api/v1/teams
POST   /api/v1/teams/{id}/members

GET    /api/v1/services
POST   /api/v1/services              ← auto-generates IntegrationKey
GET    /api/v1/services/{id}
PUT    /api/v1/services/{id}
GET    /api/v1/services/{id}/integration-key   ← returns full webhook URL

GET    /api/v1/escalation-policies
POST   /api/v1/escalation-policies
PUT    /api/v1/escalation-policies/{id}
POST   /api/v1/escalation-policies/{id}/steps
PUT    /api/v1/escalation-policies/{id}/steps/{stepId}

GET    /api/v1/schedules
POST   /api/v1/schedules
PUT    /api/v1/schedules/{id}/members        ← sets ordered rotation
POST   /api/v1/schedules/{id}/overrides
DELETE /api/v1/schedules/{id}/overrides/{overrideId}
GET    /api/v1/schedules/{id}/oncall-now     ← resolves current on-call user(s)

GET    /api/v1/incidents
GET    /api/v1/incidents/{id}
GET    /api/v1/incidents/{id}/timeline       ← IncidentEvents
POST   /api/v1/incidents/{id}/ack
POST   /api/v1/incidents/{id}/resolve
POST   /api/v1/incidents/{id}/note
```

### 2.2 — On-call resolution logic

Keep it simple for MVP: one rotation type (weekly round-robin).

```
1. Check ScheduleOverrides for now — return override user if found
2. Calculate weeks since schedule epoch → index into ordered ScheduleMembers
3. Return that user
```

No multi-layer schedules. No business-hours restrictions. Deliberate v2 cut.

---

## Phase 3 — `dazzled-backend`: Ingestion Pipeline

**Goal:** An external system can fire a webhook and an Incident appears in the DB.

### 3.1 — Ingest endpoint

```
POST /api/v1/ingest/{integrationKey}
     Headers: Content-Type: application/json
              X-Dazzled-Token: {optional bearer for extra security}
```

Steps inside the handler:
1. Look up `Service` by `IntegrationKey` — 404 if not found.
2. Detect payload shape: Grafana unified alerting format, or generic format.
3. For Grafana: iterate `alerts[]` array, create one `Alert` row per entry.
4. For generic: create one `Alert` row.
5. Rate-limit by `ServiceId` (simple in-memory sliding window, 100 alerts/minute is fine for MVP).
6. Persist `Alert`, publish `AlertReceived`. Return `202 Accepted` immediately — don't block on processing.

**Grafana payload mapping:**
```json
// Grafana sends:
{ "alerts": [{ "fingerprint": "abc123", "status": "firing",
               "labels": { "alertname": "HighCPU", "severity": "critical" },
               "annotations": { "summary": "CPU above 90%" } }] }

// Map to Alert:
Fingerprint = alert.fingerprint
DedupKey    = ServiceId + ":" + alert.fingerprint
Title       = alert.annotations.summary ?? alert.labels.alertname
Severity    = alert.labels.severity
```

**Generic payload:**
```json
{ "title": "DB connection pool exhausted",
  "severity": "critical",
  "dedupKey": "db-pool-prod",
  "description": "Free connections: 0" }
```

### 3.2 — `AlertReceivedConsumer`

```
1. Load Alert from DB
2. Query open Incidents for ServiceId WHERE DedupKey = alert.DedupKey AND Status != Resolved
3. IF found: add IncidentEvent("alert_added"), update LastSeenAtUtc, done
4. IF not found:
   a. Create Incident (Status=Triggered, CurrentStepOrder=1)
   b. Add IncidentEvent("triggered")
   c. Publish IncidentTriggered(incidentId, serviceId, policyId, stepOrder=1)
   d. (outbox handles atomicity)
```

---

## Phase 4 — `dazzled-backend`: Escalation Engine

**Goal:** When an incident triggers, the right people get notified, and if they don't respond, the next step fires.

### 4.1 — `IncidentTriggeredConsumer`

```
1. Load EscalationPolicy + step at stepOrder
2. For each target in this step:
   a. If target is User → resolve directly
   b. If target is Schedule → call OnCallResolver.GetCurrentOnCall(scheduleId)
3. For each resolved User, for each channel they have configured:
   a. Create Notification row (Status=Pending)
   b. Publish NotificationRequested(incidentId, userId, channel)
4. Enqueue Hangfire delayed job:
   BackgroundJob.Schedule<IEscalationService>(
       s => s.CheckAndEscalate(incidentId, stepOrder),
       TimeSpan.FromMinutes(step.TimeoutMinutes)
   )
5. Store the returned Hangfire job ID on the Incident row (HangfireJobId)
```

### 4.2 — `EscalationService.CheckAndEscalate(incidentId, stepOrder)`

```
1. Load Incident — if Status is Acked or Resolved, return (no-op)
2. If Incident.CurrentStepOrder != stepOrder, return (already escalated past here)
3. Load next step in policy
4. If no next step: publish IncidentTriggered with a special "escalation_exhausted"
   flag — create a final IncidentEvent and stop. Don't loop forever.
5. If next step exists:
   a. Update Incident.CurrentStepOrder to next step
   b. Add IncidentEvent("escalated", note="Step 2")
   c. Publish IncidentTriggered(incidentId, policyId, nextStepOrder)
```

### 4.3 — Ack flow

```
POST /api/v1/incidents/{id}/ack
1. Set Incident.Status = Acked, AckedAtUtc, AckedByUserId
2. Add IncidentEvent("acked")
3. BackgroundJob.Delete(incident.HangfireJobId)  ← stops escalation
4. Update Notifications for this user to Status=Acked
5. Broadcast via SignalR hub: incidentUpdated(incidentId)
```

---

## Phase 5 — `dazzled-backend`: Notification Dispatch

**Goal:** Notifications actually reach people via SMS, Voice, and Email.

### 5.1 — `NotificationRequestedConsumer` (branches by channel)

Wrap every external call in Polly: 3 retries with exponential backoff (2s, 4s, 8s). Transient Twilio/SMTP failures must not silently drop a page.

**SMS (Twilio):**
```csharp
var message = await MessageResource.CreateAsync(
    to: new PhoneNumber(user.PhoneE164),
    from: new PhoneNumber(twilioFromPhone),
    body: $"[Dazzled] {incident.Title} — {incident.Severity.ToUpper()}\n" +
          $"Service: {service.Name}\n" +
          $"Reply ACK to acknowledge."
);
// Store message.Sid as Notification.ProviderMessageId
```

**Voice (Twilio):**
```csharp
var call = await CallResource.CreateAsync(
    to: new PhoneNumber(user.PhoneE164),
    from: new PhoneNumber(twilioFromPhone),
    url: new Uri($"{publicBaseUrl}/api/v1/twiml/incident/{incidentId}/{notificationId}")
);
```

TwiML endpoint (no auth — it's a Twilio callback):
```xml
<!-- GET /api/v1/twiml/incident/{incidentId}/{notificationId} -->
<Response>
  <Say>Dazzled alert. Incident: High CPU on production. Severity: critical.</Say>
  <Gather numDigits="1" action="/api/v1/twiml/gather/{notificationId}">
    <Say>Press 1 to acknowledge. Press 9 to repeat.</Say>
  </Gather>
</Response>
```

**Email (MailKit):**
```csharp
var message = new MimeMessage();
// HTML body includes a one-click ack link:
// /api/v1/incidents/{id}/ack-link?token={hmacToken}
// HMAC-SHA256(incidentId + notificationId + secret), 2h expiry
// No login required — responders ack from their phone's email client
```

### 5.2 — Twilio webhook endpoints

```
POST /api/v1/webhooks/twilio/status
  → Updates Notification.Status (delivered / failed / undelivered)
  → Validate X-Twilio-Signature header

POST /api/v1/webhooks/twilio/sms
  → Parse Body parameter for "ACK" or "RESOLVE"
  → Look up open Notification by From phone number
  → Call AckIncident or ResolveIncident application service
  → Return TwiML: <Response><Message>Acknowledged. Incident #{id}.</Message></Response>

POST /api/v1/twiml/gather/{notificationId}
  → Parse Digits: "1" → Ack, "9" → re-read
```

### 5.3 — Email one-click ack

```
GET /api/v1/incidents/{id}/ack-link?token={hmacToken}
  → Validate HMAC, check expiry
  → Call AckIncident
  → Return 200 HTML: "Incident #{id} acknowledged. You can close this tab."
```

---

## Phase 6 — `dazzled-backend`: Grafana Integration

**Goal:** Grafana displays Dazzled's operational data without any custom plugin work.

### 6.1 — Auto-provision MSSQL data source (`dazzled-infra`)

```yaml
# grafana/provisioning/datasources/mssql.yml
apiVersion: 1
datasources:
  - name: Dazzled MSSQL
    type: mssql
    url: mssql:1433
    database: dazzled
    user: grafana_readonly
    secureJsonData:
      password: "${GRAFANA_DB_PASSWORD}"
    jsonData:
      encrypt: disable
```

Create a `grafana_readonly` SQL login in `dazzled-infra/mssql/init.sql` with SELECT on the `dbo` schema only.

### 6.2 — Pre-built dashboard panels

Commit a `incidents.json` Grafana dashboard to the infra repo. Key panels:

| Panel | Query |
|---|---|
| Incidents over time | `COUNT(*) GROUP BY DATETRUNC(hour, TriggeredAtUtc)` |
| Open incident count | `COUNT(*) WHERE Status = 'Triggered'` |
| MTTA (mean time to ack) | `AVG(DATEDIFF(second, TriggeredAtUtc, AckedAtUtc))` |
| MTTR (mean time to resolve) | `AVG(DATEDIFF(second, TriggeredAtUtc, ResolvedAtUtc))` |
| Notifications by channel | `COUNT(*) GROUP BY Channel` |
| Notification failure rate | `SUM(CASE WHEN Status='Failed' THEN 1 ELSE 0 END) / COUNT(*)` |
| On-call load per user | `COUNT(*) GROUP BY AckedByUserId JOIN Users` |

### 6.3 — Embed in frontend

Grafana runs on port 3001. The React dashboard page will embed panels via `<iframe src="http://localhost:3001/d-solo/...?orgId=1&panelId=1&theme=dark">`. Anonymous viewer role is enabled in compose for MVP — lock behind SSO for production.

---

## Phase 7 — `dazzled-frontend`: Foundation

**Goal:** A running TanStack Start app with routing, auth state, API client, and all tooling wired.

### 7.1 — Scaffold & structure

```bash
npx @tanstack/cli@latest create dazzled-frontend
cd dazzled-frontend
pnpm add zustand @tanstack/react-query @tanstack/react-table @tanstack/react-form
pnpm add -D typescript @types/react @types/node
```

```
dazzled-frontend/
├── app/
│   ├── routes/
│   │   ├── __root.tsx              # root layout, QueryClientProvider, Zustand, SignalR init
│   │   ├── _auth/
│   │   │   └── login.tsx
│   │   ├── _protected/             # all guarded routes under one layout
│   │   │   ├── index.tsx           # dashboard (Grafana embed + summary cards)
│   │   │   ├── incidents/
│   │   │   │   ├── index.tsx       # incident list
│   │   │   │   └── $incidentId.tsx # incident detail + timeline
│   │   │   ├── services/
│   │   │   │   ├── index.tsx
│   │   │   │   └── $serviceId.tsx
│   │   │   ├── escalation-policies/
│   │   │   │   ├── index.tsx
│   │   │   │   └── $policyId.tsx
│   │   │   ├── schedules/
│   │   │   │   ├── index.tsx
│   │   │   │   └── $scheduleId.tsx
│   │   │   └── settings/
│   │   │       └── index.tsx
│   ├── components/
│   │   ├── ui/                     # primitives (Button, Badge, Input, Modal)
│   │   ├── incidents/              # IncidentRow, IncidentTimeline, AckButton
│   │   ├── schedules/              # OnCallBadge, ScheduleRotationEditor
│   │   └── layout/                 # Sidebar, TopNav, PageShell
│   ├── lib/
│   │   ├── api/
│   │   │   ├── client.ts           # fetch wrapper with auth header injection
│   │   │   └── endpoints/          # typed API functions per domain
│   │   │       ├── incidents.ts
│   │   │       ├── services.ts
│   │   │       ├── schedules.ts
│   │   │       └── auth.ts
│   │   ├── query/
│   │   │   ├── client.ts           # QueryClient singleton
│   │   │   └── keys.ts             # query key factory
│   │   └── signalr/
│   │       └── hub.ts              # SignalR connection lifecycle
│   ├── stores/
│   │   ├── authStore.ts            # Zustand: token, user, login/logout
│   │   └── uiStore.ts              # Zustand: sidebar, filters, toasts
│   └── types/
│       └── api.ts                  # TypeScript types mirroring backend DTOs
├── vite.config.ts
├── tsconfig.json
├── .env.example
└── README.md
```

### 7.2 — TypeScript configuration

```json
// tsconfig.json — strict mode, no shortcuts
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "target": "ES2022",
    "lib": ["ES2022", "DOM"],
    "moduleResolution": "bundler",
    "jsx": "react-jsx"
  }
}
```

### 7.3 — Environment variables

```env
# .env.example
VITE_API_BASE_URL=http://localhost:5000
VITE_SIGNALR_URL=http://localhost:5000/hubs/incidents
VITE_GRAFANA_URL=http://localhost:3001
```

TanStack Start server functions access these server-side; Vite exposes `VITE_*` prefixed ones to the browser.

### 7.4 — Auth guard (TanStack Router)

```typescript
// app/routes/_protected.tsx
export const Route = createFileRoute('/_protected')({
  beforeLoad: ({ context }) => {
    const { token } = useAuthStore.getState()
    if (!token) throw redirect({ to: '/login' })
  },
  component: ProtectedLayout,
})
```

### 7.5 — Zustand stores

**Auth store** (`authStore.ts`) — the only client-owned state that persists:

```typescript
interface AuthState {
  token: string | null
  user: { id: string; name: string; email: string; role: 'Admin' | 'Responder' } | null
  login: (token: string, user: AuthState['user']) => void
  logout: () => void
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      token: null,
      user: null,
      login: (token, user) => set({ token, user }),
      logout: () => set({ token: null, user: null }),
    }),
    { name: 'dazzled-auth', storage: createJSONStorage(() => sessionStorage) }
  )
)
```

Use `sessionStorage` not `localStorage` — JWTs shouldn't survive a browser close for an ops tool.

**UI store** (`uiStore.ts`) — ephemeral UI state that doesn't belong in the URL and doesn't belong in the server:

```typescript
interface UIState {
  sidebarOpen: boolean
  toggleSidebar: () => void
  incidentFilters: { status: string[]; severity: string[]; serviceId: string | null }
  setIncidentFilters: (f: Partial<UIState['incidentFilters']>) => void
  toasts: Toast[]
  addToast: (t: Omit<Toast, 'id'>) => void
  removeToast: (id: string) => void
}
```

**What Zustand is NOT used for:** Any data that comes from the server (incidents list, service configs, on-call status) lives in TanStack Query. Zustand only owns state that is either derived from user interaction with the UI or persisted auth context. If you find yourself putting API response data into Zustand, stop — that's TanStack Query's job.

### 7.6 — API client with auth injection

```typescript
// lib/api/client.ts
const API_BASE = import.meta.env.VITE_API_BASE_URL

export async function apiClient<T>(
  path: string,
  options: RequestInit = {}
): Promise<T> {
  const token = useAuthStore.getState().token
  const res = await fetch(`${API_BASE}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options.headers,
    },
  })
  if (res.status === 401) {
    useAuthStore.getState().logout()
    throw new Error('Unauthorized')
  }
  if (!res.ok) throw new Error(`API error: ${res.status}`)
  return res.json()
}
```

### 7.7 — TanStack Query setup

```typescript
// lib/query/client.ts
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,
      retry: 2,
    },
  },
})

// lib/query/keys.ts — centralized query key factory
export const queryKeys = {
  incidents: {
    all: () => ['incidents'] as const,
    list: (filters: IncidentFilters) => ['incidents', 'list', filters] as const,
    detail: (id: string) => ['incidents', id] as const,
    timeline: (id: string) => ['incidents', id, 'timeline'] as const,
  },
  services: {
    all: () => ['services'] as const,
    detail: (id: string) => ['services', id] as const,
  },
  schedules: {
    oncallNow: (id: string) => ['schedules', id, 'oncall-now'] as const,
  },
} as const
```

---

## Phase 8 — `dazzled-frontend`: Core Incident Pages

**Goal:** Responders can see, ack, and resolve incidents.

### 8.1 — Incident list (`incidents/index.tsx`)

Use **TanStack Table** for the list. Key features:
- Server-side filtering by status, severity, serviceId — pass filter state from `uiStore` into the query key so TanStack Query re-fetches automatically when filters change.
- Column sorting by `TriggeredAtUtc`, `Severity`.
- Row click navigates to detail.
- Status badge with color coding: Triggered=red, Acked=yellow, Resolved=green.

```typescript
const { incidentFilters } = useUIStore()
const { data, isLoading } = useQuery({
  queryKey: queryKeys.incidents.list(incidentFilters),
  queryFn: () => fetchIncidents(incidentFilters),
  refetchInterval: 15_000,  // poll every 15s as SignalR fallback
})

const table = useReactTable({
  data: data?.incidents ?? [],
  columns,
  getCoreRowModel: getCoreRowModel(),
  getSortedRowModel: getSortedRowModel(),
})
```

### 8.2 — Incident detail (`incidents/$incidentId.tsx`)

Sections:
1. **Header** — title, severity badge, service name, status, timestamps
2. **Action bar** — Ack button (if Triggered), Resolve button (if Triggered/Acked), Add Note
3. **Timeline** — `IncidentEvents` in chronological order: triggered, notifications sent, escalated, acked by X, note added, resolved
4. **Notifications** — table of who was notified, by what channel, delivery status, ack status

Ack mutation:
```typescript
const ackMutation = useMutation({
  mutationFn: () => ackIncident(incidentId),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: queryKeys.incidents.detail(incidentId) })
    queryClient.invalidateQueries({ queryKey: queryKeys.incidents.all() })
    addToast({ type: 'success', message: 'Incident acknowledged' })
  },
})
```

### 8.3 — Service detail (`services/$serviceId.tsx`)

The most important thing on this page: the **integration key / webhook URL**, displayed prominently in a copy-to-clipboard code block. This is what engineers paste into Grafana or their monitoring tool. Surface it immediately, don't bury it.

Also display: linked escalation policy (with link to edit), current on-call user from linked schedule.

---

## Phase 9 — `dazzled-frontend`: Configuration Pages

**Goal:** Admins can configure services, escalation policies, and on-call schedules entirely through the UI.

### 9.1 — Escalation policy builder

This is the most complex form in the app. Use **TanStack Form** with a dynamic field array for steps.

```typescript
const form = useForm({
  defaultValues: {
    name: '',
    steps: [{ timeoutMinutes: 5, targets: [{ targetType: 'User', targetId: '' }] }],
  },
  onSubmit: async ({ value }) => {
    await upsertPolicy(policyId, value)
    queryClient.invalidateQueries({ queryKey: queryKeys.services.all() })
  },
})
```

UI: vertical list of steps, each showing timeout and a target picker (user dropdown or schedule dropdown). Drag-to-reorder is a v2 concern — use Up/Down arrow buttons for MVP. "Add Step" appends a new step with defaults.

### 9.2 — On-call schedule editor

Two sub-sections:
1. **Rotation** — ordered list of users, drag to reorder (or Up/Down for MVP). Shows who is currently on-call with a highlighted badge.
2. **Overrides** — date-range picker + user picker. Shows upcoming overrides in a simple list. Delete override button.

Use **TanStack Table** for the overrides list (date, user, duration, delete action column).

### 9.3 — Shared form patterns

All configuration forms follow the same pattern:
- Render inside a `<PageShell title="..." breadcrumb={[...]}>` wrapper
- Unsaved changes warning via `form.state.isDirty` check on route change (`router.subscribe`)
- Server errors surfaced inline next to the relevant field, not only in a toast
- Optimistic UI for toggle/status changes (TanStack Query `optimisticUpdate`)

---

## Phase 10 — `dazzled-frontend`: Dashboard & Real-Time

**Goal:** The home page gives responders an immediate operational picture and updates live.

### 10.1 — Dashboard page (`index.tsx`)

```
┌─────────────────────────────────────────────────────┐
│  Summary Cards: [Open Incidents] [Acked] [On-Call]  │
├─────────────────┬───────────────────────────────────┤
│  Recent         │  Grafana Panel: Incidents/hour     │
│  Incidents      │  (iframe embed)                    │
│  (last 5)       │                                    │
├─────────────────┴───────────────────────────────────┤
│  Grafana Panel: MTTA / MTTR (7-day rolling avg)     │
└─────────────────────────────────────────────────────┘
```

The summary cards are TanStack Query-fetched counts from your own API — fast, live, no Grafana dependency. The panels below are Grafana iframes — slower to load, but rich and maintainable without frontend changes.

### 10.2 — SignalR real-time (backend side — `Dazzled.Api`)

```csharp
// Hub
public class IncidentHub : Hub
{
    // Clients join a group per team, so broadcasts are scoped
    public async Task JoinTeam(string teamId) =>
        await Groups.AddToGroupAsync(Context.ConnectionId, $"team-{teamId}");
}

// In AckIncident application service, after updating DB:
await _hubContext.Clients.Group($"team-{teamId}")
    .SendAsync("incidentUpdated", new { incidentId, status = "Acked", ackedBy });
```

**Frontend SignalR setup** (`lib/signalr/hub.ts`):
```typescript
export function createIncidentHub(token: string) {
  const connection = new HubConnectionBuilder()
    .withUrl(`${import.meta.env.VITE_SIGNALR_URL}`, {
      accessTokenFactory: () => token,
    })
    .withAutomaticReconnect()
    .build()

  connection.on('incidentUpdated', ({ incidentId }) => {
    queryClient.invalidateQueries({ queryKey: queryKeys.incidents.detail(incidentId) })
    queryClient.invalidateQueries({ queryKey: queryKeys.incidents.all() })
  })

  return connection
}
```

Initialize in `__root.tsx` after auth is confirmed. The `refetchInterval: 15_000` on the incidents list query is the fallback if SignalR disconnects — don't remove it.

---

## Phase 11 — Cross-Repo Integration & Hardening

### 11.1 — CORS (backend)

```csharp
// Dazzled.Api Program.cs
builder.Services.AddCors(o => o.AddPolicy("DazzledFrontend", p =>
    p.WithOrigins(builder.Configuration["AllowedOrigins"]!.Split(','))
     .AllowAnyHeader()
     .AllowAnyMethod()
     .AllowCredentials()  // required for SignalR
));
```

Set `AllowedOrigins=http://localhost:5173` in dev, production origin in prod. Never `AllowAnyOrigin` with `AllowCredentials`.

### 11.2 — Shared types strategy

The frontend's `app/types/api.ts` must mirror the backend's DTOs. For MVP, maintain them in sync manually. A v2 improvement is generating TypeScript types from the backend's OpenAPI spec (`openapi-typescript` CLI, run as a step in the frontend's CI against the backend's `/swagger/v1/swagger.json`).

### 11.3 — Backend testing priorities

These two areas are where a silent bug means a missed page — they deserve real test coverage:

**Dedup logic (unit test):**
```
- Two alerts with the same DedupKey → one Incident created
- Two alerts with different DedupKeys → two Incidents created
- Alert for an already-Resolved incident → new Incident created
- Alert for an Acked incident → attaches to existing Incident (your business rule choice)
```

**Escalation step computation (unit test):**
```
- Step 1, no ack → step 2 fires after timeout
- Ack before timeout → escalation job is cancelled (mock Hangfire)
- Last step, no ack → escalation_exhausted, no infinite loop
- On-call override takes precedence over rotation
```

**Integration test (`WebApplicationFactory`):**
```
- POST /ingest/{key} → alert persisted, message published to RabbitMQ
- Full flow: ingest → consumer → incident created (use in-memory MassTransit test harness)
```

### 11.4 — Backend health checks

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString)
    .AddRabbitMQ(rabbitConnectionString);

// GET /health → { status: "Healthy", checks: [...] }
```

The infra `docker-compose.yml` can poll this to gate frontend container startup.

### 11.5 — Frontend error boundaries

Wrap each route's component in a TanStack Router `errorComponent` that shows a clean "Something went wrong" screen with a retry button — this prevents a single failed query from crashing the whole app. Incidents list failing to load should not prevent the sidebar from rendering.

---

## Phase 12 — `dazzled-infra`: Full Stack Compose & Deployment

### 12.1 — Add `api` and `frontend` to compose

```yaml
# docker-compose.yml additions
  api:
    build:
      context: ../dazzled-backend
      dockerfile: Dockerfile
    environment:
      ConnectionStrings__Default: "Server=mssql;Database=dazzled;..."
      RabbitMQ__Host: rabbitmq
      Twilio__AccountSid: "${TWILIO_ACCOUNT_SID}"
      Twilio__AuthToken: "${TWILIO_AUTH_TOKEN}"
      Twilio__FromPhone: "${TWILIO_FROM_PHONE}"
      Smtp__Host: mailpit
      Smtp__Port: 1025
      Jwt__SigningKey: "${JWT_SIGNING_KEY}"
      PublicBaseUrl: "${NGROK_URL}"    # set from ngrok tunnel for Twilio webhooks
    ports: ["5000:8080"]
    depends_on:
      mssql: { condition: service_healthy }
      rabbitmq: { condition: service_healthy }

  frontend:
    build:
      context: ../dazzled-frontend
      dockerfile: Dockerfile
      args:
        VITE_API_BASE_URL: http://localhost:5000
        VITE_SIGNALR_URL: http://localhost:5000/hubs/incidents
        VITE_GRAFANA_URL: http://localhost:3001
    ports: ["5173:3000"]
    depends_on:
      - api
```

### 12.2 — Backend Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish src/Dazzled.Api/Dazzled.Api.csproj -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:9.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "Dazzled.Api.dll"]
```

### 12.3 — Frontend Dockerfile

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN npm i -g pnpm && pnpm install --frozen-lockfile
COPY . .
ARG VITE_API_BASE_URL
ARG VITE_SIGNALR_URL
ARG VITE_GRAFANA_URL
RUN pnpm build

FROM node:22-alpine
WORKDIR /app
COPY --from=build /app/.output .
EXPOSE 3000
CMD ["node", "server/index.mjs"]
```

TanStack Start outputs a Node.js server via Nitro. No Nginx needed for MVP.

### 12.4 — EF Core migrations in CI

```bash
# Add to dazzled-backend CI pipeline before deploy:
dotnet ef database update --project src/Dazzled.Infrastructure --startup-project src/Dazzled.Api
```

Don't run `Database.Migrate()` in `Program.cs` in production — it's a deployment concern, not an app startup concern.

---

## Deliberate MVP Cuts (v2 backlog)

| Feature | Why it was cut |
|---|---|
| Multi-tenant organizations | Single team is enough to prove the loop |
| Layered / multi-tier on-call schedules | Weekly round-robin + overrides handles 80% of real use |
| Slack / Teams / PagerDuty-style push notifications | Clean integration point, zero architectural change needed to add |
| Maintenance windows / alert suppression | Dedup key handles the acute version of this |
| Status pages | Separate product concern, separate UI entirely |
| SSO / SAML | JWT + roles is enough for a team-sized MVP |
| Mobile app | Mobile browser + SMS ack covers the mobile use case for MVP |
| OpenAPI → TypeScript type generation | Manual sync is fine for one team; automate when it hurts |
| Separate API / Worker processes | Run both in one process, split when you need independent scaling |

---

## Build Order Summary

| Phase | Repo | Deliverable |
|---|---|---|
| 0 | infra | Full backing stack running via `docker compose up` |
| 1 | backend | Running API: auth, migrations, MassTransit, Hangfire wired |
| 2 | backend | All CRUD endpoints for Users, Services, Policies, Schedules |
| 3 | backend | Ingest endpoint → Alert → Incident via RabbitMQ consumers |
| 4 | backend | Escalation engine with Hangfire timeouts |
| 5 | backend | Twilio SMS + Voice, SMTP email dispatch + ack webhooks |
| 6 | infra | Grafana provisioned with MSSQL data source + incident dashboard |
| 7 | frontend | TanStack Start scaffold, Zustand stores, API client, auth guard |
| 8 | frontend | Incident list + detail, ack/resolve, service detail |
| 9 | frontend | Escalation policy builder, schedule editor, user management |
| 10 | frontend | Dashboard with Grafana embed, SignalR live updates |
| 11 | all | CORS, shared types, unit + integration tests, health checks |
| 12 | infra | All three services in compose, Dockerfiles, CI migration step |
