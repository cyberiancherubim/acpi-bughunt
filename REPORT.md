# Technical Report: ACPI Namespace Collisions (Sanitized)

## Executive summary
A Lenovo firmware build defining ACPI tables for platform string `LENOVO TC-O5T` exhibits duplicate namespace objects. During OS boot the ACPI interpreter raises `AE_ALREADY_EXISTS` while creating helper methods (`\GPLD`, `\GUPC`), USB root-hub metadata objects (`_UPC`, `_PLD` under `\_SB.PC00.XHCI.RHUB.HSxx/SSxx`), and a buffer field in a GPU `_DSM` method (`\_SB.PC00.PEG1.PEGP._DSM.USRG`). The behavior indicates a firmware correctness defect; no privilege escalation or memory-corruption primitive has been demonstrated.

## Evidence (summarized, raw data removed)
- Prior disassembly identified `Method (GPLD, 2, Serialized)` and `Method (GUPC, 2, Serialized)` at root scope, referenced throughout XHCI RHUB port devices.
- Boot logs recorded repeated `AE_ALREADY_EXISTS` errors for `_UPC` and `_PLD` objects across HSxx/SSxx ports and for `USRG` within the PEGP `_DSM`, causing method aborts.
- Table headers showed compiler ID `INTL 20200717` and OEM string `LENOVO TC-O5T`; raw headers and dumps have been purged.

## Impact assessment
- **Correctness/reliability:** Duplicate ACPI objects are rejected, potentially dropping USB port capability/physical-location metadata and aborting GPU-specific behavior.
- **Security relevance:** Currently indirect; loss of accurate metadata could affect policy enforcement that depends on ACPI-sourced topology, but no direct exploit chain is known.

## Redaction notes
All raw artifacts (ACPI dumps, kernel logs, hardware inventories, screenshots, MSDM content) were removed to protect host-specific identifiers and keys. Only high-level summaries remain. Researchers should keep full evidence privately when engaging with vendors.

## Reproduction guidance (execute privately)
1. Capture boot logs on the target platform and filter for `AE_ALREADY_EXISTS`.
2. Dump ACPI tables with `acpidump` and split using `acpixtract`.
3. Disassemble with `iasl -d dsdt.dat ssdt*.dat`.
4. Confirm duplicate definitions or repeated `Create*Field` operations for the objects listed above.

## Next steps for vendors or reviewers
- Validate table generation pipelines for duplicate definitions across DSDT/SSDT sets.
- Provide firmware updates that remove overlapping objects or make runtime methods idempotent.
- Request private access to the redacted evidence if additional confirmation is required.
