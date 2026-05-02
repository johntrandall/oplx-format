# Enum cross-surface mapping

OmniPlan exposes the same conceptual enums via three surfaces — XML, omniJS, and AppleScript — with **different naming conventions**. Generators interfacing with multiple surfaces must use this translation table.

## TaskType

| XML form | omniJS | AppleScript | Internal int |
|---|---|---|---|
| (omitted = default) | `TaskType.task` | `standard task` | `0` |
| `<type>milestone</type>` | `TaskType.milestone` | `milestone task` | `1` |
| `<type>group</type>` | `TaskType.group` | `group task` | `2` |
| `<type>hammock</type>` | `TaskType.hammock` | `hammock task` | `3` |

XML form is **lowercase**. Hand-writing `<type>MILESTONE</type>` (uppercase) or `<type>Milestone</type>` (mixed) **causes the entire document to fail to open silently** — the most aggressive silent-corruption mode in the format. See `silent-corruption.md`.

omniJS `task.type = TaskType.hammock` is silently rejected (the setter accepts the value but readback shows `task`). The only programmatic path to a hammock task is XML hand-write. AppleScript also refuses hammock task creation.

## ResourceType

| XML form (capitalized!) | omniJS | AppleScript | Notes |
|---|---|---|---|
| `<type>Staff</type>` | `ResourceType.staff` | `person` | Note AS uses `person`, not `staff` |
| `<type>Equipment</type>` | `ResourceType.equipment` | `equipment` | |
| `<type>Material</type>` | `ResourceType.material` | `material` | Auto-emits `<units-available>0</units-available>` |
| `<type>Group</type>` | `ResourceType.group` | `resource group` | |
| `<type>Project</type>` | (auto-assigned) | (auto-assigned) | Reserved for `r-1` (root resource) |

XML form is **capitalized** — opposite convention from TaskType. Generators producing resource XML must capitalize.

AppleScript renames `Staff` → `person` and `Group` → `resource group`. This is a third convention mismatch.

## DependencyKind

| XML attribute (uppercase!) | omniJS | AppleScript | Internal |
|---|---|---|---|
| (omitted = default FS) | `DependencyKind.FinishStart` | `finish to start` | |
| `kind="FF"` | `DependencyKind.FinishFinish` | `finish to finish` | |
| `kind="SS"` | `DependencyKind.StartStart` | `start to start` | |
| `kind="SF"` | `DependencyKind.StartFinish` | `start to finish` | |
| (sentinel — never emit) | `DependencyKind.Invalid` | (none) | For unparseable input |

XML `kind=` attribute is **CASE-SENSITIVE UPPERCASE only**. `kind="ff"` (lowercase) is silently stripped — the dependency becomes the default Finish-Start. This is silent corruption: file appears valid, dependencies look correct in the outline, but scheduling produces wrong results.

`DependencyKind.Invalid` is a read-only sentinel exposed via omniJS for legacy data; setting `dep.kind = DependencyKind.Invalid` is a silent no-op.

## ResourceAssignmentType (XML element: `<recalculate>`)

| XML form | omniJS | Internal int | Effect |
|---|---|---|---|
| `<recalculate>duration</recalculate>` (default) | `ResourceAssignmentType.adjustDuration` | `0` | Duration adjusts when assignments change |
| `<recalculate>effort</recalculate>` | `ResourceAssignmentType.adjustEffort` | `1` | Effort adjusts when units change |
| `<recalculate>units</recalculate>` | `ResourceAssignmentType.adjustAssignedUnits` | `2` | Units adjust when duration changes |

**No `none` value.** Hand-writing `<recalculate>none</recalculate>` is silently normalized to `duration` on save. (Some Omni Group docs use the term "none" for the effective behavior when no resources are assigned, but it is not a settable enum value.)

**No `assignments` value.** Earlier community documentation (including a now-corrected version of the `omniplan-concepts` skill) listed `assignments` as the third value. The actual XML form is `units` — the omniJS enum key `adjustAssignedUnits` was the source of confusion.

When `<recalculate>` is non-default, OmniPlan also emits `<fixed-duration>SECONDS</fixed-duration>` alongside `<effort>`. The two coexist with `<static-cost>` (they are not mutually exclusive).

## Lead-time percentage convention mismatch

The dependency lead-time-as-percentage value uses **different conventions** across surfaces:

| Surface | Input value | XML serialization |
|---|---|---|
| XML `<lead-time is-percentage="true">N</lead-time>` | `N` is the **fraction** (0.25 = 25%, 1.0 = 100%) | as written |
| AppleScript `set lead percentage to 0.25` | the **fraction** | XML 0.25 |
| omniJS `dep.leadTimePercentage = 25` | **integer percent** (25 = 25%) | XML 0.25 |
| omniJS `dep.leadTimePercentage = 0.25` | **silently rejected** (readback `null`, no XML) | (none) |

Generators using omniJS must convert fraction → integer-percent. omniJS rejects values with absolute value < 1 (so `0.5` → null). Decimal type is also rejected — must be a Number.

## Scheduling granularity

| XML form (in `Actual.xml`, not `__TOC.xml`) | omniJS | AppleScript |
|---|---|---|
| (omitted = default exact) | (not exposed) | `exact scheduling` |
| `<granularity>hours</granularity>` | (not exposed) | `hourly scheduling` |
| `<granularity>days</granularity>` | (not exposed) | `daily scheduling` |

omniJS does NOT expose this property. Settable via AppleScript `set scheduling granularity to ...` or by hand-edit of `Actual.xml`.

## Assignment units

| omniJS / AppleScript value | XML form |
|---|---|
| 1.0 (default) | `<assignment idref="rN"/>` (self-closing, no `units=`) |
| 0.5, 1.5, 2.0, etc. | `<assignment idref="rN" units="0.5"/>` |
| 0.0 | `<assignment>` element is **REMOVED** entirely (silent corruption — see `silent-corruption.md`) |

## Task status (read-only, computed)

| AS form | Internal int | Meaning |
|---|---|---|
| `ok` | `0` | Default, no status flag |
| `close to due date` | `1` | Approaching due date (status icon green) |
| `due now` | `2` | Due today (status icon orange) |
| `past due` | `3` | Past due (status icon red) |
| `finished` | `4` | Complete |

Used in saved `<filter>` predicates (e.g., "Due Soon" filters `status == 1`; "Overdue" filters `status == 3`).

## Other enums

- `ImageExportPortion`: `bothViews`, `visualViewOnly`, `outlineOnly` (omniJS only — image export config)
- `DeviceType`: `mac`, `iPhone`, `iPad`, `visionPro` (omniJS — for cross-platform script branching)
- `ColorSpace`: `RGB`, `White`, `CMYK`, `HSB`, `Pattern`, `Named` (omniJS — used in `<color>` elements)
- `StringEncoding`: 23 values (omniJS — for text I/O scripts; not relevant to file format)
- `FileType` / `TypeIdentifier`: 12 writable types (zip, ical, csv, mpp, mspdi, png, pdf, tiff, jpeg, ooutline, gv, planfile)

See `omnijs.md` for the complete omniJS enum surface.
