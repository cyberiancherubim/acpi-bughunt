# GitHub Security Advisory Draft for ACPI Namespace Collisions and Secure Boot Investigation Constraints

## Context and prior art

The failure mode you are seeing (“ACPI BIOS Error (bug)… AE_ALREADY_EXISTS”) is emitted by the ACPI interpreter when firmware-provided AML attempts to create an ACPI namespace object whose name already exists in the same scope. In the upstream ACPICA-derived exception catalogue used by the Linux kernel, **AE_ALREADY_EXISTS** is literally defined as an exception indicating an entity already exists. citeturn35view1

The class of collisions you surfaced (_UPC/_PLD under XHCI root hub ports, plus a buffer-field collision inside a PEGP _DSM) maps onto well-known ACPI integration pitfalls: OEMs split platform AML across a **DSDT** plus multiple **SSDT** overlays, and if those overlays are not mutually exclusive (or are loaded unintentionally together), duplicate object creation is a common outcome. This can range from “harmless log noise” to functional regressions, depending on whether the duplicates occur in optional metadata objects or in methods that drivers actively evaluate at runtime. citeturn12search1turn32view7

Your prompt also includes a Lenovo server advisory (circa 2018) describing duplicate _UPC definitions (“Some USB ports have the ACPI _UPC method defined in multiple locations… extra definitions are ignored”) and stating the messages could be ignored pending a future UEFI firmware update. That historical note matters as “prior art”: it demonstrates the same symptom pattern has existed in Lenovo firmware families before, and that vendors sometimes treat these as correctness/quality issues unless downstream impact is demonstrated.

## Technical analysis of the collisions

ACPI methods like **_UPC** (USB Port Capabilities) and **_PLD** (Physical Location of Device) are *standardised objects* used by operating systems to understand USB port properties. The ACPI specification describes **_UPC** as a package that communicates USB port capabilities (including whether a port is user-visible / connectable and connector characteristics), and **_PLD** as a structured description for physical location metadata. citeturn33view0turn33view2

Modern operating systems and device-management stacks can use these objects for experience and policy: for example, Microsoft’s guidance for USB port ACPI configuration explicitly references _UPC and _PLD as part of describing ports and their physical topology. citeturn33view1

When firmware defines **_UPC** or **_PLD** multiple times under the same device scope (or redefines the helper methods used to construct them), Linux’s ACPI interpreter will log **AE_ALREADY_EXISTS**, and the second definition is not installed. The severity depends on what the “winning” definition is and which definition gets discarded; the behaviour can be benign if the duplicates contain identical data, but it can be problematic if the discarded definition is the only correct one (e.g., external vs internal port classification). citeturn35view1turn33view2

Your logs show two distinct collision patterns (summarised below using the evidence you collected):

- **Global helper method collisions at root scope**: duplicated **\GPLD** and **\GUPC**. In your decompiled DSDT, you located these methods at `dsdt.dsl:12646` (Method GPLD) and `dsdt.dsl:12682` (Method GUPC), with many subsequent calls returning `GUPC(...)` / `GPLD(...)` inside the XHCI RHUB port devices. This strongly suggests an overlay table (SSDT) is also attempting to define those same global names, triggering the root-scope AE_ALREADY_EXISTS at boot.  
- **USB root hub port metadata collisions**: duplicated objects under `\_SB.PC00.XHCI.RHUB.HSxx` and `SSxx` devices, producing AE_ALREADY_EXISTS for `_UPC` and `_PLD` across multiple ports. Because _UPC/_PLD are standard objects expected at those device nodes, duplicates usually indicate AML that is “double declaring” identical objects (or two tables each declare them).  
- **GPU device‑specific method collision**: under `\_SB.PC00.PEG1.PEGP._DSM`, a repeated buffer-field creation named `USRG` triggers AE_ALREADY_EXISTS and causes method abort (“CreateBufferField failure… aborting method _DSM”). This is structurally different from a static Name/Method duplicate: it suggests AML that performs a **CreateField/CreateBufferField** with a fixed name on each invocation, instead of guarding against re-entry or redefinition.

The last category is the one that most plausibly *moves* this from “cosmetic” to “potentially impactful”, since `_DSM` is frequently evaluated by drivers, and aborting it can change runtime configuration behaviour for that device. The ACPI spec explicitly defines _DSM as a mechanism for device-specific functions that are outside standard ACPI objects, and it is common for vendors to route platform-specific policy through it. citeturn33view2turn33view0

## Evidence and reproducibility from your TC‑O5T platform

Your reproduced evidence bundle (kernel logs + AML disassembly) supports a “conference defensible” claim of a firmware namespace correctness defect, because it ties **runtime kernel exceptions** to **static AML proof** that the relevant objects exist and are referenced.

Key artefacts you already captured and (after sanitisation) should include in the advisory repo:

- **Boot log evidence**: repeated kernel messages of the form `ACPI BIOS Error (bug): Failure creating named object [...] AE_ALREADY_EXISTS`, including root-scope objects `\GPLD`, `\GUPC`, and per-port paths under `\_SB.PC00.XHCI.RHUB.HSxx` and `SSxx`, as well as the PEGP `_DSM.USRG` collision.
- **Table extraction and disassembly evidence**: `acpi_dump.dat`, `dsdt.dat`, `ssdt*.dat`, and `dsdt.dsl` (plus `ssdt*.dsl` where relevant). In your run, the DSDT header identifies the platform string `LENOVO TC-O5T` and the Intel compiler ID date in the AML header; include those headers in the report because they’re often what OEMs need to map to a firmware branch.
- **Minimal grep outputs** proving the presence and call-sites of GPLD/GUPC and the XHCI RHUB device topology. You already produced a compact grep transcript (`grep -RIn 'GPLD\|GUPC' ./*.dsl` and the HSxx/SSxx device listings), which is ideal for an advisory appendix because it lets reviewers confirm the namespace layout quickly without wading through multi‑MB AML.

A minimal, reproducible extraction pipeline (suitable for the advisory’s “Steps to reproduce”) is:

- capture boot log evidence (`dmesg` or `journalctl -k -b`)
- dump ACPI (`acpidump`)
- split tables (`acpixtract`)
- disassemble (`iasl -d …`)
- grep for the colliding symbols and scopes

These steps are directly grounded in standard ACPICA tooling and are therefore easy for Lenovo engineers (or independent researchers) to reproduce. citeturn33view2turn33view0

## Secure Boot and kernel lockdown constraints affecting firmware verification tooling

Your attempt to validate adjacent firmware security properties with CHIPSEC (e.g., BIOS write protection, Secure Boot variables) ran into an expected and *important* constraint: when Linux detects it booted under UEFI Secure Boot, it may enable **kernel lockdown** features that restrict low-level interfaces and require **signed kernel modules**. The `kernel_lockdown(7)` documentation describes lockdown as a mechanism to prevent tampering and information exposure, and explicitly notes that it restricts functionality such as loading unsigned modules and a number of low-level hardware interfaces. citeturn32view7turn18view2

The specific failure you hit—`insmod … chipsec.ko: Key was rejected by service`—is the canonical symptom of module signature enforcement failing when Secure Boot/lockdown is active. This exact error string is widely reported in Secure Boot module-signing contexts. citeturn37search4

CHIPSEC on Linux relies on a kernel component (commonly `chipsec.ko`) for privileged hardware access, so it sits directly in the “must be signed/trusted” path when Secure Boot policy requires module signing. CHIPSEC’s documentation and build guidance reflect that it uses OS-specific helpers/drivers to perform low-level operations. citeturn32view5

Because you explicitly do **not** want to disable Secure Boot (good), the standard approach is **MOK-based module signing**: generate a signing keypair, enrol the public key into the Machine Owner Key database via `mokutil`, and sign the module with the kernel’s `sign-file` tooling before insertion. Red Hat’s documentation for managing the UEFI Secure Boot and MOK workflow covers the enrolment mechanism and its purpose (adding owner keys so locally built modules can be trusted). citeturn36search6

This interaction is relevant to the advisory for two reasons:

- it explains why “deep inspection” with CHIPSEC is blocked in your baseline configuration, and therefore why your advisory evidence focuses on ACPI table correctness rather than flash/SMI-level security state; and  
- it highlights a practical, *secure* path for firmware/ACPI researchers to run low-level tools without weakening the boot chain of trust.

## Draft GitHub Security Advisory

Below is a **ready-to-paste** GitHub Security Advisory draft in Markdown. It is written to fit GitHub’s “ecosystem: Other / firmware” style advisories and to be compatible with a public repo that contains a sanitised evidence bundle.

```markdown
# Advisory: ACPI namespace collisions (AE_ALREADY_EXISTS) in Lenovo firmware ACPI tables cause USB port metadata and GPU _DSM method aborts

## Summary
A Lenovo platform firmware build for a system identifying as `LENOVO TC-O5T` exposes ACPI namespace collisions during Linux boot. The kernel ACPI interpreter logs repeated `AE_ALREADY_EXISTS` exceptions when loading ACPI tables, indicating duplicate object creation attempts.

Collisions are observed for:
- Root-scope helper methods `\GPLD` and `\GUPC`
- Multiple USB XHCI root hub ports under `\_SB.PC00.XHCI.RHUB.HSxx` / `SSxx` involving `_UPC` and `_PLD`
- A GPU device-specific method `_DSM` under `\_SB.PC00.PEG1.PEGP` where a buffer field `USRG` creation fails due to a name collision, causing `_DSM` to abort

## Affected systems
- Confirmed on: Lenovo system with ACPI DSDT header string `LENOVO TC-O5T` (see `artifacts/tables/dsdt.dat` header + `artifacts/dsl/dsdt.dsl`)
- Likely affected: any firmware branch that ships the same DSDT/SSDT set (to be confirmed by additional reports)

## Affected environments
- Observed on Linux (kernel boot log via `journalctl -k -b`), with ACPI interpreter rejecting duplicate namespace objects.
- The defect is firmware-resident (DSDT/SSDT AML) and may surface on other OSes depending on ACPI interpreter behaviour.

## Impact
Primary impact is correctness and reliability:
- Boot-time ACPI error spam (`AE_ALREADY_EXISTS`) and discarding of duplicate ACPI objects.
- Potential misreporting of USB port capabilities/physical location metadata where `_UPC`/`_PLD` duplicates are involved.
- GPU platform policy may be impacted because `_DSM` aborts due to a buffer-field collision; this can interfere with device initialisation and power-management behaviours.

Security impact is currently assessed as **low / indirect**:
- Some enterprise hardening relies on reliable port classification (internal vs external) to enforce USB policy. Incorrect or missing metadata could degrade such policy, depending on the policy implementation.

No working exploit is claimed in this advisory.

## Technical details
Linux reports errors similar to:
- `ACPI BIOS Error (bug): Failure creating named object [\GPLD], AE_ALREADY_EXISTS`
- `ACPI BIOS Error (bug): Failure creating named object [\GUPC], AE_ALREADY_EXISTS`
- `ACPI BIOS Error (bug): Failure creating named object [\_SB.PC00.XHCI.RHUB.HS01._UPC], AE_ALREADY_EXISTS`
- `ACPI BIOS Error (bug): Failure creating named object [\_SB.PC00.PEG1.PEGP._DSM.USRG], AE_ALREADY_EXISTS`
  followed by `_DSM` abort

Decompiled AML (`dsdt.dsl`) shows `Method (GPLD, 2, Serialized)` and `Method (GUPC, 2, Serialized)` defined at root scope, and a large XHCI root hub topology containing HSxx/SSxx devices that reference these methods.

## Steps to reproduce
1. Boot Linux normally on the target platform.
2. Capture boot log evidence:
   - `journalctl -k -b | grep -Ei 'ACPI BIOS Error|ACPI Error|AE_ALREADY_EXISTS|_UPC|_PLD|PEGP|_DSM|USRG'`
3. Dump ACPI tables:
   - `sudo acpidump > acpi_dump.dat`
   - `acpixtract -a acpi_dump.dat`
4. Disassemble:
   - `iasl -d dsdt.dat ssdt*.dat`
5. Confirm duplicated symbols and topology:
   - `grep -RIn 'Method \\(GPLD|Method \\(GUPC' dsdt.dsl`
   - `grep -RIn 'Scope \\(_SB\\.PC00\\.XHCI\\.RHUB' dsdt.dsl`
   - `grep -RIn 'Device \\(HS[0-9][0-9]\\)|Device \\(SS[0-9][0-9]\\)' dsdt.dsl`

## Mitigations / workarounds
- Firmware update: install an updated UEFI/BIOS release when available from Lenovo that removes duplicate object creation from ACPI tables.
- Operational: if there is no functional impact, errors may be treated as noise; however, note the `_DSM` abort may be impactful for GPU policy.
- Avoid “DSDT override” approaches on Secure Boot systems unless you understand the security implications; kernel lockdown may restrict ACPI table override in Secure Boot mode.

## Secure Boot note (diagnostic tooling)
CHIPSEC and similar low-level tools may be blocked by Secure Boot module signing policy (`Key was rejected by service` when inserting `chipsec.ko`). If diagnostics are required without disabling Secure Boot, use a Machine Owner Key (MOK) approach to sign and trust the module.

## Credits
Reported by: @cyberian-cherubim

## Timeline
- 2026-03-07 / 2026-03-08: Observed and extracted evidence; advisory drafted.

## References
- See `docs/` and `artifacts/` in this repository for logs and extracted tables (sanitised).
```

This draft is intentionally conservative: it **does not** claim a confirmed security exploit path, but it does clearly document the correctness defect, the `_DSM` runtime abort (most likely to be functional), and the downstream security-policy risk (“degrades USB port classification”) as a conditional risk rather than a proven exploit.

For the few spec-dependent statements inside the draft (what _UPC/_PLD are, what AE_ALREADY_EXISTS means, why Secure Boot blocks unsigned modules), the authoritative references are: ACPI spec sections for _UPC/_PLD, the Linux ACPI exception catalogue (“AE_ALREADY_EXISTS”), and kernel lockdown documentation describing Secure Boot-driven restrictions. citeturn33view2turn35view1turn32view7

## Publication checklist and repository layout

A public repo is a great idea here, but firmware evidence has **two recurring privacy traps**:

- **MSDM table disclosure**: The ACPI MSDM table is used by OEM Activation and can contain a Windows digital product key. Microsoft’s OA3 tooling guidance explicitly references the MSDM table and its role in injected OEM keys; treat vbios/ACPI artefacts as sensitive until reviewed. citeturn39search8turn39search7  
- **Platform identifiers**: ACPI (and adjacent SMBIOS) can include serial numbers, UUIDs, asset tags, and licensing markers.

A repo layout that works well for both peer review **and** OEM triage (while minimising accidental disclosure) is:

- `README.md` — plain-English overview, scope, and current status  
- `SECURITY.md` — disclosure intent (e.g., “will coordinate with Lenovo PSIRT”), contact  
- `docs/advisory.md` — the GHSA text (copy of the above)  
- `docs/technical-analysis.md` — deeper write-up, screenshots of the key log lines, and a map of colliding objects  
- `artifacts/logs/` — *sanitised* kernel logs (remove hostnames, serials, MACs if present)  
- `artifacts/tables/` — `dsdt.dat`, `ssdt*.dat` (exclude `msdm.dat`), plus a `SHA256SUMS` file  
- `artifacts/dsl/` — `dsdt.dsl`, `ssdt*.dsl` containing the relevant scopes  
- `scripts/` — redaction helpers (e.g., delete MSDM, mask serial patterns), plus `repro.sh` to regenerate the evidence bundle

Finally, the “secure boot will not be disabled” stance is compatible with doing deeper platform checks: if you later want to validate BIOS write protections or Secure Boot variable integrity on the same system, pursue MOK-based signing rather than disabling Secure Boot, because lockdown is specifically designed to prohibit untrusted ring‑0 code in Secure Boot mode. citeturn32view7turn36search6turn37search4