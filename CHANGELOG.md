# Changelog

All notable changes to this spec are documented here. See `VERSION` for the current spec version and the OmniPlan version it was verified against.

## [Unreleased]

### 2026-05-28

- `spec/actual-xml.md` — new `<attachment>` element documented under `<task>` (wire form + position in element-order + PyObjC code path for generating `<bookmarkData>`). Verified 2026-05-28 against OmniPlan 4.10.2 build 232.5.0 via manual GUI attach + hand-emit round-trip.
- `spec/coverage.md` — earlier 2026-05-27 row claiming "attachment cannot be added programmatically" expanded with the XML-emission path, the PyObjC bookmark recipe, and the silent-ignore gotcha. The scripting-surface claim still stands (AppleScript and omniJS remain non-functional); only the XML path works.
- `spec/silent-corruption.md` — new HIGH-severity entry `ATTACH-NO-BOOKMARK`: `<attachment>` without `<bookmarkData>` opens cleanly but `count attachments of task` returns 0.
- `examples/with-attachment.oplx` (+ sibling `examples/attachment-target.txt`) — 2-task fixture: `t1` has a working attachment with bookmark; `t2` has the same structure minus `<bookmarkData>` to demonstrate the silent-ignore failure mode side-by-side.
- `README.md` — `with-attachment.oplx` added to the Examples list.

### 2026-05-07

- `LICENSE` replaced with the canonical SPDX CC-BY-4.0 text so GitHub's Licensee detector recognizes the license correctly. Trademark / scope / methodology content from the previous LICENSE file migrated to a new sibling `NOTICE.md`. README License section updated accordingly.

## [0.1.0] — 2026-05-02

Initial public release. Verified against OmniPlan 4.10.2 on macOS 24 (Sequoia).

### Coverage
- File format: directory bundle and zip variants; minimum-viable doc structure; bundle/zip layout differences
- Tasks: all 4 types, all element forms, ordering rules, 3-pt estimation, completion, all 4 date constraints, locked dates
- Resources: all 4 types, full element form set, per-resource schedule overrides
- Dependencies: 4 kinds + Invalid sentinel, lead-time as duration and percentage, multi-prereq
- Assignments: units form rules
- Scenarios: granularity element, fixed-end-mode, hand-edit baseline workflow
- AppleScript: 24 commands annotated working/broken in 4.10.2
- omniJS: 35+ Task properties, Dependency/Assignment/Resource/Scenario/Schedule surfaces
- Silent-corruption catalog with severity tiers
- Cross-surface enum mapping (XML / omniJS / AppleScript)
- Internal property names from `__changelog.xml` and saved `<filter>` bplists

### Known gaps (see `spec/coverage.md`)
- HTML export crashes on stale (null) template path in 4.10.2
- MSPDI export errors with available file extensions
- `subtract work time on date` errors -10000 (use `weekday N` form)
- omniJS `task.split` doesn't persist
- Note rich-text with formatting (bold/color/alignment) — `<style>` block grammar not yet captured

### Acknowledgements
This spec is community-maintained reverse-engineering work. The Omni Group has not reviewed or endorsed it. Where this spec disagrees with future OmniPlan behavior, OmniPlan wins.
