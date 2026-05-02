# `__changelog.xml`

Operational mutation log. Records every property change made to the document, batched into change-sets per editing session.

## Minimum form

```xml
<?xml version="1.0" encoding="UTF-8"?>
<changelog xmlns="http://www.omnigroup.com/namespace/OmniPlan/v2">
  <version>4.0</version>
</changelog>
```

A doc with no changes since creation has just `<version>4.0</version>` and no `<change-set>` entries. This is sufficient for OmniPlan to open the doc.

## Full structure

```xml
<changelog xmlns="...">
  <version>4.0</version>

  <change-set user="..." date="ISO-8601-UTC" order="N">
    <change idref="tN" [to="tParent"] [to-position="N"]>
      <change idref="tN" attribute="..." type="real|date|string|..." to="VALUE"/>
      ...
    </change>
    ...
  </change-set>
  ...
</changelog>
```

## `<change-set>` element

Groups all mutations from one logical operation (typically one user action or one scripted batch).

| Attribute | Meaning |
|---|---|
| `user` | The actor that made the change. UI sets this to the OS user's full name; AppleScript `change mark from "..."` updates it for subsequent change-sets. Generators commonly use `"claude"`, `"gen_oplx.py"`, etc. |
| `date` | ISO-8601 UTC with milliseconds |
| `order` | Monotonically increasing integer across the doc's lifetime. OmniPlan increments this; generators producing fresh changelogs can use `1, 2, 3, ...` |

## `<change>` element (nested)

Two semantic forms:

### Outer `<change>`: object lifecycle

```xml
<change idref="t500" to="t-1" to-position="5">
  ...inner changes...
</change>
```

Records the creation/move/delete of the object identified by `idref`:
- `to="<parent-id>"` and `to-position="N"`: insertion as the Nth child (0-indexed) of the parent
- (no `to`): mutation of an existing object

### Inner `<change>`: property mutation

```xml
<change idref="t500" attribute="title" type="string" to="Task 2"/>
<change idref="t500" attribute="effort" type="real" to="3600"/>
<change idref="t500" attribute="resourceAssignmentType" type="real" to="1"/>
```

| Attribute | Meaning |
|---|---|
| `idref` | Target object id |
| `attribute` | Internal property name (often differs from XML element name; see below) |
| `type` | Value type: `string`, `real`, `date`, `boolean`, etc. |
| `to` | New value |

Inner `<change>` elements are nested inside their outer `<change>` (the object-lifecycle wrapper).

## Internal property name table

The changelog reveals OmniPlan's internal property names. These often differ from the XML element names — sometimes meaningfully (e.g., `<recalculate>` is internally `resourceAssignmentType`).

### Task properties

| Internal property | Type | XML element / API surface | Notes |
|---|---|---|---|
| `title` | string | `<title>` | |
| `effort` | real (seconds) | `<effort>` | |
| `effortDone` | real (seconds) | `<effort-done>` | omniJS uses same name |
| `internalDuration` | string | (not directly serialized) | Format: `"3600s"` with literal `s` suffix |
| `internalPrerequisitesString` | string | (computed from `<prerequisite-task>` siblings) | Format: `"57FS+0.25%, 58FF+3600"` |
| `resourceAssignmentType` | real | `<recalculate>` | 0=duration, 1=effort, 2=units |
| `priority` | real | `<priority>` | |
| `noEarlierThanConstraintDate` | date | `<start-no-earlier-than>` or `<end-no-earlier-than>` | Single-field repr; flag below distinguishes |
| `noEarlierThanConstrainedTaskEnd` | real (0/1) | (combines with above) | When 1, constraint applies to task END (so → `<end-no-earlier-than>`) |
| `noLaterThanConstraintDate` | date | `<start-no-later-than>` or `<end-no-later-than>` | Same single-field pattern |
| `noLaterThanConstrainedTaskEnd` | real (0/1) | (combines with above) | When 1, applies to task END |
| `manualDate` | date | `<locked-start-date>` | |
| `manualDateConstrainsTaskEnd` | real (0/1) | (no separate XML element) | When 1, the manual date constrains end (start back-computed); when 0, constrains start |
| `resourceLeveledDate` | date | `<leveled-start>` | Set after leveling |
| `taskType` | int | `<type>` | 0=task, 1=milestone, 2=group, 3=hammock |
| `staticCost` | Decimal | `<static-cost>` | |
| `expectedEffortEstimate` | real | `<expected-estimate>` | 3-pt PERT |
| `minEffortEstimate` | real | `<min-estimate>` | 3-pt PERT |
| `maxEffortEstimate` | real | `<max-estimate>` | 3-pt PERT |
| `status` | int | (not directly serialized; computed) | Task-status enum |
| `completedPercentage` | int 0..100 | (computed from effort/effortDone) | |
| `isOnCriticalPath` | bool | (computed) | |
| `assignedResources` | collection | (computed from `<assignment>` siblings) | |
| `selectedAtTimeOfFiltering` | bool | (UI state, internal) | |
| `selectedDependencyChain` | string | (UI state, internal) | "dependents" or "prerequisites" |

### Resource properties

| Internal property | XML element |
|---|---|
| `name` | `<name>` |
| `type` | `<type>` (capitalized: Staff/Equipment/Material/Group/Project) |
| `email` | `<email-address>` |
| `efficiency` | `<efficiency>` |
| `unitsAvailable` | `<units-available>` |
| `costPerHour` | `<cost-per-hour>` |
| `costPerUse` | `<cost-per-use>` |
| `note` | `<note>` |

## Implications for generators

A doc generated programmatically can have an empty changelog — OmniPlan does not require change history. Generators producing fresh docs typically emit:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<changelog xmlns="http://www.omnigroup.com/namespace/OmniPlan/v2">
  <version>4.0</version>
</changelog>
```

If the generator wants to identify itself, it can add one synthetic change-set with `user="my-tool"`. The user attribution will then appear in OmniPlan's Change Tracking UI for any further user edits.

## Implications for parsers

The changelog is the **only** place where many internal property names are exposed. If a parser needs to track property changes over time (audit, ETL pipelines), the changelog is the source of truth — `Actual.xml` only shows current state.

## Implications for diff tools

Two `.oplx` documents with identical `Actual.xml` may have different `__changelog.xml` contents (different mutation history → same final state). For semantic-equivalence checks, diff `Actual.xml` and ignore `__changelog.xml`. For audit checks, diff `__changelog.xml`.
