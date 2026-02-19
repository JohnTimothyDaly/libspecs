# New Polyglot Parser Core (MK1)

## 1) Introduction

### Purpose
Polyglot Parser Core (MK1) defines a small, reusable parser architecture that can be implemented consistently in:
- Ruby
- Python
- JavaScript
- TypeScript
- Gleam
- Elixir
- PHP
- Rust
- C++
- Go

It is designed around the strongest shared patterns observed in:
- `cpp_json_examples/json/parser.cpp`
- `Elson/include/Parser.hpp`
- `MissMatch/lib/MissMatch.js` (`makeParser`)
- `PebbleScript/include/Parser.h`

### Design goals
1. **Simplicity first**: minimal primitives, small API surface.
2. **Speed with safety**: linear-time cursor scanning and zero-copy slices when practical.
3. **Predictable errors**: structured, position-aware parse errors.
4. **Language portability**: same conceptual model across OO and functional/procedural languages.
5. **Deterministic behavior**: same input + grammar = same AST + same diagnostics.

### Non-goals
- Not a parser generator.
- Not a full parser-combinator framework.
- Not tied to one grammar (JSON, DSL, etc.).

---

## 2) Scope and Core Concepts

### Required primitives
1. **Cursor management**
   - index, line, column, source length
   - peek/next/consume/has_next
   - checkpoint + rewind for speculative parse
2. **Token/state helpers**
   - character class helpers (`is_whitespace`, `is_digit`, `is_ident_start`, etc.)
   - scanner helpers (`consume_while`, `consume_if`, `expect_char`, `match_string`)
3. **Recursive descent scaffolding**
   - parse entrypoint
   - rule functions returning `Result<Node, ParseError>` (or equivalent)
   - controlled backtracking via checkpoints
4. **Structured parse errors**
   - expected token(s), found token/char, message, position span, rule context

### Recommended split (delivery model)
- **parser-core-cpp**: cursor, scanner, recursive descent base (high-performance baseline)
- **parser-core-js**: ergonomic DSL helpers and lightweight combinator-like wrappers
- **shared assets**:
  - parser design guide
  - cross-language test corpus format

---

## 3) Data Model Specification

### 3.1 Source and position

```text
Position:
  offset: u32|usize
  line: u32
  column: u32

Span:
  start: Position
  end: Position
```

Rules:
- `offset` is byte-based by default.
- `line` and `column` are 1-based.
- For UTF-8 inputs, `column` may be byte column unless language implementation opts into grapheme-aware columns (must document choice).

### 3.2 Cursor state

```text
CursorState:
  source: immutable string/bytes
  offset: integer
  line: integer
  column: integer
```

### 3.3 Parse error

```text
ParseError:
  code: enum/string (e.g., UnexpectedToken, UnexpectedEOF, InvalidEscape)
  message: string
  expected: list<string>
  found: optional<string>
  span: Span
  rule_stack: list<string>
  severity: Error|Warning (Error required; Warning optional)
```

### 3.4 Parse result envelope

```text
ParseResult<T>:
  ok: bool
  ast: optional<T>
  errors: list<ParseError>
  warnings: list<ParseError>
  consumed_all_input: bool
```

Policy:
- Hard-fail mode: stop at first error.
- Recovery mode (optional): continue with synchronization tokens.

---

## 4) Core Algorithm (Language-agnostic)

### 4.1 High-level algorithm
1. Initialize cursor at start.
2. Optionally consume BOM and leading whitespace/comments according to grammar profile.
3. Call root rule (`parse_root`).
4. Consume trailing whitespace/comments.
5. If input remains, raise `UnexpectedCharacters`.
6. Return `ParseResult`.

### 4.2 Rule function contract
Each rule:
1. Captures start position.
2. Tries to parse expected structure.
3. On success, returns node + span(start..current).
4. On failure, returns structured `ParseError` and does not leave cursor in inconsistent state.

### 4.3 Checkpoint/rewind behavior
- Use checkpoint for alternatives:
  - Save cursor state.
  - Attempt rule A.
  - If fail and recoverable, rewind and attempt rule B.
- Do not rewind after irreversible commits unless explicitly designed.

### 4.4 Error strategy
- Prefer most-specific error at farthest cursor position.
- Aggregate expected tokens for same position.
- Include rule stack for context.

### 4.5 Complexity target
- Time: O(n) for deterministic grammars without pathological backtracking.
- Memory: O(depth + AST size).
- Avoid unbounded recursion where host language stack limits are low (offer iterative alternative for deeply nested structures).

---

## 5) API Draft (OO style)

Use in Ruby, Python, TypeScript/JavaScript classes, PHP, C++, and optionally Go structs with methods.

```text
class Cursor {
  constructor(source)
  has_next() -> bool
  peek(ahead = 0) -> char|nil
  next() -> char|nil
  consume() -> void
  consume_if(predicate) -> bool
  consume_while(predicate) -> string/slice
  expect_char(ch) -> Result<void, ParseError>
  match_string(s) -> bool
  checkpoint() -> CursorSnapshot
  rewind(snapshot) -> void
  position() -> Position
}

class ParserCore<TAst> {
  constructor(source, options)
  parse() -> ParseResult<TAst>
  parse_root() -> Result<TAst, ParseError>   // implemented by grammar-specific parser
  push_rule(name) -> void
  pop_rule() -> void
  error(code, message, expected, found, start, end) -> ParseError
}
```

Options:
- `allow_trailing_commas` (example grammar option)
- `max_depth`
- `recovery_mode`
- `track_warnings`
- `unicode_mode` (byte column vs scalar/grapheme policy)

---

## 6) API Draft (Non-OO / Functional / Procedural)

Use in Gleam, Elixir, Rust, Go, and functional-style JavaScript/TypeScript.

```text
type Cursor
type CursorSnapshot
type ParseError
type ParseResult(a)

new_cursor(source) -> Cursor
has_next(cursor) -> bool
peek(cursor, ahead) -> Option<char>
next(cursor) -> { Option<char>, Cursor }
consume_if(cursor, predicate) -> { bool, Cursor }
consume_while(cursor, predicate) -> { slice, Cursor }
checkpoint(cursor) -> CursorSnapshot
rewind(cursor, snapshot) -> Cursor
position(cursor) -> Position

parse(source, options, parse_root_fn) -> ParseResult(ast)

// parse_root_fn signature
parse_root_fn(cursor, context) -> Result({ ast, cursor }, ParseError)
```

Recommended context structure:
- `context.rule_stack`
- `context.options`
- `context.memo` (optional packrat/memoization table)

---

## 7) Algorithms and Data Structures

### 7.1 Required algorithms
1. **Single-pass cursor scan** for terminals.
2. **Recursive descent rule dispatch** for non-terminals.
3. **Checkpoint-rewind alternative parsing** for ambiguous prefixes.
4. **Farthest-error selection** when multiple failures occur.

### 7.2 Optional algorithms
1. **Selective memoization** (packrat-lite)
   - Key: `(rule_id, offset)`
   - Value: success/failure + resulting offset + node/error
   - Use only where backtracking hotspots exist.
2. **Panic-mode recovery**
   - skip until synchronization token set (`]`, `}`, `,`, `;`, newline, etc. per grammar)

### 7.3 Data structures
1. Cursor state struct/class (fixed fields).
2. Rule stack (vector/list).
3. Error list (vector/list).
4. Optional memo table:
   - hash map/dictionary for dynamic languages
   - hash map with compact key structs in Rust/C++/Go
5. AST node shape:

```text
Node:
  kind: string|enum
  span: Span
  children: list<Node>
  value: optional scalar/string/number/object
```

### 7.4 Performance guidance
- Prefer source slicing over string concatenation while scanning.
- Avoid per-character object allocations.
- Keep helper predicates pure and inlineable where possible.
- Minimize exceptions for normal control flow (return result types where language permits).

---

## 8) Security and Sound Engineering Constraints

These constraints are mandatory to address architectural/security concerns:

1. **Depth limits**
   - Configurable `max_depth` to prevent stack exhaustion.
2. **Input size limits**
   - Configurable max source bytes/chars.
3. **Time bounds (host-level)**
   - Parser itself should be linear where possible.
   - Host application should enforce wall-time budget for untrusted inputs.
4. **No eval/exec**
   - Parser core must never execute parsed content.
5. **Unicode safety**
   - Explicitly reject invalid escape sequences and invalid code points.
6. **Deterministic memory behavior**
   - Avoid unbounded growth in error recovery mode.
   - Cap number of collected errors (e.g., 100) before aborting.

---

## 9) Language-specific Implementation Notes

## 9.1 Ruby
- Use `Struct` or small immutable classes for `Position`, `Span`, `ParseError`.
- Avoid exception-driven parsing for common failures; prefer result objects.
- For speed, access bytes (`String#byteslice`) when grammar is ASCII-oriented.

## 9.2 Python
- Use `dataclasses` for model types.
- Prefer explicit return tuples or `Result`-like classes over heavy exceptions in hot paths.
- CPython recursion limits: provide iterative fallback for deep grammars.

## 9.3 JavaScript
- Use plain objects for low overhead; freeze only public API outputs if needed.
- Keep scanner loops monomorphic to help V8 optimization.
- Avoid regex-heavy tokenization if it obscures position tracking.

## 9.4 TypeScript
- Define discriminated unions for AST and `ParseError.code`.
- Enable strict mode (`strict`, `noUncheckedIndexedAccess`) in reference implementation.
- Expose both ESM and CJS builds only if required by consumers.

## 9.5 Gleam
- Model parser state as immutable record, threading through functions.
- Use algebraic data types for result and error enums.
- Keep helper functions small and composable; avoid deep recursion where BEAM stack concerns appear.

## 9.6 Elixir
- Use structs for error/span types.
- Keep parser state immutable; return `{result, state}` tuples.
- For performance, prefer binary pattern matching and integer indices over repeated string splitting.

## 9.7 PHP
- Use value objects for spans/errors in modern PHP.
- Avoid excessive temporary string creation.
- Be explicit about multibyte behavior (`mb_*`) if column semantics are character-based.

## 9.8 Rust
- Use `&str` + byte offset cursor; validate UTF-8 boundaries before slicing.
- Prefer `Result<T, ParseError>` and enums for node/error kinds.
- Avoid `unsafe` unless benchmark-backed and encapsulated.
- Optional: arena allocation for large ASTs.

## 9.9 C++
- Use `std::string_view` for zero-copy scanning.
- Return `expected<T, ParseError>` (or equivalent) instead of exceptions in hot path.
- Guard against integer overflow in offsets and lengths.
- Keep API exception-safe with clear ownership semantics.

## 9.10 Go
- Use byte offsets with explicit rune decoding where needed.
- Return `(node, error)` and carry structured error type with span.
- Favor simple structs and slices; avoid interface-heavy AST where performance matters.

---

## 10) Testing Specification

### 10.1 Required test categories
1. **Cursor tests**
   - `peek`, `next`, `consume_while`, checkpoint/rewind, line/column updates.
2. **Rule tests**
   - each grammar rule success and failure cases.
3. **Error quality tests**
   - expected tokens, found token, span accuracy, farthest-error selection.
4. **Limits tests**
   - max depth, max size, max errors.
5. **Fuzz/property tests** (where ecosystem allows)
   - no panics/crashes, deterministic output.

### 10.2 Shared cross-language corpus format

Use line-delimited JSON (`.jsonl`) for compatibility:

```json
{"id":"valid_001","source":"...","expect_ok":true,"expect_ast_kind":"Root"}
{"id":"invalid_001","source":"...","expect_ok":false,"expect_error_code":"UnexpectedToken","expect_error_offset":12}
```

Fields:
- `id` (string)
- `source` (string)
- `expect_ok` (bool)
- `expect_ast_kind` (optional)
- `expect_error_code` (optional)
- `expect_error_offset` (optional)
- `notes` (optional)

### 10.3 Golden output format
For deterministic comparison, implementations should expose:
- canonical JSON for AST (stable key order)
- canonical JSON for errors

---

## 11) AI/Human Code Generation Checklist (Acceptance)

A generated implementation is acceptable only if all checks pass:

### API and behavior
- [ ] Cursor supports peek/next/consume/checkpoint/rewind.
- [ ] Root parse enforces full input consumption (except allowed trailing trivia).
- [ ] Parse errors include `code`, `message`, `expected`, `found`, and `span`.
- [ ] Farthest-error strategy is implemented.

### Safety and robustness
- [ ] Depth limit is enforced.
- [ ] Input size guard is enforceable.
- [ ] Error count cap exists in recovery mode.
- [ ] No code execution (`eval`/`exec`) in parser core.

### Performance and quality
- [ ] Complexity is linear on representative grammar workloads.
- [ ] No avoidable per-char allocations in hot scanner loop.
- [ ] Benchmarks included for at least small/medium/large corpus sizes.

### Portability and consistency
- [ ] Shared test corpus runs in target language.
- [ ] AST and error golden outputs match expected canonical form.
- [ ] Language-specific caveats documented (Unicode column policy, recursion behavior).

---

## 12) Minimal Implementation Plan (for teams)

1. Implement cursor + error primitives.
2. Implement parser skeleton + root function contract.
3. Implement at least one reference grammar (small JSON subset or DSL subset).
4. Add corpus runner.
5. Add limits and safety guards.
6. Optimize only after benchmark evidence.

---

## 13) Recommended Directory Layout (reference)

```text
polyglot-parser-core/
  spec/
    New_polyglot_parser_core_mk1.md
    corpus/
      cases.jsonl
  implementations/
    ruby/
    python/
    javascript/
    typescript/
    gleam/
    elixir/
    php/
    rust/
    cpp/
    go/
```

This keeps one source-of-truth specification and shared cross-language validation assets.
