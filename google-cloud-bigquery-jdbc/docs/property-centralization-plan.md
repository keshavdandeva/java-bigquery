# BigQuery JDBC property parsing centralization plan

## Problem statement

`DataSource` and `BigQueryConnection` both maintain connection property behavior today, but in different ways:

- `DataSource` manually serializes Java bean fields into `Properties` in `createProperties()`.
- `BigQueryConnection` manually parses URL properties in its constructor.

This creates a **dual maintenance path** where adding a new property can easily be done in one place and forgotten in the other.

## Design goal

Create a single source of truth for JDBC connection properties so that:

1. Property names, default values, and type conversion rules are declared once.
2. URL parsing and DataSource `Properties` generation both use the same metadata.
3. Adding a new property requires one metadata change plus optional wiring to runtime consumers.

## Recommended approach

## 1) Introduce a typed property registry

Add a new internal abstraction (for example `BigQueryConnectionOptions` + `PropertyKey<T>`) that centralizes:

- canonical property name (`ProjectId`, `Location`, etc.),
- property type (`String`, `boolean`, `int`, `long`, `List<String>`, `Map<String,String>`),
- default value,
- parser/validator,
- serializer (`T -> String`) for connection `Properties`.

Example shape:

- `PropertyKey<T>`
  - `name`
  - `defaultValue`
  - `parse(String raw)`
  - `serialize(T value)`

- `BigQueryConnectionOptions`
  - typed fields or map keyed by `PropertyKey<T>`
  - `fromUrl(String url, String loggerContext)`
  - `fromDataSource(DataSource dataSource)`
  - `toProperties()`

## 2) Make `BigQueryConnection` depend on parsed options

Refactor constructor flow from:

- many direct calls to `BigQueryJdbcUrlUtility.parse*Property(...)`

to:

- `BigQueryConnectionOptions options = BigQueryConnectionOptions.fromUrl(...)`
- consume typed options (`options.projectId()`, `options.enableHighThroughputAPI()`, etc.)

This isolates conversion/validation logic from connection behavior.

## 3) Make `DataSource` emit options via the same registry

Refactor `DataSource.createProperties()` to:

- build `BigQueryConnectionOptions` from DataSource fields,
- return `options.toProperties()`.

That removes the repeated `if (field != null) setProperty(...)` list as the long-term maintenance surface.

## 4) Use `VALID_PROPERTIES` as metadata seed (optional migration bridge)

`BigQueryJdbcUrlUtility.VALID_PROPERTIES` and `VALID_OAUTH_PROPERTIES` already hold user-facing metadata.

Use this list as a migration bridge:

- augment entries with type/default/parser metadata,
- eventually converge this with the new `PropertyKey<T>` registry.

This avoids defining the same property catalogue twice.

## Incremental rollout strategy

To reduce risk, ship in 4 safe phases:

1. **Phase A: Introduce registry + tests only**
   - no behavior changes yet.
2. **Phase B: Move `DataSource.createProperties()` to registry**
   - verify generated `Properties` are byte-for-byte equivalent.
3. **Phase C: Move `BigQueryConnection` URL parsing to registry**
   - keep existing public behavior and defaults.
4. **Phase D: Delete old parsing branches in `BigQueryJdbcUrlUtility`**
   - keep utility methods that remain broadly useful.

## Test plan

Add focused tests for centralization invariants:

1. **Property completeness parity**
   - Every supported DataSource-settable property exists in registry.
2. **Round-trip tests**
   - URL -> options -> properties preserves value semantics.
3. **Defaulting tests**
   - Missing properties resolve to expected defaults.
4. **Validation tests**
   - invalid booleans/ints/list formats fail with clear errors.
5. **Backwards compatibility tests**
   - current known JDBC URL examples continue to work unchanged.

## Property categories to prioritize

Start with the high-churn/high-risk categories first:

- Core routing: `ProjectId`, `DefaultDataset`, `Location`
- Auth/OAuth properties
- Query behavior: dialect/cache/large results
- HTAPI and write API controls
- Proxy/SSL/timeouts

Then migrate lesser-used or advanced properties.

## Key implementation principles

- Keep property names and defaults in one place only.
- Prefer typed access over stringly-typed map lookups.
- Keep parsing/serialization deterministic and side-effect free.
- Separate **option parsing** from **BigQuery client construction**.

## Expected outcomes

- Reduced bugs from missing dual updates.
- Faster onboarding for adding new properties.
- Cleaner constructor logic in `BigQueryConnection`.
- Better unit testability for connection option behavior.
