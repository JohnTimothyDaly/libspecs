# Configuration Layering with Type Safety Specification (MK2.1)

## 1. Introduction

This specification defines a **Configuration Layering with Type Safety** library that is portable across:

- Ruby
- Python
- JavaScript
- TypeScript
- Gleam
- Elixir
- PHP
- Rust
- Go
- C++

Primary goals:

1. **Deterministic layering** across multiple sources (defaults, file, env, db/session/runtime, secrets, overrides).
2. **Type-safe access** with explicit schema validation and coercion.
3. **Operational explainability** with source-layer tracing and redaction support.
4. **Connector bootstrap ergonomics** via schema-validated connector factories.
5. **Portable API** for OO and functional ecosystems.

---

## 2. Scope and Non-Goals

### In scope

- Layered config resolution with deterministic precedence.
- Schema-based validation, coercion, and normalization.
- Key-level caching and invalidation where implemented.
- Diagnostics and structured error reporting.
- Connector bootstrap (`schema -> resolved config -> connector instance`).

### Out of scope

- UI or CLI config editors.
- Full secret vault implementation (only adapter hooks).
- Guaranteeing identical runtime internals across languages (only identical observable semantics).

---

## 3. Core Concepts

### 3.1 Config layers

A configuration is composed from ordered layers. Recommended precedence (lowest to highest):

1. **Defaults**
2. **File**
3. **Environment variables**
4. **Database/provider-backed dynamic layer**
5. **Session**
6. **Runtime overrides**

Notes:

- Resolve by **provider order** and choose the **last non-`undefined`/non-`nil` value**.
- File providers pre-resolve `common + environment` by deep-merging map/hash nodes.

### 3.2 Schema and types

Each config key is described by a schema entry.

```text
FieldSpec {
  path: string
  type: string           // string|integer|float|boolean|url|duration|secret
  required: bool
  default: any
  enum_values?: any[]
  pattern?: regex
  lock_after?: layer     // TS-style lock boundary semantics
  secret: bool
}
```

### 3.3 Validation and normalization

Validation is deterministic and path-oriented. Coercion/normalization behavior proven in code:

- `boolean`: accepts `true|1|yes|y|on` and `false|0|no|n|off` (case-insensitive, trimmed)
- `integer`, `float`: parsed explicitly from string/numeric input
- `duration`: accepts two forms:
  - **Integer or float input**: treated as milliseconds (e.g. `5000` → 5 seconds).
  - **String input**: one or more `{integer}{unit}` tokens, where unit ∈ `ms | s | m | h | d`, evaluated additively (e.g. `"1h30m"` = 5400 s, `"500ms"`, `"2h"`). Leading/trailing whitespace is trimmed; case-insensitive units.
  - Coerced result: the language-native duration type where one exists (e.g. `time.Duration` in Go, `ActiveSupport::Duration` in Ruby); integer milliseconds otherwise.
- `url`: requires `http://` or `https://`
- env-key normalization: uppercase + non-alnum to `_`, dot/underscore mapping (Gleam)

Important edge case: empty environment variables (`""`) are treated as **set values**, not missing values.

### 3.4 Caching model

Caching is key-level and optional.

- TS/Ruby: cache stores resolved winner + timestamp; TTL optional.
- Invalidation patterns: `invalidate(key)`, namespace invalidation, full reload.

### 3.5 Connector Factory (Bootstrap Pattern)

Connector bootstrap is a first-class concept:

1. Register schema under connector/service name.
2. Register connector builder/factory under the same name.
3. Validate schema before construction.
4. Resolve a typed/normalized map and construct connector.
5. Provide redacted config views for support/debugging.

This pattern is production-proven in TS (`ConnectorFactory`) and Ruby (`ConnectorFactory`).

---

## 4. Resolution Algorithm

### 4.1 High-level algorithm

1. Build provider chain in explicit precedence order.
2. For file-backed provider(s), pre-resolve `common + environment` map.
3. For each key, scan providers and keep the latest non-empty-by-nullability value:
   - TS: ignore only `undefined`
   - Ruby: ignore only `nil`
   - Gleam: ignore only missing/`None`
4. Apply lock semantics. Two models exist; implementations choose one:
   - **Cutoff model** (TS/Ruby): `lockAfter` names a layer; any provider at a higher precedence than that layer is ignored for the key, and the conflict is surfaced as a diagnostic.
   - **Explicit lock model** (Gleam): `locked_keys[key] = layer` declares that only the named layer may supply the value; all other providers are ignored for the key.
   Both models emit the same `LockConflict` diagnostic shape and are unified in the pseudocode via `lock_rule.blocks(provider_index, provider.layer)`.
5. Apply schema default only when no value is present after lock/precedence.
6. Coerce to target type and validate `required`, `enum`, `pattern`, constraints.
7. Return immutable snapshot/result plus diagnostics/errors.

### 4.2 Merge semantics

- **Primary cross-provider merge mode:** key-wise winner selection (higher precedence layer wins).
- **File layer pre-merge (TS/Ruby):** deep merge for map/hash values.
- **Scalars and lists/arrays:** replaced by higher layer value.
- **`chain_load_all` style map merges (Gleam):** right-hand provider wins on key collision.

### 4.3 Provider `get` vs `load_all`

The resolution pass uses two provider access modes:

- **`get(key)`** — point lookup used in the per-key loop (§4.4). All providers must implement this.
- **`load_all()`** — returns the provider's full key-value map. Used when bulk pre-fetching is more efficient (file providers, in-memory hash providers). Implementations may call `load_all` once per provider at the start of resolution and wrap the result as a `HashProvider`, replacing per-key `get` calls for that provider. Providers that cannot enumerate all keys (e.g., live DB or secrets adapters) may omit `load_all` and rely solely on `get`.

The `chain_load_all` merge pattern (Gleam §4.2) calls `load_all` on each provider in order and merges the maps right-to-left, producing a single flat snapshot before the validation pass.

### 4.4 Pseudocode

```text
function resolve(plan, schema_registry, opts): ConfigResult
  cache = opts.cache
  output = empty_map
  diagnostics = []
  errors = []

  for key in schema_registry.all_keys_sorted():
    if cache.has_fresh(key):
      winner = cache.get(key)
    else:
      winner = { value: MISSING, source: null, locked_conflict: false }
      lock_rule = schema_registry.lock_rule_for(key)  // lockAfter or explicit lock layer

      for (index, provider) in plan.providers_in_order():
        candidate = provider.get(key)

        // Important: empty string is NOT treated as missing
        if candidate is MISSING_BY_NULLABILITY:   // undefined | nil | None only
          continue

        if lock_rule.blocks(index, provider.layer):
          winner.locked_conflict = true
          continue

        winner = { value: candidate, source: provider.name, locked_conflict: winner.locked_conflict }

      cache.put(key, winner)

    entry = schema_registry.entry_for(key)
    effective = (winner.value is MISSING) ? entry.default : winner.value

    if effective is MISSING and entry.required:
      errors.push(error_required(key))
      continue

    if effective is MISSING:
      // Optional key with no value and no default — omit from output entirely.
      if winner.locked_conflict:
        diagnostics.push(lock_conflict_issue(key, entry.lock_rule))
      continue

    coerced = coerce(effective, entry.type)
    if coerced is COERCE_ERROR:
      errors.push(error_type(key))
      continue   // do not store type-incorrect value in output

    validate_enum_pattern_constraints(key, coerced, entry, errors)

    output[key] = coerced   // store the coerced, normalized value
    if winner.locked_conflict:
      diagnostics.push(lock_conflict_issue(key, entry.lock_rule))

  // On failure, `output` contains only keys that were successfully coerced.
  // Callers may inspect it for partial diagnostics but must not treat it as
  // a valid config snapshot.
  if errors.any():
    return failure(output, errors, diagnostics)

  return success(freeze(output), diagnostics)
```

---

## 5. API Drafts

### 5.1 Object-oriented API

```text
class ConfigLoader {
  constructor(schemaRegistry: SchemaRegistry, providers: Provider[], opts?: LoaderOptions)

  get(key: string, opts?: GetOptions): unknown
  require(key: string, opts?: GetOptions): unknown
  validateSchema(schemaName: string): true
  resolveSchema(schemaName: string): map<string, any>
  explain(key: string, schemaName?: string): ExplainResult
  dump(opts?: DumpOptions): map<string, DumpEntry>
  reload(): this
  invalidate(key: string): void
  invalidateNamespace(prefix: string): void
  setRuntime(key: string, value: any): void   // optional — may be omitted in read-only implementations
}

class ConnectorFactory {
  constructor(configLoader: ConfigLoader)

  register(name: string, builder: (resolved: map<string, any>) => Connector): void
  create(name: string): Connector
  redactedConfig?(name: string): map<string, any>
}

class Layer {
  id: string
  source: Source
  precedence: number
  merge_policy: MergePolicy
}
```

### 5.2 Non-OO / functional API

```text
resolve(plan, schemaRegistry, opts) -> Result<ConfigSnapshot, ConfigError[]>
get(config, key, expectedType) -> Result<T, ConfigError>
validate(config, schemaName) -> Result<ValidatedMap, ConfigError>
invalidate(cache, key_or_prefix) -> cache
reload(system) -> system

register_connector(factory, name, builder) -> factory
create_connector(factory, name, configSystem) -> Result<Connector, ConfigError>
```

Rust-like sketch:

```text
fn resolve(schema: &SchemaRegistry, plan: &Plan, cache: &mut Cache) -> Result<Snapshot, Vec<ConfigError>>
```

Go-like sketch:

```text
func Resolve(schema SchemaRegistry, plan Plan, cache Cache) (Snapshot, []ConfigError)
```

Gleam-like sketch:

```text
resolve(plan, schema, cache) -> Result(ConfigSnapshot, List(ConfigError))
```

---

## 6. Algorithms and Data Structures

### 6.1 Data structures

- **ConfigMap**: `map<string, Value>`
- **Provider**: `{name, get/load, load_all, optional set/reload}`
- **SchemaRegistry**: map of schema name -> schema definition
- **SchemaEntry**: `{key, type, required, default, enum/pattern/constraints, secret, lock_rule}`
- **ConfigSnapshot**: immutable view + typed getters
- **CacheStore**: key -> `{value, source, timestamp, lock_conflict}`
- **DiagnosticsReport**: validation/lock/source diagnostics
- **ConnectorFactoryRegistry**: connector name -> builder fn

### 6.2 Complexity targets

Let:

- `P` = number of providers
- `K` = number of keys
- `R` = average number of validation rules per key (enum values, pattern check, min/max, length bounds)

Target complexity:

- Key resolution pass: $O(K \times P)$
- Validation: $O(K \times R)$
- File deep-merge per reload: $O(N)$ where `N` is merged node count
- Cache lookup: $O(1)$ average

---

## 7. Security and Reliability Requirements

### Canonical Error Model

All implementations must map failures into this canonical taxonomy:

1. `SchemaNotFound`
2. `KeyNotFound` / `MissingRequired`
3. `TypeMismatch`
4. `ConstraintViolation` (`enum`, `pattern`, min/max/length)
5. `ProviderError` (I/O, parse, upstream adapter)
6. `LockConflict` / `Locked`
7. `ConnectorNotRegistered`

Cross-language projection:

- **TypeScript/Ruby:** raise typed exceptions (`ValidationError`, `MissingKeyError`, connector errors) carrying detail arrays.
- **Gleam:** return `Result(_, ConfigError)` using ADTs (`ValidationFailed`, `ProviderError`, `Locked`, etc.).
- **Interchange format for logs/telemetry:**

```text
ConfigErrorEnvelope {
  code: string
  key?: string
  schema?: string
  layer?: string
  provider?: string
  expected?: string
  got?: string
  details?: string[]
}
```

### Security

1. Use safe parser modes for untrusted config files.
2. Redact `secret` fields in dumps/explain/diagnostics unless explicit unsafe reveal is enabled.
3. Normalize and whitelist environment key mapping (prefix + canonical key transform).
4. Keep runtime override writes explicit and auditable.

### Reliability

1. Never silently swallow provider failures during validation.
2. Required-key failures must be deterministic and path-specific.
3. Lock rule violations should be diagnosable (`lockedConflict`/`Locked`).
4. Reload clears or invalidates caches to avoid stale winner values.

---

## 8. Language-Specific Notes

Production reference implementations exist for **TypeScript**, **Ruby**, and **Gleam**. The notes for those languages reflect patterns proven in those implementations. All other languages (Python, JavaScript, Elixir, PHP, Rust, Go, C++) are **aspirational targets**: the observable semantics defined by this spec must be met, but idiomatic patterns for those ecosystems are left to the implementer.

### Ruby

- Use namespaced modules/classes (`Conf::Framework::*`) with provider inheritance from `BaseProvider`.
- Define schemas via DSL blocks (`Schema.define` / `SchemaRegistry#register` with block) and `Struct` entries.
- Normalize external keys at boundaries (`to_s`, hash key stringification) to avoid symbol/string drift.
- Apply provider precedence by ordered list scan where last non-`nil` value wins.
- Use exception hierarchy (`Error`, `ValidationError`, `MissingKeyError`, `ConnectorNotRegisteredError`) for control flow.
- Connector bootstrap idiom: `factory.register(:name) { |resolved| ... }` then `factory.create(:name)` after `validate_schema!`.

### Python

- No production reference implementation in this corpus.
- For parity, mirror canonical model: explicit provider chain, typed coercion, structured error envelope.

### JavaScript

- No production reference implementation in this corpus.
- For parity, follow TypeScript runtime semantics: explicit coercion, provider-order precedence, redaction defaults.

### TypeScript

- Use explicit interfaces/unions (`ConfigProvider`, `SchemaEntry`, `ConfigType`, `LockLayer`) and runtime coercion from `unknown`.
- Build schemas through `defineSchema`/`SchemaBuilder` and maintain a `SchemaRegistry` keyed by connector/service name.
- Resolve precedence by ordered provider scan and keep last non-`undefined` winner.
- Implement lock boundary via `lockAfter` and emit lock conflict diagnostics (`doctor`).
- Model providers as composable classes (`HashProvider`, `FileLayerProvider`, `EnvProvider`, `DbProvider`, `SessionProvider`, `RuntimeProvider`).
- Use typed error classes (`ValidationError`, `MissingKeyError`, `ConnectorNotRegisteredError`) and redacted dump/explain support.

### Gleam

- provider chain resolves from the ordered list where caller controls precedence by construction order.
- Model domain as ADTs (`ConfigType`, `ConfigValue`, `ConfigError`, `ValidationError`) and return `Result` instead of exceptions.
- Use opaque `Secret` type to prevent accidental secret exposure.
- Compose systems with immutable builder pipeline: `system.new() |> add_provider |> register_schema |> ...`.
- Implement provider contracts as first-class function records (`Provider(load, load_all)`), and chain providers explicitly.
- Keep validation in schema module with constraint combinators and `Result`-based accumulation.
- Use layer ADT + precedence function for deterministic ordering; represent locks as `Dict(key -> Layer)`.

### Elixir

- No production reference implementation in this corpus.
- Recommended parity target: tagged tuples for error model and immutable provider composition.

### PHP

- No production reference implementation in this corpus.
- Recommended parity target: strict scalar coercion helpers, redaction-by-default dumps, canonical error envelope.

### Rust

- No production reference implementation in this corpus.
- Recommended parity target: enums for error taxonomy + `Result`-based resolver/validator pipeline.

### Go

- No production reference implementation in this corpus.
- Recommended parity target: explicit error returns with structured error codes and deterministic provider ordering.

### C++

- No production reference implementation in this corpus.
- Recommended parity target: explicit error objects and deterministic provider scan semantics.

---

## 9. Built-in Normalizers and Validators

Minimum built-ins (aligned to production code):

- `normalize_env_key` (uppercase, `_` substitution, prefix)
- `normalize_file_keys` (stringify nested map keys)
- `coerce_string`
- `coerce_secret`
- `coerce_integer`
- `coerce_float`
- `coerce_boolean` (true/false token sets)
- `coerce_duration`
- `coerce_url` (`http(s)` check)
- `validate_required`
- `validate_enum`
- `validate_pattern`
- `validate_min_max` (where language implementation supports it)
- `validate_length_bounds` (where language implementation supports it)

---

## 10. Acceptance Checklist

Use this checklist to verify implementation quality:

- [ ] Provider order and precedence are deterministic and documented.
- [ ] Nullability semantics are explicit (`undefined`/`nil`/`None` only count as missing).
- [ ] File-layer `common + environment` merge behavior is consistent.
- [ ] Lock rules (`lockAfter` or explicit locked layer) are enforced and diagnosable.
- [ ] Schema validation reports required/type/constraint errors with path detail.
- [ ] Secrets are redacted in explain/dump paths by default.
- [ ] ConnectorFactory bootstraps only after schema validation.
- [ ] Cache invalidation/reload behavior prevents stale resolved values.
- [ ] OO and functional APIs are both documented.

---

## 11. Minimal Implementation Plan

1. Implement canonical models: `SchemaEntry`, `Provider`, `ConfigErrorEnvelope`, `ConnectorFactory` contract.
2. Implement provider-chain key resolution with explicit nullability semantics.
3. Implement optional lock semantics (`lockAfter` and/or explicit lock map) plus diagnostics.
4. Implement file-layer deep-merge helper (`common + environment`) for file providers.
5. Implement coercion + validation pipeline (required/type/enum/pattern/constraints).
6. Implement redaction-aware explain/dump APIs.
7. Implement cache/invalidation hooks and connector bootstrap integration tests.

This design preserves MK2 rigor while adopting the production-proven patterns of TS, Ruby, and Gleam implementations.
