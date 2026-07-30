# `$ref` test matrix and coverage plan

This document is a structured planning resource for `$ref` test coverage.
See [`docs/ref-semantics.md`](./ref-semantics.md) for the design principles that this
matrix operationalizes.

---

## 1. Purpose and background

### Why a matrix?

`$ref` touches many independent semantic dimensions simultaneously:

- where a ref lives (inline, `$defs`, external file, …)
- what kind of schema is referenced (primitive, format-backed, object, enum, …)
- what constraints the schema carries
- whether the author expressed explicit modeling intent
- where the ref appears at use-site (top-level property, array item, oneOf branch, …)
- how many times the ref is reused and in what scope

Ad-hoc regression tests catch one specific symptom at a time.
A matrix helps us reason about coverage across the full space:
which combinations have been tested, which are still dark, and which are
highest priority.

### Relationship to `docs/ref-semantics.md`

`docs/ref-semantics.md` defines the intended product semantics:
`$ref` should be transparent by default; backend materialization should
not redefine semantic behavior.

This document turns that principle into a concrete test plan.
It records which combinations of dimensions have test coverage and which
do not, so contributors can fill gaps incrementally rather than
continuing to add one-off patches.

### Goal

Test `$ref` behavior intentionally:

- prove semantic parity across ref shapes (inline ≈ `$defs` ≈ external)
- confirm explicit modeling intent is preserved when the author requested it
- confirm generation stability across all combinations (no crashes, no
  impossible output, no silent validator loss)

---

## 2. Testing philosophy

Two distinct kinds of test assertions are useful, and they should not be
conflated.

### Semantic tests

Semantic tests assert *user-visible behavior*: does the generated code
round-trip correctly, validate the same inputs, and preserve the same field
types regardless of how the schema was structured?

Examples:

- an inline string-with-pattern field and its `$ref` equivalent should
  reject the same invalid inputs at runtime
- a `date-time` field should parse and serialize identically whether the
  format annotation is inline or behind an external ref

Semantic tests are the most durable. They survive refactors and remain
meaningful even when the generator's internal topology changes.

### Implementation-stability tests

Implementation-stability tests assert *generation-level properties*:
does the generator produce output at all, does it produce a named symbol
when one is expected, does it not accidentally produce a duplicate symbol
when none was requested?

Examples:

- generating a schema with a named `title` behind a ref should still produce
  a named Go type
- generating an external primitive ref that qualifies for semantic inlining
  should not leave behind an unreachable standalone declaration

Implementation-stability tests are more fragile under refactors, but
they catch regressions in declaration topology and naming.

### Guideline

**Most `$ref` tests should be semantic tests.** Only add declaration-topology
assertions when the specific goal is to catch a known topology regression
(for example, leftover materialization that should have been eliminated).

Not every test combination needs to assert declaration shape.

---

## 3. Test dimensions

The following dimensions define the full test space.

### 3.1 Ref shape / location

| Value | Notes |
|---|---|
| Inline (no ref) | Baseline / reference case |
| Same-file `$defs` internal ref | `#/$defs/Foo` |
| Same-file `definitions` (legacy) | `#/definitions/Foo` |
| External root ref | `./thing.schema.json` |
| External `$defs` ref | `./thing.schema.json#/$defs/Foo` |
| Nested / transitive external refs | External file that itself refs another file |

### 3.2 Schema kind / materialization class

| Value | Notes |
|---|---|
| Primitive / value-like | `string`, `integer`, `number`, `boolean` |
| Format-backed primitive | `date-time`, `date`, `time`, `ipv4`, `ipv6`, `duration` |
| Specialized integer | Integer with explicit sizing (`int8`–`int64`, etc.) |
| Object | Struct-like schema |
| Enum | String or integer enum |
| Array / collection | Simple array, nested container |
| `additionalProperties` map-like | Object with free-form values |

### 3.3 Constraint family

| Value | Example fields |
|---|---|
| String pattern | `pattern` |
| String length | `minLength`, `maxLength` |
| String const | `const` on string |
| Numeric bounds | `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum` |
| Numeric multiple | `multipleOf` |
| Numeric const | `const` on number / integer |
| Boolean const | `const: true / false` |
| Format / parsing | `format: date-time` etc. |
| Object-level required | `required: [...]` |
| Combination | Two or more of the above |

### 3.4 Explicit modeling intent

| Value | Notes |
|---|---|
| None (pure reuse intent) | No title, no extensions |
| `title` | Author named the concept |
| `goJSONSchema.type` | Author mapped to a Go type path |
| `x-go-type` | Author requested a specific Go type |
| `x-go-ref` | Author requested cross-package type import |

### 3.5 Use-site context

| Value | Notes |
|---|---|
| Top-level property | Most common |
| Nested object property | `$.properties.outer.properties.inner` |
| Array item | `items: { "$ref": "…" }` |
| oneOf branch | `oneOf: [{ "$ref": "…" }, …]` |
| allOf / anyOf branch | Composition contexts |
| `x-go-oneof-envelope` branch | Discriminator envelope |

### 3.6 Reuse pattern

| Value | Notes |
|---|---|
| Single use | Ref appears once |
| Repeated in same scope | Same ref used for two or more sibling properties |
| Repeated across nested scopes | Ref used in parent and child objects |
| Repeated across branches | Ref used in multiple oneOf / anyOf branches |
| Repeated across files / packages | Ref used from multiple root schemas |

### 3.7 Assertion type

| Value | Notes |
|---|---|
| Runtime semantic parity | Round-trip / unmarshal test |
| Generated field shape | Field Go type matches expectation |
| Declaration / symbol preservation | Named symbol exists when expected |
| Generation stability | No crash, no impossible / uncompilable output |

### 3.8 Additional practical dimensions

| Value | Notes |
|---|---|
| Nullable / pointer behavior | Field is optional or pointer-wrapped |
| Package / output boundary | Schema produces output in a different package |
| Recursive / self-referencing | Schema refs itself directly or transitively |
| Mixed primitive + object in same scope | Both kinds of ref in the same parent object |

---

## 4. Canonical buckets and recommended baseline suites

Testing every combination of all dimensions above is not tractable.
Instead, the strategy is to cover each conceptually distinct behavioral area
with a small canonical representative set and then add regression tests
only for bugs or edge cases that fall outside the canonical coverage.

The buckets below are the recommended baseline suites.

### Bucket A – Primitive transparency

**Goal:** Confirm that inline, `$defs`, and external primitive refs are
semantically equivalent for the core constraint families.

Representative dimensions:

- Schema kind: string / integer / number / boolean
- Ref shape: inline + same-file `$defs` + external root + external `$defs`
- Constraints: pattern, length bounds, numeric bounds, const
- Intent: none (pure reuse)
- Assertion: runtime semantic parity

A single canonical schema per constraint family exercised across all four
ref shapes is enough.

### Bucket B – Format-backed primitive transparency

**Goal:** Confirm that format-backed refs (`date-time`, `date`, `time`, `ipv4`,
`ipv6`, `duration`) remain semantically transparent even though their backend
Go representation is a named/specialized type (`time.Time`, `netip.Addr`, etc.).

Representative dimensions:

- Schema kind: format-backed primitive
- Ref shape: inline + external root + external `$defs`
- Constraints: format only (parsing/serialization semantics)
- Intent: none
- Assertion: field Go type matches expected named type; round-trip parity

### Bucket C – Specialized integer transparency

**Goal:** Confirm that integer refs that materialize as non-default Go integer
types (`int64`, `int32`, etc.) do not silently drop numeric constraints.

Representative dimensions:

- Schema kind: integer with explicit sizing
- Ref shape: inline + external root
- Constraints: numeric bounds, const
- Intent: none
- Assertion: runtime semantic parity; validator is generated

### Bucket D – Explicit naming intent preservation

**Goal:** Confirm that when the author signals a named concept (`title`,
`goJSONSchema.type`, `x-go-type`), the generated symbol boundary is preserved.

Representative dimensions:

- Schema kind: string, object
- Ref shape: same-file `$defs` + external root
- Constraints: pattern (string), required (object)
- Intent: `title` / `goJSONSchema.type` / `x-go-type`
- Assertion: named Go symbol exists; field references that symbol

### Bucket E – Unnamed object contextual materialization

**Goal:** Confirm that an object ref without explicit modeling intent
materializes consistently whether the schema is inline, `$defs`, or external.

Representative dimensions:

- Schema kind: object
- Ref shape: inline + same-file `$defs` + external root
- Intent: none
- Assertion: generation stability; field type is consistent Go struct reference

### Bucket F – Named object shared symbol preservation

**Goal:** Confirm that a named object used in multiple places shares a single
generated symbol rather than generating per-use copies.

Representative dimensions:

- Schema kind: object with `title`
- Ref shape: external root
- Reuse: repeated across sibling properties / across parent + child
- Assertion: single declaration emitted; all use-sites reference the same symbol

### Bucket G – Nested and container contexts

**Goal:** Confirm ref transparency holds when refs appear as array items or
inside nested objects.

Representative dimensions:

- Schema kind: primitive + object
- Ref shape: external root
- Use-site: array item, nested object property
- Assertion: field Go type matches expectation; runtime parity

### Bucket H – Composition contexts

**Goal:** Confirm ref transparency holds inside `allOf`, `anyOf`, `oneOf`.

Representative dimensions:

- Schema kind: primitive + object
- Ref shape: same-file `$defs` + external root
- Use-site: `allOf` / `anyOf` / `oneOf` branch
- Assertion: generation stability; field type matches inline equivalent

### Bucket I – oneOf-envelope stress cases

**Goal:** Confirm that `x-go-oneof-envelope` dispatch remains correct when
envelope branches reference external schemas, including mixed
primitive + object branch combinations.

Representative dimensions:

- Schema kind: object branch + primitive branch (mixed)
- Ref shape: external `$defs`
- Use-site: `x-go-oneof-envelope` branch
- Reuse: each branch ref used once
- Assertion: generation stability; discriminator dispatch is correct

### Bucket J – x-go-ref / imported type mapping

**Goal:** Confirm that `x-go-ref` remaps cross-package type references
correctly and that order of schema processing does not matter.

Representative dimensions:

- Schema kind: string + object
- Ref shape: external root
- Intent: `x-go-ref` + `x-go-type`
- Assertion: generated import path is correct; symbol is referenced (not re-declared)

### Bucket K – Compatibility / legacy refs

**Goal:** Confirm that `definitions` (legacy pre-draft-2019) refs work
equivalently to `$defs` refs.

Representative dimensions:

- Schema kind: primitive + object
- Ref shape: same-file `definitions`
- Assertion: generation stability; semantic parity with `$defs` equivalent

---

## 5. TODO checklist

The items below are grouped by bucket and intended to be checked off by
individual PRs. Each item targets one concrete coverage gap.

### Bucket A – Primitive transparency

- [ ] Add canonical inline / `$defs` / external-root parity test for
      string pattern constraints
- [ ] Add canonical inline / `$defs` / external-root parity test for
      string length constraints (`minLength` / `maxLength`)
- [ ] Add canonical inline / `$defs` / external-root parity test for
      string `const`
- [ ] Add canonical inline / `$defs` / external-root parity test for
      integer bounds (`minimum` / `maximum` / `exclusiveMaximum`)
- [ ] Add canonical inline / `$defs` / external-root parity test for
      numeric `multipleOf`
- [ ] Add canonical inline / `$defs` / external-root parity test for
      numeric `const`
- [ ] Add canonical inline / `$defs` / external-root parity test for
      boolean `const`
- [ ] Add external-`$defs` (`#/$defs/Foo`) parity case alongside existing
      external-root cases

### Bucket B – Format-backed primitive transparency

- [ ] Add explicit parity test for `date-time` external ref (inline ≈ external)
      that asserts round-trip correctness, not just generation success
- [ ] Add `date` and `time` external ref parity tests
- [ ] Add `ipv4` and `ipv6` external ref parity tests
- [ ] Add `duration` external ref parity test
- [ ] Confirm that no unnecessary standalone declaration is emitted for
      semantically transparent format-backed refs

### Bucket C – Specialized integer transparency

- [ ] Add external primitive integer ref test with numeric bounds, confirming
      validators are not dropped when the backend type is `int64` or narrower
- [ ] Add external integer ref test for `const` constraint confirming parity
- [ ] Add `minSizedInts`-mode external ref parity test for each integer
      constraint family (bounds, exclusive, multipleOf)

### Bucket D – Explicit naming intent preservation

- [ ] Add named-string (`title`) internal ref test confirming symbol is kept
- [ ] Add named-string (`title`) external ref test confirming symbol is kept
- [ ] Add `goJSONSchema.type` external ref test confirming symbol is kept
- [ ] Add `x-go-type` external ref test confirming symbol is kept
- [ ] Add test confirming that adding `title` to a previously anonymous schema
      behind a ref changes behavior predictably

### Bucket E – Unnamed object contextual materialization

- [ ] Add inline / `$defs` / external-root parity test for unnamed object ref
      (generation stability; no extra duplicate declaration)
- [ ] Add test confirming that unnamed external object ref does not generate
      an extra orphan symbol at the root level

### Bucket F – Named object shared symbol preservation

- [ ] Add test for named object ref repeated in two sibling properties —
      confirm single symbol emitted
- [ ] Add test for named object ref repeated in parent + child object —
      confirm single symbol emitted
- [ ] Add test for named object ref repeated across different root schemas —
      confirm cross-package symbol reference (not re-declaration)

### Bucket G – Nested and container contexts

- [ ] Add inline / external parity test for primitive ref as array item
- [ ] Add inline / external parity test for object ref as array item
- [ ] Add inline / external parity test for primitive ref as nested property
      (two-level object nesting)

### Bucket H – Composition contexts

- [ ] Add primitive ref in `allOf` branch — inline / `$defs` parity test
- [ ] Add object ref in `anyOf` branch — generation stability test
- [ ] Add primitive ref in `oneOf` branch — inline / `$defs` / external parity
- [ ] Add mixed primitive + object refs in `oneOf` — generation stability test

### Bucket I – oneOf-envelope stress cases

- [ ] Add `x-go-oneof-envelope` test where all branches use external object refs
- [ ] Add `x-go-oneof-envelope` test with one integer-ref branch and one
      object-ref branch (mixed materialization classes)
- [ ] Add `x-go-oneof-envelope` test where the same external schema is referenced
      from two different branches (shared ref in envelope)
- [ ] Add `x-go-oneof-envelope` test where branches use external `$defs` refs
      (not root refs)

### Bucket J – x-go-ref / imported type mapping

- [ ] Add `x-go-ref` test for a string type mapping — confirm import is emitted
- [ ] Add `x-go-ref` test for an object type mapping — confirm no re-declaration
- [ ] Add test confirming order-independence: producer schema processed after
      consumer schema still produces correct output

### Bucket K – Compatibility / legacy refs

- [ ] Add `definitions` (legacy) inline / `$defs` parity test for a primitive ref
- [ ] Add `definitions` (legacy) test for object ref — generation stability
- [ ] Add test confirming that mixed use of `definitions` and `$defs` in the
      same file does not cause a crash or silent name collision

---

## 6. Existing coverage and highest-value gaps

### Strongest existing anchors

| Test | What it covers |
|---|---|
| `core/refSemanticInline` | External primitive refs (string/integer/number/boolean/datetime) with constraints; inline ≈ external semantic parity; runtime unmarshal assertions |
| `core/refSemanticNamed` | Named refs (`title`, `goJSONSchema.type`, `x-go-type`) symbol-preservation behavior |
| `validation/primitive_defs` | Same-file `$defs` primitive refs with full constraint family coverage |
| `core/refToPrimitiveString` | External root ref to a plain string schema |
| `core/refToEnum` | External root ref to an enum schema |
| `core/refExternalFormatTransparent` | External `date-time`-only format transparency (generation) |
| `core/refExternalFile` | Basic external file ref to an object schema |
| `core/refExternalFileNestedRefs` | Transitive external ref chain |
| `minSizedInts/externalRefConstInteger` | External integer ref with const under `minSizedInts` mode |
| `core/refXGoImport` | `x-go-ref` cross-package import remapping |
| `core/refOld` | Legacy `definitions`-based ref |

### Highest-value gaps to fill first

1. **External-`$defs` ref parity for primitives** — most existing external ref
   tests use root external refs. The `#/$defs/Foo` form of external ref is
   under-tested, especially for primitives with constraints.

2. **Format-backed primitive runtime parity** — `refExternalFormatTransparent`
   tests generation but not round-trip correctness. A runtime unmarshal test
   for `date-time`, `date`, and `ipv4` would close this gap.

3. **Specialized integer ref validator retention** — no existing test asserts
   that numeric constraints survive when an external integer ref materializes
   as `int64` or another non-default Go type. This is the most likely source
   of silent validator loss.

4. **oneOf-envelope with external `$defs` refs** — current envelope tests use
   inline schemas or same-file refs. Mixed external `$defs` refs in envelope
   branches have no coverage.

5. **Named object shared-symbol deduplication** — no test currently asserts
   that a named object ref repeated in multiple use-sites produces exactly one
   declaration (not N per use-site).

6. **Legacy `definitions` parity** — `refOld` confirms basic generation but
   does not compare semantics against a `$defs` equivalent. A parity assertion
   would catch regressions if `definitions` support is ever inadvertently broken.
