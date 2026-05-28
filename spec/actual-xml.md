# `Actual.xml` (and other scenario files)

The scenario file contains the entire task tree, resource tree, dependency graph, schedule, and per-scenario settings.

## Root structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<scenario xmlns="http://www.omnigroup.com/namespace/OmniPlan/v2"
          id="<scenario-id>"
          [fixed-end="true"]>
  <start-date>ISO-8601-UTC</start-date>
  [<end-date>ISO-8601-UTC</end-date>]
  [<granularity>hours|days</granularity>]

  <prototype-task>...</prototype-task>...      <!-- optional -->
  <prototype-resource>...</prototype-resource>... <!-- optional -->

  <top-resource idref="r-1"/>
  <resource id="r-1">...</resource>             <!-- root resource -->
  <resource id="rN">...</resource>...           <!-- non-root resources -->

  <top-task idref="t-1"/>
  <task id="t-1">...</task>                     <!-- root task (always type=group) -->
  <task id="tN">...</task>...                   <!-- non-root tasks -->

  [<critical-path root="-1" enabled="true|false" resources="false">
    <color .../>
  </critical-path>]
</scenario>
```

The `id` attribute on the root `<scenario>` MUST match the `id` in `__TOC.xml/<project>/<scenario>`.

### Scheduling direction

| Mode | XML signal | Notes |
|---|---|---|
| Forward from fixed start (default) | `<start-date>` only | Scenario root has no `fixed-end` attribute |
| Backward from fixed end | `<end-date>` + `fixed-end="true"` on `<scenario>` | `<start-date>` may also be present |
| Generic dates ("Day 1", "Day 12") | `hasGenericDates=true` (omniJS) | XML form not yet captured |

### Granularity

- `<granularity>hours</granularity>` — tasks start/end on hour boundaries
- `<granularity>days</granularity>` — tasks start/end at day boundaries
- (omitted, default) — exact (down to the second)

Only set via AppleScript `set scheduling granularity to ...` or by hand-edit; omniJS does not expose this property. **The element lives in `Actual.xml` (per scenario), NOT in `__TOC.xml`.**

## `<task>` element

```xml
<task id="tN">
  <title>...</title>                                  <!-- always present, may be self-closing for empty -->
  [<type>milestone|group|hammock</type>]               <!-- omitted for default 'task' (standard) -->
  [<priority>0..N</priority>]                          <!-- integer; default 0 omitted -->
  [<effort>SECONDS</effort>]                           <!-- omitted for milestones, groups; PERT-computed when 3-pt set -->
  [<effort-done>SECONDS</effort-done>]                 <!-- progress; default 0 omitted -->
  [<min-estimate>SECONDS</min-estimate>]               <!-- 3-point estimation: pessimistic-min -->
  [<expected-estimate>SECONDS</expected-estimate>]     <!-- 3-point: most-likely -->
  [<max-estimate>SECONDS</max-estimate>]               <!-- 3-point: pessimistic-max -->
  [<schedule-type>asap</schedule-type>]                <!-- present in fixed-end mode only -->
  [<locked-start-date>ISO-UTC</locked-start-date>]     <!-- manual scheduling -->
  <recalculate>duration|effort|units</recalculate>     <!-- always; default 'duration' -->
  [<fixed-duration>SECONDS</fixed-duration>]           <!-- emitted alongside non-default <recalculate> -->
  <static-cost>0</static-cost>                         <!-- always (Decimal) -->
  [<leveled-start>ISO-UTC</leveled-start>]             <!-- present after leveling has run -->

  [<child-task idref="..."/>...]                       <!-- groups only -->
  [<note>...</note>]                                   <!-- rich text; see Notes section -->

  [<start-no-earlier-than>ISO-UTC</start-no-earlier-than>]
  [<start-no-later-than>ISO-UTC</start-no-later-than>]
  [<end-no-earlier-than>ISO-UTC</end-no-earlier-than>]
  [<end-no-later-than>ISO-UTC</end-no-later-than>]

  [<prerequisite-task idref="..." [kind="FF|SS|SF"]/>...]  <!-- self-closing for plain prereq -->
  [<prerequisite-task idref="...">                          <!-- non-empty when lead-time present -->
    <lead-time is-percentage="false">SECONDS</lead-time>
   </prerequisite-task>]

  [<attachment uri="file:///abs/path">                  <!-- file or HTTP attachment; multiple allowed -->
    <bookmarkData>BASE64_NSURL_BOOKMARK</bookmarkData>  <!-- required for file:// URIs -->
   </attachment>]

  [<assignment idref="rN" [units="0.5"]/>]              <!-- self-closing default; units=N omitted when 1.0 -->

  [<user-data>...</user-data>]                          <!-- custom data -->
</task>
```

### Element ordering rules (Verified)

- `<title>` is first
- `<type>` (if present) immediately after `<title>`
- `<priority>` (if non-default) after `<type>`, before `<effort>`
- `<effort>` and 3-pt estimates: `effort → min-estimate → expected-estimate → max-estimate`
- `<schedule-type>` after `<effort>`, before `<recalculate>` (fixed-end mode only)
- `<locked-start-date>` between `<effort>` and `<recalculate>` (NOT after `<static-cost>`)
- `<recalculate>` always present
- `<fixed-duration>` after `<recalculate>` when emitted
- `<static-cost>` after `<recalculate>`/`<fixed-duration>`
- `<leveled-start>` after `<static-cost>`
- `<child-task>` references after the value elements (groups only)
- `<note>` after `<static-cost>` (or after `<child-task>` for groups)
- The 4 `*-no-*-than` constraints AFTER `<static-cost>` (not before like locked-start-date)
- `<attachment>` siblings after `<static-cost>`, before `<prerequisite-task>` (Verified 2026-05-28)
- `<prerequisite-task>` siblings after constraints (and after any `<attachment>` siblings)
- `<assignment>` siblings after prerequisites
- `<user-data>` typically last

### `<note>` rich-text format

Plain multi-line note `"Line 1\nLine 2"`:

```xml
<note>
  <text>
    <p>
      <run><lit>Line 1</lit></run>
    </p>
    <p>
      <run><lit>Line 2</lit></run>
    </p>
  </text>
</note>
```

Each line → `<p>` containing `<run>` containing `<lit>`.

#### Styled runs

Styled runs add a `<style>` block inside `<run>` before the `<lit>`. The grammar is:

```xml
<run>
  <style>
    <value key="KEY">VALUE</value>
    <value key="KEY2">VALUE2</value>
  </style>
  <lit>styled text</lit>
</run>
```

Verified 2026-05-27 (VM cross-check) — the canonical wire-format keys are:

| Key | Value type | Verified values |
|---|---|---|
| `paragraph-alignment` | enum | `left` (default — stripped), `center`, `right` |
| `font-family` | string | PostScript family name (e.g. `Helvetica`) |
| `font-weight` | integer | `9` for bold; `5` (default normal) is stripped |
| `font-italic` | boolean | `yes` (no = default = stripped) |
| `font-size` | integer | Point size (e.g. `18`) |

OmniPlan also accepts `<value key="font-name">PostScriptFontName</value>` as input — it normalizes on save by splitting the PostScript name into `font-family` + `font-weight` (or `font-italic` if the face is oblique). `font-name` is write-only; output always uses the split form.

Invalid keys silently strip the entire `<style>` block on save: `font`, `bold`, `italic`, `font-style`, `font-color`, `foreground-color`, `text-color`, `color`, `underline`, `size`. The wire-format key for text color in 4.10.2 is **not yet identified** — all obvious candidates strip.

Multiple `<value>` children inside one `<style>` are allowed and round-trip together.

**No scripting surface supports styled notes** — see `coverage.md` Verified entry under "Edge cases" (omniJS `task.note` is a plain `String`; AppleScript `word N of note of task` errors `-1700` "can't make into type specifier"). XML hand-write is the only programmatic path; the GUI is the only interactive path.

### `<user-data>` (custom data) form

```xml
<user-data>
  <key>BudgetCode</key>
  <string>BC-2026</string>
  <key>CostCenter</key>
  <string>ABC-100</string>
</user-data>
```

Interleaved `<key>NAME</key>` followed by `<string>VALUE</string>`. **Verified 2026-05-27:** only `<string>` survives as a value type in OmniPlan 4.10.2; injecting `<number>`, `<date>`, or `<boolean>` causes file-level rejection at open time (silent `-10000`). The AppleScript `custom data` surface declares value as `type="any"` and the internal model supports more, but only `<string>` persists through the wire format in 4.10.2. Order is not predictable across saves; do not rely on alphabetical or insertion-order.

The keys in the doc must also be registered in `__TOC.xml/<project>/<task-user-data-keys>` as `<key>NAME</key><null/>` pairs. **This is not optional** (Verified 2026-05-27): without TOC registration, OmniPlan silently strips the `<user-data>` block on save. Position also matters — the block must come AT THE END of the task element (after `<note>`, `<assignment>`, etc.); injected earlier in the task it is silently stripped on save. `<user-data>` is not valid on the root task `t-1` — placing it there causes file-level rejection.

### `<attachment>` element

Verified 2026-05-28 against OmniPlan 4.10.2 build 232.5.0 by manual GUI attach + round-trip hand-emission.

```xml
<attachment uri="file:///Users/me/Documents/spec.pdf">
  <bookmarkData>YnBsaXN0MDDUAQIDBAUGBwhfEBhib29rbWFy...</bookmarkData>
</attachment>
```

- Element name: **`<attachment>`** (singular). Lives directly inside `<task>`. Multiple `<attachment>` siblings allowed per task.
- `uri` attribute (required): standard URL. For local files, the `file://` form macOS emits via `NSURL.absoluteString()` (includes trailing percent-encoding and a `/` on directories).
- `<bookmarkData>` child (required for `file://` URIs): base64-encoded macOS NSURL bookmark data. The bookmark encodes inode + path + volume UUID so the link survives file renames. **An `<attachment>` without `<bookmarkData>` is silently ignored on load** — the document opens, the lint surface does not complain, but `count attachments of <task>` returns 0 (see `silent-corruption.md` `ATTACH-NO-BOOKMARK`).
- Element position: after `<static-cost>` (and after any constraint dates) and before `<prerequisite-task>` / `<assignment>` / `<note>` / `<user-data>`. The full verified `<task>` order observed in the 2026-05-28 experiment was: `<title> → <type> (if not default) → <effort> (if not group) → <recalculate> → <static-cost> → <attachment>... → <prerequisite-task>... → <assignment>... → <note> → <user-data>`.
- Scripting surfaces (as of 4.10.2): AppleScript declares `attachment.file` as `access="r"` (read-only); omniJS has no `Attachment` class. **XML emission is the only programmatic path** to add an attachment. See `coverage.md` Verified entry.
- HTTP/HTTPS URIs: OmniPlan accepts `http://` / `https://` URIs without `<bookmarkData>` in the AppleScript / GUI surfaces, but the round-trip wire form is not yet enumerated here — verify against your installed build before relying on it.

#### Generating `<bookmarkData>` (macOS, PyObjC)

```python
from Foundation import NSURL  # pip install pyobjc-framework-Cocoa
import base64

url = NSURL.fileURLWithPath_(abspath)
bookmark, err = url.bookmarkDataWithOptions_includingResourceValuesForKeys_relativeToURL_error_(
    0, None, None, None,
)
# err is None on success; bookmark is an NSData (~1.1 KB for a typical ~/Documents/ path)
b64 = base64.b64encode(bytes(bookmark)).decode("ascii")
uri = url.absoluteString()  # file:///Users/.../file.ext
```

The bookmark binary is an Apple-internal NSKeyedArchiver-style format. Treat as opaque base64; if you ever need to resolve one back to a path, route through `CFURLCreateByResolvingBookmarkData`. Generate fresh on emit rather than attempting to round-trip parsed bookmarks.

## `<resource>` element

```xml
<resource id="rN">
  <name>...</name>                                     <!-- may be self-closing for empty -->
  <type>Staff|Equipment|Material|Group|Project</type>  <!-- capitalized! -->
  [<note>...</note>]
  [<cost-per-use>NUMBER</cost-per-use>]
  [<cost-per-hour>NUMBER</cost-per-hour>]              <!-- Decimal -->
  [<units-available>NUMBER</units-available>]          <!-- always for Material (default 0); omitted for others when 1 -->
  [<efficiency>0.0..1.0</efficiency>]                  <!-- default 1.0 omitted -->
  [<email-address>...</email-address>]                 <!-- NOT <email> -->
  [<schedule>...</schedule>]                           <!-- per-resource calendar override -->
</resource>
```

### Resource types

| XML form | Meaning | omniJS enum | AppleScript enum |
|---|---|---|---|
| `Staff` | Person/team | `ResourceType.staff` | `person` |
| `Equipment` | Machinery/equipment | `ResourceType.equipment` | `equipment` |
| `Material` | Consumable | `ResourceType.material` | `material` |
| `Group` | Resource group container | `ResourceType.group` | `resource group` |
| `Project` | Reserved for `r-1` (root resource) | (auto-assigned) | (auto-assigned) |

**Note**: AppleScript uses `person` where omniJS/XML use `Staff`/`staff`. See `enums.md`.

### `<schedule>` (resource calendar)

```xml
<schedule>
  <schedule-day day-of-week="sunday|monday|...|saturday">
    <time-span start-time="SECONDS_FROM_MIDNIGHT" end-time="SECONDS_FROM_MIDNIGHT"/>
    <time-span .../>...                                <!-- multiple spans per day -->
  </schedule-day>
  ...
  <calendar name="Overtime" editable="yes" overtime="yes"/>
  <calendar name="Time Off" editable="yes" overtime="no"/>
</schedule>
```

When a resource has any non-default work hours, OmniPlan emits the FULL day's hours (default + custom) for that weekday. Default work hours: 8:00-12:00 (`28800-43200`) and 13:00-17:00 (`46800-61200`), Mon-Fri only.

The two system calendars `Overtime` and `Time Off` are auto-emitted whenever a `<schedule>` block exists. User-named calendars (e.g., synced from iCal) are read-only via omniJS and AppleScript.

To modify schedules: AppleScript `add work time s from "HH:MM" to "HH:MM" weekday N` and `subtract work time` (with `weekday` form Verified; `on date` form errors -10000 in 4.10.2).

## `<prerequisite-task>` element

Plain prerequisite (default kind = Finish-Start):

```xml
<prerequisite-task idref="tN"/>
```

With a non-default kind:

```xml
<prerequisite-task idref="tN" kind="FF|SS|SF"/>
```

With a lead-time (duration):

```xml
<prerequisite-task idref="tN" [kind="..."]>
  <lead-time is-percentage="false">SECONDS</lead-time>
</prerequisite-task>
```

With a lead-time (percentage):

```xml
<prerequisite-task idref="tN" [kind="..."]>
  <lead-time is-percentage="true">DECIMAL</lead-time>
</prerequisite-task>
```

`is-percentage="true"` value is the FRACTION (e.g., `0.25` for 25%). **The omniJS `dep.leadTimePercentage` setter takes INTEGER PERCENT (25 = 25%); AppleScript `set lead percentage to 0.25` takes the fraction.** This is a cross-surface convention mismatch — see `silent-corruption.md`.

Multi-prerequisite: emit multiple sibling `<prerequisite-task>` elements in insertion order.

`kind="..."` values are CASE-SENSITIVE uppercase only: `FF`, `SS`, `SF`. Lowercase or other values are silently stripped (file appears valid but dependencies become Finish-Start).

## `<assignment>` element

Default 1.0 units:

```xml
<assignment idref="rN"/>
```

Non-default units:

```xml
<assignment idref="rN" units="0.5"/>
```

`units="0"` is **silent corruption**: setting `unitsAssigned = 0` via omniJS REMOVES the assignment entirely. Do not emit `units="0"` in generated XML.

## Special tasks

- `t-1` is the **root task** (a group containing all top-level tasks). MUST exist; referenced via `<top-task idref="t-1"/>`.
- `t-2`, `t-3`, `t-4` are conventional prototype-task IDs (the templates OmniPlan offers in the UI). Optional.

The root task's body is a `<task type="group">` with `<child-task>` references for every top-level task. **Tasks not reachable via the `<top-task>` → `<child-task>` chain from `t-1` are silently DROPPED on save.**

## Special resources

- `r-1` is the **root resource** (always type `Project`, name conventionally `Project` but may be empty `<name/>`).
- `r-2`, `r-3` are conventional prototype-resource IDs.

## Date formats

Always ISO-8601 UTC with milliseconds: `2026-06-01T13:00:00.000Z`. Non-UTC offsets are accepted on input but normalized to UTC on save (Verified for `<start-no-earlier-than>`).
