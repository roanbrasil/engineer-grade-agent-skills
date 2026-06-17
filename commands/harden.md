# /harden — Resilience, Security, Observability

Make the implementation production-ready: resilient under failure, observable in production, secure by design.

## What this command does

1. Audits resilience: circuit breakers, retries, timeouts, bulkheads present and configured
2. Audits observability: structured logging with correlation IDs, metrics, traces instrumented
3. Audits security: input validation, auth, secrets not in code, no injection vectors
4. Reviews error handling: no swallowed exceptions, appropriate error responses
5. Reviews API contract: backward-compatible, documented, versioned

## Skills activated

- `resilience-patterns` — circuit breaker, retry+jitter, bulkhead, timeout, rate limiter
- `observability-excellence` — structured logs, metrics (RED/USE), distributed traces
- `api-contract-design` — error format (RFC 7807), versioning, breaking change detection
- `kafka-mastery` — DLT (dead letter topic), retry topics, consumer error handling
- `microservices-excellence` — inter-service auth, data ownership violations

## Usage

```
/harden <component or service>
```

Examples:
- `/harden the payment service client`
- `/harden the Kafka consumer pipeline`
- `/harden the user API endpoints`

## Hardening Checklist

### Resilience
- [ ] Every external call has a timeout configured
- [ ] Retries use exponential backoff with jitter
- [ ] Circuit breaker protects downstream dependencies
- [ ] Bulkheads isolate failure domains
- [ ] Dead letter queue / topic for unprocessable messages

### Observability
- [ ] All requests have a correlation/trace ID
- [ ] Logs are structured JSON (no plain string concatenation)
- [ ] RED metrics instrumented (rate, errors, duration)
- [ ] Distributed trace spans for all external calls
- [ ] Health check endpoint (liveness + readiness)

### Security
- [ ] No secrets in source code or logs
- [ ] All user input validated at boundary
- [ ] Authentication checked before business logic
- [ ] Authorization checked at use case level
- [ ] No verbose error messages in production responses

### Error Handling
- [ ] All exceptions caught at appropriate level
- [ ] Error responses follow RFC 7807 Problem Details
- [ ] Transient vs permanent errors handled differently
