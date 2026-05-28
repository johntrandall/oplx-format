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
- [ ] ~~`<next-task-id>` is recomputed to max-used+1 on every save~~ — **RETRACTED 2026-05-27 (VM cross-check)**. The prior claim was wrong. See Verified entry under "Internal" below for actual behavior.
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

- [x] **VERIFIED 2026-05-27 (VM cross-check)** — `<recalculate>` invalid values all normalize to `duration` (no file-level rejection). Tested 5 variants in one fixture: `none`, `assignments`, `BOGUS_VALUE`, `DURATION` (uppercase), empty `<recalculate></recalculate>` — all five became `<recalculate>duration</recalculate>` on save. File opened cleanly (count=1). Whether "normalized" vs "stripped + default-fill" is still indistinguishable from output alone — but the practical effect is identical: invalid input → `duration` output, no error.
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — `<next-task-id>` is **preserved verbatim from input**, NOT recomputed on every save (correcting prior wrong claim). Test 1: built a fixture with `<next-task-id>2</next-task-id>` and tasks `t1`/`t10`/`t99` (max-used = 99). Save with no edits → output `<next-task-id>2</next-task-id>` unchanged. Test 2: same fixture, opened, added one task via AppleScript `make new task` → new task got id `t2` (the value OmniPlan read from the counter, NOT max-used+1) and the counter advanced to `3`. **Generators should set this to `max(used-id) + 1` to avoid future collisions** but OmniPlan does not enforce that — it trusts the value verbatim and will produce `t2` even when `t99` already exists. Same pattern observed across 4 saved fixtures this session (recalc-test, hammock-multi, dates3, nextid). The 2019-era Cycle 0011 note "preserved by coincidence" was misreading; preservation is the normal behavior, not coincidence.
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — OmniPlan auto-corrects `<resource id="r-1"><name/></resource>` → `<name>Project</name>` AND `<type>Group</type>` → `<type>Project</type>`. Observed in the hammock-multi-test fixture: input had `<resource id="r-1"><name/><type>Group</type>...</resource>`; post-save XML showed `<name>Project</name><type>Project</type>`. The root resource has its own dedicated `Project` type value (not in the Staff/Equipment/Material/Group enum used by user-created resources).
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — omniJS `task.resourceAssignmentType` ↔ wire `<recalculate>` round-trips in both directions. Mapping: `ResourceAssignmentType.adjustDuration` ↔ `<recalculate>duration</recalculate>`, `.adjustEffort` ↔ `<recalculate>effort</recalculate>`, `.adjustAssignedUnits` ↔ `<recalculate>units</recalculate>` (note: the omniJS enum is `adjustAssignedUnits`, NOT `adjustUnits`). Tested both directions: (a) hand-built fixture with all three `<recalculate>` values on assigned tasks → all three preserved across save; (b) omniJS write `t.resourceAssignmentType = ResourceAssignmentType.adjustAssignedUnits` on a task that was `<recalculate>duration</recalculate>` → post-save XML shows `<recalculate>units</recalculate>`. The earlier Cycle 0011 "inferred from t506's saved state" claim is superseded. The omniJS object's `toString()` form is `[object ResourceAssignmentType: <enum-name>]` (not a `.name` property); `instanceof`-style detection via `Object.getPrototypeOf(rat).constructor.name === "ResourceAssignmentType"`.

## Inferred (never directly tested)

- [x] **VERIFIED 2026-05-27 (VM cross-check)** — `<recalculate>BOGUS_VALUE</recalculate>` normalizes to `duration` (see Verified row above). Same fixture also confirmed `none`, `assignments`, `DURATION` (uppercase), and empty `<recalculate></recalculate>` all normalize identically.
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — Resource type lowercase rejection: a fixture with `<resource><type>staff</type></resource>`, `<type>equipment</type>`, and `<type>material</type>` (all lowercase) was silently rejected at open time (`count documents` returned 0; the byte-identical file proves OmniPlan never saved). Same pattern as TaskType `<type>` rejection: case is enforced at the resource layer too. Use exactly `Staff`/`Equipment`/`Material`/`Group` (capitalized).
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — Malformed `<effort>` causes file-level rejection. Three variants tested in one fixture: `<effort>three</effort>` (string literal), `<effort>999999999999</effort>` (huge number, ~31708 years), `<effort>-3600</effort>` (negative). ALL THREE caused silent rejection (`count documents` returned 0). The integer-seconds form `<effort>3600</effort>` is the only valid input; anything else makes OmniPlan refuse to open the file.
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — Date format normalization: hand-written `<locked-start-date>` accepts three variants and normalizes ALL to canonical `YYYY-MM-DDThh:mm:ss.000Z` form. Tested: `2026-06-08T08:00:00.000-05:00` (EST -5 offset) → `2026-06-08T13:00:00.000Z` (UTC-converted), `2026-06-09T13:00:00Z` (no milliseconds) → `2026-06-09T13:00:00.000Z` (millis added), `2026-06-10T13:00:00.000Z` (canonical) → unchanged. **Critical caveat:** the date elements are silently stripped if element-order is wrong; see new Verified silent-corruption entry. The Verified element order for tasks with date pinning is `title → effort → locked-start-date → recalculate → static-cost` (Verified by `with-baseline.oplx` corpus + this VM test).

## Open (not yet tested)

### Cross-format / interop
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — MSPDI export is **broken in OmniPlan 4.10.2 regardless of extension**. Tested `as "MSPDI"` against three file extensions (`.mpp`, `.xml`, `.mspdi`) in one VM session — every combination returns AppleScript error `(6)` "could not be exported as '<filename-root>'" with NO file produced. The cross-test confirms the error is per-format, not per-extension: same fixture exported as `MPP` to those same three extensions ALL succeed and produce a valid MS-Office binary (magic `d0cf11e0`, ~232KB, mime `application/vnd.ms-office`). The error message references the filename root (not the format), which earlier led to speculation that the extension contract was wrong — that speculation is now falsified. Workaround for MS Project interop: use `as "MPP"` (produces real `.mpp` binary; extension does not affect content).
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — HTML export is **unconditionally broken** in OmniPlan 4.10.2: the "(null) template" dialog appears regardless of whether `OPHTMLTemplate` is unset, deleted, OR set to a valid bundled-template path. Tested all three preference states; the dialog text is "The HTML template you are trying to use is out of date. The customized HTML template at <PATH> is from an older version of OmniPlan and may no longer work correctly." with two buttons: "Use Default Template" and "Try Customized Template". **Clicking "Use Default Template" CRASHES OmniPlan** (Exception 0x00000002, OmniGroupCrashCatcher launches; AppleScript returns `-609`; document is closed). **Clicking "Try Customized Template" returns a 0-byte file** (no crash, AppleScript returns success — the export completes but writes nothing because the template-loading path silently produces empty output). The bundled templates exist at `/Applications/OmniPlan.app/Contents/Resources/<locale>.lproj/HTMLTemplates/{Bright,Zebra,Printer Friendly,SeriousBusiness}/` — but OmniPlan considers them "out of date" against its internal version check, so even pointing OPHTMLTemplate at them does not prevent the dialog. **Workaround: use `as "CSV"` instead** (CSV export is working). The applescript.md Broken entry's claim that "selecting any option crashes" is half wrong — only "Use Default Template" crashes; "Try Customized Template" silently fails with empty output.
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — iCal export works cleanly. `tell application "OmniPlan" to export front document to POSIX file "/tmp/out.ics" as "ICAL"` produces a valid RFC-5545 VCALENDAR file (~606 bytes for a 3-task fixture). One `VEVENT` per task with `DTSTART`/`DTEND`/`DTSTAMP` in UTC, `SUMMARY` from task title, `UID` of the form `<scenario-id>.<task-numeric-id>` (e.g. `hammock-multi-test.1`). Hammock tasks with degenerate duration (start=end) emit as zero-duration events. The exporter writes the file extension you supply (use `.ics`).
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — `omniplan://localhost/omnijs-run?script=...` URL invocation works (fire-and-forget). Test: `open "omniplan://localhost/omnijs-run?script=actual.rootTask.title"` from shell → no error, OmniPlan executes the JS in the active document. The URL is fire-and-forget — no return-value capture path (`open` returns shell exit code, not the JS expression result). Use this for "do something then close" workflows where return value is irrelevant; use `evaluate javascript` via AppleScript for return values (already documented in `applescript.md`).
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — `make new document` via AppleScript: returns AppleScript error `-10000 "Instantiating a document with properties {} resulted in no windows"` BUT post-error inspection shows `count documents = 1` and `name of every document = "Untitled"`. So the document IS created in memory; the failure is window-opening, not document-creation. Workaround: ignore the -10000, then immediately operate on `front document` (works). To save the new doc to disk use `save front document in POSIX file "/path/to/file.oplx"`. The `omniplan-mcp` JXA bridge documents this exact workaround; the gotcha is also already in `applescript.md` under Broken / partially-working commands.
- [ ] Hand-write filter bplist (vs reading existing)

### Unexplored API
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — `attachment` cannot be added programmatically. The AppleScript `attachment` class declares `file` as `access="r"` (read-only) — no documented writable setter exists. omniJS surface has NO `Attachment` class at all (not in `OP-API.md`'s 80+ class list). `make new attachment with properties {file:...}` errors with `-1700 "Can't make {file:...} into type properties of attachment"`. Reading attachments works (`count attachments of t1` returns 0 cleanly). **The only programmatic path to attach a file is via XML emission or the GUI** — neither scripting surface supports it in 4.10.2. Likely a documented-but-incomplete API surface.
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — `change mark from "name"` is user-attribution for changelog entries (NOT a marker XML element). The string argument feeds the `user` attribute of subsequent `<change-set>` elements in `__changelog.xml`: `<change-set user="NAME" date="..." order="N">...</change-set>`. **In the MDM-managed Tart VM**, the value is dominated by the system identity `"Managed via Tart"` regardless of the `change mark from` call — verified by inspecting the changelog after multiple AppleScript-driven changes; all entries showed `user="Managed via Tart"` even after `change mark from "TestUser-Claude"`. The `change mark from` setter likely DOES work on a non-managed Mac (the saved `Modifier` field would also need to differ); needs a non-MDM probe to confirm full effect. The MDM-managed-identity collision is the same root cause as `reverse-engineer-file-format` gotcha #17.
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — `fix <violation> with action "..."` action strings enumerated for three common violation types. The valid strings are exposed via the `actions` property on each violation (`get actions of first violation of front document`). Three violation types tested:

  | Violation type | Description | Valid `actions` strings |
  |---|---|---|
  | Scheduling | Hammock task doesn't have start and end prerequisites | `setToTask` |
  | Scheduling | Manual date set earlier than no-earlier-than constraint date | `automatic`, `moveStart`, `removeConstraint` |
  | Constraints | This task is involved in a dependency loop | `remove`, `removeOther-<N>` (parametric — the suffix indexes which "other" task in the loop to remove) |

  Each `fix` call with a valid action cleared its violation (count decremented). **Invalid action strings are SILENTLY IGNORED** — `fix first violation of front document with action "bogus-action-name"` returns no error but does nothing (silent-corruption mode worth recording). The vocabulary observed (`automatic`, `setToTask`, `moveStart`, `removeConstraint`, `remove`, `removeOther-<N>`) is non-exhaustive — other violation types not exercised here may expose additional action strings; always enumerate via `actions of <violation>` before calling `fix`.
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — `Project.title` setter persists to `__TOC.xml/<project>/<title>NAME</title>`. Test: `tell project of front document to set title to "TestProjectTitle-2026"` then save; post-save TOC contained `<project><title>TestProjectTitle-2026</title>...`. Does NOT write to `Actual.xml` (no scenario-level title element). The setter on `document.name` is read-only (-10006); use `project.title` instead.

### Edge cases
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — Hammock without deps: OmniPlan computes effort = **1 work week** = `hours-per-week` × 3600s = 144000s for the default `hours-per-week="40"`. Test: converted a normal task (original effort 7200) to `<type>hammock</type>` with no `<prerequisite-task>` elements, opened in OmniPlan, saved; post-save `<effort>` was 144000. Task is accepted (no rejection), `<recalculate>duration</recalculate>` is preserved. The original task's effort value is overwritten by the recalculation.
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — Hammock with multiple prereqs: `start = max(predecessor end dates)` for FS dependencies (the LATEST finish wins). Test: minimal 3-task fixture with t1 (effort 3600s, no prereqs), t2 (effort 28800s, no prereqs, finishes later than t1), t3 (`<type>hammock</type>` with `<prerequisite-task idref="t1"/>` + `<prerequisite-task idref="t2"/>`, both FS default kind). omniJS readback: `t3.startDate = t2.endDate` (Jun 02 12:00Z), confirming the LATER predecessor defines the start. t3 effort stayed at 0 (no successor → no end-side constraint → degenerate hammock).
- [x] **VERIFIED 2026-05-27** — `<numbering-style>` accepts `wbs` (Hierarchical Numbering) AND `flat` (Flat Numbering). Test: injected `<numbering-style>flat</numbering-style>` in a hand-built `.oplx` TOC, opened in OmniPlan 4.10.2, View → Task Outline submenu shows "Flat Numbering" checked, "Hierarchical Numbering" unchecked. (Reverse case: `wbs` → Hierarchical checked.) The "Show/Hide Numbering" toggle is independent — separate XML element not yet identified. The AppleScript dictionary exposes no `numbering style` enum (the XML element is wire-only). Whether other enum values exist (e.g. for the show/hide toggle) — Open.
- [x] **VERIFIED 2026-05-27** — `<page-adornment>` variables observed across local corpus (funding-pipeline.oplx + with-baseline.oplx + Gantt Timeline BAK + spacessync-launch.oplx): `OPDocumentTitleVariableIdentifier`, `OPDocumentFilenameVariableIdentifier`, `OPPageNumberVariableIdentifier`, `OPPrintJobTimestampVariableIdentifier`. Likely not exhaustive — additional variables may exist for `pageCount`, custom data, or scenario name; not observed yet. Search pattern: `grep -oE "OP[A-Z][a-zA-Z]+VariableIdentifier" __TOC.xml`.
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — Note rich-text `<style>` grammar enumerated. The correct shape is `<run><style><value key="KEY">VALUE</value>...</style><lit>TEXT</lit></run>` — NOT the `<attribute name="KEY"><value key="..."></value></attribute>` form the spawning session tried (which silently strips). Hand-built fixture with 10 candidate keys, opened + saved + diffed against the corpus pattern from `examples/with-baseline.oplx`. Canonical wire-format keys (each Verified to round-trip):

  | Key | Type | Verified values | Notes |
  |---|---|---|---|
  | `paragraph-alignment` | enum | `left`, `center`, `right` | Pre-existing in `with-baseline.oplx` |
  | `font-family` | string | `Helvetica` | PostScript family name |
  | `font-weight` | integer | `9` (bold), `5` is default → stripped | CSS-style weight; non-default values preserved |
  | `font-italic` | boolean | `yes` | Verified via input `Helvetica-Oblique` → output `yes` |
  | `font-size` | integer | `18` | Point size |

  **Input shortcuts that normalize on save:** `<value key="font-name">Helvetica-Bold</value>` → splits to `font-family=Helvetica` + `font-weight=9`. `Helvetica-Oblique` → `font-family=Helvetica` + `font-italic=yes`. So `font-name` is write-only (input grammar accepted, output grammar uses the split form).

  **Silently stripped invalid keys (tested):** `font` (use `font-name`), `bold`, `italic`, `font-style`, `font-color`, `foreground-color`, `text-color`, `color`, `underline`, `size`. The `color` key in particular is NOT yet identified — common candidates all stripped; color grammar remains a sub-open question.

  **Multiple `<value>` children allowed:** `<style><value key="font-weight">9</value><value key="paragraph-alignment">center</value></style>` round-trips both keys cleanly.

  **No scripting-surface API exists:** omniJS `task.note` is a plain String (no `.style`); AppleScript `word N of note of task` errors `-1700` ("can't make into type specifier") — the .sdef-declared rich-text element accessors are non-functional. XML hand-write is the only programmatic path; GUI keystroke (Cmd-B/I via VNC) is the only interactive path.
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — `<user-data>` accepts **`<string>` only** in OmniPlan 4.10.2. Non-string value tags (`<number>`, `<date>`, `<boolean>`) cause OmniPlan to reject the entire file with silent AppleEvent error `-10000` at open time. Earlier host-only result that claimed all four types worked was a **false positive** — the test grep matched the input fragment (the file was rejected; OmniPlan never saved it). VM cross-check on `oplx-verify-2026-05-27` (image `macos-15.7-l3-omni-suite:v2-tcc-granted-20260513`) tested each type individually: `<string>` round-trips cleanly; `<number>42</number>`, `<date>2026-05-27T12:00:00.000Z</date>`, `<boolean>true</boolean>` each caused `count documents` to return 0 after `open -a OmniPlan`. The AppleScript surface declares `custom data` value as `type="any"` and the internal model supports more, but the WIRE format only persists `<string>` in 4.10.2. Three additional requirements verified the same session:
  1. **Position:** `<user-data>` must appear at the END of the task element (after `<note>`, `<assignment>`, etc.). Earlier-in-task position causes the block to be silently stripped on save.
  2. **TOC registration required:** the key must be listed in `__TOC.xml/<project>/<task-user-data-keys>` as `<key>NAME</key><null/>`. Without registration, the value block is silently stripped on save (even with correct position).
  3. **Root task `t-1`:** cannot carry `<user-data>` — injecting there causes file-level rejection.
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — `<user-data>` cold-write (key not in `__TOC.xml/<task-user-data-keys>`): the `<user-data>` block is **silently stripped on save**. Tested with `<key>BudgetCode</key><string>BC-2026</string>` injected on a non-root task at end-of-task position with valid `<string>` value but no TOC registration; post-save user-data count was 0. TOC registration as `<key>NAME</key><null/>` pairs is mandatory (Verified silent-corruption, documented in silent-corruption.md). Note the element name is `<user-data>` (not `<custom-data>` — the Open question's wording was incorrect; both AppleScript and the wire format use `user-data`).
- [x] **VERIFIED 2026-05-27 (VM cross-check)** — `subtract work time` and `add work time` are **both universally broken** in OmniPlan 4.10.2 regardless of date form. Five date forms tested: `weekday N` (integer), mutated `current date` (canonical AppleScript date object), `date "Wednesday, June 17, 2026 at 12:00:00 AM"` (full form), `date "June 17, 2026"` (short form), `date "6/17/2026"` (numeric form), and ISO-style `"2026-06-17"` string. ALL forms error `-10000 "AppleEvent handler failed"` — for both commands, on both project-level and per-resource schedules, on both hand-built fixtures and a real OmniPlan-saved bundle (`examples/with-baseline.oplx`). The prior applescript.md claim that the `weekday N` form was "Verified working" is **retracted** — it fails too. The cocoa key for the `on` parameter is `Date` (per .sdef), so a typed AppleScript date should work — but the handler aborts before reading the parameter. Alternate access paths (`set duration of week day schedule N`, `set duration of first week day schedule of s`) error `-10008` ("property not element") or `-1700` ("can't make item N into type specifier") — there is no working setter on the schedule surface from AppleScript. **Workaround: hand-edit `<schedule>` XML directly** (which IS Verified for per-resource overrides emitting full day's hours — see the resources Verified section).
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
