# HTTP API Patterns (go-kratos)

## Handler responsibilities

In go-kratos, HTTP handlers usually live in `internal/service` and should:

- parse/validate request transport objects
- call biz use-cases
- map results/errors to response DTOs

Avoid putting business branching logic here.

## Suggested flow

1. `service` receives request DTO.
2. `service` invokes `biz` method with context.
3. `biz` orchestrates domain rules and calls repo interfaces.
4. `data` persists/fetches.
5. `service` maps domain output to response DTO.

## Error mapping

- Use domain errors in `biz` (e.g., not found, conflict, invalid state).
- Convert to transport status in `service` (HTTP status or gRPC codes).
- Keep message stable and non-sensitive.

## Middleware baseline

- recovery
- logging
- tracing
- metadata / request-id propagation
- authn/authz (as needed)
- rate limiting (public APIs)
