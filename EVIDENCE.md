## Evidence summary (raw artifacts removed)

- Prior disassembly identified serialized helper methods `GPLD` and `GUPC` at root scope, invoked by XHCI RHUB USB devices.
- The RHUB topology included devices `HS01`–`HS14` and `SS01`–`SS10`, each returning `_UPC` and `_PLD` data constructed via the helper methods.
- Boot-time interpreter output (now redacted) reported `AE_ALREADY_EXISTS` while creating those helper methods and the per-port `_UPC/_PLD` objects, indicating duplicate definitions across the DSDT/SSDT set.
- A device-specific method `_DSM` under `\_SB.PC00.PEG1.PEGP` attempted to create a buffer field `USRG` on each invocation; repeated creation triggered `AE_ALREADY_EXISTS` and aborted the method.

## Runtime observations (summarized)

- `AE_ALREADY_EXISTS` for `\GPLD` and `\GUPC` during ACPI table load.
- `AE_ALREADY_EXISTS` for `_UPC` and `_PLD` under multiple RHUB port paths.
- `CreateBufferField` collision for `USRG` inside `_DSM`, followed by method abort.

## Redaction notice

All underlying ACPI dumps, kernel logs, and disassembly outputs have been removed to prevent leakage of host identifiers, MSDM data, serials, or HWIDs. The above bullets describe the retained conclusions without exposing raw artefacts.

## Validation limitations

Unsigned `chipsec.ko` could not be loaded under Secure Boot; deeper ring-0 checks were not performed in the sanitized dataset.
