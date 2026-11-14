# SpecKitFinex

A small electronic financial exchange with administrator and participant REST APIs,
price-time order matching, and concurrency-safe, replaceable in-memory storage.

The project is currently in the specification and design phase. The planned stack is
Spring Boot 4.1.0 and JDK 21; application implementation has not started.

- [Project constitution](.specify/memory/constitution.md)
- [Feature specification](specs/001-exchange-simulator/spec.md)
- [Requirements traceability](specs/001-exchange-simulator/requirements-traceability.md)
- [Implementation plan](specs/001-exchange-simulator/plan.md)
- [HTTP API contract](specs/001-exchange-simulator/contracts/api.md)
- [OpenAPI contract](specs/001-exchange-simulator/contracts/openapi.json)
- [Future implementation validation guide](specs/001-exchange-simulator/quickstart.md)

Administrator resources use `/internal`. Participant resources are at the service
root. Handled business rejections return HTTP 200 with an explicit application
result code and `success=false`.
