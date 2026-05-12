# Senior Python API & System Design Interview Cheat Sheet

## 30-second intro
I’m a Python developer that specializes in building data pipelines and backend APIs. Most of my work has been Python, FastAPI, ETL, async services, Postgres, and system integrations. At Quantum Health I built pipelines and async APIs that pulled data from multiple systems into one place for reporting and operations. I like messy backend problems — APIs, sync jobs, reliability, and keeping systems cheap and maintainable.

## Tell me about your projects
I’ve worked most deeply on a Security Data Fabric project. I built Python ingestion pipelines to pull data from systems like ServiceNow, Defender, and Grafana, normalized it through bronze/silver/gold layers, and exposed it through async FastAPI services. That gave leadership and security teams one source of truth instead of manual reporting across disconnected tools.

I’ve also done automation around ticket triage, infrastructure cost reduction, and integration-heavy workflows. So my background is really Python backend work, ETL, async APIs, and handling unreliable systems cleanly.

## How I design APIs
I start with business requirements and resource boundaries first. I figure out who uses it, what actions they need, what the source of truth is, and what fields matter. Then I define request and response contracts before I think too hard about implementation.

For example, for a B2B claims system I’d probably start with resources like customers, claims, appointments, and sync-jobs. Then endpoints like:
- POST /claims
- GET /claims/{id}
- PATCH /claims/{id}
- GET /claims?status=open&cursor=abc&limit=50
- POST /claims/{id}/resync

Then I define typed contracts with validation, version responses carefully, keep writes idempotent, and isolate third-party integrations behind service or adapter layers so the API stays stable even if vendors change.

## Contracts
Contracts are the request and response shapes. What fields I accept, what types they are, what’s required, what I return, and what errors look like. I like locking those down early with Pydantic because frontend and backend can work independently once the shape is agreed on.

Example:
- request: customer_id, incident_date, glass_type
- response: claim_id, status, created_at, sync_status

I care about backwards compatibility, so I prefer additive changes over breaking changes.

## Third-party API schema changes
I wouldn’t couple my application directly to a third-party schema. I’d isolate that dependency behind an adapter layer and map vendor payloads into a stable internal contract. If their schema changes, I update the adapter, support both versions during migration if needed, and keep my API stable for consumers. To avoid disruption, I use idempotent writes, retries with backoff, checkpointed sync state, and structured logging with correlation IDs. If sync still fails, I mark it for reconciliation instead of risking silent data loss or taking the whole flow down. That’s similar to how I’ve approached integration-heavy work in my own Python pipeline and API projects — keep the core contract stable, treat dependencies as unreliable, and make failures recoverable.

## Async / ASGI
I use async when the bottleneck is waiting on I/O — database calls, third-party APIs, queues, cloud services. ASGI helps because requests don’t block a worker while they’re waiting. That improves concurrency without just throwing more processes at the problem.

If the work is CPU-heavy, like large transforms or ML scoring, I’d move that to a worker or batch job instead of doing it inline in the request.

## Why FastAPI
FastAPI is a good fit because it’s async-friendly, gives typed request validation, has clean OpenAPI docs, and makes contract-first API work easy. It’s fast to build with, but still structured.

## Keeping Python cheap and efficient
I focus on simple wins first:
- don’t over-fetch
- batch where possible
- paginate results
- stream large datasets with generators
- use async for I/O-bound work
- keep payloads small
- use connection pooling
- move expensive jobs off the request path

Usually cost and performance problems come from too much data movement, too many queries, or doing heavy work synchronously.

## Generators
I use generators in ETL jobs when I’m processing large datasets so I don’t load everything into memory at once. I’d rather stream records, transform them in batches, and write incrementally.

## Pagination
I don’t return giant result sets. For APIs I prefer cursor pagination over offset at scale because it’s more stable and avoids duplicate or skipped records when data changes during reads.

## Decorators
I use decorators for cross-cutting concerns, not core business logic. Good examples are auth, retries, timing, logging, and tracing. It keeps handlers and services cleaner.

## Logging
I keep logs structured and minimal. I want request_id, correlation_id, endpoint, status, duration, and failure reason. Enough to trace a problem without dumping sensitive payloads.

If I need full detail, I’d rather log a reference ID and look up the record safely than spray raw data into logs.

## Error handling
I separate expected operational errors from unexpected failures.
- bad input: 400
- auth/permission: 401 or 403
- missing record: 404
- downstream timeout: 503
- unexpected exception: 500

I return clean errors to the client, but log enough internal context to investigate. For flaky downstreams I use retries with backoff, timeouts, and sometimes circuit-breaker style behavior.

## How I start designing projects
I start with the business problem and the data flow. I want to know:
- who uses it
- what actions matter
- what systems are involved
- what the source of truth is
- latency needs
- failure tolerance
- security/compliance needs
- what success looks like

Then I define the contracts and schemas, then the service boundaries, then the storage and sync strategy, then observability. I try not to jump straight into code.

## ETL thinking
I think about ETL in layers:
- ingest raw data reliably
- normalize it into a stable internal schema
- enrich and validate it
- expose only what consumers need

The big things I care about are schema consistency, idempotent loads, retry safety, observability, and reconciliation when systems drift.

## System design fundamentals to mention
- functional and non-functional requirements
- resource modeling
- contract-first APIs
- versioning
- idempotency
- pagination
- filtering/sorting
- auth and RBAC
- async vs sync boundaries
- source of truth
- eventual consistency
- retries and backoff
- reconciliation jobs
- observability
- rate limiting
- caching strategy
- queue/worker separation
- failure domains
- horizontal scaling
- indexing and query patterns
- connection pooling
- secure defaults

## Redis caching
I’d use Redis only where latency matters and stale data is acceptable for a short window, like hot reference data, read-heavy endpoints, rate limiting, or idempotency keys. I wouldn’t use Redis as the system of record.

Tradeoff:
- lower latency and less DB load
- but added invalidation complexity and possible stale reads

## Load balancer
For a FastAPI web service, I’d usually put it behind an application load balancer or API gateway, then scale stateless app instances horizontally.

## CAP theorem
For cross-system integrations, I usually assume partitions and downstream instability will happen. So I bias toward availability and eventual consistency, then restore correctness with idempotency, retries, and reconciliation.

## Database fundamentals
I try to design around access patterns early — what gets queried most, what needs indexes, and what should be normalized versus precomputed for reads.

For a system like this:
- Postgres as source of truth
- Redis for cache where useful
- queue for async sync
- workers for ETL and sync jobs

## Queue and worker design
I don’t put slow third-party sync work directly in the request path if I can avoid it. I prefer:
request → API → DB → queue → worker → sync result update

Why:
- better response times
- safer retries
- clearer failure handling
- less coupling to vendor uptime

Key terms:
- dead-letter handling
- retry policy
- poison message
- backpressure
- replay/reprocess
- checkpointing

## Observability
If a sync fails, I want to know which record failed, where it failed, how many times it retried, and whether it needs reconciliation — without searching random logs for 30 minutes.

Use:
- logs
- metrics
- tracing
- correlation IDs
- audit trail
- dashboards and alerts

## Security fundamentals
Security-wise I validate inputs at the contract layer, enforce authorization at service boundaries, avoid logging sensitive payloads, and keep secrets/config outside code.

Mention:
- RBAC
- least privilege
- secrets management
- input validation
- audit logging
- PII minimization
- secure defaults
- rate limiting

## Performance fundamentals
I usually get the best performance wins by shrinking the amount of data moved, not by trying clever tricks too early.

Mention:
- async I/O
- batching
- payload minimization
- query tuning
- caching
- streaming/generators
- avoiding N+1 queries
- connection pooling

## Low-level system design answer shape
1. Clarify requirements
- core resources
- traffic shape
- latency target
- source of truth
- sync vs async expectations
- consistency requirements
- failure tolerance

2. Model resources
- customers
- claims
- appointments
- sync-jobs

3. Define API contracts
- request/response models
- validation
- versioning

4. Describe data flow
API request → FastAPI service → validate → Postgres write → queue → worker → legacy adapter → sync status update

5. Describe failure handling
If downstream times out, the request is still persisted, worker retries with backoff, and if it still fails the record is marked pending_reconciliation instead of disappearing.

6. Describe scale
- stateless API layer
- load balancer
- pagination
- Redis for hot reads if needed
- move slow work off request path

7. Describe observability and security
- structured logs
- metrics
- request IDs
- RBAC
- no sensitive data in logs

## Tie-back to my past work
### Security Data Fabric
In my Security Data Fabric work, I had multiple external systems with different schemas and update patterns. I normalized them into an internal data model, separated ingestion from API access, and exposed the unified result through async FastAPI services. That taught me to care a lot about stable internal contracts, observability, retries, and not letting external inconsistency leak into the product layer.

### TF2S3 migration
In the Terraform Cloud to S3 migration work, the main system design lesson was safe automation at scale — dry runs first, state validation, rollback thinking, checkpointing, secret sanitization, and making large operations restartable instead of brittle.

### OhioHealth / device troubleshooting
At OhioHealth, debugging endpoint and IoT communication issues taught me to treat external systems and protocols as unreliable until proven otherwise. That mindset carries directly into API integrations — define boundaries clearly, validate inputs, and always have a fallback plan when dependencies behave badly.

## CARL examples
### Changing requirements
Context: I was working with multiple source systems that all represented similar security data differently.

Action: I created a normalized internal schema and separated source-specific ingestion logic from the API layer so changes in one upstream system didn’t ripple through everything else.

Result: That reduced manual reporting and made downstream analytics much more consistent.

Learning: Schema discipline early saves a lot of pain later. Without it, every consumer ends up coupled to source-specific quirks.

### Reliability tradeoff
Context: Some data came from systems that weren’t reliable enough to trust synchronously.

Action: Instead of making clients wait on upstream availability, I persisted the request, processed sync asynchronously, and tracked sync status explicitly.

Result: The API stayed responsive even when dependencies were flaky.

Learning: For integration-heavy systems, eventual consistency plus good reconciliation is often better than pretending everything can be strongly consistent.

### Cost/performance tradeoff
Context: We had workflows and infrastructure patterns creating unnecessary spend.

Action: I focused on automation, better backend choices, and removing waste instead of adding more infrastructure.

Result: That migration reduced annual cloud spend by $126k.

Learning: The cheapest system design improvement is often reducing complexity and unnecessary operations before scaling hardware.

## Best phrases to use naturally
- internal contract
- adapter layer
- source of truth
- eventual consistency
- idempotent writes
- checkpointed sync state
- reconciliation job
- structured logging
- correlation ID
- move slow work off the request path
- stateless API layer
- clear ownership of data
- graceful degradation
- backpressure
- retry with backoff
- dead-letter handling
- contract-first design

## Final rule
For every answer, include:
1. concrete example
2. failure case
3. tradeoff
4. why I chose it
