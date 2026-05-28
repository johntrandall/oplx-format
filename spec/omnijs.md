# omniJS surface — reference

Omni Automation (omniJS) is the JavaScript automation layer in OmniPlan. The Omni Group publishes [official docs](https://omni-automation.com/omniplan/) covering the documented public API. This file annotates the API with **observed behaviors, undocumented surfaces, and known bugs** discovered during reverse-engineering.

For the canonical reference, read Omni's docs first.

## Globals (verified by introspection in OmniPlan 4.10.2)

```
actual            // shorthand for the active scenario
document          // shorthand for the active PlanDocument
app               // shorthand for the application
title             // doc title (string)
baselineNames     // Array of String — baseline scenario names
console           // standard
__omnijs__        // internal

// Classes:
Application, Document, PlanDocument, Window, Selection,
Project, Scenario,
Task, TaskType, Resource, ResourceType, ResourceCalendar, ResourceAssignmentType,
Assignment, Dependency, DependencyKind,
Schedule, Calendar, CalendarEvent,
Color, ColorSpace, Decimal, Duration, Date, DateComponents, TimeZone,
FileType, FileWrapper, FileSaver, FilePicker, TypeIdentifier, StringEncoding,
URL, Data, ImageExportPortion, ImageExportView,
PlugIn, Form, Formatter, Image, Audio, Pasteboard, SharePanel, Speech,
Alert, Notification, Credentials, Crypto, Email, MenuItem, ToolbarItem,
Device, DeviceType, Locale, Preferences, Timer, Version
```

## Task instance properties

35+ properties via introspection. Categories below.

### Core identity

- `uniqueID` — string per docs ("used in `omniplan:///task/<uniqueID>` URL"); JXA introspection returns Number. Treat as opaque ID.
- `title` (settable string)
- `type` (TaskType — settable for task/milestone/group; **silently rejected for hammock**)
- `note` (string — writes to rich-text XML format)
- `priority` (number — settable; integer for resource leveling)

### Scheduling — time

- `startDate`, `endDate` (Date r/o — computed)
- `duration` (Duration)
- `effort`, `effortDone`, `effortRemaining` (Number r/o — derived; settable form: `task.effort = N`, `task.effortDone = N`)
- `manualStartDate`, `manualEndDate` (Date or null — settable; persists as `<locked-start-date>` in XML; **manualEndDate back-computes to a locked-START-date** with no separate `<locked-end-date>` element)
- `lockedStartDate`, `lockedEndDate` — **Dropped in OmniPlan 4.x** per official docs; XML element name is the legacy artifact

### Scheduling — constraints

- `startNoEarlierThanDate`, `startNoLaterThanDate`, `endNoEarlierThanDate`, `endNoLaterThanDate` (Date or null — settable)
- `currentStartLimit`, `currentEndLimit` (Date r/o — slack relative to connected tasks)
- `absoluteStartLimit`, `absoluteEndLimit` (Date r/o — slack relative to overall project)
- `freeSlack`, `totalSlack` (Number r/o)

### 3-point estimation

- `minEffortEstimate`, `expectedEffortEstimate`, `maxEffortEstimate` (Number — settable in person-seconds)

When all three are set, `effort` is auto-computed via PERT: `effort = (min + 4*expected + max) / 6`. Generators emitting 3-pt estimates should NOT also write a different `<effort>` value — OmniPlan will overwrite.

### Resources

- `resourceAssignmentType` (ResourceAssignmentType — **this IS the `<recalculate>` field**, despite the unobvious name; Cycle 0008's "genuine gap" claim was wrong). Mapping Verified 2026-05-27 (VM cross-check) in both read and write directions:
  - `ResourceAssignmentType.adjustDuration` ↔ `<recalculate>duration</recalculate>` (default)
  - `ResourceAssignmentType.adjustEffort` ↔ `<recalculate>effort</recalculate>`
  - `ResourceAssignmentType.adjustAssignedUnits` ↔ `<recalculate>units</recalculate>` (note: enum name is `adjustAssignedUnits`, NOT `adjustUnits`)

  The omniJS object's `toString()` form is `[object ResourceAssignmentType: <enum-name>]` — there is no `.name` property; introspect via `Object.getPrototypeOf(value).constructor.name`.
- `resourceLeveledDate` (Date or null r/o)
- `resourceLevelingDelay` (Number r/o)
- `assignments` (Array of Assignment r/o)
- `assignmentsCost` (Number r/o)

### Hierarchy

- `subtasks` (Array of Task r/o — NOT `children`)
- `prerequisites` (Array of Dependency r/o — settable via `addPrerequisite`)
- `dependents` (Array of Dependency)

> **TODO when next OmniPlan release ships** (RT #3107771, 2026-05-05):
> Omni Group support confirmed Task (and Resource) will gain a `parent`
> accessor and a `move` method that reparents without churning
> `uniqueID`. Test builds available now at
> <https://omnistaging.omnigroup.com/omniplan/>. Update this section to
> Verified once the formal release ships and the live surface is tested.
> Until then the 4.10.2-Verified gap statement (no reparent path on the
> hierarchy described above) stands.

### Costs

- `staticCost` (Decimal — settable)
- `totalCost` (Number r/o = static + assignment costs)

### Custom data

- `customData` (read-only proxy — use `customValue(forKey)` and `setCustomValue(forKey, to)` methods)

### Methods on Task.prototype

- `addSubtask() → Task`
- `addAssignment(to: Resource) → Assignment`
- `addPrerequisite(to: Task) → Dependency` — **returns the Dependency, but the `kind` parameter is silently dropped**. Workaround: set `dep.kind = DependencyKind.X` on the returned Dependency.
- `addDependent(to: Task) → Dependency`
- `customValue(forKey: String) → String or null`
- `setCustomValue(forKey: String, to: String)`
- `descendents() → Array of Task`
- `clearResourceLeveledDate()`
- `remove()` — deletes task and its dependencies/assignments
- `split(at: Date, resumingAt: Date) → Boolean` — **returns true but doesn't persist to XML in 4.10.2** (silent corruption)

## Dependency instance properties

7 settable properties:
- `kind` (DependencyKind: FinishStart | FinishFinish | StartStart | StartFinish | Invalid)
- `prerequisiteAlignedToEnd` (Boolean) — coupled with `kind`
- `dependentAlignedToEnd` (Boolean) — coupled with `kind`
- `leadTimeDuration` (Duration or null) — needs `Duration.workSeconds(N)` not raw Number
- `leadTimePercentage` (Number — **integer percent**, not fraction; values <1 silently rejected; Decimal type rejected)
- `prerequisite`, `dependent` (Task refs — read-only)

Three equivalent ways to set non-default kinds programmatically:
1. `dep.kind = DependencyKind.<X>` (this cycle's preferred form)
2. `dep.prerequisiteAlignedToEnd = X; dep.dependentAlignedToEnd = Y` (Boolean coupling)
3. AppleScript `depend ... upon ...; set dependency type of dep to <enum>`

## Assignment instance properties

- `resource`, `task` (refs — read-only)
- `unitsAssigned` (Number — **setting to 0 REMOVES the assignment**, silent corruption)
- `isLocal` (Boolean r/o)
- `specificAssignments` (Array — when assigned to a Resource Group, the actual member resources)
- `totalCost`, `totalDuration`, `totalEffort` (read-only computed)

## Resource instance properties

- `name`, `email`, `note`, `efficiency`, `unitsAvailable`, `costPerHour`, `costPerUse`, `type` — all settable
- `schedule` (Schedule r/o)
- `assignments`, `members`, `customData` (collections)
- `totalHours`, `totalCost`, `completedCost` — read-only computed
- `uniqueID` (Number)

Methods:
- `addMember() → Resource` — for Group resources
- `setCustomData(...)`, `customValue(...)`, `setCustomValue(...)`
- `descendents() → Array of Resource`
- `remove()`

> **TODO when next OmniPlan release ships** (RT #3107771, 2026-05-05):
> Omni Group support confirmed Resource (alongside Task) will gain a
> `parent` accessor and a `move` method addressing the same
> reparent-without-`uniqueID`-churn gap on the resource tree. Test
> builds at <https://omnistaging.omnigroup.com/omniplan/>. Update this
> section to Verified once the formal release ships and the live
> surface is tested. The 4.10.2 documentation above remains accurate
> for that version.

## Scenario (`actual`) properties

- `name` (String)
- `startDate`, `endDate` (Date — settable)
- `hasFixedEndDate` (Boolean — toggle for forward/backward scheduling)
- `hasGenericDates` (Boolean — "Day 1" / "Day 12" mode)
- `completed` (Number — overall percentage 0..1)
- `milestones` (Array r/o)
- `rootTask`, `rootResource` (refs)
- `schedule` (Schedule r/o)
- `totalCost` (Number r/o)

Methods:
- `taskNamed(name: String) → Task or null`
- `resourceNamed(name: String) → Resource or null`
- `customValue(forKey)`, `setCustomValue(forKey, to)`

## Project properties (sparse)

- `actual` (Scenario r/o)
- `baselineNames` (Array of String r/o)
- `document` (PlanDocument or null)
- `title` (String — settable, persistence form TBD)

Methods:
- `baselineNamed(name: String) → Scenario or null`
- `customValue`, `setCustomValue`

## Document (PlanDocument) properties

- `name`, `fileType` (read-only)
- `project`, `windows`, `writableTypes`
- `canUndo`, `canRedo`

Methods:
- `save()` — saves to current file. **No args; ignores any URL/type passed.**
- `makeFileWrapper(baseName: String, type: String or null) → Promise<FileWrapper>` — async; for save-as-other-format. JXA bridge can't await; use from a Plug-in or via `omniplan://localhost/omnijs-run?script=...` URL invocation.
- `fileWrapper(...)` — **Deprecated**, use `makeFileWrapper`
- `close(didCancel)`, `undo()`, `redo()`, `show()`
- `setImageExportSettings(...)`

## PlanDocument class methods

- `PlanDocument.makeNew(resultFunc) → PlanDocument` — programmatic doc creation; alternative to AppleScript `make new document` (which has the -10000 quirk after cold start)
- `PlanDocument.makeNewAndShow(resultFunc) → PlanDocument`

## Schedule class

```js
schedule.calendars                                    // Array of ResourceCalendar (the [Overtime, Time Off] system pair)
schedule.endDate(start, duration) → Date              // when does it end?
schedule.startDate(end, duration) → Date              // when must it start?
schedule.durationBetween(start, end) → Duration       // work-time between two clock times
```

The 3 date-math functions respect the schedule (skip weekends, time-off). Useful for verification batteries that need to compute task end dates correctly.

## ResourceCalendar / CalendarEvent

```js
calendar.name      // "Overtime" or "Time Off" (system-generated) or user-named (from iCal sync)
calendar.overtime  // Boolean
calendar.events    // Array of CalendarEvent (READ-ONLY)

event.start, event.end, event.title  // all read-only
```

**Calendar modification is read-only via omniJS.** Use AppleScript `add work time` / `subtract work time` instead. Per Omni's docs: "scripts can NOT create external resource calendars or the events they display."

## Duration class

Static methods:
- `Duration.workSeconds(N) → Duration`
- `Duration.workHours(N) → Duration`
- `Duration.elapsedDays(N)`, `elapsedHourMinSec(...)`, `elapsedYearMonthDay(...)`

`Duration.zero` is `null` (not a Duration instance) — there's no zero-Duration sentinel.

Cannot be constructed via `Duration.fromString(...)` — that method only exists on `Decimal`.

## Decimal class

`Decimal.fromString(string)` is the canonical constructor for Decimal values.

Used for `staticCost`, `costPerHour`, `costPerUse`, etc. **Do NOT pass Decimal where Number is expected** (e.g., `dep.leadTimePercentage` rejects Decimal).

> **TODO when next OmniPlan release ships** (RT #3107771, 2026-05-05):
> Omni Group support confirmed `Decimal.fromString("100.00").toString()`
> is the canonical round-trip pattern, and that the absence of a direct
> `Decimal → Number` coercion is by design (Decimal wraps
> `NSDecimalNumber`; binary floating-point can't represent every decimal
> value precisely, so a silent coercion would lose precision). Update
> this section to document the canonical pattern as Verified once the
> next release ships and the live behavior is reconfirmed. The 4.10.2
> regex-based `toString` workaround above continues to work in the
> interim.

## URL invocation pattern

```
omniplan://localhost/omnijs-run?script=<URL-encoded JS>
```

Allows external invocation of async omniJS scripts (the path JXA can't reach). Useful for save-as-zip via `makeFileWrapper().write(URL)` chains. Example pattern documented in `applescript.md`.

## Known omniJS bugs in 4.10.2

| Bug | Workaround |
|---|---|
| `addPrerequisite(other, kind)` silently drops the `kind` arg | Use `dep.kind = DependencyKind.X` post-creation |
| `task.type = TaskType.hammock` silently no-op | Hand-write `<type>hammock</type>` in XML |
| `dep.kind = DependencyKind.Invalid` silently no-op | Don't try; `Invalid` is a read sentinel only |
| `dep.leadTimePercentage = 0.25` rejected (silently null) | Use integer percent: `dep.leadTimePercentage = 25` |
| `task.split(at, resumingAt)` returns true but doesn't persist | (no known programmatic workaround in 4.10.2) |
| `document.save(url, type)` ignores both args | Use `makeFileWrapper` or AppleScript `save in <file>` |
| `document.makeFileWrapper(...)` returns Promise — JXA can't await | Use Plug-in or `omniplan://` URL invocation |
| `assignment.unitsAssigned = 0` removes the assignment | Use a small positive value or `assignment.remove()` |
