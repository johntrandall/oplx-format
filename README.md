# oplx-format

**Community-maintained file-format specification for OmniPlan `.oplx` documents.**

> **Naming note**: `oplx` is the file extension OmniPlan uses (`.oplx`). This project is **not** related to Yamaha OPL audio synthesis chips (OPL2/OPL3/OPL4) which share a similar string in some retro-audio communities.

OmniPlan is The Omni Group's project planning app. Its `.oplx` document format is a directory bundle (or alternatively a flat zip) containing XML files that describe scenarios, tasks, resources, schedules, and dependencies. Omni publishes excellent docs for [Omni Automation](https://omni-automation.com/omniplan/) (the omniJS scripting layer) and the AppleScript dictionary, but **does not publish a specification of the file format itself**. This repo fills that gap.

## Status

- **Verified against**: OmniPlan 4.10.2 (macOS, build 2026-05-01)
- **Format version**: `file-format-version="3"` (per `__TOC.xml`)
- **Coverage**: substantially complete for core entities (tasks, resources, dependencies, assignments, schedules, baselines). Some edge cases still tagged Observed-not-Verified — see `spec/coverage.md`.
- **Pre-release builds (2026-05-06)**: Omni Group test builds at <https://omnistaging.omnigroup.com/omniplan/> expand the omniJS surface — `parent` accessor and `move` method on both Task and Resource (per Omni Group RT #3107771). Verified-against remains 4.10.2; pre-release additions are documented in `spec/omnijs.md` and tracked in `spec/coverage.md` as Inferred until the test builds are run end-to-end.
- **Maintainership**: best-effort, single-version snapshot. Not affiliated with The Omni Group.

## Audience

You will find this useful if you:

- Want to **generate `.oplx` files programmatically** (without invoking OmniPlan)
- Want to **parse `.oplx` files** for downstream processing (BI dashboards, integrations, archival)
- Are writing tools that interoperate with OmniPlan
- Want to understand silent-corruption modes that may bite hand-edited or generated files

## Ecosystem

This spec has a Python reference implementation and a sibling MCP server:

| Repo | What it is | When to use |
|---|---|---|
| 📖 [**oplx-format**](https://github.com/johntrandall/oplx-format) (this repo) | The file-format **specification** (CC-BY-4.0) | Read this if you're writing any `.oplx` tool in any language |
| 🐍 [**oplx-tools**](https://github.com/johntrandall/oplx-tools) | **Python**: generate / lint / parse — **no OmniPlan needed** | Headless `.oplx` workflows: CI/CD, batch generation, agent file-write |
| 🤖 [**omniplan-mcp**](https://github.com/johntrandall/omniplan-mcp) | **MCP server** for live OmniPlan automation (requires OmniPlan running) | Conversational task management with Claude — "schedule a task tomorrow at 2pm" |
| 📦 [**lash**](https://github.com/johntrandall/lash) | The installer used to wire `oplx-tools`'s lint hook into Claude Code | One-shot setup for the agent integration |

The MCP and `oplx-tools` are complementary, not competitive: MCP for live runtime queries; `oplx-tools` for headless file mutation. Both implement this spec.

## Quick orientation

An `.oplx` document has two equivalent forms:

| Form | UTI | Layout |
|---|---|---|
| **Directory bundle** (default) | `com.omnigroup.omniplan2.planfile` | A folder containing `__TOC.xml`, `__changelog.xml`, `Actual.xml`, `QuickLook/Preview.png` |
| **Zip variant** | `com.omnigroup.omniplan2.planfile-zip` | A zip file containing the same XML files at the top level + `Preview.png` (NO `QuickLook/` subdir) |

OmniPlan accepts both interchangeably. Generators producing the zip form yield documents 3-10× smaller for the same content.

The `.oplx` extension covers both — OmniPlan introspects the contents, not the extension.

## Specification index

- [`spec/overview.md`](spec/overview.md) — bundle layout, scenarios, prototypes, and how it all fits together
- [`spec/actual-xml.md`](spec/actual-xml.md) — `Actual.xml` (and other scenario files): tasks, resources, dependencies
- [`spec/toc-xml.md`](spec/toc-xml.md) — `__TOC.xml`: project-level metadata, window state, filters, styling
- [`spec/changelog-xml.md`](spec/changelog-xml.md) — `__changelog.xml`: operational mutation log
- [`spec/enums.md`](spec/enums.md) — every enum across XML / omniJS / AppleScript with their cross-surface mappings
- [`spec/silent-corruption.md`](spec/silent-corruption.md) — patterns that fail silently (uppercase `<type>` rejection, units=0 deletion, etc.)
- [`spec/applescript.md`](spec/applescript.md) — annotations on the AppleScript dictionary (which commands work, which are broken in 4.10.2)
- [`spec/omnijs.md`](spec/omnijs.md) — omniJS API surface (Task, Resource, Dependency, Assignment, Scenario, Schedule)
- [`spec/coverage.md`](spec/coverage.md) — what's Verified, what's Observed, what's still open

## Examples

- [`examples/minimum-viable.oplx`](examples/minimum-viable.oplx) — smallest valid `.oplx` (1.1 KB zip, 3 hand-coded XML files)
- [`examples/with-baseline.oplx`](examples/with-baseline.oplx) — multi-scenario doc with one baseline
- [`examples/with-hammock.oplx`](examples/with-hammock.oplx) — hammock task pattern (the only way to create one — the omniJS API silently refuses)

## Tooling

A separate companion repo, `oplx-tools`, provides Python-based reference implementations:

- A reference generator that produces minimum-viable `.oplx` from a YAML description
- A linter that validates a hand-written or generated `.oplx` against this spec
- A reader that extracts data from `.oplx` for downstream tools

## Caveats

- This spec is **derived from empirical reverse-engineering**, not from Omni Group internal documents. The Omni Group has not reviewed or endorsed it. Where this spec disagrees with future OmniPlan behavior, OmniPlan wins.
- The format is stable across patch versions of OmniPlan 4.10.x in our testing, but a major-version bump (4.11+, 5.x) could invalidate parts.
- Some elements documented here have **silent-corruption modes** — patterns that OmniPlan accepts then mis-interprets. Hand-editing without a linter is risky. See `spec/silent-corruption.md`.

## Verification environment

Findings are tested against OmniPlan 4.10.2. Two environments:

- **Host:** the contributor's working Mac (macOS 15.x), used for the bulk of menu-state probes and save-roundtrip diffs.
- **VM cross-check (when needed):** a **persistent** Tart VM named `oplx-spec-verify` lives on `susanbones`, cloned from `umbridge.tail486ac0.ts.net:5051/tart/macos-15.7-l3-omni-suite:v2-tcc-granted-20260513`. It has all 4 Omni apps installed and licensed (OmniPlan, OmniGraffle, OmniFocus, OmniOutliner) and has the additional TCC Accessibility + Automation grants for `sshd-keygen-wrapper` that the base image doesn't yet bake. Lifecycle:

  ```
  tart-vm status                   # check if running
  tart-vm start oplx-spec-verify   # resume (or initial clone — see below)
  tart-vm stop  oplx-spec-verify   # park; preserves the manual TCC grants
  tart-vm ssh   oplx-spec-verify   # use it
  # DO NOT tart-vm destroy oplx-spec-verify  ← loses the manual TCC grants
  ```

  **Re-cloning from scratch (only if the VM is lost):**
  ```
  tart-vm start oplx-spec-verify \
    --from umbridge.tail486ac0.ts.net:5051/tart/macos-15.7-l3-omni-suite:v2-tcc-granted-20260513 \
    --as local-runtime
  # Then VNC in and add /usr/libexec/sshd-keygen-wrapper to:
  #   System Settings → Privacy & Security → Accessibility
  #   System Settings → Privacy & Security → Automation (OmniPlan + System Events)
  open vnc://admin:admin@$(tart-vm ip oplx-spec-verify)
  ```

  **Two real blockers encountered (2026-05-27) and how they were resolved:**

  1. **Orchard schedules VMs headless.** Document-level AppleScript commands (`count documents`, `save front document`) hung with `-1712 AppleEvent timed out` even with `with timeout of 120 seconds`. Application-level commands (`get version`) worked. **Fix:** bypass Orchard and use the `tart-vm` wrapper directly (`tart-vm start NAME --from IMAGE --as ephemeral`). It runs with the Aqua session on (per `tart-vm-management` skill, "tart-vm has no `--gui` flag. Aqua session is on by default.").

  2. **TCC Accessibility not granted for sshd-keygen-wrapper.** Even with Aqua running, all System Events / AppleEvents commands targeting OmniPlan documents timed out. Inspection of `/Library/Application Support/com.apple.TCC/TCC.db` showed `sshd-keygen-wrapper` had `kTCCServiceSystemPolicyAllFiles = 2` (Full Disk Access) but `kTCCServiceAccessibility = 0` (DENIED). The `v2-tcc-granted` image granted Full Disk Access but NOT Accessibility for SSH-driven scripts. **Fix:** VNC into the VM (`open vnc://admin:admin@<vm-ip>`) and add `/usr/libexec/sshd-keygen-wrapper` to System Settings → Privacy & Security → Accessibility. After the grant, `count documents` returned correctly and `save front document` completed cleanly.

  After both blockers were resolved, the save-roundtrip tests (`<numbering-style>flat</numbering-style>`, `<user-data>` value types) ran successfully. The `<user-data>` test result OVERTURNED an earlier host-only "all 4 types Verified" claim — see the corrected entry in `spec/coverage.md`. **Methodology lesson:** if your host test relies on grepping the file after a save, also check that the save actually ran (e.g. byte-diff against the pre-open state, or look for first-save normalization markers like changed `<gantt-view>` dimensions). A silently-rejected file looks byte-identical to the input, which can falsely "verify" content that was never actually saved.

## License

Specification text is licensed CC-BY-4.0 (see [LICENSE](LICENSE)). See [NOTICE.md](NOTICE.md) for trademark and scope notes — OmniPlan is a trademark of The Omni Group; this spec is empirical and not affiliated with or endorsed by Omni.

## Contributing

Pull requests welcome for:
- Verification of Observed claims in `spec/coverage.md`
- New version snapshots (test against OmniPlan 4.11+ when released)
- Edge cases not yet covered

Please include evidence: reproducible test, before/after XML, OmniPlan version.
