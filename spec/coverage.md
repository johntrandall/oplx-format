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

## Observed (correlation, not isolated proof)

- [ ] `<recalculate>` invalid values (none, assignments, BOGUS) → normalize to `duration`. Saved doc shows `duration`; cannot distinguish "normalized" from "stripped + default-fill".
- [ ] `next-task-id` reset behavior on every save (Cycle 0011 had it preserved by coincidence in Cycle 0010)
- [ ] OmniPlan auto-corrects `<resource id="r-1"><name/></resource>` → `<name>Project</name>`
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
- [ ] `Project.title` setter — where does it persist?

### Edge cases
- [ ] Hammock without deps — what effort does OmniPlan compute?
- [ ] Hammock with multiple prereqs — which one defines the start?
- [ ] Other `<numbering-style>` enum values besides `wbs`. UI evidence (reference manual `gantt-view.md`, `menu-commands-and-keyboard-shortcuts.md`) shows three user-facing states: **Show/Hide Numbering** toggle, **Flat Numbering** (1, 2, 3, …), **Hierarchical Numbering** (1, 1.1, 1.1.1, …). The Verified `wbs` value corresponds to Hierarchical; a sibling value for Flat is implied but its literal XML token is untested. The AppleScript dictionary exposes no `numbering style` enum (the element is XML-only). Hypothesis to test: `<numbering-style>flat</numbering-style>` and/or a separate `<show-numbering>` boolean.
- [ ] `<page-adornment>` complete variable list (only `OPDocumentTitleVariableIdentifier`, `OPPrintJobTimestampVariableIdentifier` seen)
- [ ] Note rich-text with formatting (bold/italic/color/alignment) — `<style>` block grammar
- [ ] `<user-data>` value types beyond `<string>` — does `<number>` / `<date>` work? AppleScript surface declares `custom data` and the `value` field of `<custom data entry>` (`class OPKeyValuePair`) as type `any`, suggesting non-string values are supported at the model layer. Wire form not yet observed for non-`<string>` children; needs an isolated write/save round-trip to verify.
- [ ] `<custom-data>` cold-write (key not in `__TOC.xml/<task-user-data-keys>`)
- [ ] `subtract work time on date "..."` — find the working date format
- [ ] AppleScript `lead percentage` boundaries (does it match omniJS integer or stay fraction with truncation?)

### Future-version surveillance

> Status check **2026-05-27**: public OmniPlan release is still **4.10.2** (`defaults read /Applications/OmniPlan.app/Contents/Info CFBundleShortVersionString`). All four items below remain pending; re-check when 4.10.3 / 4.11 ships and the RT #3107771 fixes land in a public build.
>
> **Expiry rule.** *Trigger:* re-run the `defaults read` check at every spec refresh. *Removal:* delete this block (and promote any newly-Verified items) once all items in the surveillance list below resolve.

- [ ] **TODO when next OmniPlan release ships** — Verify Task `parent` accessor + `move` method against the live release. Vendor confirms fix in next release per RT #3107771 (2026-05-05); update to Verified once release ships and tested.
- [ ] **TODO when next OmniPlan release ships** — Verify Resource `parent` accessor + `move` method against the live release. Vendor confirms fix in next release per RT #3107771; update to Verified once release ships and tested.
- [ ] **TODO when next OmniPlan release ships** — Reconfirm `Decimal.fromString("100.00").toString()` is the canonical round-trip pattern across all Decimal-typed properties (`staticCost`, `costPerHour`, `costPerUse`) against the live release. Vendor confirms this is by-design per RT #3107771; update to Verified once release ships and tested.
- [ ] Verify broader spec against OmniPlan 4.11+ when released.
- [ ] OmniPlan 5.x format-version changes (currently `file-format-version="3"`)

## Methodology lessons

Embedded in private research notes. The summary version:

1. **"No-op" claims must diff ALL bundle XML files** (not just `__TOC.xml`).
2. **Default values produce no changelog entries** — proving "default was set" via changelog is impossible by design.
3. **"Round-trip Verified"** must specify scope (file-format vs XML vs semantic).
4. **Test artifacts must literally match** the documented "minimum" — don't claim extras are unnecessary unless you've tested without them.
5. **Batched experiments** yield ambiguous causal evidence. For surface enumeration, batching is fine; for "X causes Y" claims, isolate.
6. **Inspect real-world examples** before declaring something "UI-only" — an existence proof may already exist on disk.
