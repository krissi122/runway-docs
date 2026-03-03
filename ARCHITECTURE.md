# Backend Architecture Reference

## Package Structure

### Backend — by layer

All entities share the same layer packages. The projection engine and its
orchestration service live naturally here without cross-feature import issues.

```
com.runway/
  models/
    Card.kt                  ← Domain model
    CardRow.kt               ← DB row model
    FixedExpense.kt
    FixedExpenseRow.kt
    ...
  dao/
    CardDao.kt               ← JDBI SQL Object interface
    FixedExpenseDao.kt
    ...
  accessors/
    CardAccessor.kt          ← Wraps DAO, owns Row ↔ domain conversion
    FixedExpenseAccessor.kt
    ...
  services/
    CardService.kt           ← Business logic
    ProjectionService.kt     ← Orchestrates across multiple accessors
    ...
  resources/
    CardResource.kt          ← JAX-RS resource + request/response types
    FixedExpenseResource.kt
    ...
```

### Frontend — by feature (page)

Each top-level directory in the frontend corresponds to a page or major view.
See `runway-ui` for frontend structure details.

---

## Layers

```
HTTP Request
    ↓
Resource       — HTTP concerns only: routing, parsing, serialization
    ↓ domain model
Service        — Business logic and orchestration
    ↓ domain model
Accessor       — Data access: wraps DAO, converts Row ↔ domain
    ↓ row model
DAO            — Pure SQL via JDBI SQL Object interface
    ↓
Database
```

### Resource
- Owns request/response types (co-located in the same file or feature package)
- Maps request → domain model **before** calling the service
- Maps domain model → response **before** returning to the client
- No business logic, no SQL awareness

### Service
- Speaks domain models only — no Row models, no HTTP types
- Orchestrates calls to one or more Accessors
- Home for any business rules that don't belong in the domain model itself

### Accessor
- Holds an `onDemand` DAO proxy (thread-safe, acquires/releases connection per call)
- Owns the Row ↔ domain model conversion
- Exposes a clean domain-typed API to the Service

### DAO
- JDBI SQL Object interface only — annotated with `@SqlQuery` / `@SqlUpdate`
- Speaks `Row` models exclusively
- No logic: if it isn't SQL, it doesn't belong here

---

## Model Types

| Type | Example | Purpose |
|---|---|---|
| Domain model | `Card` | Business representation; used across Service and Accessor |
| Row model | `CardRow` | Direct DB column mapping; used only inside Accessor and DAO |
| Request type | `CreateCardRequest` | Inbound API payload; lives with the Resource |
| Response type | `CardResponse` | Outbound API payload; lives with the Resource |

---

## Row ↔ Domain Mapping

Use **extension functions** by default. Place them in the feature package,
close to the models they concern.

```kotlin
// CardRow.kt or a CardMappings.kt in the cards/ package
fun CardRow.toDomain(): Card = Card(
    id = id,
    name = name,
    balance = balance,
    // ...
)

fun Card.toRow(): CardRow = CardRow(
    id = id,
    name = name,
    balance = balance,
    // ...
)
```

**Exception:** when mapping requires non-trivial field manipulation (e.g.,
derived values, unit conversions, conditional logic), map inline inside the
Accessor method instead of forcing it into an extension function.

---

## Request ↔ Domain Mapping

Done in the **Resource layer**, before calling the service.

```kotlin
@POST
fun create(request: CreateCardRequest): Response {
    val card = Card(
        id = UUID.randomUUID(),
        name = request.name,
        balance = request.balance,
        // ...
    )
    val created = service.create(card)
    return Response.ok(CardResponse.from(created)).build()
}
```

---

## JDBI Conventions

- Install `KotlinPlugin`, `KotlinSqlObjectPlugin`, and `PostgresPlugin` at startup
- `KotlinPlugin` maps `snake_case` column names to `camelCase` properties automatically
- Accessors hold an `onDemand` proxy — do not pass raw `Handle` or `Jdbi` into Services or Resources
- Use `jdbi.inTransaction` in the Accessor (or a coordinating Service) when multiple DAO calls must be atomic

```kotlin
class CardAccessor(jdbi: Jdbi) {
    private val dao = jdbi.onDemand<CardDao>()

    fun findById(id: UUID): Card? = dao.findById(id)?.toDomain()
    fun listActive(): List<Card> = dao.listActive().map { it.toDomain() }
    fun insert(card: Card): Card { dao.insert(card.toRow()); return card }
}
```

---

## Dependency Injection

**Phase 2–3:** `Jdbi` is constructed in `RunwayApplication.run()` and passed
directly to Accessors, which are passed to Services, which are passed to
Resources. No DI framework yet.

**Phase 4+:** Dagger is introduced when the dependency graph grows complex
enough (engine wiring, multiple accessors feeding a single service, etc.).
