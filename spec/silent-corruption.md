# Silent corruption modes

Patterns where OmniPlan accepts input without complaint but produces wrong output. These are the most important things to avoid in generators or hand-edited files.

## Severity tiers

| Tier | Symptom | Detectability |
|---|---|---|
| **CRITICAL** | Document silently fails to open (no error dialog) | Caller can detect (file → New Project chooser instead of doc) |
| **HIGH** | Whole tasks/elements stripped from saved XML | Diffable post-save |
| **MEDIUM** | Value normalized to default (semantic loss) | Diffable post-save if you know the original |
| **LOW** | Cosmetic/metadata loss | Generally tolerable |

## CRITICAL: file-level rejection

### Uppercase `<type>` for tasks

```xml
<type>MILESTONE</type>     <!-- whole doc fails to open -->
<type>Milestone</type>     <!-- whole doc fails to open -->
```

OmniPlan silently refuses to load the file. No error dialog; the front-of-app New Project chooser appears instead. Generators must always emit lowercase: `task | milestone | group | hammock`.

### Resource type case sensitivity (suspected)

The XML form uses **capitalized** resource types: `Staff | Equipment | Material | Group`. Whether lowercase causes file-level rejection or just attribute stripping is **untested**; either way, generators must capitalize.

## HIGH: element-level drops

### Orphaned tasks (not reachable from `<top-task>`)

```xml
<top-task idref="t-1"/>
<task id="t-1">
  <child-task idref="t1"/>
  <!-- no reference to t999 -->
</task>
<task id="t999">...</task>     <!-- silently dropped on save -->
```

Tasks not reachable from `t-1` via the `<child-task>` chain are silently removed. Generators must ensure every task is referenced as `<child-task>` of some ancestor reachable from `t-1`.

### `<assignment units="0"/>` removes the assignment

Setting `unitsAssigned = 0` via omniJS or AppleScript REMOVES the `<assignment>` element entirely. If you intend "zero allocation", that's not representable — instead, omit the assignment, or use a small positive value.

### omniJS `task.split(at, resumingAt)` silently fails

Returns `true` (success) but the split is not persisted to XML. Possibly an in-memory-only operation in 4.10.x.

### omniJS `dep.leadTimePercentage` < 1 silently rejected

```js
dep.leadTimePercentage = 0.25;   // readback: null, NO XML emitted
```

Values with absolute magnitude < 1 are silently rejected. Use integer-percent (`25` for 25%) for omniJS. AppleScript uses the fraction convention (`0.25`).

### omniJS `task.type = TaskType.hammock` silently rejected

Readback shows `TaskType.task`. The only programmatic path to a hammock task is XML hand-write. This holds even when the task has prereq + dependent connections.

### `DependencyKind.Invalid` silently no-op

Setting via omniJS keeps the prior value. Generators should never emit `Invalid`.

## MEDIUM: value normalization

### Lowercase `kind="ff"` → default Finish-Start

```xml
<prerequisite-task idref="t1" kind="ff"/>     <!-- ff stripped, becomes FS -->
```

`kind` attribute is CASE-SENSITIVE uppercase. Lowercase or invalid values are silently stripped, dropping the dependency to the default Finish-Start. Scheduling produces wrong results.

### `<recalculate>` invalid values → `duration`

```xml
<recalculate>none</recalculate>           <!-- → duration -->
<recalculate>assignments</recalculate>    <!-- → duration -->
<recalculate>BOGUS</recalculate>          <!-- → duration -->
```

Only `duration | effort | units` are valid. Anything else silently normalizes to `duration`.

### `manualEndDate` back-computed to `<locked-start-date>`

Setting `task.manualEndDate = D` does NOT produce `<locked-end-date>` (no such element exists). Instead, OmniPlan back-computes a `<locked-start-date>` value of `D - duration`. If the duration changes later, the locked-start-date does NOT update — the task may now end at a different date than the original `manualEndDate`.

### `<next-task-id>` recomputed to max-used+1

Hand-bumping `<next-task-id>` to a high value does NOT survive — OmniPlan recomputes it on every save. Generators must work with the actual current max ID.

### `scheduling granularity` value lives in `Actual.xml`

Cycle 0015 of the research repo wrongly claimed the AppleScript `set scheduling granularity` setter was a no-op because `__TOC.xml` was unchanged. The setter actually persists to `<granularity>` in `Actual.xml`. Methodology lesson: when checking if a setter is a no-op, diff ALL bundle XML files, not just one.

## LOW: cosmetic / metadata

### Custom-data key order is non-deterministic

`<user-data>` elements interleave `<key>NAME</key>` and `<string>VALUE</string>` pairs, but the order across saves is hash-iteration order, not alphabetical or insertion-order. Tools that diff `Actual.xml` between revisions should normalize user-data ordering before comparison.

### `Preview.png` quality varies

OmniPlan re-renders Preview.png on save with potentially different sizes (28KB → 177KB observed in one round-trip). The XML content is unaffected.

### `__changelog.xml` history grows

Each save appends a new `<change-set>`. Long-lived docs accumulate history. Generators producing fresh docs can emit an empty changelog (`<changelog><version>4.0</version></changelog>`).

### `<note>` plain-string round-trip is lossy through the XML form

Setting `task.note = "foo\nbar"` saves as:

```xml
<note>
  <text>
    <p><run><lit>foo</lit></run></p>
    <p><run><lit>bar</lit></run></p>
  </text>
</note>
```

A naive parser reading `<note>/<text>/<p>/<run>/<lit>` text and concatenating with `\n` will recover the original. Parsers that strip line breaks or that fail to handle multi-`<p>` notes will lose the structure.

## Recommendations for generators

1. **Validate against this list before producing XML.** A linter that catches all the patterns above is the highest-value tool to build alongside generation.

2. **Normalize ALL enum values** to their canonical XML form before emitting:
   - Task types: lowercase
   - Resource types: capitalized
   - Dependency kinds: uppercase
   - Recalculate: `duration` | `effort` | `units`

3. **Never emit `units="0"`** — omit the `<assignment>` instead.

4. **Always reference every task via `<child-task>`** from the root group `t-1`.

5. **Use ISO-8601 UTC with milliseconds** for all dates (`2026-06-01T13:00:00.000Z`).

6. **Convert lead-time percentage to fraction** when writing XML directly (e.g., `0.25` for 25%). For the omniJS API, use integer percent (`25`).

7. **Don't rely on `<next-task-id>`** for cross-save ID coordination — OmniPlan recomputes it. If you need stable external IDs, use `<user-data>` keys.

## Recommendations for parsers

1. **Diff `Actual.xml`, not `__changelog.xml`**, for semantic equivalence.

2. **Normalize `<user-data>` ordering** before comparing customizations between revisions.

3. **Watch for the encoding of constraint dates**: a single internal field `noEarlierThanConstraintDate` plus a `noEarlierThanConstrainedTaskEnd` flag (0/1) maps to either `<start-no-earlier-than>` or `<end-no-earlier-than>`. The XML elements look distinct; the internal model treats them as the same field.

4. **Don't trust default-shaped values** to round-trip through editing. If a value is at the OmniPlan default and nothing changes it, OmniPlan may strip it on save.
