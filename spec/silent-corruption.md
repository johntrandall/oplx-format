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

### Hand-injected wrong-name `<gantt-view>` toggle elements are silently stripped

Hand-writing `<critical-paths/>` (plural) instead of `<critical-path/>` (singular) inside `<gantt-view>` does NOT activate the View → Gantt → Critical Paths toggle. The wrong-name element is silently dropped by OmniPlan on first save with no warning. Same pattern for any naively-guessed sibling toggle name — the menu-name-to-element-name mapping is irregular (see `toc-xml.md` § "`<gantt-view>` toggle children" for the verified 5-element table). Methodology lesson: don't extrapolate element names from menu names; verify each via menu-toggle → save → diff. Verified 2026-05-27 by injecting `<critical-paths/>` into a fixture; OmniPlan stripped it on save and emitted `<critical-path/>` only after the menu was actually toggled.

### `<user-data>` with non-`<string>` value tags rejects the entire file

OmniPlan 4.10.2 accepts ONLY `<string>` as the value type for `<user-data>` keys. Injecting `<number>`, `<date>`, or `<boolean>` causes OmniPlan to silently reject the entire file at open time — `count documents` returns 0, no error dialog, the only signal is the AppleScript `-10000` error if you `tell application "OmniPlan" to open POSIX file`. Verified 2026-05-27 by individually injecting each type into `<user-data>` at the canonical end-of-task position with correct TOC key registration; all three non-string variants reproduced the rejection.

### `<user-data>` placed before `<note>` in a task is silently stripped on save

`<user-data>` must come AT THE END of a task element (after `<note>`, after `<assignment>`, etc.). Injecting it earlier — even just before `<note>` — causes OmniPlan to silently drop the entire block on save. The file opens cleanly; the user-data values are simply missing in the post-save XML. Verified 2026-05-27.

### `<user-data>` without TOC key registration is silently stripped on save

`<user-data>` keys in a task must be registered in `__TOC.xml/<project>/<task-user-data-keys>` for the values to survive a save. Without registration, OmniPlan silently strips the block on save even when position is correct and value type is `<string>`. The spec's `actual-xml.md` previously documented this as "to be visible in OmniPlan's UI" — empirically, the keys aren't merely hidden; they're DELETED from the saved doc. Verified 2026-05-27.

### `<user-data>` on the root task `t-1` causes file-level rejection

Hand-writing `<user-data>` inside the `<task id="t-1">` element (the implicit root) causes OmniPlan to reject the file with silent `-10000` at open. `<user-data>` is only valid on non-root tasks. Verified 2026-05-27.

### Hand-guessed styled-note `<style>` grammar is silently flattened

Hand-writing `<run><style><attribute name="font-traits"><value key="bold">1</value></attribute></style><lit>TEXT</lit></run>` (and other guessed `attribute name=...` variants) does NOT produce bold/italic in the saved note. OmniPlan parses the doc cleanly, then on first save **collapses all separately-styled `<run>` siblings into a single plain `<run>`** with the lit-text concatenated and the bold/italic attributes silently dropped. No errors, no warnings, no leftover indication of the failure. Verified 2026-05-27 with a 5-run injection containing alternating bold/italic/plain runs: post-save was one `<run>` with one `<lit>plain BOLD normal ITALIC end</lit>`. The real `<style>` block grammar for font traits is still Open — needs GUI-driven content as a reference. Methodology lesson: when guessing wire-format grammar from .sdef field names or external conventions, ALWAYS check that the post-save XML preserves the structure you wrote — OmniPlan accepts then collapses without complaint.

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
