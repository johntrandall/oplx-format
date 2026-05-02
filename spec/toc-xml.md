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

ID counters. **OmniPlan recomputes these to `max(used-id) + 1` on every save** — hand-bumped values are NOT preserved. Generators producing fresh docs can set them to any plausible starting value (e.g., `2` for an empty doc).

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

Hand-writing new filters requires hand-encoding NSKeyedArchiver bplists — non-trivial. Generators that produce minimal docs typically omit `<filter>` blocks; OmniPlan will not auto-add the default set.

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

## File-format-version

Always `"3"` in OmniPlan 4.10.x. The attribute is required on the `<omniplan>` root.
