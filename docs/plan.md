# TicketHub: Implementation Plan

TicketHub is a REST API for reserving seats at events such as concerts and theatre shows.
Its central problem is concurrency: two users trying to book the same seat at the same moment.
The database decides who gets the seat, not application code, and an integration test proves it
by firing 50 simultaneous requests against a real PostgreSQL instance.

## How this plan is used

- Work happens one pull request at a time, in the order below, on a branch named `feat/NN-short-name`.
- Each PR has one open design question. It is answered before any code is written and recorded in
  [`decisions.md`](decisions.md).
- Each PR ticks its own box in the Progress list.
- Tests must pass before every commit. From PR 02 on, CI runs them on every pull request.
- Commits follow Conventional Commits (`feat:`, `fix:`, `test:`, `docs:`, `build:`, `ci:`).
- Each PR description has three sections: **What**, **Decisions**, **How to test**.
- Guiding rule: the simplest tool that does the job.

## Progress

- [ ] 01 · Project skeleton
- [ ] 02 · CI on pull requests
- [ ] 03 · Event and seat schema
- [ ] 04 · Admin endpoints
- [ ] 05 · Public browsing
- [ ] 06 · User registration
- [ ] 07 · Login with JWT and roles
- [ ] 08 · Seat booking
- [ ] 09 · Concurrency test
- [ ] 10 · Idempotency key
- [ ] 11 · My bookings and cancellation
- [ ] 12 · Docker image and GHCR
- [ ] 13 · Azure deployment
- [ ] 14 · README

## Scope of version 1

**Accounts.** Registration and login with JWT. Two roles: `USER` and `ADMIN`.

**Admin.** Creates events (name, venue, date and time) and their seats (row, number).

**User.**

- Lists events (paginated) and sees which seats of an event are available.
- Books one seat per request, sending an `Idempotency-Key` header.
- Lists their own bookings and cancels one; the seat becomes available again.

**Business rules.**

- A seat has at most one active booking at any time. The database enforces this.
- Bookings are accepted only for events that have not started yet ([D-013](decisions.md)).
- Only the owner can cancel a booking, and only before the event starts ([D-014](decisions.md)).
- A repeated booking request with the same `Idempotency-Key` returns the original result and never
  creates a second booking.

## Out of scope (level 2)

Considered only after version 1 works end to end: temporary seat holds that expire, booking several
seats at once, Redis for caching and rate limiting, Mockito-based unit tests, load tests with k6,
refresh tokens, Azure Database for PostgreSQL and Azure Container Registry instead of the free options.

---

## PR details

### 01 · Project skeleton

Branch: `feat/01-project-skeleton`

**Scope**

- Maven project with the Maven Wrapper: Java 25, Spring Boot 4.1, with Spring Web, Validation,
  Data JPA, Flyway, the PostgreSQL driver and springdoc-openapi. Check the latest patch versions
  first, and pick a springdoc-openapi version that supports Spring Boot 4.
- Base package `io.github.augustinbuzatu.tickethub`. Feature packages (`auth`, `event`, `booking`,
  `common`) are added as each one gets its first class.
- `docker-compose.yml` with PostgreSQL, using the same major version as production (Supabase).
- `application.yml` that reads the database connection from environment variables, plus
  `.env.example` with placeholder values.
- First Flyway migration `V1__init.sql`, so the migration pipeline works end to end. Domain tables
  arrive in PR 03.
- Health check and Swagger UI.
- A short "Run locally" section in the README.

**Decision to make:** health check from Spring Boot Actuator (`/actuator/health`, includes a database
check) or a custom `/health` controller?

**Done when:** `docker compose up -d` followed by `./mvnw spring-boot:run` starts the application,
Flyway applies `V1`, the health check reports `UP` and Swagger UI loads.

### 02 · CI on pull requests

Branch: `feat/02-ci-pull-requests`

**Scope**

- `.github/workflows/ci.yml`: on every pull request to `main` and every push to `main`, set up
  Java 25, cache Maven dependencies, run `./mvnw verify`.
- First integration test: the application context starts against PostgreSQL in Testcontainers and
  the Flyway migrations apply.

**Decision to make:** one PostgreSQL container shared by all integration tests (fast), or a new
container per test class (full isolation)?

**Done when:** the CI check runs and passes on this PR.

### 03 · Event and seat schema

Branch: `feat/03-event-seat-schema`

**Scope**

- Flyway migration for events and seats with `NOT NULL`, foreign key and uniqueness constraints
  (a seat's row and number are unique within its event or venue).
- JPA entities and Spring Data repositories. Hibernate only validates the schema
  (`ddl-auto: validate`); it never creates or changes it.
- Repository integration tests on Testcontainers.

**Decision to make:** is the venue a text column on `events` or its own `venues` table? Do seats
belong to an event or to a venue?

**Done when:** migrations apply on an empty database, the entity mapping validates and repository
tests pass.

### 04 · Admin endpoints

Branch: `feat/04-admin-endpoints`

**Scope**

- Endpoints to create an event and its seats. They stay unsecured until PR 07.
- Request and response DTOs, Bean Validation (the event date must be in the future).
- Global error handling with Problem Details (RFC 9457): `400` for validation errors, `404` for
  unknown resources, `409` for duplicate seats.

**Decision to make:** does the admin send an explicit list of seats, or a generator such as
"rows A to J, 20 seats each"?

**Done when:** integration tests cover success, validation errors and duplicates, and every error
response is a Problem Details document.

### 05 · Public browsing

Branch: `feat/05-public-browsing`

**Scope**

- Paginated list of upcoming events, event details, seats of an event.

**Decision to make:** what shape does a paginated response have (Spring's `PagedModel` or our own page
DTO)? Are an event's seats paginated too, or returned whole as a seat map, as a deliberate exception
to [D-008](decisions.md)?

**Done when:** tests cover pagination parameters, the seat list and `404` for an unknown event.

### 06 · User registration

Branch: `feat/06-user-registration`

**Scope**

- Flyway migration for users: unique email, role column, password stored as a BCrypt hash.
- Registration endpoint. It always creates a `USER`. Validation, and `409` when the email is
  already in use.
- The password or its hash never appears in a response or a log line.

**Decision to make:** how does the first `ADMIN` account come to exist: created at startup from
environment variables, inserted by a Flyway migration, or promoted manually with SQL?

**Done when:** tests cover successful registration, duplicate email and invalid input.

### 07 · Login with JWT and roles

Branch: `feat/07-jwt-login-roles`

**Scope**

- Login endpoint that returns a short-lived JWT. The signing key comes from an environment variable.
- Stateless security. Proposed access rules, confirmed in this PR: admin endpoints require `ADMIN`,
  booking endpoints require a logged-in user, event browsing is public.
- `401` and `403` responses as Problem Details.
- Swagger UI "Authorize" button for bearer tokens.

**Decision to make:** validate tokens with Spring Security's built-in JWT support (OAuth2 Resource
Server) or with a custom filter and the jjwt library? How long does a token live?

**Done when:** tests cover no token (`401`), invalid or expired token (`401`), `USER` on an admin
endpoint (`403`) and `ADMIN` on an admin endpoint (success).

### 08 · Seat booking

Branch: `feat/08-seat-booking`

**Scope**

- Flyway migration for bookings. The database guarantees at most one active booking per seat. If
  seats belong to a single event, the constraint can be on the seat alone.
- Booking endpoint, one seat per request. A violated constraint becomes `409 Conflict`.
- Events that have already started reject bookings ([D-013](decisions.md)).
- The seat list from PR 05 shows each seat as available or taken.

**Decision to make:** what happens to a cancelled booking: the row is deleted, or it stays with status
`CANCELLED` and uniqueness becomes a partial unique index over active bookings only? This is decided
here because it shapes the schema.

**Done when:** tests cover a successful booking (`201`), a second booking for the same seat (`409`),
an event that has already started (rejected) and seat availability.

### 09 · Concurrency test

Branch: `feat/09-concurrency-test`

**Scope**

- Integration test on real PostgreSQL: 50 different users, 50 threads released at the same moment,
  all booking the same seat.
- It asserts exactly one `201`, exactly 49 `409` responses and exactly one booking row in the database.
- A [`decisions.md`](decisions.md) entry explaining why a unique constraint was chosen over pessimistic
  locking (`SELECT ... FOR UPDATE`) and optimistic locking (`@Version`), including deadlocks.

**Decision to make:** does the test go through HTTP against the fully started application (realistic,
slower), or call the service layer directly (faster, skips part of the stack)?

**Done when:** the test passes on 10 consecutive runs and in CI.

### 10 · Idempotency key

Branch: `feat/10-idempotency-key`

**Scope**

- `Idempotency-Key` header required on booking requests (`400` when missing).
- Repeating a request with the same key returns the original booking and creates nothing new.
- Reusing a key for a different seat is rejected.
- Note: after a failed statement PostgreSQL aborts the rest of the transaction, so the existing
  booking must be looked up in a separate transaction. The code must also tell which constraint
  failed: seat already taken or key already used.

**Decision to make:** store the key on the booking row (unique per user and key), or in a separate
table that stores the full original response?

**Done when:** tests cover a replayed request, two identical requests sent concurrently, key reuse
with a different seat and a missing header.

### 11 · My bookings and cancellation

Branch: `feat/11-bookings-cancellation`

**Scope**

- Paginated list of the current user's bookings.
- Cancellation: only the owner, only before the event starts ([D-014](decisions.md)). The seat
  becomes available again.

**Decision to make:** `DELETE /bookings/{id}` or `POST /bookings/{id}/cancel`? Does a user who tries
to cancel someone else's booking get `403`, or `404` (which does not reveal that the booking exists)?

**Done when:** tests cover cancelling by the owner, someone else's booking, an event that has already
started, and booking the same seat again after a cancellation.

### 12 · Docker image and GHCR

Branch: `feat/12-docker-image`

**Scope**

- Container image for the application. The application is added to `docker-compose.yml`, so the
  whole stack starts with one command.
- On every merge to `main`, GitHub Actions builds the image and publishes it to GitHub Container
  Registry, tagged with the commit SHA.

**Decision to make:** a multi-stage `Dockerfile` written by hand (easy to explain), or an image built
by Spring Boot with Cloud Native Buildpacks (no `Dockerfile`)?

**Done when:** `docker compose up` runs the application and the database, and the image appears in
GHCR after the merge.

### 13 · Azure deployment

Branch: `feat/13-azure-deployment`

**Scope**

- Supabase PostgreSQL project. The application connects through the Supabase session pooler
  (port 5432, IPv4, supports prepared statements), not the transaction pooler (port 6543).
- Azure resources created by hand in the portal: resource group, Container Apps environment and
  container app, scaling to zero.
- Environment variables and secrets set on the container app: database connection, JWT signing key,
  first admin account.
- GitHub Actions deploys the new image to Azure Container Apps on every merge to `main`.
- Keep-alive: a scheduled workflow calls an endpoint that reads from the database once a day, so the
  free Supabase project is not paused for inactivity.

**Decision to make:** how does GitHub Actions authenticate to Azure: OpenID Connect (federated
credentials, no stored secret) or a service principal secret stored in GitHub Secrets?

**Done when:** the public URL serves Swagger UI and the full flow (register, login, book, cancel)
works against the deployed API.

### 14 · README

Branch: `feat/14-readme`

**Scope**

- Architecture diagram in Mermaid (rendered by GitHub), the key decisions in short, link to the live
  API, example `curl` requests, how to run locally and run the tests, a note that the first request
  after a quiet period can take a few seconds.

**Decision to make:** does the README open with the concurrency story (problem, solution, the
50-request test) or with the feature list?

**Done when:** someone new can clone, run and test the project by following only the README.
