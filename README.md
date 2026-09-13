# Real-Time Core Payment Engine & Transaction Gateway (RTCPE)

A microservice-based portfolio project built to practice Java/Spring Boot skills and
the tooling expected of a Senior Software Engineer in the Philippine bank/fintech market.

## Goal

- Get back into deep Java/Spring Boot work after a year in DevOps
- Build hands-on experience with the patterns fintech/bank job specs actually ask for:
  event-driven architecture, idempotency, distributed transactions (Sagas), service-to-service
  auth, observability
- Timeline: weekends, ongoing until a new role is landed
- Deployment target: local Docker Compose to start; Kubernetes later if time allows

## Architecture Overview

```
Client (mobile/web/partner API)
        │
        ▼
[API / Transaction Gateway]  ← Spring Cloud Gateway, rate limiting, request validation
        │
        ▼
[Auth Service]  ← OAuth2/OIDC token issuance & validation
        │
        ▼
[Payment Orchestration Service]  ← validates request, generates idempotency key, kicks off flow
        │
   ┌────┼────────────┬───────────────┐
   ▼    ▼             ▼               ▼
[Account Svc] [Fraud/Risk Svc] [Ledger Svc]   [Notification Svc]
   │              │                │               │
   └──────────────┴────► Kafka ◄───┴───────────────┘
                          │
                  [Reconciliation/Settlement Svc] (batch, EOD)
                  [Audit/Compliance Svc] (immutable log)
```

**Cross-cutting concerns:** Postgres per service (database-per-service pattern), Redis
(idempotency keys + caching), Prometheus/Grafana, ELK/Loki, Jaeger/Zipkin (distributed
tracing), Spring Cloud Config.

**Communication style:**
- Synchronous (REST/gRPC) where an immediate response is required: gateway → auth →
  orchestrator
- Asynchronous (Kafka) for money-movement side effects: ledger updates, fraud checks,
  notifications, audit logging

## Hard Parts (ranked by interview relevance)

1. **Idempotency** — a client retries a payment after a timeout; must not double-charge.
   Requires an idempotency key persisted (Redis/Postgres) before processing begins.
2. **Ledger consistency under concurrency** — two transactions hitting the same account
   balance simultaneously. Naive read-then-write balance updates break here.
3. **Distributed transaction coordination** — a payment touches Account, Ledger, Fraud,
   and Notification services. No single DB transaction spans all of them → needs a Saga
   (orchestrated or choreographed).
4. **Message ordering & exactly-once-ish delivery** — Kafka guarantees order only within
   a partition; requires partitioning by account ID and idempotent consumers.
5. **Auth for both humans and services** — user-facing OAuth2/OIDC is separate from
   service-to-service auth (mTLS or client-credentials JWT) between microservices.
6. **Auditability/compliance** — BSP-regulated fintechs expect immutable, tamper-evident
   audit trails, not plain log files.

## Riskiest Piece: Money Movement Consistency

Two approaches worth building, to compare trade-offs (and have a strong interview story):

### Approach A — Orchestrated Saga (synchronous coordinator)
A central Payment Orchestrator calls Account, Ledger, and Fraud services in sequence via
REST/gRPC, issuing compensating calls (reversal entries) if a later step fails.

- **Pros:** Easy to reason about; single place to see the whole flow; straightforward to
  debug/test; maps naturally onto a Spring Boot service-layer method with rollback logic.
- **Cons:** Orchestrator becomes a bottleneck and single point of coordination logic;
  synchronous calls increase latency and risk cascading failures if a downstream service
  is slow.

### Approach B — Choreographed Saga (event-driven via Kafka)
Payment Service publishes `PaymentInitiated`; Ledger Service consumes it, writes a pending
entry, publishes `LedgerReserved`; Fraud Service consumes and publishes `FraudCleared` or
`FraudRejected`; on rejection, a compensating event triggers ledger reversal.

- **Pros:** Services are decoupled; scales better; closer to how real high-throughput
  payment systems (and most fintech job specs) operate.
- **Cons:** Harder to trace "what happened to transaction X" without solid tracing and
  correlation IDs; testing requires simulating event sequences; must handle out-of-order
  and duplicate events explicitly (outbox pattern + idempotent consumers).

**Plan:** Build Approach A first to get the ledger logic correct, then refactor the same
flow into Approach B. The refactor itself — what broke, what was learned — is a strong
interview story.

## Current Focus: Auth Service

### Decision: Spring Authorization Server vs. Keycloak

| | Spring Authorization Server | Keycloak |
|---|---|---|
| Pros | Deep Java reps — implement token issuance, refresh, revocation, custom claims yourself | Production-realistic; most banks run Keycloak or a commercial IAM (Okta, ForgeRock); learn realms, clients, roles, token mappers |
| Cons | Less representative of day-to-day bank work | Less "look at my code," more "look at my config" |

**Decision:** Build with Spring Authorization Server to demonstrate OAuth2 protocol depth,
while being able to speak to Keycloak as the production-standard alternative.

### Setup Order

1. **Project skeleton**
   - Spring Boot 3.x, Java 21
   - Single deployable `auth-service` module (don't over-decompose yet)
   - Postgres for user/client storage (not H2)

2. **Core dependencies**
   - `spring-boot-starter-oauth2-authorization-server`
   - `spring-boot-starter-security`
   - `spring-boot-starter-data-jpa` + `postgresql` driver
   - `spring-boot-starter-validation`

3. **First working slice — Client Credentials grant**
   - Register a client (e.g., `payment-orchestrator-service`) with client ID/secret
   - Stand up the token endpoint; confirm a JWT via `curl`
   - Validate the JWT on a protected endpoint in a second toy service

4. **Second slice — Authorization Code + PKCE**
   - End-user login flow (what a mobile banking app would use)
   - Custom claims (e.g., `accountId`, `roles`) via a `TokenCustomizer`

5. **Docker Compose from day one**
   - Run Postgres + `auth-service` together
   - Leaves room to add Kafka, Redis, and other services later without restructuring

### Things to Practice / Be Ready to Discuss

- Token lifetimes and refresh token rotation strategy
- Client secret storage (env vars now; Vault/Secrets Manager as the production answer)
- JWK rotation (signing key rollover without breaking already-issued tokens)
- Opaque tokens vs. JWTs, and when to choose one over the other

## Progress Log

- [ ] Auth Service — project skeleton + Postgres
- [ ] Auth Service — client credentials grant working end-to-end
- [ ] Auth Service — authorization code + PKCE flow
- [ ] Auth Service — Docker Compose setup
- [ ] Payment Orchestration Service — scaffold
- [ ] Approach A (orchestrated Saga) — ledger + account + fraud happy path
- [ ] Approach A — compensating transactions on failure
- [ ] Approach B (choreographed Saga) — refactor to Kafka events
- [ ] Observability stack (Prometheus/Grafana, tracing)
- [ ] Reconciliation/Settlement Service
- [ ] Audit/Compliance Service
