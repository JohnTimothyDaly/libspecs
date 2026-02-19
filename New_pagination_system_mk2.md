# Language-Agnostic Pagination + Cursor Navigation Specification (MK2)

## 1) Introduction

This document specifies a **portable pagination and cursor navigation library** designed for implementation in:

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

---

## 2) Scope and Goals

### Goals

1. Support both navigation styles:
   - **Offset/limit pagination** (page-based)
   - **Cursor-based pagination** (seek-based)
2. Provide deterministic metadata for UI and APIs:
   - `self`, `first`, `prev`, `next`, `last` links
   - page windows (first/middle/last ranges)
3. Be framework-agnostic and backend-agnostic.
4. Have a language-neutral conformance test suite.

### Non-goals

- No DB adapter implementation in core library.
- No web framework coupling.
- No hard dependency on specific serialization formats beyond JSON/base64url primitives.

---

## 3) Design Principles

1. **Stable ordering is mandatory** for cursor mode.
2. **Sanitize and clamp all client inputs** (`limit`, `offset`, `page`, cursor payload).
3. **Never trust client-generated cursor data** without verification.
4. **Predictable metadata over clever behavior**.
5. **Pure functions first**, optional OO wrapper second.

---

## 4) Core Types

## 4.1 PaginationMode

- `offset`
- `cursor`

## 4.2 PaginationRequest

Common fields:

- `mode: PaginationMode`
- `limit: int`
- `max_limit: int`
- `default_limit: int`

Offset mode fields:

- `offset?: int`
- `page?: int` (1-based; optional convenience input)

Cursor mode fields:

- `after?: string` (opaque cursor)
- `before?: string` (opaque cursor)
- `direction?: "forward" | "backward"` (derived if omitted)

## 4.3 PaginationState (Normalized)

- `limit: int` (clamped to `1..max_limit`)
- `offset: int` (>= 0; for offset mode)
- `page: int` (>= 1; for offset mode)
- `mode`
- `cursor_after?: CursorPayload`
- `cursor_before?: CursorPayload`

## 4.4 CursorPayload (decoded)

- `v: int` (cursor version)
- `key: array` (ordered sort key values)
- `dir: "f" | "b"` (forward/backward marker)
- `f: string` (fingerprint of query/filter/sort)
- `exp?: int` (optional expiry unix timestamp)

## 4.5 PageWindow

- `pages: int[]` (visible page numbers)
- `show_left_gap: bool`
- `show_right_gap: bool`
- `first_pages: int[]`
- `middle_pages: int[]`
- `last_pages: int[]`

## 4.6 PageMeta

- `total_items?: int` (optional in cursor mode)
- `total_pages?: int` (offset mode if total known)
- `current_page?: int`
- `has_prev: bool`
- `has_next: bool`
- `links: { self, first, prev, next, last }` (URLs or null)
- `range_start?: int` (1-based inclusive)
- `range_end?: int`

---

## 5) Algorithms

## 5.1 Normalize Input (all modes)

Input: user params + config (`default_limit`, `max_limit`).

Algorithm:

1. Parse numeric inputs as integers; invalid -> defaults.
2. `limit = clamp(limit || default_limit, 1, max_limit)`.
3. Offset mode:
   - If `page` provided: `page = max(1, page)` and `offset = (page - 1) * limit`.
   - Else: `offset = max(0, offset || 0)` and `page = floor(offset / limit) + 1`.
4. Cursor mode:
   - Decode and verify cursor(s) if present.
   - Reject invalid signature/version/fingerprint/expiry.

Complexity: `O(1)`.

## 5.2 Compute Total Pages

For known `total_items`:

- `total_pages = ceil(total_items / limit)`
- if `total_items == 0`, then `total_pages = 0`

Complexity: `O(1)`.

## 5.3 Compute Page Window (UI-friendly)

Inspired by existing PHP and Ember patterns:

Inputs:

- `current_page`
- `total_pages`
- `max_pages` (default 10)
- `small_section_size` (default 3)
- `large_section_size` (default 5)

Rules:

1. If `total_pages <= max_pages`: show full contiguous range.
2. Else split into first/middle/last sections.
3. Near boundaries: render two sections (`first + last`) only.
4. In middle zone: render three sections with gaps.
5. Never emit duplicate page numbers.

Complexity: `O(max_pages)`.

## 5.4 Offset Navigation Links

Inputs:

- `base_url`
- query params
- `limit`, `offset`, `total_items`

Rules:

- `self`: current offset
- `first`: omit offset or set `0`
- `prev`: `max(0, offset - limit)` if offset > 0 else null
- `next`: `offset + limit` only if `< total_items`
- `last`:
  - if `total_items == 0`: null
  - else aligned last start: `((total_items - 1) / limit) * limit` (integer division)

This keeps robust behavior for non-aligned offsets.

Complexity: `O(1)`.

## 5.5 Cursor Navigation

Prerequisites:

- Deterministic sort list ending with unique tiebreaker (e.g., primary key).
- Query fingerprint generated from normalized filters/sort/search.

Forward query pattern:

- Predicate: `(sort_key_tuple) > cursor.key` for ascending (or equivalent lexicographic condition)
- Fetch `limit + 1` rows
- `has_next = rows.length > limit`
- Return first `limit` rows

Backward query pattern:

- Predicate inverse of forward
- Fetch `limit + 1`
- Reverse result set for client presentation

Cursor encode:

1. Build payload `{v,key,dir,f,exp?}`
2. JSON serialize (canonical key order)
3. Sign with HMAC-SHA256 using secret
4. Output: `base64url(payload) + "." + base64url(signature)`

Cursor decode:

1. Split token
2. Verify signature constant-time
3. Parse JSON, validate schema/version
4. Verify `f` (query fingerprint) and `exp`

Complexity:

- Encode/decode `O(k)` where `k` is payload size.
- Query complexity depends on datastore; intended seek is `O(log n + limit)` with index support.

## 5.6 Query State Serialization

State includes:

- search term
- selected filters/categories
- sort field/order
- pagination params (`page/offset/limit` OR `after/before/limit`)

Rules:

1. Canonical ordering of keys before serialization.
2. Drop null/empty defaults where possible.
3. Preserve unknown query params if configured (`passthrough=true`).
4. Provide parse + serialize round-trip guarantees.

---

## 6) Data Structures

1. **Config**
   - immutable struct/object with defaults and limits.
2. **Normalized request/state**
   - validated struct/object used by all algorithms.
3. **Window representation**
   - arrays for `first/middle/last`; booleans for gaps.
4. **Cursor payload**
   - compact signed map/object; versioned.
5. **Link map**
   - fixed keys (`self/first/prev/next/last`) with nullable values.

---

## 7) API Draft

## 7.1 OO Style (Ruby/Python/TypeScript/PHP/C++)

```text
class PaginationConfig {
  int defaultLimit;
  int maxLimit;
  int maxPages;
  int smallSectionSize;
  int largeSectionSize;
  string? cursorSecret;
  int? cursorTtlSeconds;
}

class PaginationRequest {
  string mode; // "offset" | "cursor"
  int? page;
  int? offset;
  int? limit;
  string? after;
  string? before;
  map<string, any> queryState;
}

class PaginationResult<T> {
  list<T> data;
  PageMeta meta;
}

class Paginator {
  PaginationState normalize(PaginationRequest req, PaginationConfig cfg);

  PageWindow buildWindow(int currentPage, int totalPages, PaginationConfig cfg);

  PageMeta buildOffsetMeta(
    string baseUrl,
    map<string, any> query,
    PaginationState state,
    int totalItems
  );

  CursorToken encodeCursor(CursorPayload payload);
  CursorPayload decodeCursor(string token, string expectedFingerprint);

  PaginationResult<T> paginateOffset<T>(list<T> items, PaginationState state, int totalItems, ...);
  PaginationResult<T> paginateCursor<T>(list<T> itemsPlusOne, PaginationState state, ...);
}
```

## 7.2 Non-OO / Functional Style (Gleam/Elixir/Rust/Go/JS FP)

```text
normalize_request(req, cfg) -> Result(PaginationState, Error)
page_count(total_items, limit) -> Int
page_window(current_page, total_pages, cfg) -> PageWindow
build_offset_links(base_url, query, state, total_items) -> LinkMap
encode_cursor(payload, secret) -> String
decode_cursor(token, secret, expected_fingerprint, now_ts) -> Result(CursorPayload, Error)
serialize_query_state(state, opts) -> String
parse_query_state(query_string, opts) -> Result(QueryState, Error)
```

For Rust/Go, return explicit `Result`/`error` values (no hidden exceptions).

---

## 8) Error Model

Standardized error codes:

- `ERR_INVALID_LIMIT`
- `ERR_INVALID_OFFSET`
- `ERR_INVALID_PAGE`
- `ERR_INVALID_CURSOR_FORMAT`
- `ERR_CURSOR_SIGNATURE`
- `ERR_CURSOR_VERSION`
- `ERR_CURSOR_EXPIRED`
- `ERR_CURSOR_FINGERPRINT_MISMATCH`
- `ERR_UNSTABLE_SORT`

Error values should include machine code + human message.

---

## 9) Security and Correctness Requirements

1. Clamp all numeric inputs.
2. Disallow negative `limit`, `offset`, `page`.
3. Cursor tokens must be signed (HMAC-SHA256 minimum).
4. Signature checks must be constant-time.
5. Cursor payload must include query fingerprint to prevent replay across different filters/sorts.
6. Cursor mode requires stable ordering with unique tiebreaker.
7. Link generation must preserve unrelated query params safely.
8. URL building must use standard encoders (no string concatenation for unsafe values).
9. Avoid SQL injection by requiring parameterized queries in adapter layer.

---

## 10) Performance Requirements

1. All metadata/window operations are `O(1)` or `O(max_pages)`.
2. Offset mode should avoid expensive `COUNT(*)` when caller opts out of totals.
3. Cursor mode should prefer indexed seek (`WHERE tuple > key LIMIT limit+1`).
4. Do not materialize full result sets in memory for pagination calculations.

---

## 11) Language-Specific Implementation Notes

## Ruby
- Prefer immutable structs/value objects.
- Use Rack/URI helpers for query encoding.
- Use `OpenSSL::HMAC` + secure compare.

## Python
- Use dataclasses or pydantic models.
- Use `hmac` + `hashlib.sha256` + `hmac.compare_digest`.
- Be explicit about timezone/epoch handling for cursor expiry.

## JavaScript / TypeScript
- Keep pure core functions; optional class wrapper.
- In TS, strongly type mode unions and result types.
- Use `URLSearchParams`; avoid ad-hoc query-string logic.

## Gleam
- Model errors with custom result types.
- Keep serialization deterministic.
- Avoid runtime exceptions; fail via `Result`.

## Elixir
- Prefer pure modules and structs.
- Use binary-safe base64url and `:crypto.mac`.
- Pattern-match error paths explicitly.

## PHP
- Use strict typing where possible (`declare(strict_types=1)` in modern code).
- Never trust `$_GET` raw values; normalize first.
- Use hash_equals for signature checks.

## Rust
- Represent modes with enums.
- Use serde for canonical serialization and explicit error enums.
- Return `Result<T, PaginationError>` everywhere.

## C++
- Use strong value types for state/config.
- Avoid implicit narrowing in arithmetic; handle overflow checks.
- Prefer tested URL/query helper library; do not hand-roll escaping.

## Go
- Use structs + explicit validation functions.
- Return `(value, error)` consistently.
- Use `crypto/hmac` and constant-time compare.

---

## 12) Conformance Test Specification

A shared test matrix must be implemented in each language.

## 12.1 Input Normalization Tests

- missing limit -> default
- limit > max -> clamp
- negative page/offset -> clamp to 0/1 semantics
- page to offset conversion correctness

## 12.2 Page Count + Window Tests

- zero results
- one page only
- total pages below threshold
- total pages above threshold, current near start
- current in middle
- current near end
- no duplicate pages in sections

## 12.3 Offset Link Tests

- first page: prev null
- middle page: prev/next set
- last page: next null
- non-aligned offset (e.g., 5 with limit 10)
- preserve unrelated query params

## 12.4 Cursor Tests

- round-trip encode/decode success
- tampered payload/signature rejected
- expired cursor rejected
- fingerprint mismatch rejected
- forward and backward traversal correctness
- stable order requirement enforced

## 12.5 Query State Tests

- parse -> serialize -> parse equality
- canonical key order output
- optional passthrough behavior

---

## 13) Implementation Checklist (AI/Human Verification)

Use this checklist to validate generated code.

## Core behavior

- [ ] Supports both `offset` and `cursor` modes.
- [ ] Normalizes/clamps all numeric inputs.
- [ ] Returns deterministic `self/first/prev/next/last` links.
- [ ] Provides page window metadata (`first/middle/last` or equivalent).
- [ ] Preserves query params safely during link generation.

## Cursor correctness

- [ ] Cursor payload is signed and verified.
- [ ] Signature compare is constant-time.
- [ ] Cursor includes version and query fingerprint.
- [ ] Cursor mode enforces stable sort + unique tiebreaker.

## Safety

- [ ] No raw string SQL concatenation in adapter examples.
- [ ] Invalid inputs map to structured errors.
- [ ] No uncaught exceptions on malformed cursor strings.

## Performance

- [ ] Uses `limit + 1` probing for next/prev in cursor mode.
- [ ] Avoids loading full result sets for metadata.
- [ ] Algorithms remain `O(1)` or `O(max_pages)` for metadata.

## Portability

- [ ] Uses only language-standard crypto/url/encoding primitives or clearly documented equivalents.
- [ ] Core logic implemented as pure functions or pure value-object methods.
- [ ] Conformance tests cover all sections in 12.

---

## 14) Minimal Reference Defaults

Recommended defaults:

- `default_limit = 20`
- `max_limit = 100`
- `max_pages = 10`
- `small_section_size = 2`
- `large_section_size = 8`
- cursor TTL: `15m` (optional; can be disabled for internal APIs)

These defaults should be configurable in all language implementations.
