# Data & Repository Patterns (go-kratos)

## Layer contract

- `biz` defines repository interfaces near use-cases.
- `data` implements interfaces with SQL/Redis/other backends.
- `biz` never imports concrete database drivers.

## Constructor conventions

- Provide `NewData(...)` for shared clients (DB, cache, broker).
- Provide `New<Domain>Repo(data *Data, logger log.Logger)` per repository.
- Add providers to wire set and regenerate injectors.

## Transactions

- Keep transaction boundaries in `data` layer.
- Expose higher-level atomic operations to `biz` when needed.
- Avoid leaking transaction objects into `biz`.

## Cache strategy

- Prefer cache-aside for read-heavy entities.
- Define TTL by domain tolerance, not arbitrary constants.
- On writes, update or invalidate cache deterministically.
