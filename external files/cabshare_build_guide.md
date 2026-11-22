# Passenger-to-Passenger Cab-Sharing Platform

## End-to-End Build Guide: Step 1 to Deployment and Monitoring

**Project type:** Passenger-to-passenger shared-cab matching
marketplace\
**Initial launch geography:** Kochi, Kerala, India\
**Primary goal:** Help people travelling in compatible directions and
time windows find one another, form a group, agree trip details
democratically, and independently arrange a lawful taxi booking.

> **Important product boundary:** The MVP is a passenger-matching and
> coordination platform. It does not dispatch drivers, onboard taxis,
> book vehicles, collect ride fares, or settle ride payments. This
> boundary does not itself establish a legal exemption. Obtain written
> advice from a Kerala transport lawyer and clarification from the
> relevant transport authority before facilitating real paid shared
> journeys.

------------------------------------------------------------------------

# 0. Product scope and decisions

## 0.1 What we are building

A user can:

1.  Register and verify their phone number.
2.  Create an instant or scheduled travel request.
3.  Enter pickup, destination, departure window, seats, luggage, and
    sharing preferences.
4.  Discover compatible passengers based on route overlap, time, pickup
    distance, capacity, and detour constraints.
5.  Invite or accept another passenger.
6.  Form a shared group.
7.  Discuss trip details in group chat.
8.  Vote on important group decisions.
9.  Independently arrange a taxi and settle costs with other passengers
    outside the platform.
10. Record the agreed trip details and optionally review/report other
    members.

The platform should support passengers who share only part of a route.
They do not all need an identical destination.

## 0.2 Confirmed product decisions

-   Support both **instant** and **scheduled** matching.
-   Group decisions are democratic. Important trip conditions require
    approval from the group.
-   Taxi booking and payment are independent passenger decisions. The
    platform does not prescribe one booking owner or collect ride
    payments in the MVP.
-   Start with one deployable modular backend and separate worker
    processes, not microservices.
-   Use open-source components where practical and isolate external
    providers behind interfaces.

## 0.3 Explicitly out of scope for the first release

-   Driver onboarding or driver dispatch.
-   Vehicle fleet management.
-   In-app taxi booking.
-   Ride-fare collection, escrow, or payment settlement.
-   Dynamic pricing.
-   Machine-learning-based matching.
-   Native Android/iOS applications.
-   Kubernetes, Kafka, Elasticsearch, or a microservice fleet.

## 0.4 MVP success criteria

A small invited test group should be able to:

-   Sign up and verify an account.
-   Create, edit, cancel, and view a travel request.
-   Find plausible matches.
-   Accept an invitation and form a group without overfilling seats.
-   Propose and vote on important changes.
-   Exchange messages and recover missed messages after reconnecting.
-   Leave a group, report a user, and block a user.
-   Use the application on a mobile browser.
-   Understand clearly that booking and payment happen independently
    outside the platform.

------------------------------------------------------------------------

# 1. Architecture

## 1.1 Recommended architecture

Build a **modular monolith** with independently runnable API and worker
processes.

``` text
Web browser
    |
    v
Next.js frontend
    |
    | HTTPS / WebSocket
    v
FastAPI application
    |
    +--> Identity module
    +--> Travel request module
    +--> Matching module
    +--> Groups and voting module
    +--> Messaging module
    +--> Safety module
    +--> Fare-split estimation module
    +--> Notifications module
    |
    +--> PostgreSQL + PostGIS (durable source of truth)
    |
    +--> Valkey (cache, presence, ephemeral coordination)
    |
    +--> Outbox table
              |
              v
         Background workers
              |
              +--> matching jobs
              +--> notification delivery
              +--> event publishing
```

The API and worker can be deployed as separate processes from the same
codebase. They may scale independently later.

## 1.2 Why not microservices initially?

The first release needs strong consistency around group membership,
seats, votes, and agreements. A modular monolith allows these changes to
be committed in database transactions without introducing distributed
transactions and network failure modes.

Keep module boundaries clean so that a module can be extracted later if
it develops a real independent scaling or deployment requirement.

## 1.3 Technology stack

  -----------------------------------------------------------------------
  Layer                   Recommended technology  Purpose
  ----------------------- ----------------------- -----------------------
  Frontend                Next.js + TypeScript    Responsive web
                                                  application

  UI                      Tailwind CSS +          Accessible components
                          shadcn/ui               

  Backend                 Python + FastAPI +      HTTP and WebSocket API
                          Pydantic                

  Database                PostgreSQL + PostGIS    Durable data and
                                                  spatial queries

  ORM/migrations          SQLAlchemy 2 + Alembic  Persistence and schema
                                                  changes

  Cache                   Valkey                  Cache, presence, rate
                                                  limiting

  Realtime                WebSockets              Live group chat and
                                                  updates

  Event fanout            Valkey Pub/Sub          Realtime event
                                                  distribution

  Background work         Python worker +         Reliable async jobs
                          transactional outbox    

  Map display             MapLibre GL JS          Interactive maps

  Map data                OpenStreetMap           Geographic data

  Routing                 Valhalla or replaceable Road route and
                          routing API             travel-time estimates

  Reverse proxy           Caddy                   HTTPS and reverse proxy

  Containers              Docker Compose          Reproducible deployment

  Tests                   Pytest, HTTPX,          Unit, integration,
                          Playwright, k6          browser, load tests

  Monitoring              Prometheus + Grafana +  Metrics and
                          OpenTelemetry           observability

  Logs                    Structured JSON logs    Debugging and audit
                                                  trail
  -----------------------------------------------------------------------

**Database choice:** Use PostgreSQL with PostGIS as the source of truth.
Do not add MongoDB merely because chat or matching needs to be fast. Use
indexes, pagination, caching, and efficient queries first.

**Valkey note:** Use Valkey rather than assuming Redis is an
unrestricted open-source dependency. Check the exact licence of any
version and component before distribution.

## 1.4 Provider abstraction

Business logic should depend on interfaces, not vendor SDKs.

Suggested ports:

``` python
class RoutingPort(Protocol):
    async def get_route(self, origin, destination): ...

class NotificationPort(Protocol):
    async def send(self, recipient, message): ...

class CachePort(Protocol):
    async def get(self, key): ...
    async def set(self, key, value, ttl_seconds=None): ...

class EventPublisherPort(Protocol):
    async def publish(self, topic, event): ...

class OtpProviderPort(Protocol):
    async def send_otp(self, phone_number, code): ...
```

Implement adapters such as `ValhallaRoutingAdapter`,
`ExternalRoutingAdapter`, `EmailNotificationAdapter`, and
`SmsOtpAdapter`.

Select adapters through application configuration at startup. Avoid
building a dynamic third-party plugin marketplace before there is a real
need for it.

------------------------------------------------------------------------

# 2. Development environment

## 2.1 Laptop requirements

An i3 laptop can build the MVP. With 8 GB RAM, run only the services
needed for the current task. 16 GB RAM is more comfortable for Docker,
the browser, frontend, backend, and database together.

Do not run a full self-hosted routing engine or a large map extract on
the development laptop initially. Use a routing adapter backed by a
hosted service or a small test fixture.

## 2.2 Install tools

Install:

-   Git
-   VS Code
-   Python 3.12+
-   Node.js LTS
-   Docker Engine / Docker Desktop
-   Docker Compose
-   PostgreSQL client tools
-   A browser with developer tools

Use a virtual environment for Python.

``` bash
python -m venv .venv
```

Activate it:

``` bash
# Windows PowerShell
.venv\Scripts\Activate.ps1

# Linux/macOS
source .venv/bin/activate
```

## 2.3 Repository layout

Start with a monorepo:

``` text
cabshare/
├── backend/
│   ├── app/
│   │   ├── domain/
│   │   │   ├── identity/
│   │   │   ├── travel/
│   │   │   ├── matching/
│   │   │   ├── groups/
│   │   │   ├── messaging/
│   │   │   └── safety/
│   │   ├── application/
│   │   │   ├── ports/
│   │   │   ├── use_cases/
│   │   │   └── dto/
│   │   ├── infrastructure/
│   │   │   ├── persistence/
│   │   │   ├── cache/
│   │   │   ├── routing/
│   │   │   ├── events/
│   │   │   └── notifications/
│   │   ├── interfaces/
│   │   │   └── api/v1/
│   │   ├── bootstrap/
│   │   └── main.py
│   ├── workers/
│   ├── migrations/
│   ├── tests/
│   ├── pyproject.toml
│   └── Dockerfile
├── frontend/
│   ├── src/
│   ├── public/
│   └── package.json
├── deploy/
│   ├── Caddyfile
│   └── compose.prod.yaml
├── docs/
├── .env.example
├── .gitignore
├── compose.yaml
└── README.md
```

## 2.4 Initialise Git

``` bash
mkdir cabshare
cd cabshare
git init
```

Create separate backend and frontend directories. Commit small, working
increments. Never commit `.env`, private keys, OTP credentials, database
passwords, production dumps, or user data.

------------------------------------------------------------------------

# 3. Start local infrastructure

## 3.1 Development Compose file

Create `compose.yaml`:

``` yaml
services:
  postgres:
    image: postgis/postgis:16-3.4
    environment:
      POSTGRES_DB: cabshare
      POSTGRES_USER: cabshare
      POSTGRES_PASSWORD: dev_only_change_me
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U cabshare -d cabshare"]
      interval: 5s
      timeout: 5s
      retries: 10

  valkey:
    image: valkey/valkey:8
    command: ["valkey-server", "--appendonly", "yes"]
    ports:
      - "6379:6379"
    volumes:
      - valkey_data:/data
    healthcheck:
      test: ["CMD", "valkey-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  postgres_data:
  valkey_data:
```

Use pinned, tested image versions in deployment rather than floating
tags. Confirm the chosen versions and licences before release.

Start:

``` bash
docker compose up -d
docker compose ps
```

Stop:

``` bash
docker compose down
```

Do not use `docker compose down -v` unless you intentionally want to
delete development database volumes.

## 3.2 Configuration

Use typed settings loaded once at application startup. Do not scatter
direct environment reads throughout business logic.

Example `.env.example`:

``` dotenv
APP_ENV=development
APP_NAME=CabShare
SECRET_KEY=replace-with-a-long-random-secret
DATABASE_URL=postgresql+asyncpg://cabshare:dev_only_change_me@localhost:5432/cabshare
VALKEY_URL=redis://localhost:6379/0
FRONTEND_ORIGIN=http://localhost:3000
ACCESS_TOKEN_MINUTES=15
REFRESH_TOKEN_DAYS=14
ROUTING_PROVIDER=mock
ROUTING_BASE_URL=
OTP_PROVIDER=development
LOG_LEVEL=INFO
```

For local development, create `.env` from this template and use a
development-only secret. Production secrets must be configured in the
hosting platform's secret manager/environment settings.

------------------------------------------------------------------------

# 4. Backend foundation

## 4.1 Create the FastAPI project

Install core packages (adjust versions through a lockfile):

``` bash
pip install fastapi uvicorn[standard] pydantic-settings
pip install sqlalchemy[asyncio] asyncpg alembic
pip install valkey
pip install python-jose[cryptography] pwdlib[argon2]
pip install pytest pytest-asyncio httpx ruff mypy
```

Use a dependency manager and lockfile for reproducible builds. Do not
rely on unpinned ad-hoc installs in production.

## 4.2 Application composition

Create:

-   `config.py` for typed settings.
-   `bootstrap/container.py` for dependency construction.
-   `main.py` for application lifespan and router registration.
-   Router modules that validate HTTP input and call application use
    cases.
-   Use cases that orchestrate domain operations.
-   Domain objects that hold business rules.
-   Infrastructure adapters that implement ports.

Keep business logic out of FastAPI routers and SQLAlchemy models.

## 4.3 Health endpoints

Provide at least:

-   `GET /health/live`: process is running.
-   `GET /health/ready`: essential dependencies are ready.
-   `GET /metrics`: Prometheus metrics, protected or network-restricted
    in production.

Do not expose database passwords, stack traces, or internal
infrastructure details in health responses.

## 4.4 Authentication and account security

Implement:

-   Phone verification using an OTP provider adapter.
-   Rate limits for OTP generation and verification.
-   OTP expiry and single-use semantics.
-   Short-lived access tokens.
-   Rotating refresh tokens with revocation.
-   Secure password hashing if passwords are supported (Argon2id).
-   Session/device management.
-   Account deletion and data export processes where required.
-   Authorization checks on every protected resource.

Do not use a hardcoded OTP in production. A development OTP may exist
only behind an explicit development configuration and must never be
enabled in production.

SMS is not inherently free or open source. Use a compliant provider and
account for Indian telecom/OTP requirements.

------------------------------------------------------------------------

# 5. Domain model and database

## 5.1 Core tables

Create the schema incrementally with Alembic migrations.

  -----------------------------------------------------------------------
  Table                               Responsibility
  ----------------------------------- -----------------------------------
  `users`                             Passenger profiles and account
                                      status

  `user_verifications`                OTP/verification records

  `travel_requests`                   Pickup, destination, time window,
                                      seats, preferences

  `route_geometries`                  Optional route geometry and routing
                                      metadata

  `match_candidates`                  Candidate pairs, score, status,
                                      expiry

  `sharing_groups`                    Group lifecycle

  `group_members`                     Passenger membership and seats

  `group_invitations`                 Invitations and responses

  `group_agreements`                  Current agreed trip
                                      configuration/version

  `group_proposals`                   Proposed group changes

  `proposal_eligible_voters`          Frozen voter set for a proposal

  `proposal_votes`                    One vote per eligible passenger

  `group_messages`                    Durable chat history

  `trip_records`                      Optional independently arranged
                                      trip details

  `fare_contributions`                Agreed indicative
                                      split/contribution records

  `reviews`                           Post-trip feedback

  `reports`                           Safety reports

  `blocks`                            User blocking relationships

  `notifications`                     Notification records

  `outbox_events`                     Reliable event publication

  `audit_logs`                        Sensitive state changes and
                                      administrative actions
  -----------------------------------------------------------------------

Use UUID primary keys, UTC timestamps, foreign keys, check constraints,
and explicit status enums. Define retention and deletion rules before
collecting personal data.

## 5.2 Travel request fields

A travel request should include:

-   Passenger ID.
-   Pickup label and PostGIS point.
-   Destination label and PostGIS point.
-   Requested departure time.
-   Earliest and latest departure time.
-   Instant/scheduled mode.
-   Seats required.
-   Luggage information.
-   Maximum detour.
-   Maximum waiting time.
-   Request status.
-   Optional preferences.
-   Created/updated timestamps.

Store coordinates in a consistent SRID (normally EPSG:4326). Validate
coordinates and avoid exposing exact home addresses to other users
before necessary.

## 5.3 Spatial indexes

Use PostGIS `GEOGRAPHY(POINT, 4326)` or an appropriate geometry model.
Add GiST indexes for spatial search.

Example migration SQL:

``` sql
CREATE INDEX ix_travel_requests_pickup
ON travel_requests USING GIST (pickup_location);

CREATE INDEX ix_travel_requests_destination
ON travel_requests USING GIST (destination_location);
```

The exact DDL depends on your ORM model and whether the column is
geometry or geography. Verify the migration against the real schema.

## 5.4 Index for common access patterns

Likely indexes include:

-   Travel request status + departure window.
-   Pickup geography.
-   Destination geography or route geometry.
-   Group membership by group and passenger.
-   Messages by `(group_id, created_at, id)`.
-   Proposals by group and status.
-   Outbox events by status and creation time.

Do not add every possible index. Measure using
`EXPLAIN (ANALYZE, BUFFERS)` and review write overhead.

------------------------------------------------------------------------

# 6. Build features in dependency order

## Phase 1 --- Identity and profiles

Implement:

-   Registration.
-   OTP verification.
-   Login/refresh/logout.
-   Passenger profile.
-   Account status.
-   Basic privacy settings.

Tests:

-   Invalid/expired OTP is rejected.
-   OTP cannot be reused.
-   OTP rate limits work.
-   Refresh-token rotation and revocation work.
-   A user cannot read another user's private profile fields.

## Phase 2 --- Travel requests

Implement:

-   Create request.
-   List my requests.
-   Get request details.
-   Edit request.
-   Cancel request.
-   Validate time windows, seats, coordinates, and request status.

Suggested endpoints:

``` text
POST   /api/v1/travel-requests
GET    /api/v1/travel-requests/me
GET    /api/v1/travel-requests/{request_id}
PATCH  /api/v1/travel-requests/{request_id}
DELETE /api/v1/travel-requests/{request_id}
```

Do not let users edit a request after it has entered a state where the
change would silently invalidate an active group. Require a group
proposal or leave/re-match workflow where appropriate.

## Phase 3 --- Maps and routing adapter

Implement a `RoutingPort` before choosing a provider permanently.

The adapter should return:

-   Encoded or structured route geometry.
-   Route distance.
-   Estimated travel duration.
-   Provider/source metadata.
-   Timestamp and freshness information.

Use MapLibre for map display. OpenStreetMap data can be used, but the
public OSM tile service is not an unlimited production tile API. Review
tile usage policy and attribution. For production, choose a provider or
host tiles appropriately.

Do not use straight-line distance as a substitute for road route
distance. It can be a cheap first-stage filter, followed by route
comparison.

## Phase 4 --- Matching engine

Build a deterministic rule-based matching engine first.

### Hard filters

Reject candidates that fail:

-   Compatible travel date/time windows.
-   Pickup proximity.
-   Route direction/overlap.
-   Seat capacity.
-   Request/group status.
-   User block/report restrictions.
-   Maximum detour or waiting constraints.

### Ranking

An initial score can be:

``` text
score =
    0.30 * route_overlap
  + 0.25 * time_compatibility
  + 0.20 * pickup_proximity
  + 0.15 * destination_compatibility
  + 0.10 * capacity_fit
```

These are initial tunable weights, not validated facts. Store component
scores so you can explain why a match was suggested. Apply hard
constraints before ranking.

### Instant matching

-   Run matching when a request is created or changed.
-   Keep requests active for a user-selected matching window.
-   Refresh candidate discovery periodically during that window.
-   Expire stale candidates.

### Scheduled matching

-   Store the request until its future departure window.
-   Trigger matching when a relevant request is created/updated.
-   Run periodic jobs for upcoming requests.
-   Recheck compatibility before group formation.

### Concurrency

Prevent two concurrent acceptances from overfilling a group. Use
transactions, row locks or equivalent safe conditional updates, unique
constraints, and idempotency keys.

Avoid duplicate candidate pairs with a normalized request-pair key and a
unique constraint.

## Phase 5 --- Groups and democratic decisions

Implement:

-   Group creation and membership.
-   Invitations and acceptance.
-   Seat capacity.
-   Group status lifecycle.
-   Proposals and votes.
-   Agreement version history.
-   Leaving/cancellation behaviour.

Important decisions should require approval from the eligible members.
Examples:

-   Departure time.
-   Pickup/drop-off changes.
-   Maximum detour.
-   Fare-split method.
-   Booking coordination responsibility, if members choose to record
    one.
-   Adding a passenger.

Use a clear approval rule, such as unanimity for core trip conditions.
Less critical preferences may use majority approval.

For each proposal:

1.  Record proposer and proposed value.
2.  Snapshot eligible voters.
3.  Allow one vote per eligible voter.
4.  Set an expiry.
5.  Resolve only once.
6.  Apply the accepted change transactionally.
7.  Record the agreement version and audit event.

If group membership changes while a proposal is open, define whether the
proposal is invalidated or resolved against the frozen voter set. Do not
silently change the electorate.

## Phase 6 --- Messaging and realtime

Persist every chat message in PostgreSQL before broadcasting it.

Suggested message fields:

-   Message UUID.
-   Group UUID.
-   Sender UUID.
-   Content/type.
-   Created/edited/deleted timestamps.
-   Client-generated idempotency UUID.

Use cursor-based pagination, not unbounded message history retrieval.

Suggested query index:

``` sql
CREATE INDEX ix_group_messages_cursor
ON group_messages (group_id, created_at DESC, id DESC);
```

Realtime flow:

1.  Authenticate WebSocket connection.
2.  Verify current group membership.
3.  Validate and persist the message.
4.  Publish a message event through Valkey.
5.  Deliver to connected group members.
6.  On reconnect, client sends its last received message ID/time cursor.
7.  Backend fetches missed messages from PostgreSQL.

**Never use Valkey Pub/Sub as the only message store.** Pub/Sub can lose
messages for disconnected subscribers. PostgreSQL remains the durable
source of truth.

## Phase 7 --- Fare-split helper and independent booking record

The MVP may provide an indicative calculator:

-   Equal split.
-   Distance-based split.
-   Custom agreed contributions.

Clearly distinguish:

-   Estimated total fare.
-   Proposed split.
-   Group-agreed split.
-   Actual fare reported by passengers.

The platform cannot verify off-platform payment. Do not describe a
proposed split as a collected payment or guaranteed fare.

Allow passengers to independently arrange a taxi through a lawful
channel. If they choose, record a booking reference and trip details. Do
not implement driver dispatch or fare collection in this MVP.

## Phase 8 --- Safety and moderation

Implement before inviting the public:

-   Report user/group/trip.
-   Block user.
-   Leave group.
-   Cancellation/no-show reporting.
-   Basic moderation/admin workflow.
-   Audit trail for administrative changes.
-   Privacy controls.
-   Clear community rules and safety guidance.

Keep phone numbers private by default. Do not expose precise live
location to other group members without explicit consent. If live
location is added later, provide start/stop controls, limited retention,
and clear visibility.

------------------------------------------------------------------------

# 7. Frontend implementation

## 7.1 Main screens

Build mobile-first:

1.  Landing page and product explanation.
2.  Sign up / OTP verification.
3.  Profile setup.
4.  Create travel request.
5.  Map-based pickup/destination selection.
6.  My requests.
7.  Match results.
8.  Match details and invitation acceptance.
9.  Group details and members.
10. Group chat.
11. Proposals and voting.
12. Trip agreement and fare-split estimate.
13. Safety/report/block.
14. Settings and account deletion.

## 7.2 UX requirements

-   Display time zones clearly; store timestamps in UTC.
-   Make instant and scheduled modes distinct.
-   Show why a match is suggested (route overlap, time compatibility,
    pickup proximity).
-   Show that route and fare estimates may be inaccurate.
-   Make all important group changes visible and auditable.
-   Show clear loading, empty, retry, and error states.
-   Prevent double submission.
-   Support low-bandwidth mobile connections.
-   Do not display exact private addresses unnecessarily.

## 7.3 Frontend API client

Centralise API calls and authentication handling. Use typed
request/response schemas generated from or checked against the backend
OpenAPI specification. Do not duplicate business rules in frontend code;
frontend validation is for usability, backend validation is
authoritative.

------------------------------------------------------------------------

# 8. Testing strategy

## 8.1 Test layers

  -----------------------------------------------------------------------
  Layer                   Tool                    What to test
  ----------------------- ----------------------- -----------------------
  Domain unit             Pytest                  Matching rules, votes,
                                                  state transitions

  Application             Pytest                  Use-case orchestration

  API integration         HTTPX                   Auth, validation,
                                                  authorization

  Database integration    Testcontainers or test  Constraints,
                          DB                      transactions, PostGIS

  Realtime                WebSocket tests         Auth, membership,
                                                  reconnect recovery

  Frontend                Vitest                  Components and client
                                                  logic

  End-to-end              Playwright              Complete passenger
                                                  journey

  Load                    k6                      API, matching,
                                                  WebSocket workloads

  Security                Dependency/scanner      Vulnerable packages and
                          tools                   common web risks
  -----------------------------------------------------------------------

## 8.2 Critical scenarios

Test at least:

-   Two users accept the same final seat concurrently.
-   The same invitation is accepted twice.
-   A user outside a group attempts to read its chat.
-   A blocked user attempts to invite/message another user.
-   A proposal receives duplicate votes.
-   A proposal expires while votes are being submitted.
-   Membership changes while a proposal is open.
-   A WebSocket disconnects and reconnects.
-   Valkey restarts while PostgreSQL remains healthy.
-   Outbox delivery fails and is retried.
-   A user cancels after group formation.
-   A scheduled request becomes stale.
-   A user submits invalid coordinates or extreme time windows.

## 8.3 CI pipeline

On every pull request:

1.  Format/lint.
2.  Type check.
3.  Run unit tests.
4.  Run integration tests.
5.  Build backend image.
6.  Build frontend.
7.  Run dependency/security checks.
8.  Block merge on failures.

Use Ruff for Python lint/format, mypy where practical, and Playwright
for end-to-end smoke tests.

------------------------------------------------------------------------

# 9. Deployment: free testing first

There are two different goals:

-   **Free public demo:** prove that the website works with a small
    invited audience.
-   **Reliable production service:** keep the app available, protect
    data, back it up, and support real users.

A free tier may be suitable for a demo but can have sleeping services,
low resource limits, ephemeral storage, database expiry, bandwidth
limits, or changing terms. Do not promise uninterrupted service on a
free plan.

## 9.1 Suggested low-cost/free deployment shape

For a small MVP, deploy:

``` text
User browser
    |
    v
Frontend hosting (static/Next.js-capable host)
    |
    v
Backend hosting (FastAPI container)
    |
    +--> PostgreSQL + PostGIS
    |
    +--> Valkey (if supported; otherwise start without cache)
    |
    +--> Worker process
```

Potential hosting categories:

-   Frontend: a free static/Next.js hosting tier, subject to framework
    and usage limits.
-   Backend: a container/PaaS free tier if available, accepting
    sleep/cold-start limits.
-   Database: a PostgreSQL provider's free developer tier, after
    checking storage, connection, expiry, backup, and PostGIS support.
-   Worker: a separate process if the platform supports it; otherwise
    use a small always-on host or temporarily run a limited worker.
-   Domain: begin with provider subdomains; buy a custom domain when
    useful.
-   Maps/routing: use a provider with a development quota or host a
    limited service; check commercial terms and usage limits.

Provider pricing and free-tier terms change frequently. Verify current
official pricing, region availability, PostGIS support, persistent
storage, outbound networking, and acceptable-use policies before
choosing.

**Important:** Do not put production secrets or real personal data into
an untrusted demo deployment. For an early public test, use invited
testers, minimal data collection, clear deletion rules, and a privacy
notice.

## 9.2 Simplest deployment option for a first demo

If you want to minimise infrastructure work:

1.  Deploy the Next.js frontend to a frontend hosting provider.
2.  Deploy FastAPI as a Docker web service.
3.  Use a managed PostgreSQL database that supports PostGIS.
4.  Initially omit Valkey if the free deployment budget is too
    constrained.
5.  Run matching/outbox tasks in a separate worker if available. For a
    very small closed demo, a scheduled worker can be temporarily
    co-located, but avoid blocking API requests.
6.  Configure the frontend API base URL and allowed CORS origin.
7.  Add a health check and test the full signup-to-group flow.

Do not assume that every free PostgreSQL tier supports PostGIS or
permanent storage. Confirm before committing to a provider.

## 9.3 Deployment environment variables

Set production values in the host's secret/environment configuration:

``` dotenv
APP_ENV=production
SECRET_KEY=<long-random-secret>
DATABASE_URL=<managed-postgres-connection-string>
VALKEY_URL=<private-valkey-connection-string>
FRONTEND_ORIGIN=https://your-frontend-domain
ROUTING_PROVIDER=<selected-provider>
ROUTING_API_KEY=<secret-if-required>
OTP_PROVIDER=<selected-provider>
OTP_API_KEY=<secret-if-required>
LOG_LEVEL=INFO
```

Never place secrets in frontend environment variables unless they are
explicitly designed to be public. Never commit production `.env` files.

## 9.4 Production Docker considerations

-   Use multi-stage builds where helpful.
-   Run as a non-root user.
-   Pin base image versions.
-   Install only runtime dependencies in the final image.
-   Use health checks.
-   Set CPU/memory limits where supported.
-   Keep database and Valkey private; do not expose their ports
    publicly.
-   Use persistent storage for self-hosted stateful services.
-   Keep API and worker commands separate.
-   Handle SIGTERM and graceful shutdown.
-   Run migrations as a controlled release step, not concurrently from
    every API replica.

Example process commands:

``` bash
# API
uvicorn app.main:app --host 0.0.0.0 --port 8000

# Worker (illustrative; implement this module)
python -m workers.outbox_worker
```

## 9.5 Custom VPS deployment

When free tiers become restrictive, a small VPS is a straightforward
next step.

On a VPS:

1.  Create a non-root deployment user.
2.  Configure SSH keys; disable password login where appropriate.
3.  Enable a firewall; expose only SSH, HTTP, and HTTPS.
4.  Install Docker Engine and Compose.
5.  Deploy the API, worker, PostgreSQL/PostGIS, Valkey, frontend (if
    self-hosted), and Caddy.
6.  Use persistent volumes and private internal networks.
7.  Configure DNS to point to the VPS.
8.  Configure Caddy for HTTPS.
9.  Set secrets securely.
10. Run migrations and smoke tests.
11. Configure automated backups and test restoration.
12. Configure monitoring and alerts.

A single VPS is a useful first production deployment, but it is also a
single point of failure. Keep backups off the VPS. For more demanding
production workloads, separate the database from application compute and
consider managed database hosting.

## 9.6 Caddy reverse proxy example

Illustrative `Caddyfile`:

``` caddyfile
app.example.com {
    reverse_proxy frontend:3000
}

api.example.com {
    reverse_proxy backend:8000
}
```

Caddy can obtain and renew certificates automatically when DNS and
network prerequisites are satisfied. Configure WebSocket proxying and
timeouts according to the chosen deployment setup.

## 9.7 Database migration and release sequence

For each release:

1.  Build and tag immutable frontend/backend images.
2.  Run automated tests.
3.  Back up the database before risky schema changes.
4.  Run Alembic migrations once.
5.  Deploy backend and worker.
6.  Deploy frontend.
7.  Run smoke tests.
8.  Watch error rate, latency, worker backlog, and database health.
9.  Roll back application images if necessary.
10. Use backward-compatible migrations so old and new application
    versions can coexist during rollout.

Do not make destructive schema changes in the same release that first
stops using the old schema. Use expand-and-contract migrations.

------------------------------------------------------------------------

# 10. Monitoring and operations

Monitoring is part of the application, not a final optional feature.

## 10.1 What to monitor

### Application metrics

-   Request count by route/status.
-   HTTP error rate.
-   p50, p95, and p99 latency.
-   WebSocket active connections and disconnects.
-   Authentication/OTP failures.
-   Match candidates generated.
-   Match acceptance rate.
-   Group creation failures.
-   Proposal resolution time.
-   Message delivery latency.
-   Outbox pending count and oldest event age.
-   Worker job failures/retries.

### Infrastructure metrics

-   CPU and memory.
-   Disk usage.
-   Database connections.
-   Database query latency.
-   Slow queries and lock waits.
-   Cache memory and hit rate.
-   Container restarts.
-   Backup age and backup failures.

### Product metrics

-   Registrations and verified users.
-   Active travel requests.
-   Match discovery-to-acceptance conversion.
-   Groups formed.
-   Group completion/cancellation rates.
-   Reports and blocks.
-   Time to find a compatible group.

Use aggregated metrics. Avoid putting phone numbers, message bodies,
exact pickup addresses, access tokens, or other personal information
into metric labels.

## 10.2 Suggested monitoring stack

For a self-managed deployment:

-   Prometheus: metrics collection.
-   Grafana: dashboards and alert visualisation.
-   OpenTelemetry: traces and instrumentation.
-   Structured JSON logs: searchable application events.
-   An error tracking service or self-hosted alternative, after
    reviewing privacy and retention.

For a tiny free demo, begin with platform logs, health checks, uptime
monitoring, and basic application metrics. Add the full
Prometheus/Grafana stack when the host has enough memory and operational
capacity.

## 10.3 Alerts

Create alerts for:

-   API unavailable.
-   Elevated 5xx rate.
-   p95 latency above your service objective.
-   Database unavailable or near connection limit.
-   Disk usage high.
-   Worker stopped or backlog growing.
-   Oldest outbox event too old.
-   Backup job failed or backup too old.
-   Repeated OTP abuse.
-   Unusual login or account activity.

Alerts should go to a channel someone actively monitors. Test alert
delivery rather than assuming it works.

## 10.4 Backups and recovery

-   Automate PostgreSQL backups.
-   Keep backups off the application server.
-   Encrypt backups.
-   Restrict access.
-   Define retention.
-   Test restoration regularly.
-   Document recovery steps and expected recovery time.
-   Back up object storage separately if it is later introduced.

A backup that has never been restored is not a verified recovery plan.

## 10.5 Logging and privacy

Log:

-   Request/correlation ID.
-   Route and response status.
-   Duration.
-   User ID only where justified and access-controlled.
-   Event/job ID.
-   Exception class and safe diagnostic details.

Do not log:

-   OTP codes.
-   Passwords or tokens.
-   Full phone numbers.
-   Full message contents by default.
-   Precise location history.
-   Payment credentials.

Define log retention and access permissions.

------------------------------------------------------------------------

# 11. Security and privacy checklist

Before inviting external testers:

-   [ ] HTTPS enabled.
-   [ ] Secrets are outside source control.
-   [ ] Database and cache are not public.
-   [ ] CORS is restricted to the frontend origin.
-   [ ] Authentication and authorization tested.
-   [ ] Group membership checked on every group/chat request.
-   [ ] Rate limiting on login, OTP, invitations, and messaging.
-   [ ] Request size and upload limits configured.
-   [ ] SQL injection protections through parameterised queries/ORM.
-   [ ] Output encoding and frontend XSS protections reviewed.
-   [ ] CSRF protections considered for cookie-based authentication.
-   [ ] Refresh-token rotation and revocation implemented.
-   [ ] Admin actions audited.
-   [ ] Report/block workflows available.
-   [ ] Privacy policy and terms are published.
-   [ ] Data collection minimised.
-   [ ] Account deletion and retention rules documented.
-   [ ] Backup access is restricted and restoration tested.
-   [ ] Dependency scanning and security updates scheduled.
-   [ ] Incident response contact/process documented.

Do not collect identity documents or precise live location unless there
is a clear, reviewed need and suitable safeguards.

------------------------------------------------------------------------

# 12. Legal and operational readiness in Kerala

This is a critical launch gate, not a task to postpone until after
public launch.

Before facilitating real paid shared journeys:

1.  Ask a Kerala transport lawyer to review the exact
    passenger-to-passenger workflow.
2.  Seek written clarification from the Kerala Transport Commissioner /
    State Transport Authority or relevant RTO.
3.  Explain that passengers independently seek and arrange one taxi and
    that the platform only matches passengers and coordinates group
    decisions.
4.  Ask specifically about aggregator classification, contract-carriage
    rules, permit requirements, reimbursement between passengers,
    fare-split calculators, insurance, and liability.
5.  Review applicable central and state laws and the latest notified
    policy, not only draft policies or commentary.
6.  Record the written advice and product restrictions.
7.  Do not onboard drivers or vehicles, dispatch taxis, collect ride
    fares, or represent the platform as a licensed transport operator
    without the required approvals.

A no-payment design does not automatically mean the platform is outside
transport regulation. A draft policy is not binding law unless and until
it is finalised and notified.

For an early prototype, use synthetic data and simulated journeys. A
real-world paid pilot should wait until the legal model is cleared.

------------------------------------------------------------------------

# 13. Suggested 90-day build plan

This is a planning template, not a promise that every feature will be
production-ready in 90 days.

  -----------------------------------------------------------------------
  Period                  Main work               Exit criteria
  ----------------------- ----------------------- -----------------------
  Days 1--7               Product requirements,   Running
                          legal questions, repo,  frontend/backend and
                          architecture, local     database
                          infrastructure          

  Days 8--18              Identity, profiles,     Authenticated users can
                          travel request CRUD     create/manage requests

  Days 19--30             Map UI, routing         Requests display and
                          adapter, PostGIS        candidate search works
                          filtering               

  Days 31--42             Matching engine,        Deterministic matching
                          instant/scheduled       tests pass
                          triggers                

  Days 43--54             Groups, invitations,    Concurrent acceptance
                          voting, agreement       and voting tests pass
                          versions                

  Days 55--64             Persistent chat,        Reconnect recovery and
                          WebSockets, outbox,     retries work
                          notifications           

  Days 65--72             Safety, reporting,      Safety flows and
                          blocking, cancellation  authorization tests
                                                  pass

  Days 73--80             E2E, load, security,    Critical test suite and
                          accessibility           smoke tests pass

  Days 81--86             Staging deployment,     Restore test, alerts,
                          backups, monitoring     dashboards work

  Days 87--90             Closed invited test,    Testers complete core
                          feedback, fixes         journey without
                                                  assistance
  -----------------------------------------------------------------------

Prioritise a reliable narrow workflow over a large feature list.

------------------------------------------------------------------------

# 14. Release checklist

## Before staging

-   [ ] Core user journey works locally.
-   [ ] Migrations are repeatable.
-   [ ] Unit and integration tests pass.
-   [ ] Frontend build succeeds.
-   [ ] API and worker can run separately.
-   [ ] No secrets in Git.
-   [ ] Error states and loading states implemented.

## Before invited testing

-   [ ] Staging deployment works over HTTPS.
-   [ ] Database backups configured.
-   [ ] Restore test completed.
-   [ ] Logs and basic monitoring available.
-   [ ] User reporting/blocking works.
-   [ ] Privacy notice and terms published.
-   [ ] Testers understand booking/payment are outside the app.
-   [ ] Synthetic or minimal personal data used.
-   [ ] Legal review completed for any real-world journey being
    facilitated.

## Before general public launch

-   [ ] Written regulatory/legal clearance for the operating model.
-   [ ] Security review completed.
-   [ ] Load test reflects expected usage.
-   [ ] Incident response and support process documented.
-   [ ] Data retention and deletion implemented.
-   [ ] Monitoring alerts tested.
-   [ ] Recovery objectives documented.
-   [ ] Production hosting and provider terms reviewed.
-   [ ] Operational owner assigned.

------------------------------------------------------------------------

# 15. Engineering rules to preserve

1.  PostgreSQL is the source of truth for all durable business data.
2.  Valkey is a performance/coordination layer, not the only store for
    messages or agreements.
3.  Persist chat messages before broadcasting.
4.  Use an outbox for reliable event publication.
5.  Make write endpoints idempotent where clients may retry.
6.  Use database transactions for group membership, capacity, voting,
    and agreement changes.
7.  Every group resource access must verify membership and permissions.
8.  Keep domain logic independent of FastAPI, SQLAlchemy, Valkey, and
    provider SDKs.
9.  Use provider interfaces for routing, OTP, notifications, cache, and
    event publishing.
10. Add infrastructure only when a measured need justifies it.
11. Use migrations and backward-compatible releases.
12. Treat monitoring, backups, privacy, and security as core product
    features.
13. Do not launch a paid shared-ride service until the legal model is
    reviewed and cleared.

------------------------------------------------------------------------

# 16. Immediate next actions

Start in this order:

1.  Create the Git repository and the folder structure.
2.  Write a concise product requirements document and state machine for
    travel requests/groups.
3.  Create `compose.yaml` with PostgreSQL/PostGIS and Valkey.
4.  Create the FastAPI app, typed settings, dependency container, and
    health endpoints.
5.  Create the Next.js app and a mobile-first landing page.
6.  Add Alembic and the first `users` migration.
7.  Implement phone verification and profile creation.
8.  Implement travel request creation and listing.
9.  Add map selection and the routing provider interface.
10. Build and test matching before adding chat or advanced features.

**First milestone:** A verified passenger can create an instant or
scheduled travel request, and the backend can find compatible requests
using deterministic rules and PostGIS.

That is the foundation. Do not start with chat, microservices, ML, or a
complex deployment before this vertical slice works.
