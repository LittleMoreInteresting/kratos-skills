# Ops & Resilience Patterns (go-kratos)

## Configuration

- Keep runtime config in `configs/` and `internal/conf` structs.
- Validate critical config at startup (ports, DSN, registry endpoints).
- Separate dev/staging/prod values.

## Observability

- Structured logging with correlation IDs.
- OpenTelemetry traces on inbound and outbound calls.
- RED metrics (rate, errors, duration) on key endpoints.

## Resilience

- Set server/client timeouts explicitly.
- Add circuit breaker or bulkhead at downstream boundaries.
- Use graceful shutdown to drain traffic and close resources.

## Deployment checklist

- health/readiness endpoints configured
- migration strategy defined
- rollback plan documented
- alert thresholds reviewed with SLO
