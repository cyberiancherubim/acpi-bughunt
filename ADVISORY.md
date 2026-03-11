# ACPI namespace collisions in Lenovo firmware trigger `AE_ALREADY_EXISTS`, USB metadata conflicts, and GPU `_DSM` aborts

## Summary
ACPI tables for a Lenovo platform identifying as `LENOVO TC-O5T` define duplicate namespace objects. During OS boot the ACPI interpreter raises `AE_ALREADY_EXISTS` while creating root helper methods (`\GPLD`, `\GUPC`), USB root-hub metadata objects (`_UPC`, `_PLD` under `\_SB.PC00.XHCI.RHUB.HSxx/SSxx`), and a buffer field inside a GPU `_DSM` (`\_SB.PC00.PEG1.PEGP._DSM.USRG`). The issue is a firmware correctness defect; Linux merely surfaces it.

## Severity
**Medium** – correctness/reliability defect with potential downstream impact where policy relies on accurate ACPI metadata. No code execution or privilege escalation is demonstrated.

## Impact
- Loss or rejection of USB port capability/physical-location metadata.
- Abort of the GPU `_DSM` method due to repeated field creation.
- Boot log noise and potential device-initialization quirks; security impact currently indirect.

## Affected scope
- Observed on firmware reporting OEM string `LENOVO TC-O5T` and compiler ID `INTL 20200717`.
- Any firmware branch shipping the same DSDT/SSDT set may be affected.

## Evidence (summarized)
- Disassembly showed serialized helper methods `GPLD` and `GUPC` at root scope, referenced across RHUB devices `HS01`–`HS14` and `SS01`–`SS10`.
- Boot logs recorded `AE_ALREADY_EXISTS` when creating `_UPC/_PLD` under those ports and when creating `USRG` inside the PEGP `_DSM`, causing that method to abort.

## Reproduction (collect privately)
1. Capture boot logs and filter for `AE_ALREADY_EXISTS`, `_UPC`, `_PLD`, `_DSM`, or `USRG`.
2. Dump ACPI tables (`acpidump`; `acpixtract`).
3. Disassemble (`iasl -d dsdt.dat ssdt*.dat`) and confirm duplicate definitions or repeated `Create*Field` operations for the objects above.
4. Keep raw outputs private; do not publish MSDM data, serials, or hostnames.

## Mitigation
Update to firmware that removes duplicate definitions or guards runtime field creation. Preserve ACPI dumps and logs privately for vendor triage.

## Credit
Independent researcher (identity withheld for privacy).

## Data handling
All raw ACPI dumps, kernel logs, hardware inventories, and MSDM content have been removed from the public repository to protect host-specific identifiers. Summarized observations remain; vendors may request private evidence if required.
