## Local evidence

### ACPI helper methods present in DSDT

- `dsdt.dsl:12646: Method (GPLD, 2, Serialized)`
- `dsdt.dsl:12682: Method (GUPC, 2, Serialized)`

### USB RHUB topology objects present

- `dsdt.dsl:12715: Device (HS01)`
- `dsdt.dsl:12936: Device (HS14)`
- `dsdt.dsl:12975: Device (SS01)`
- `dsdt.dsl:13164: Device (SS10)`

### Repeated helper invocations under RHUB

Examples:
- `dsdt.dsl:12720: Return (GUPC (One, Zero))`
- `dsdt.dsl:12725: Return (GPLD (One, One))`
- `dsdt.dsl:12805: Return (GUPC (Zero, 0xFF))`
- `dsdt.dsl:12810: Return (GPLD (Zero, Zero))`

### Additional SSDT references

- `ssdt4.dsl:188` through `ssdt4.dsl:217` reference `HS01` through `HS14` and `SS01` through `SS10`

## Runtime observations

Observed boot-time ACPI errors include:
- `ACPI Error: [GPLD] Namespace lookup failure, AE_ALREADY_EXISTS`
- `ACPI Exception: AE_ALREADY_EXISTS, During name lookup/catalog`
- `ACPI Error: 1 table load failures, ... successful`

## Validation limitations

Unsigned `chipsec.ko` could not be loaded under Secure Boot:
- `insmod: ERROR: could not insert module ... Key was rejected by service`

This did not affect ACPI table extraction or ASL-level namespace analysis.
