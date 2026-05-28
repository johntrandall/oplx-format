# `__TOC.xml`

Project-level metadata, scenario index, window state, default styling, and saved filters.

## Minimum viable form

```xml
<?xml version="1.0" encoding="UTF-8"?>
<omniplan xmlns="http://www.omnigroup.com/namespace/OmniPlan/v2" file-format-version="3">
  <project>
    <next-task-id>2</next-task-id>
    <next-resource-id>2</next-resource-id>
    <scenario id="<scenario-id>" name="Actual" filename="Actual.xml"/>
  </project>
</omniplan>
```

That's it. OmniPlan opens this and fills in everything else (`<window>`, styling, filters, etc.) on first save.

## Full structure (post-save by OmniPlan)

```xml
<omniplan xmlns="..." file-format-version="3">

  <window>
    <editing-scenario>...</editing-scenario>
    <view>task|resource</view>
    <change-tracking/>
    <status-display>basic</status-display>
    <task-view>...</task-view>           <!-- outline columns, gantt scales, task-bar-labels -->
    <resource-view>...</resource-view>
    <network-view>...</network-view>
    <project-outline-view>...</project-outline-view>
  </window>

  <project>
    <next-task-id>N</next-task-id>
    <next-resource-id>N</next-resource-id>

    <scenario id="..." name="Actual" filename="Actual.xml"/>
    <scenario id="..." name="Baseline ..." filename="Baseline-...xml"/>...

    <date-display dates="true" times="true" seconds="false"/>
    <numbering-style>wbs</numbering-style>
    <critical-path-slack>0</critical-path-slack>
    <currency-format>$1,234.56</currency-format>
    <duration-format hours-per-day="8" hours-per-week="40" hours-per-month="160" hours-per-year="1920" hours="true" days="true" weeks="true"/>
    <effort-format hours-per-day="8" hours-per-week="40" hours-per-month="160" hours-per-year="1920" hours="true" days="true" weeks="true"/>

    <base-style>...</base-style>
    <column-title-style>...</column-title-style>
    <note-style>...</note-style>
    <standard-task-style>...</standard-task-style>
    <overdue-task-style>...</overdue-task-style>
    <completed-task-style>...</completed-task-style>
    <group-task-style>...</group-task-style>
    <milestone-task-style>...</milestone-task-style>
    <hammock-task-style>...</hammock-task-style>
    <resource-style>...</resource-style>
    <not-editable-style>...</not-editable-style>

    <task-user-data-keys>
      <user-data>
        <key>BudgetCode</key><null/>
        <key>CostCenter</key><null/>
      </user-data>
    </task-user-data-keys>

    <filter name="..." flat="0|1" id="...">[base64-bplist]</filter>...

    <subscribe-refresh>0</subscribe-refresh>
    <leveling>
      <constrains-completion-date/>
    </leveling>
    <page-adornment>...</page-adornment>
  </project>

</omniplan>
```

## `<project>` block elements

### `<next-task-id>`, `<next-resource-id>`

ID counters. OmniPlan **preserves these verbatim from input** (Verified 2026-05-27 — correcting prior claim that they were recomputed to `max(used-id) + 1` on save). Hand-bumped values DO survive: a doc opened with `<next-task-id>2</next-task-id>` and existing tasks `t1`/`t10`/`t99` saves with the counter still at `2`. When the counter is too low and OmniPlan later creates a new task internally, it uses the counter value as-is (producing `t2` in this case, coexisting with `t99`) — there is no validation against existing IDs. Generators producing fresh docs should set the counter to `max(used-id) + 1` for forward correctness; OmniPlan does not do this for you.

### `<scenario>` references

Index of all scenarios in the bundle. The first conventionally `name="Actual"`. Each subsequent entry corresponds to a baseline scenario file in the bundle. Multiple `<scenario>` entries enable multi-baseline workflows.

### `<numbering-style>`

Verified value: `wbs`. Other values unknown. The corresponding AppleScript `numbering style` enum from the dictionary lists no other named values.

### `<critical-path-slack>`

Slack threshold (in seconds) below which a task is considered on the critical path. Default `0`.

### `<currency-format>`

Locale-display placeholder string (e.g., `"$1,234.56"`). The literal text is the format pattern OmniPlan uses to render currency values.

### `<duration-format>` / `<effort-format>`

Define the unit conversions and which units are visible:

```xml
<duration-format
  hours-per-day="8" hours-per-week="40"
  hours-per-month="160" hours-per-year="1920"
  hours="true" days="true" weeks="true"/>
```

Both elements have identical structure. The `hours-per-*` attributes set the display conversions (e.g., `1d = 8h`); the boolean attrs (`hours`, `days`, `weeks`, optional `months`, `years`) toggle which units appear in the UI.

### `<*-task-style>` / `<*-style>` blocks

9+ default-styling blocks for the various task/resource categories. Generators producing minimal docs can omit these — OmniPlan fills in defaults on save. If preserving custom styling matters, copy these blocks verbatim from a known-good source bundle.

### `<task-user-data-keys>`

Registry of custom-data keys that exist in the doc. Each `<key>NAME</key>` is paired with `<null/>` (placeholder for type/default — type system not yet documented). Tasks can use `<user-data>` keys not registered here, but they may not surface in the UI's Custom Data Inspector until registered.

### `<filter>` blocks

Saved smart-filter definitions. The element body is a base64-encoded Apple binary plist (NSKeyedArchiver of an NSPredicate). Decoding requires Python's `plistlib.loads(base64.b64decode(...))`.

System-default filters in a fresh OmniPlan doc:

| Filter name | Property tested | Operator | Value |
|---|---|---|---|
| Dependents of Selection | `selectedDependencyChain` | CONTAINS | `"dependents"` |
| Due Soon | `status` | == | `1` |
| In-Progress Tasks | `completedPercentage` | (< 100 OR > 0) | |
| Incomplete Tasks | `completedPercentage` | < | `100` |
| Milestones Only | `taskType` | == | `1` |
| Overdue | `status` | == | `3` |
| Prerequisites for Selection | `selectedDependencyChain` | CONTAINS | `"prerequisites"` |
| Project's Critical Path | `isOnCriticalPath` | == | `True` |
| Selected Tasks Only | `selectedAtTimeOfFiltering` | == | `True` |
| Tasks Only | `taskType` | == | `0` |
| Unassigned | `assignedResources[SIZE]` | == | `0` |
| Unstarted Tasks | `completedPercentage` | == | `0` |

Filters reveal **internal property names** that aren't surfaced as XML elements:

- `taskType` (int): 0=task, 1=milestone, 2=group, 3=hammock
- `status` (int): per `task status` AppleScript enum (1=close-to-due, 3=past-due, etc.)
- `completedPercentage` (int 0..100)
- `isOnCriticalPath` (bool)
- `assignedResources` (collection — supports `SIZE`)
- `selectedDependencyChain` (string `"dependents"` or `"prerequisites"`)
- `selectedAtTimeOfFiltering` (bool)

Hand-writing new filters from scratch requires hand-encoding NSKeyedArchiver bplists — non-trivial. Generators that produce minimal docs typically omit `<filter>` blocks; OmniPlan will not auto-add the default set.

#### Modifying an existing filter (Verified 2026-05-27)

Hand-modifying an existing filter's predicate is feasible and round-trips cleanly. Recipe (Python 3):

```python
import base64, plistlib, re

# 1. Read TOC and locate the filter
with open("__TOC.xml") as f:
    toc = f.read()
m = re.search(r'<filter name="Due Soon"[^>]*>([^<]+)</filter>', toc)
b64 = m.group(1)

# 2. Decode the bplist
plist = plistlib.loads(base64.b64decode(b64))

# 3. Mutate the desired field. The constant-value index depends on
#    the predicate's structure (compound vs simple vs function-expr) —
#    inspect plist['$objects'] to locate it. For a simple equality
#    predicate like "status == 1", the constant is the integer object
#    referenced by NSConstantValueExpression.
plist['$objects'][15] = 99   # was 1; index discovered via inspection

# 4. Re-serialize as binary plist and re-encode base64
new_b64 = base64.b64encode(
    plistlib.dumps(plist, fmt=plistlib.FMT_BINARY)
).decode('ascii')

# 5. Substitute back and re-zip the bundle
with open("__TOC.xml", "w") as f:
    f.write(toc.replace(b64, new_b64))
```

OmniPlan accepts the differently-sized bplist (the Python serializer produces a slightly different byte layout than OmniPlan's) without complaint, and the modification survives a save round-trip. Sibling filters are not affected — OmniPlan reads each filter independently by `id=`.

**Constraint:** the exact object index of a value depends on the predicate type. Always locate via `plistlib.loads` + dict inspection; never hard-code an index for a different predicate.

### `<leveling>`

Resource-leveling rules. Default body:

```xml
<leveling>
  <constrains-completion-date/>
</leveling>
```

Empty marker children indicate which constraints leveling respects. Other markers (e.g., `<constrains-start-date/>`, `<constrains-resource/>`) are unverified.

### `<page-adornment>`

Print-time page headers/footers. Uses an OmniPlan variable system with placeholders like `OPDocumentTitleVariableIdentifier`, `OPPrintJobTimestampVariableIdentifier`. Generators not concerned with print output can omit.

## `<window>` block

Records UI state as of last save: which view is active, column widths, gantt zoom levels, task-bar-label configuration per task-type. Not required for generators — OmniPlan fills in defaults.

The `<scale scale-name="..." full-day-width="N">` entries set the gantt zoom levels (Automatic, Day, Hour, Minute, Month, Quarter, Week, Year). One has `<selected/>` indicating the active zoom.

### `<column>` entries inside `<outline>`

Each column in the outline displays as `<column name="X" width="N"/>`. The `name` attribute uses the human-readable column name with capitalized casings (Verified 2026-05-27 across `funding-pipeline.oplx`, `Gantt Timeline BAK 11.11.oplx`, and a spacessync-launch.oplx-derived fixture). Known built-in names: `Violations`, `Notes`, `Title`, `Effort`, `Start`, `End`, `Prerequisites`, `Assigned`, `Type`, `Resource`, `Status`, `%Done`. Non-exhaustive — OmniPlan exposes more via View → Columns.

**Pro custom-data fields surface as `<column name="<KeyName>" .../>` entries** matching their custom-data key verbatim. Verified 2026-05-27 (VM `oplx-spec-verify`): `t.setCustomValue("BudgetCode","BC-9999")` via omniJS emitted `<column name="BudgetCode" width="94"/>` and `<column name="Department" width="89"/>` in the saved `__TOC.xml`, plus matching keys registered in `<task-user-data-keys>`.

Wrong attribute (`key=` instead of `name=`, or lowercase column names) causes file-level rejection with silent `-10000`. See `silent-corruption.md`.

### `<gantt-view>` toggle children — View → Gantt menu state

The `<gantt-view>` element holds the Gantt-area UI configuration. Five empty-element children represent the state of toggleable items in **View → Gantt** menu. When the toggle is ON, the element is present; when OFF, the element is absent. **Menu item is `enabled=true` in BOTH cases** — the element controls the CHECKED state of the toggle, not the ENABLED state. (Verified 2026-05-27 via single-variable AppleScript probe against OmniPlan 4.10.2.)

| Menu item | XML element | Notes |
|---|---|---|
| Dependency Lines | `<dependency-lines/>` | Verified — controls toggle checked state |
| Critical Paths | `<critical-path/>` | **Singular** (not `critical-paths/`). Verified by menu-toggle → save → diff |
| Slack Lines | `<slack/>` | **Drops "lines"**. Verified by menu-toggle → save → diff |
| Group Shading | `<group-shading/>` | Verified |
| Constraints | `<constraints/>` | Verified — corpus (BAK + with-baseline) + isolation |

**Element placement:** all five appear as direct children of `<gantt-view>`, between `<view-mode>actual</view-mode>` and the first `<scale>` entry. Order observed across saved files: `dependency-lines`, `constraints`, then the others when present.

**Menu-name-to-element-name is NOT a simple pattern.** Don't assume kebab-case-of-menu-name — `Critical Paths → critical-path` (singular) and `Slack Lines → slack` (drops the noun) both break that pattern. When emitting from a generator, use exactly the names above. When extending to a future toggle, do a toggle-via-menu → save → diff test to discover the actual element name, don't guess.

**For a generator:** the safe default is to OMIT all five elements (toggle defaults are OFF). Hand-generated files like `with-hammock.oplx` and `minimum-viable.oplx` open fine without them — the menu items appear enabled and unchecked. Include them only if you want a specific toggle to be ON when the user first opens the file.

**Test corpus (2026-05-27):**
- `~/dev/_funding/funding-pipeline.oplx` (OmniPlan-saved): contains none of the five
- `~/dev/oplx-format/examples/with-baseline.oplx` (OmniPlan-saved): `<dependency-lines/>` + `<constraints/>`
- `~/Documents/Documents - ChoChang/Gantt Timeline BAK 11.11.oplx` (OmniPlan-saved): `<dependency-lines/>` + `<constraints/>`

## File-format-version

Always `"3"` in OmniPlan 4.10.x. The attribute is required on the `<omniplan>` root.
