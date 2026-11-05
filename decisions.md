# Decision Log

Every design decision in TicketHub, with the options that were considered and the reason for the
choice. Entries are numbered and never rewritten: a changed decision gets a new entry that points
to the one it replaces.

Entry format:

```markdown
### D-NNN · Short title

- **Date:** YYYY-MM-DD
- **PR:** NN (or "baseline" for decisions taken before the first PR)
- **Decision:** what was chosen.
- **Options considered:** the alternatives, each with its main trade-off.
- **Reason:** why this option fits this project.
```

---

## Baseline

### D-001 · Monolith organised by feature

- **Date:** 2026-11-04
- **PR:** baseline
- **Decision:** a single Spring Boot application, split into feature packages (`auth`, `event`,
  `booking`, `common`). Each feature is layered controller → service → repository.
- **Options considered:** package by layer (all controllers together, all services together), which
  scatters one feature across the codebase; microservices, which add network calls, separate
  deployments and distributed transactions.
- **Reason:** one deployable unit is the simplest thing to build, run and explain. Feature packages
  keep the code for one use case in one place.

### D-002 · DTOs separate from JPA entities

- **Date:** 2026-11-04
- **PR:** baseline
- **Decision:** the API only reads and returns DTOs. Entities never leave the service layer.
- **Options considered:** returning entities directly, which is less code but ties the API contract
  to the database schema.
- **Reason:** the schema can change without breaking clients, sensitive fields such as password
  hashes cannot leak by accident, and lazy-loaded relations never surprise the JSON serializer.

### D-003 · Double bookings prevented by the database

- **Date:** 2026-11-04
- **PR:** baseline
- **Decision:** a uniqueness constraint in PostgreSQL guarantees at most one active booking per seat.
  When it is violated, the API answers `409 Conflict`.
- **Options considered:** check-then-insert in Java (two requests can both read "free" before
  either writes); pessimistic locking with `SELECT ... FOR UPDATE` (correct, but holds locks and can
  deadlock); optimistic locking with `@Version` (needs a row to version and retries on conflict).
- **Reason:** the constraint is atomic, holds across any number of application instances and needs
  no locks. The full comparison is written up in PR 09.

### D-004 · Stateless JWT authentication

- **Date:** 2026-11-04
- **PR:** baseline
- **Decision:** login returns a short-lived JWT. The server keeps no sessions.
- **Options considered:** server-side sessions, which need shared session storage once there is more
  than one instance.
- **Reason:** every instance can verify a token on its own. Refresh tokens are left for level 2.

### D-005 · Standard error format

- **Date:** 2026-11-04
- **PR:** baseline
- **Decision:** errors are Problem Details documents (RFC 9457) with the correct status code: `400`,
  `401`, `403`, `404`, `409`.
- **Options considered:** a custom error format.
- **Reason:** a published standard that Spring supports out of the box. Clients parse one format for
  every error.

### D-006 · Schema managed by Flyway

- **Date:** 2026-11-04
- **PR:** baseline
- **Decision:** the schema changes only through versioned SQL migrations. Hibernate validates the
  mapping and never modifies the schema.
- **Options considered:** letting Hibernate generate or update the schema (`ddl-auto: update`).
- **Reason:** every change is reviewable SQL and every environment has the same schema. Constraints
  such as partial unique indexes are written exactly as PostgreSQL needs them.

### D-007 · Integration tests on real PostgreSQL

- **Date:** 2026-11-04
- **PR:** baseline
- **Decision:** integration tests run against PostgreSQL started by Testcontainers.
- **Options considered:** H2 in memory, which is faster to start but behaves differently from
  PostgreSQL for constraints, partial indexes and locking.
- **Reason:** the concurrency guarantee lives in the database, so it must be tested on the real one.

### D-008 · Pagination on list endpoints

- **Date:** 2026-11-04
- **PR:** baseline
- **Decision:** every endpoint that returns a list is paginated.
- **Options considered:** returning complete lists.
- **Reason:** response size and query cost stay bounded as data grows. Any exception is recorded as
  its own decision.

### D-009 · Configuration from environment variables

- **Date:** 2026-11-04
- **PR:** baseline
- **Decision:** the same container image runs locally and in Azure. Only environment variables
  differ (database connection, JWT signing key). No secrets in the repository; `.env` is ignored by
  Git and `.env.example` holds placeholder values.
- **Options considered:** per-environment configuration files built into the image.
- **Reason:** one artifact for every environment, and secrets never reach version control.

### D-010 · Technology stack

- **Date:** 2026-11-04
- **PR:** baseline
- **Decision:** Java 25 (LTS), Spring Boot 4.1 with Maven, PostgreSQL. Swagger UI through
  springdoc-openapi. JUnit 5 and Testcontainers for tests.
- **Options considered:** Java 21 with Spring Boot 3.x, the previous generation.
- **Reason:** the current LTS release and the current Spring generation, all within one ecosystem.

### D-011 · Zero-cost hosting

- **Date:** 2026-11-04
- **PR:** baseline
- **Decision:** API on Azure Container Apps (scales to zero), image in GitHub Container Registry,
  database on Supabase (free tier). Azure resources are created by hand in the portal; GitHub
  Actions deploys on every merge.
- **Options considered:** Azure Database for PostgreSQL and Azure Container Registry, which keep
  everything inside Azure but cost money (kept for level 2).
- **Reason:** hands-on Azure experience at no cost. Trade-offs: the first request after a quiet
  period waits for the application to start, and a free Supabase project is paused after a week of
  low activity, which a daily keep-alive request prevents (PR 13).

### D-012 · CI from the second PR

- **Date:** 2026-11-05
- **PR:** baseline
- **Decision:** the workflow that runs the tests on every pull request arrives in PR 02. Building and
  publishing the container image stays in PR 12.
- **Options considered:** adding CI together with the Docker work, near the end of version 1.
- **Reason:** every later PR, including the concurrency test, is verified automatically from the
  start.

### D-013 · No bookings for events that have started

- **Date:** 2026-11-05
- **PR:** baseline (implemented in PR 08)
- **Decision:** a booking is accepted only if the event has not started yet.
- **Options considered:** accepting bookings at any time.
- **Reason:** a seat at an event that is under way or over has no value, and accepting the booking
  would leave meaningless data behind.

### D-014 · No cancellation after the event starts

- **Date:** 2026-11-05
- **PR:** baseline (implemented in PR 11)
- **Decision:** a booking can be cancelled only by its owner and only before the event starts.
- **Options considered:** allowing cancellation at any time.
- **Reason:** a seat released after the start can no longer be used by anyone, and the booking
  history should reflect who actually held the seat.

---

## Decisions by PR

<!-- New entries go below, starting with D-015. -->
