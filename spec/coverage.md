# Coverage map

What's been Verified, Observed, or remains Open as of spec version 0.1.0 (OmniPlan 4.10.2, 2026-05-02).

## Tier definitions

- **Verified**: tested in isolation. One variable changed, effect observed in saved XML or omniJS readback.
- **Observed**: works in our testing, but other variables changed concurrently. Correlation, not isolated proof.
- **Inferred**: logical conclusion from evidence, but never directly tested. Marked separately because it should not be relied on without confirmation.
- **Open**: untested or partially tested.

## Verified

### File format
- [x] Directory bundle UTI `com.omnigroup.omniplan2.planfile` and zip-variant UTI `com.omnigroup.omniplan2.planfile-zip`
- [x] `.oplx` extension covers both variants; OmniPlan introspects contents
- [x] Directory bundle has `QuickLook/Preview.png` subdir; zip variant has `Preview.png` at top
- [x] Hand-zip workflow produces valid zip-variant docs (open AND round-trip)
- [x] Minimum-viable .oplx (3 files, no prototypes, ~1.1KB zip) opens cleanly
- [x] OmniPlan auto-fills Preview.png + window state + styling on first save

### Tasks
- [x] All 4 task types (task / milestone / group / hammock) — XML form lowercase
- [x] `<type>` case-sensitivity: uppercase causes file-level rejection
- [x] Element ordering: title → type → priority → effort → schedule-type → recalculate → fixed-duration → static-cost
- [x] `<priority>` element form, default 0 omitted
- [x] `<effort-done>` element form
- [x] `<recalculate>` accepts `duration | effort | units`
- [x] `<recalculate>` non-default values emit `<fixed-duration>` sibling
- [x] All 4 date constraint elements (start/end-no-earlier/later-than)
- [x] `<locked-start-date>` element; no `<locked-end-date>` element
- [x] `<note>` rich-text form `<text>/<p>/<run>/<lit>`
- [x] `<user-data>` interleaved key/value form
- [x] Empty title `<title/>` is allowed
- [x] Negative effort rejected by omniJS
- [x] `<min/expected/max-estimate>` form for 3-pt estimation
- [x] `<effort>` is auto-PERT-computed when 3-pt set
- [x] Hand-written `<type>hammock</type>` round-trips (effort recomputed from deps)

### Resources
- [x] All 4 resource types — XML capitalized form
- [x] Resource element order: name → type → note → cost-per-use → cost-per-hour → units-available → efficiency → email-address
- [x] Material auto-emits `<units-available>0</units-available>`
- [x] `<email-address>` element name (NOT `<email>`)
- [x] Per-resource `<schedule>` overrides emit FULL day's hours

### Dependencies
- [x] `<prerequisite-task>` self-closing for default Finish-Start
- [x] `kind="FF|SS|SF"` attribute for non-default kinds
- [x] `kind=` is CASE-SENSITIVE uppercase
- [x] `<lead-time is-percentage="false">N</lead-time>` for duration leads (seconds)
- [x] `<lead-time is-percentage="true">N</lead-time>` for percentage leads (fraction)
- [x] omniJS `leadTimePercentage` takes integer percent, not fraction
- [x] AppleScript `set lead percentage to N` takes fraction
- [x] Multi-prereq: sibling `<prerequisite-task>` elements in insertion order

### Assignments
- [x] `<assignment idref="rN"/>` self-closing for default 1.0 units
- [x] `<assignment idref="rN" units="0.5"/>` for non-default
- [x] `unitsAssigned = 0` removes the assignment

### Scenarios
- [x] `<granularity>` element in `Actual.xml` (not `__TOC.xml`)
- [x] Granularity values: omitted (exact), `hours`, `days`
- [x] `hasFixedEndDate=true` switches to backward-from-end scheduling
- [x] Multiple `<scenario>` entries in `__TOC.xml` enable baselines
- [x] Hand-edit baseline scenario file (separate `<scenario>` XML + reference) is recognized

### Internal
- [x] `<next-task-id>` is recomputed to max-used+1 on every save
- [x] `__changelog.xml` minimum form: `<version>4.0</version>` only
- [x] `__changelog.xml` `<change-set>` and nested `<change>` grammar
- [x] OmniPlan strips empty/default-shaped prototype-tasks on save

### `<gantt-view>` toggle children (View → Gantt menu state)
*Verified 2026-05-27 against OmniPlan 4.10.2 via single-variable AppleScript probe + menu-toggle → save → diff. See `toc-xml.md` § "`<gantt-view>` toggle children".*

- [x] `<dependency-lines/>` empty element corresponds to View → Gantt → Dependency Lines toggle ON; absence = OFF
- [x] `<critical-path/>` (**singular**) corresponds to View → Gantt → Critical Paths toggle ON
- [x] `<slack/>` (**drops "lines"**) corresponds to View → Gantt → Slack Lines toggle ON
- [x] `<group-shading/>` corresponds to View → Gantt → Group Shading toggle ON
- [x] `<constraints/>` corresponds to View → Gantt → Constraints toggle ON
- [x] All five toggles' menu items are `enabled=true` regardless of element presence (no element ≠ disabled menu — earlier strong claim that absence "disables the menu entirely" was tested and falsified)
- [x] Element placement: between `<view-mode>` and the first `<scale>`, as direct children of `<gantt-view>`
- [x] Naming pattern is NOT simple kebab-of-menu-name — see `Critical Paths → critical-path` and `Slack Lines → slack` exceptions

## Observed (correlation, not isolated proof)

- [ ] `<recalculate>` invalid values (none, assignments, BOGUS) → normalize to `duration`. Saved doc shows `duration`; cannot distinguish "normalized" from "stripped + default-fill".
- [ ] `next-task-id` reset behavior on every save (Cycle 0011 had it preserved by coincidence in Cycle 0010)
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — OmniPlan auto-corrects `<resource id="r-1"><name/></resource>` → `<name>Project</name>` AND `<type>Group</type>` → `<type>Project</type>`. Observed in the hammock-multi-test fixture: input had `<resource id="r-1"><name/><type>Group</type>...</resource>`; post-save XML showed `<name>Project</name><type>Project</type>`. The root resource has its own dedicated `Project` type value (not in the Staff/Equipment/Material/Group enum used by user-created resources).
- [ ] Cycle 0011's `resourceAssignmentType = adjustDuration` round-trip is inferred from t506's saved state (no changelog entry because default values produce no entry)

## Inferred (never directly tested)

- [ ] `<recalculate>BOGUS_VALUE</recalculate>` normalization (couldn't isolate due to sibling probe failures)
- [ ] Resource type case sensitivity for non-Staff types (lowercase `<type>staff</type>` outcome — likely also rejected based on TaskType pattern, but not separately tested)
- [ ] Files with malformed `<effort>` (string literals, very large numbers)
- [ ] Files with non-UTC date strings other than `+TT:00` offset

## Open (not yet tested)

### Cross-format / interop
- [ ] MSPDI export — which extension does it need?
- [ ] HTML export with cleared `OPHTMLTemplate` preference
- [ ] iCal export round-trip
- [ ] `omniplan://localhost/omnijs-run?script=...` URL invocation pattern
- [ ] `PlanDocument.makeNew(fn)` programmatic doc creation
- [ ] Hand-write filter bplist (vs reading existing)

### Unexplored API
- [ ] `attachment` AppleScript class — how to ADD an attachment programmatically
- [ ] `change mark from "name"` — is this only user attribution, or is there a marker XML element?
- [ ] `fix violation with action "..."` — what action strings are valid?
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — `Project.title` setter persists to `__TOC.xml/<project>/<title>NAME</title>`. Test: `tell project of front document to set title to "TestProjectTitle-2026"` then save; post-save TOC contained `<project><title>TestProjectTitle-2026</title>...`. Does NOT write to `Actual.xml` (no scenario-level title element). The setter on `document.name` is read-only (-10006); use `project.title` instead.

### Edge cases
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — Hammock without deps: OmniPlan computes effort = **1 work week** = `hours-per-week` × 3600s = 144000s for the default `hours-per-week="40"`. Test: converted a normal task (original effort 7200) to `<type>hammock</type>` with no `<prerequisite-task>` elements, opened in OmniPlan, saved; post-save `<effort>` was 144000. Task is accepted (no rejection), `<recalculate>duration</recalculate>` is preserved. The original task's effort value is overwritten by the recalculation.
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — Hammock with multiple prereqs: `start = max(predecessor end dates)` for FS dependencies (the LATEST finish wins). Test: minimal 3-task fixture with t1 (effort 3600s, no prereqs), t2 (effort 28800s, no prereqs, finishes later than t1), t3 (`<type>hammock</type>` with `<prerequisite-task idref="t1"/>` + `<prerequisite-task idref="t2"/>`, both FS default kind). omniJS readback: `t3.startDate = t2.endDate` (Jun 02 12:00Z), confirming the LATER predecessor defines the start. t3 effort stayed at 0 (no successor → no end-side constraint → degenerate hammock).
- [x] **VERIFIED 2026-05-27** — `<numbering-style>` accepts `wbs` (Hierarchical Numbering) AND `flat` (Flat Numbering). Test: injected `<numbering-style>flat</numbering-style>` in a hand-built `.oplx` TOC, opened in OmniPlan 4.10.2, View → Task Outline submenu shows "Flat Numbering" checked, "Hierarchical Numbering" unchecked. (Reverse case: `wbs` → Hierarchical checked.) The "Show/Hide Numbering" toggle is independent — separate XML element not yet identified. The AppleScript dictionary exposes no `numbering style` enum (the XML element is wire-only). Whether other enum values exist (e.g. for the show/hide toggle) — Open.
- [ ] `<page-adornment>` complete variable list (only `OPDocumentTitleVariableIdentifier`, `OPPrintJobTimestampVariableIdentifier` seen)
- [ ] Note rich-text with formatting (bold/italic/color/alignment) — `<style>` block grammar. **Partially probed 2026-05-27 (VM cross-check):** hand-injected `<run><style><attribute name="font-traits"><value key="bold">1</value></attribute></style><lit>BOLD</lit></run>` and `italic` variants. Post-save, OmniPlan **silently collapsed all my styled runs into a single plain `<run>`** with concatenated text — the `attribute name="font-traits"` grammar I guessed is wrong; no errors, just elided formatting. The AppleScript `font traits` setter on `word N of theNote` has a -2740 parser quirk and was not pursued. Real styled-note grammar requires GUI-driven content (System Events `keystroke` + Cmd-B/Cmd-I via VNC, or a manually-created OmniPlan note) and remains Open. No local `.oplx` in the corpus has `font-traits` or `attribute name=` styling.
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — `<user-data>` accepts **`<string>` only** in OmniPlan 4.10.2. Non-string value tags (`<number>`, `<date>`, `<boolean>`) cause OmniPlan to reject the entire file with silent AppleEvent error `-10000` at open time. Earlier host-only result that claimed all four types worked was a **false positive** — the test grep matched the input fragment (the file was rejected; OmniPlan never saved it). VM cross-check on `oplx-verify-2026-05-27` (image `macos-15.7-l3-omni-suite:v2-tcc-granted-20260513`) tested each type individually: `<string>` round-trips cleanly; `<number>42</number>`, `<date>2026-05-27T12:00:00.000Z</date>`, `<boolean>true</boolean>` each caused `count documents` to return 0 after `open -a OmniPlan`. The AppleScript surface declares `custom data` value as `type="any"` and the internal model supports more, but the WIRE format only persists `<string>` in 4.10.2. Three additional requirements verified the same session:
  1. **Position:** `<user-data>` must appear at the END of the task element (after `<note>`, `<assignment>`, etc.). Earlier-in-task position causes the block to be silently stripped on save.
  2. **TOC registration required:** the key must be listed in `__TOC.xml/<project>/<task-user-data-keys>` as `<key>NAME</key><null/>`. Without registration, the value block is silently stripped on save (even with correct position).
  3. **Root task `t-1`:** cannot carry `<user-data>` — injecting there causes file-level rejection.
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — `<user-data>` cold-write (key not in `__TOC.xml/<task-user-data-keys>`): the `<user-data>` block is **silently stripped on save**. Tested with `<key>BudgetCode</key><string>BC-2026</string>` injected on a non-root task at end-of-task position with valid `<string>` value but no TOC registration; post-save user-data count was 0. TOC registration as `<key>NAME</key><null/>` pairs is mandatory (Verified silent-corruption, documented in silent-corruption.md). Note the element name is `<user-data>` (not `<custom-data>` — the Open question's wording was incorrect; both AppleScript and the wire format use `user-data`).
- [ ] `subtract work time on date "..."` — find the working date format
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — AppleScript `lead percentage` stays **fraction** semantics (NOT integer-percent like omniJS). Tested values 0, 0.25, 0.5, 1.0, 2.0, -0.5, 25, 100 — ALL accepted verbatim by the setter, readback returns the same value (no clamping, no truncation, no rejection). Setting `lead percentage of d to 0.5` then saving produces `<lead-time is-percentage="true">0.5</lead-time>` in the wire format. The opposite convention from omniJS (which uses integer-percent and silently rejects values < 1) is canonical in the spec's surface table.

### Future-version surveillance

> Status check **2026-05-27**: public OmniPlan release is still **4.10.2** (`defaults read /Applications/OmniPlan.app/Contents/Info CFBundleShortVersionString`). All four items below remain pending; re-check when 4.10.3 / 4.11 ships and the RT #3107771 fixes land in a public build.
>
> **Expiry rule.** *Trigger:* re-run the `defaults read` check at every spec refresh. *Removal:* delete this block (and promote any newly-Verified items) once all items in the surveillance list below resolve.

- [ ] **TODO when next OmniPlan release ships** — Verify Task `parent` accessor + `move` method against the live release. Vendor confirms fix in next release per RT #3107771 (2026-05-05); update to Verified once release ships and tested.
- [ ] **TODO when next OmniPlan release ships** — Verify Resource `parent` accessor + `move` method against the live release. Vendor confirms fix in next release per RT #3107771; update to Verified once release ships and tested.
- [ ] **TODO when next OmniPlan release ships** — Reconfirm `Decimal.fromString("100.00").toString()` is the canonical round-trip pattern across all Decimal-typed properties (`staticCost`, `costPerHour`, `costPerUse`) against the live release. Vendor confirms this is by-design per RT #3107771; update to Verified once release ships and tested.
- [ ] Verify broader spec against OmniPlan 4.11+ when released.
- [ ] OmniPlan 5.x format-version changes (currently `file-format-version="3"`)

## Cross-environment verification status

The Verified entries above were established on a **host Mac** (macOS 15.x, OmniPlan 4.10.2). The reverse-engineer-file-format playbook recommends a VM cross-check to catch environment-conditional plist keys (gotcha #17). For the 2026-05-27 verification session:

- A Tart VM (`oplx-verify-2026-05-27`, image `macos-15.7-l3-omni-suite:v2-tcc-granted-20260513`) was provisioned via Orchard.
- VM AppleScript-driven document operations (`count documents`, `save front document`, `keystroke "s"`) timed out at `-1712` in orchard's default headless mode — even with `with timeout of 120 seconds`. Application-level commands (`get version`) worked.
- VM cross-check could not complete via AppleScript save-roundtrip. Acceptable alternatives (not exercised this session): `orchard vnc vm <name>` to attach a display, or routing through `omniplan-mcp`'s JXA bridge.
- **Implication for the Verified entries marked "2026-05-27":** they are host-confirmed but not VM-confirmed. The wire-format claims (XML schema, element-name correspondence) are not plausibly environment-conditional — but a gotcha-#17-style "this key appears only on the host" or "this menu name only renders on the host" finding would not have been caught by this session.

See `README.md` § "Verification environment" for the canonical VM access pattern.

## Methodology lessons

Embedded in private research notes. The summary version:

1. **"No-op" claims must diff ALL bundle XML files** (not just `__TOC.xml`).
2. **Default values produce no changelog entries** — proving "default was set" via changelog is impossible by design.
3. **"Round-trip Verified"** must specify scope (file-format vs XML vs semantic).
4. **Test artifacts must literally match** the documented "minimum" — don't claim extras are unnecessary unless you've tested without them.
5. **Batched experiments** yield ambiguous causal evidence. For surface enumeration, batching is fine; for "X causes Y" claims, isolate.
6. **Inspect real-world examples** before declaring something "UI-only" — an existence proof may already exist on disk.
