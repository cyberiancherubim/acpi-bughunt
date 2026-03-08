## Timeline

### Prior art

- 2017 onward; upstream Linux and distro bug trackers document ACPI namespace lookup failures, duplicate `_UPC` definitions, and related ACPICA load issues across multiple vendors.
- 2018; Lenovo published a support notice acknowledging duplicate USB ACPI object definitions causing `AE_ALREADY_EXISTS` class errors on certain systems.

### Current finding

- 2026-03-07 to 2026-03-08; ACPI tables extracted from a Lenovo platform.
- DSDT and SSDTs decompiled and inspected.
- Duplicate namespace-relevant USB helper and topology patterns documented.
- Secure Boot prevented unsigned CHIPSEC kernel module loading, limiting deeper ring-0 validation but not ACPI namespace analysis.
