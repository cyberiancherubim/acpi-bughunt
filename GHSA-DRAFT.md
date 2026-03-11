## Summary

A Lenovo platform firmware build exposes ACPI namespace collisions during table load and namespace construction. The issue manifests as duplicate object definitions involving USB topology helper methods and port capability objects, producing ACPICA load failures such as `AE_ALREADY_EXISTS` during boot.

## Severity

Moderate; firmware correctness defect with potential downstream security relevance, but no demonstrated code execution or privilege escalation.

## Affected component

Platform firmware ACPI implementation; DSDT and SSDT namespace construction.

## Technical details

Decompiled ACPI source and boot logs indicate duplicate helper method and USB topology object creation under the XHCI root hub namespace. Observed failures include `AE_ALREADY_EXISTS` during namespace lookup and ACPI table load failure counts during boot.

## Impact

The operating system may lose correct USB port capability and physical topology metadata for affected ports. This can degrade device initialization fidelity, power behavior, and hardware identity semantics. No direct exploit chain has been demonstrated.

## Evidence

See `EVIDENCE.md` and `REPORT.md` for summarized observations; raw dumps were removed for privacy.

## Historical context

Lenovo has previously acknowledged ACPI firmware defects involving duplicate USB ACPI object definitions producing `AE_ALREADY_EXISTS` class failures. Upstream Linux and ACPICA bug history also documents this defect family across vendors and firmware generations.

## Mitigation

Install corrected firmware when available. Preserve ACPI dumps, decompiled tables, and kernel boot logs for vendor triage.

## Credit

Independent researcher (identity withheld for privacy)

## Data handling

All raw ACPI dumps, kernel logs, hardware inventories, and MSDM content were removed from the public repository to avoid exposing host-specific identifiers or keys. Only summarized findings remain; vendors can request private evidence if needed for validation.
