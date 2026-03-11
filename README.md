# ACPI Namespace Collision Report (Sanitized)

This repository provides a privacy-safe summary of ACPI namespace collisions observed on a Lenovo platform. All system-identifying artifacts (hostnames, UUIDs, MSDM table contents, kernel logs, raw ACPI tables, screenshots) have been removed.

## What remains
- `ADVISORY.md` – concise statement of the defect and its impact.
- `EVIDENCE.md` – summarized observations from prior disassembly and logs (no raw dumps).
- `GHSA-DRAFT.md` – sanitized GitHub Security Advisory draft.
- `TIMELINE.md` – disclosure and analysis milestones.
- `LIMITATIONS.md` – scope boundaries and caveats.
- `REPORT.md` – structured technical report in third-person narrative.

## Key finding
Firmware-level duplication of ACPI namespace objects (e.g., `\GPLD`, `\GUPC`, USB RHUB `_UPC/_PLD`, and a GPU `_DSM` field) triggers `AE_ALREADY_EXISTS` errors during OS boot. The evidence points to a correctness and reliability defect; no exploit primitive has been demonstrated.

## Privacy and redaction notes
- Raw dumps, hardware inventories, and verbose boot logs have been purged to avoid leaking serials, product keys, hostnames, or HWIDs.
- References to line numbers or table names are retained only in summarized form.
- Contributors should keep any full evidence sets private and avoid reintroducing sensitive artifacts into this repository.

## Independent reproduction (keep private)
1. Collect ACPI tables on the target platform (`acpidump`, `acpixtract`).
2. Disassemble with `iasl -d dsdt.dat ssdt*.dat`.
3. Inspect boot logs for `AE_ALREADY_EXISTS` and grep disassembly for duplicate `_UPC/_PLD` objects or repeated `Create*Field` operations.
4. Store raw outputs privately; share only sanitized excerpts when reporting.
