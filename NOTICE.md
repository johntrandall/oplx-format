# NOTICE

This NOTICE accompanies the `LICENSE` file for [`oplx-format`](https://github.com/johntrandall/oplx-format).
The canonical license text (CC-BY-4.0) is in `LICENSE`; this file documents
trademark, scope, and methodology context that does not belong in the
license text itself.

## 1. Trademarks

**OmniPlan** and **The Omni Group** are trademarks of The Omni Group.
This specification uses those names solely to identify the product whose
behavior is being documented (nominative fair use, per *Toyota Motor Sales,
U.S.A., Inc. v. Tabari*, 610 F.3d 1171 (9th Cir. 2010)). This project is
**not affiliated with, sponsored by, or endorsed by** The Omni Group.

## 2. Format not subject to copyright

The OmniPlan `.oplx` file format itself is a functional system — a "method
of operation" excluded from copyright protection by **17 U.S.C. §102(b)**.
Element grammar, ordering rules, enum values, dependency-kind tokens, and
the cross-surface naming conventions described in this work are facts
about a publicly observable system and are not subject to copyright.

What IS copyrighted in this repository is the **prose, examples, diagrams,
and organizational structure** that describe and explain the format. Those
elements — the spec authors' original expression — are licensed
**CC-BY-4.0** as stated in `LICENSE`.

## 3. Methodology

This specification is derived from **empirical reverse-engineering of
publicly-observable behavior** of OmniPlan 4.10.2 (build 232.5.0, macOS
15.7.3). Methodology:

- OmniPlan was legally obtained through normal commercial channels.
- No NDA, employment relationship, or confidential disclosure was the
  source of any documented behavior.
- No Omni Group source code was accessed, decompiled, or relied upon.
- No DMCA §1201 anticircumvention measures were bypassed; documented
  behavior is observable through the application's normal usage and
  through the introspection facilities (omniJS console, AppleScript
  dictionary, saved-document XML inspection) the application itself
  provides.

For the broader legal context, see the [reverse-engineering and interop
resource doc](https://github.com/johntrandall/dev/blob/main/_resources/licensing/reverse-engineering-and-interop.md)
in the umbrella `dev` knowledge layer.

## 4. Vendor wins

Where this specification disagrees with actual OmniPlan behavior,
**OmniPlan wins.** This document describes what one careful observer
saw at one specific version; The Omni Group is the authoritative source
for the format and is free to change it. Treat this spec as a *useful
approximation that is most-likely-correct for the documented version*,
not as an authoritative standard.

If you find a discrepancy, please file an issue at
<https://github.com/johntrandall/oplx-format/issues> — corrections
welcome.

## 5. No endorsement

The Omni Group has not reviewed, validated, or endorsed this work. The
spec exists to support an interoperable third-party tool ecosystem; it
does not represent the vendor's view of their own format and should not
be cited as such.

## 6. Scope

This specification documents:

- The on-disk structure of `.oplx` documents (directory bundle and zip
  variant)
- Observable behavior of the omniJS scripting surface and the
  AppleScript dictionary in OmniPlan 4.10.2
- Cross-surface naming and enum mappings across XML / omniJS /
  AppleScript

It does NOT document:

- The Omni Group's source code, internal algorithms, or proprietary
  methods
- Behavior of OmniPlan versions other than 4.10.2 (where pre-release
  behavior is noted, that is marked explicitly per-section)
- Behavior under license-key states other than a fully-licensed copy

## 7. Vendor relationship

The Omni Group has been responsive to clarifying questions filed via
their official support channels. Two threads in May 2026 are
acknowledged in this repository's documentation:

- **RT #3107771 (2026-05-05).** Two omniJS gaps documented in
  `spec/omnijs.md` were confirmed by Omni Group support: there was no
  way to reparent a Task without changing its uniqueID, and there was
  no documented Decimal-to-Number accessor for cost values. The next
  release of OmniPlan adds a `parent` accessor and `move` method to
  both Task and Resource; test builds are available at
  <https://omnistaging.omnigroup.com/omniplan/> as of 2026-05-06. The
  Decimal point was clarified as by-design (`NSDecimalNumber`
  semantics; binary floats can't represent all decimal values
  exactly), with the canonical round-trip pattern being
  `Decimal.fromString("100.00").toString()`.
- **RT #3108330 (2026-05-05).** A licensing question about running
  OmniPlan inside ephemeral macOS VMs for automated testing of
  `oplx-tools` was answered with confirmation of the standard seat
  policy and a proactive upgrade to a three-seat team license to
  support that scenario.

These engagements are documented as factual paper trail; The Omni
Group's responses do not constitute endorsement of this spec, and the
"vendor wins" rule (§4) and non-affiliation disclaimer (§1, §5)
continue to apply.
