# Composable Validation and Error Handling Specification (MK3)

## 1. Introduction

This specification defines a **cross-language, composable validation and error-handling library** that can be implemented in:

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

1. **Composable**: rules and handlers can be combined, nested, and reused.
2. **Predictable**: deterministic execution and deterministic error output ordering.
3. **Secure by default**: no unsafe dynamic execution, safe error surfaces.
4. **Fast enough for hot paths**: low allocations, short-circuit options, precompiled rules.
5. **Portable**: same semantics across OO and non-OO language ecosystems.

---

## 2. Scope and Non-Goals

### In scope

- Validation of maps/objects/structs and scalar values.
- Normalization/coercion + validation pipeline.
- Structured error model with machine-readable codes.
- Error classification and policy-based handling (retry, fail-fast, collect-all).
- Optional async rule execution.
- Localized error message rendering.

### Out of scope

- UI form rendering.
- Database migrations/ORM concerns.
- Full policy engines (authorization beyond validation).

---

## 3. Core Concepts

### 3.1 Pipeline stages

Each validation run executes three stages:

1. **Normalize** input into canonical representation.
2. **Validate** with composable rules.
3. **Finalize** with structured result + error policy actions.

### 3.2 Rule model

A rule is a pure unit with:

- `id` (stable machine key)
- `kind` (built-in/custom)
- `path` (target field path)
- `params` (rule parameters)
- `when` predicate (optional)
- `severity` (`error`, `warning`, `info`)

Rules can be composed with:

- `allOf` (AND)
- `anyOf` (OR)
- `not`
- `if/then/else`
- `forEach` (array/object element validation)
- `dependsOn` (cross-field)

### 3.3 Error model

All failures are represented as structured records, never plain strings internally.

```text
ValidationError {
  code: string                // e.g. "required", "type.invalid", "len.max"
  message_key: string         // i18n lookup key
  message: string | null      // optional pre-rendered fallback
  path: string                // dot/json-pointer style path
  rule_id: string
  severity: error|warning|info
  category: validation|normalization|system|dependency|security
  retryable: bool
  details: map<string, any>   // safe structured metadata
  cause: ErrorRef | null      // internal linkage (no sensitive leak)
}
```

### 3.4 Result model

```text
ValidationResult<T> {
  ok: bool
  value: T | null                 // normalized/canonical value
  errors: ValidationError[]       // stable order
  warnings: ValidationError[]
  stats: {
    rules_total: int
    rules_executed: int
    duration_ms: number
  }
}
```

### 3.5 Error policy model

Inspired by the event processor runtime pattern:

- `collect_all` (default): run all applicable rules.
- `fail_fast`: stop on first non-retryable `error`.
- `budgeted`: stop after `max_errors`.
- `retryable_boundary`: retries only for marked transient/system failures.

---

## 4. Validation Execution Algorithm

## 4.1 High-level algorithm

1. Build immutable `ValidationContext` (input, locale, clock, options, correlation_id).
2. Apply normalizers in deterministic order.
3. Resolve enabled rules (conditionals + feature flags).
4. Execute rules by path groups (stable path order, then declaration order).
5. For each rule result:
   - success: continue
   - failure: append structured error
   - exception: map to structured `system` error
6. Apply error policy (`collect_all`, `fail_fast`, etc.).
7. Produce `ValidationResult` with canonical value + diagnostics.

## 4.2 Pseudocode

```text
function validate(schema, input, options): ValidationResult
  ctx = mk_context(schema, input, options)
  value_or_error = run_normalizers(ctx)

  if value_or_error is fatal:
    return fail_with_normalization_error(value_or_error)

  value = value_or_error.value
  rules = resolve_rules(schema, value, options)
  acc = new_error_accumulator(options)

  for group in group_by_path_stable(rules):
    for rule in group.rules:
      if !predicate_passes(rule.when, value, ctx):
        continue

      outcome = execute_rule(rule, value, ctx)

      match outcome:
        case PASS:
          continue
        case FAIL(err):
          acc.add(err)
        case EXCEPTION(ex):
          acc.add(map_exception(ex, rule, ctx))

      if should_stop(acc, options.policy):
        return finalize(value, acc, ctx)

  return finalize(value, acc, ctx)
```

## 4.3 Determinism guarantees

- Rules execute in deterministic order.
- Errors are emitted in deterministic order.
- Message rendering is separate from error identity.

---

## 5. API Drafts

## 5.1 Object-oriented API (Ruby/PHP/TypeScript/C++ style)

```text
class ValidationEngine {
  constructor(schema: Schema, opts?: EngineOptions)

  registerRule(name: string, fn: RuleFn): void
  registerNormalizer(name: string, fn: NormalizerFn): void
  registerErrorMapper(fn: ErrorMapperFn): void

  validate<T>(input: unknown, opts?: RunOptions): ValidationResult<T>
  validateAsync<T>(input: unknown, opts?: RunOptions): Promise<ValidationResult<T>>
}

interface Schema {
  fields: map<string, FieldSpec>
  rules: RuleSpec[]
  normalizers?: NormalizerSpec[]
  messages?: map<string, string>
}

interface ErrorRuntime {
  classify(err: unknown): ErrorClass
  withBoundary<T>(opName: string, fn: () => T): RuntimeResult<T>
  withRetry<T>(policy: RetryPolicy, fn: () => T): RuntimeResult<T>
}
```

Example rule DSL (string-compatible for legacy migration from Audaris style):

```text
"?set|len:<=100|exp"
```

Recommended canonical schema form:

```json
{
  "field": "user.email",
  "rule": "string.email",
  "params": {"allow_plus": true},
  "severity": "error"
}
```

## 5.2 Non-OO / functional API (Gleam/Elixir/Rust/Go friendly)

```text
validate(schema, input, opts) -> ValidationResult(input_type)
register_rule(registry, name, fn) -> registry
register_normalizer(registry, name, fn) -> registry
with_retry(policy, fn) -> Result(value, RuntimeError)
classify_error(err) -> ErrorClass
```

Rust-like signature sketch:

```text
fn validate<T>(schema: &Schema<T>, input: Value, opts: &RunOptions) -> ValidationResult<T>
```

Go-like signature sketch:

```text
func Validate[T any](schema Schema[T], input any, opts RunOptions) ValidationResult[T]
```

Elixir-like sketch:

```text
validate(schema, input, opts) :: {:ok, value, warnings} | {:error, errors, warnings}
```

Gleam-like sketch:

```text
validate(schema, input, opts) -> Result(Validated(input), ValidationErrors)
```

---

## 6. Algorithms and Data Structures

## 6.1 Data structures

- **RuleRegistry**: hash map `rule_name -> function`.
- **PathIndex**: ordered map `path -> [rules]` for stable grouped execution.
- **ErrorAccumulator**: append-only vector/list (plus severity counters).
- **NormalizationPipeline**: array of pure transformation functions.
- **MessageCatalog**: map `message_key -> template`.

## 6.2 Complexity targets

Let:

- `R` = number of active rules
- `N` = number of input fields visited
- `E` = number of emitted errors

Target complexity:

- Rule indexing: $O(R)$
- Validation pass: $O(R + N)$ (assuming $O(1)$ per rule primitive)
- Error accumulation: $O(E)$
- Memory overhead: $O(R + E)$

## 6.3 Performance guidance

- Precompile regex and parsed rule expressions.
- Avoid reflection in hot loops.
- Reuse immutable schema and compiled plan.
- Avoid allocating message strings during validation unless requested.

---

## 7. Security and Reliability Requirements

## 7.1 Security requirements

1. No dynamic code execution from schema/rule strings.
2. Strict allowlist for built-in rules and normalizers.
3. Error details must avoid secrets/PII by default.
4. Canonical Unicode normalization for text inputs before rule checks.
5. Constant-time compare helper for sensitive token validation use-cases.
6. Regex timeout/guard strategy where runtime supports it (or safe regex subset).

## 7.2 Reliability requirements

1. Rule exceptions must never crash the host process.
2. Exceptions are converted to `system` category errors.
3. Retry policy must only apply to retryable/transient classes.
4. Backoff strategy: exponential with jitter for async/dependency checks.

---

## 8. Language-Specific Implementation Notes

## 8.1 Ruby

- Use symbols for internal keys, JSON-compatible string keys at boundaries.
- Keep `LocalizedString`-style locale allowlist and fallback chain.
- Prefer immutable value objects for schema/rules where practical.

## 8.2 Python

- Use `dataclasses` or `pydantic`-like typed records for error/result payloads.
- Avoid exception-driven control flow in tight loops.
- Provide both sync and `async` rule execution paths.

## 8.3 JavaScript

- Export ESM-first build with CJS compatibility only if required.
- Preserve stable error ordering; avoid object key-order dependence.
- Provide tree-shakeable rule packs.

## 8.4 TypeScript

- Use generics for typed validated output (`ValidationResult<T>`).
- Model error codes as string literal unions.
- Keep invariant helpers (`assertCondition` style) for internal consistency checks.

## 8.5 Gleam

- Represent rules as algebraic data types.
- Return `Result` values; avoid exceptions.
- Compose rule execution through pipelines and pattern matching.

## 8.6 Elixir

- Use tuples (`{:ok, value}` / `{:error, errors}`) and structs for schemas/errors.
- Isolate side-effecting/async rules in separate modules.
- Keep transformations pure and composable in pipelines.

## 8.7 PHP

- Keep backward-compatible rule-string parser for Audaris-like migration.
- Support UTF-8-safe checks (`mb_*` functions).
- Never concatenate SQL with unvalidated values in helper integrations.

## 8.8 Rust

- Model rule outcomes with enums (`Pass | Fail(Error) | System(Error)`).
- Avoid heap allocations in hot paths where possible (borrowed data/views).
- Use trait-based rule registry for extensibility.

## 8.9 Go

- Prefer explicit error returns over panics.
- Use interfaces for pluggable rules and normalizers.
- Keep zero-value-safe option structs and deterministic map iteration via sorted keys.

## 8.10 C++

- Use `std::variant`/`expected`-style outcomes (or equivalent).
- Keep ownership explicit (`unique_ptr`/value semantics) and avoid shared mutable state.
- Precompile regex and guard expensive operations.

---

## 9. Reference Rule Set (minimum built-ins)

Required built-in rules:

- `required` / `set`
- `type.string`, `type.number`, `type.int`, `type.bool`, `type.date`
- `len.eq`, `len.min`, `len.max`, `len.range`
- `string.alnum` (unicode aware)
- `string.safe_text` (expanded character class like Audaris `exp`)
- `number.currency` (strict decimal policy)
- `enum.in`
- `path.valid` / `name.valid` (GraphQL-like constraints from tmp_prj)

Required normalizers:

- `trim`
- `empty_to_null`
- `unicode_normalize`
- `number_decimal_normalize`
- `date_parse_iso`
- `locale_value_fallback`

---

## 10. Compatibility and Migration

1. Include adapter to parse legacy rule strings (`"?set|len:<=10|aln"`) into canonical rule specs.
2. Keep legacy message templates but map to stable machine `code`s.
3. Provide migration tooling to emit warnings for ambiguous or unsafe legacy rules.

---

## 11. Test Strategy

## 11.1 Test layers

- **Unit tests:** each built-in rule and normalizer.
- **Property tests:** deterministic ordering, idempotent normalization.
- **Integration tests:** full schema validation with nested objects and arrays.
- **Resilience tests:** injected rule exceptions, retryable dependency failures.
- **Security tests:** regex abuse, malformed Unicode, sensitive data leak checks.

## 11.2 Golden tests

Store golden files for:

- Canonical error payload JSON.
- Localized rendering snapshots.
- Legacy rule parser output.

---

## 12. Acceptance Checklist (AI/Human Validation)

Use this checklist to verify generated implementations:

- [ ] Input is normalized before validation.
- [ ] Rule execution order and error ordering are deterministic.
- [ ] Output errors are structured with `code`, `path`, `severity`, `category`.
- [ ] Exceptions are mapped to structured `system` errors.
- [ ] Retry policy applies only to retryable/transient errors.
- [ ] Legacy rule strings can be parsed or explicitly rejected with clear errors.
- [ ] i18n message rendering is separated from error identity.
- [ ] Sensitive values are redacted in logs/error details.
- [ ] Built-in rules cover required/min/max/type/enum/path/name/date/number/currency.
- [ ] Unit/integration tests include invalid Unicode, malformed numbers, and date edge cases.
- [ ] Performance for compiled schema avoids per-call rule parsing.
- [ ] OO and non-OO API surfaces both exist and are documented.

---

## 13. Minimal Implementation Plan

1. Implement core data models (`RuleSpec`, `ValidationError`, `ValidationResult`).
2. Implement normalizer pipeline + built-in normalizers.
3. Implement rule registry + minimum built-in rule set.
4. Implement engine with deterministic ordering and policy control.
5. Implement error runtime (`classify`, `boundary`, `retry`).
6. Add legacy parser adapter and golden tests.
7. Add language-specific adapters/wrappers.

This sequence prioritizes **simplicity and speed** while preserving security and sound engineering practices.
