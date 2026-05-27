# AppleScript surface — annotations

OmniPlan's AppleScript dictionary (in `OmniPlan.app/Contents/Resources/OmniPlan.sdef`) declares 24 commands and 20+ classes. This file documents what works in OmniPlan 4.10.2, which commands are broken, and notable workarounds.

The full sdef dump is included in companion repos as a reference. Below is a working/broken matrix.

## Working commands

| Command | Signature | Notes |
|---|---|---|
| `open` | `open file` | Standard file open. **Caveat**: when 0 docs are open, OmniPlan shows a New Project chooser dialog that blocks AppleScript with `-609 Connection is invalid` until dismissed. |
| `close` | `close document [saving yes\|no\|ask]` | Closes a document. |
| `save` | `save document [in file]` | **No `as <type>` parameter** — cannot change format. Saves to current file in current format. To change format, see Workflow notes below. |
| `print` | `print document` | UI-gated by TCC. |
| `quit` | `quit` | Standard. |
| `count` | `count <class> in <container>` | Standard. |
| `make` | `make new <class> with properties {...}` | Standard. **Caveat**: `make new document` errors `-10000` after fresh OmniPlan launch with 0 docs; workaround is to open an existing bundle instead. |
| `delete`, `duplicate`, `move`, `exists` | Standard | All work as expected. |
| `evaluate javascript` | `evaluate javascript "..."` | Runs omniJS. **Cannot await Promises** — async results return as `[object Object]` and the operation may not complete. |
| `assign` | `assign <task> to <resource> [units N]` | Assignment with optional units. Default 1.0. |
| `depend` | `depend <task> upon <task> [type <kind>]` | Dependency creation. Kind enum: `finishstart \| finishfinish \| startstart \| startfinish` (lowercase, no `task` suffix). |
| `level` | `level <project>` | Resource leveling. |
| `lookup` | `lookup <project> value <V> in <KEY>` | Find first task by custom-data key/value. |
| `add work time` | `add work time <schedule> from "HH:MM" to "HH:MM" [weekday N \| on <date>]` | Modify working hours. **`weekday` form Verified; `on date` errors `-10000`** in 4.10.2. |
| `subtract work time` | (same params as `add work time`) | Inverse. Same `on date` issue. |
| `lookup` | `lookup <project> value <V> in <KEY>` | Find task by custom-data key. |
| `change mark` | `change mark <project> [from "name"]` | Sets the user attribution for subsequent change-sets. No visible XML output by itself. |
| `undo`, `redo` | `undo <document>`, `redo <document>` | Standard. |
| `export` | `export <document> to <file> as <type>` | Export to alternate formats: `CSV`, `MPP`, `MSPDI`, `ICAL`, `PNG`, `PDF`, `TIFF`, `JPEG`, `OmniOutliner v3`, `OmniGraffle`, `HTML Task List`, `HTML Resource List`, `HTML Full Report`. |

## Broken / partially-working commands

| Command | Status | Workaround |
|---|---|---|
| `baseline` | **Errors `-1708` or `-10000`** at runtime despite being in the dictionary. | Hand-edit XML: copy `Actual.xml` to `Baseline.xml` with new scenario id, add `<scenario>` entry to `__TOC.xml`. Spec: see `actual-xml.md`. |
| `subtract work time on date` | **Errors `-10000`** with text date strings AND with mutated `current date`. | Use `weekday N` form instead; OR hand-edit the resource `<schedule>` block to add a `<calendar>` event. |
| `make new document` (after 0-docs cold start) | Errors `-10000 "no windows"` | Use `open -a OmniPlan /path/to/X.oplx` (Launch Services) instead. |
| `export ... as "MSPDI"` | Errors `"could not be exported"` regardless of file extension | Use `as "MPP"` for MS Project interop (produces a real `.mpp` binary). |
| `export ... as "HTML Task List"` (and other HTML variants) | Triggers a "(null) template" dialog; selecting any option **CRASHES OmniPlan** | Avoid until the broken `OPHTMLTemplate*` preference is cleared. Use CSV export. |
| `fix violation with action "..."` | Untested in our cycles. | (open) |
| `set task type of <task> to hammock task` | **Errors `-10000` "AppleEvent handler failed"** despite `task type` being declared writable in `.sdef` (no `access="r"`). The `.sdef`-declared writability is documented-but-broken for the `hammock task` value. Verified 2026-05-27 by repeated direct probe against OmniPlan 4.10.2. | Hand-edit XML: emit `<type>hammock</type>` directly in `Actual.xml` (the only working programmatic path). |
| `make new task ... with properties {task type: hammock task}` | **Silently ignored** — call succeeds with no error, but resulting task has `task type` of `standard task`. Same documented-but-broken pattern as the post-creation setter above. Verified 2026-05-27. | Hand-edit XML as above. |

## Working classes

`application`, `document`, `window`, `project`, `scenario`, `task`, `milestone` (subclass of task), `resource`, `child task`, `child resource`, `assignment`, `prerequisite` (subclass of dependency), `dependent` (subclass of dependency), `dependency`, `violation`, `currency`, `schedule`, `attachment`, `custom data entry`, `style`, `attribute`, `named style`, `rich text` (with subdivisions: character, paragraph, word, attribute run), `file attachment`.

## Notable enum naming differences vs omniJS / XML

| Concept | AppleScript | omniJS | XML |
|---|---|---|---|
| Resource type | `person` | `staff` | `Staff` |
| Resource type | `resource group` | `group` | `Group` |
| Task type | `standard task` (with " task" suffix) | `task` | (omitted) |
| Task type | `milestone task` | `milestone` | `milestone` |
| Lead-time percentage value | fraction (0.25) | integer percent (25) | fraction (0.25) |

See `enums.md` for the complete cross-surface mapping.

## Idiomatic patterns

### Open a doc and run omniJS

```applescript
tell application "OmniPlan"
  open POSIX file "/path/to/doc.oplx"
  evaluate javascript "actual.rootTask.subtasks.length"
end tell
```

### Save under a new path (preserves format)

```applescript
tell application "OmniPlan"
  save (front document) in (POSIX file "/path/to/copy.oplx")
end tell
```

### Modify a resource's schedule

```applescript
tell application "OmniPlan"
  set s to schedule of (first resource of project of front document)
  add work time s from "18:00" to "20:00" weekday 3
end tell
```

### Look up a task by custom data

```applescript
tell project of front document
  set t to lookup it value "ABC-100" in "CostCenter"
  return name of t
end tell
```

### Set scheduling granularity

```applescript
tell project of front document
  set scheduling granularity to hourly scheduling
  -- emits <granularity>hours</granularity> in Actual.xml
end tell
```

## Workflow: save-as-zip from AppleScript

Direct save-as-zip is not supported by the `save` command (no `as <type>` parameter). Workflows:

1. **Open an existing zip-variant doc**: subsequent saves preserve the zip format. So if you have ANY zip-variant `.oplx`, opening it and saving over the top stays zip.
2. **Hand-zip after directory save**: save the doc as the directory bundle, then post-process with shell `zip` (see `actual-xml.md` for layout details).
3. **omniJS `makeFileWrapper` async path**: documented to work but JXA bridge can't await Promises; achievable via a registered Plug-in inside OmniPlan or via the `omniplan://localhost/omnijs-run?script=...` URL invocation.
