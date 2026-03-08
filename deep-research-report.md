# ACPI Namespace Collision Finding on a Lenovo Platform

## Executive summary

Your boot logs show repeated **ACPICA** loader/runtime exceptions of the form **`AE_ALREADY_EXISTS`** while creating ACPI namespace objects such as `\GPLD`, `\GUPC`, USB root-hub port objects like `\_SB.PC00.XHCI.RHUB.HSxx._UPC/_PLD`, and a GPU-related object `\_SB.PC00.PEG1.PEGP._DSM.USRG`. These messages are consistent with **ACPI namespace collisions**: AML attempted to create a named object that already exists in the namespace. The ACPICA exception string for `AE_ALREADY_EXISTS` (“an entity already exists”) matches this interpretation. citeturn3search1turn31view1

Two distinct bug patterns appear likely from your artefacts:

- **Table-load collisions** (during ACPI table install): the same *NameString* is defined in more than one definition block (e.g., an SSDT redefines an object already defined in the DSDT or another SSDT). This is exactly the kind of situation Linux ACPI developers describe when duplicated AML is executed/installed and namespace object conflicts lead to `AE_ALREADY_EXISTS`. citeturn32search0turn31view1  
- **Runtime idempotency bug inside a method** (during method execution): `Create*Field` ops inside `_DSM` create a **named** object (`USRG`) that can collide on subsequent invocations if the method does not guard against re-creation. The ACPI-defined `_DSM` method is expected to be callable by the OS multiple times, so a method that cannot safely handle re-entry will fail on later calls. citeturn27search2

The net effect is typically **firmware correctness/reliability impact** (missing/aborted device objects or device-specific methods), with potential downstream knock-on effects in kernel drivers that rely on those objects. Whether it is *security-relevant* depends on demonstrable consequences (e.g., unintended exposure of internal ports as user-accessible, misreported device location/attributes, or a chain leading to a kernel vulnerability). citeturn24search1turn31view1

## Background and technical context

### How DSDT/SSDTs form one namespace

On ACPI platforms, the firmware provides a set of ACPI System Description Tables. The OS loads the DSDT plus (often) multiple SSDTs into a **single hierarchical namespace**. In Linux documentation: “The data block of the DSDT along with the contents of SSDTs represents… the ACPI namespace.” citeturn31view1

Because everything merges into **one namespace**, **names must be unique within their scope**. The Linux ACPI documentation explicitly describes namespace objects as “identified by names and paths” and lays out naming conventions. A duplicate definition in the same scope is a fundamental conflict. citeturn31view1

### What `AE_ALREADY_EXISTS` means

In ACPICA’s exception definitions, `AE_ALREADY_EXISTS` corresponds to a condition where “an entity already exists.” This is the canonical signal you get when AML/ACPICA tries to create a namespace node (Name/Device/Method/Field, etc.) that is already present. citeturn3search1

Linux ACPI maintainers have discussed real-world cases where duplicated AML execution/installation produces “namespace object conflicts” and yields `AE_ALREADY_EXISTS`. citeturn32search0

### Why `_UPC` and `_PLD` matter for USB port modelling

For USB ports represented in ACPI, `_UPC` (“USB Port Capabilities”) and `_PLD` (“Physical Location of Device”) are used to describe properties such as whether a port is user visible, connectable, and how it maps physically. The ACPI specification defines `_PLD` as a mechanism to convey physical location information to the OS. citeturn22view2  
The ACPI specification also defines `_UPC` for USB port capabilities modelling. citeturn24search0  
Microsoft’s ACPI guidance highlights how `_UPC` and `_PLD` are used to describe USB ports and their properties to the OS. citeturn24search1

So, collisions involving `XHCI.RHUB.HSxx._UPC/_PLD` are not just “noise”: they can break or degrade USB port metadata and, depending on which definitions “win” or are aborted, can change how ports are presented to the OS. citeturn24search0turn24search1

### `_DSM` and why a `CreateWordField` collision is plausible

`_DSM` (“Device Specific Method”) is a standard ACPI mechanism for vendor/device-specific functions identified by UUID/revision/function index. The ACPI spec defines its interface and expected usage patterns. citeturn27search2  
If AML inside `_DSM` uses `CreateWordField(..., USRG)` unconditionally, then *subsequent invocations* can attempt to create `USRG` again and fail with `AE_ALREADY_EXISTS`, matching what your logs show.

## Findings and evidence framing

### Observed error signatures and what they imply

From your captured kernel output (as shared in your session), representative messages include:

- `ACPI BIOS Error (bug): Failure creating named object [\GPLD], AE_ALREADY_EXISTS`
- `ACPI BIOS Error (bug): Failure creating named object [\GUPC], AE_ALREADY_EXISTS`
- `ACPI BIOS Error (bug): Failure creating named object [\_SB.PC00.XHCI.RHUB.HS01._UPC], AE_ALREADY_EXISTS`
- `ACPI BIOS Error (bug): Failure creating named object [\_SB.PC00.PEG1.PEGP._DSM.USRG], AE_ALREADY_EXISTS`

Interpreting those messages in a standards-aligned way:

- Any failure “creating named object” with `AE_ALREADY_EXISTS` is a strong indicator of **duplicate namespace entries**—either defined in multiple tables, or created repeatedly at runtime. citeturn3search1turn32search0  
- The presence of fully-qualified paths like `\_SB.PC00.XHCI.RHUB.HS01._UPC` indicates the collision is not generic; it is tied to specific device nodes and objects in the namespace. This aligns with the ACPI namespace model described in Linux documentation. citeturn31view1

### Likely collision classes in your case

Based on patterns you surfaced (DSDT/SSDT disassembly work + boot-time loader errors), an evidence-driven write-up can separate the issue into:

- **Class A: table-load collisions** for `\GPLD`, `\GUPC`, and many `XHCI.RHUB.HSxx/SSxx` objects. These are typically produced when an SSDT defines objects already defined by the DSDT or another SSDT installed earlier. Linux ACPI discussions document how running/including SSDTs twice (or installing overlapping objects) produces `AE_ALREADY_EXISTS`. citeturn32search0turn31view1  
- **Class B: runtime method non-idempotency** inside `_DSM` for the GPU path (the `USRG` field creation). The `_DSM` contract implies repeated calls can occur, so AML should be robust to re-entry. citeturn27search2

This split matters for remediation and for how you communicate severity.

### CHIPSEC driver and Secure Boot side finding

You also observed that building/loading `chipsec.ko` under Secure Boot failed with “Key was rejected by service.” That message is consistent with kernel module signature enforcement: a kernel configured to require signed modules (often with Secure Boot/lockdown) rejects unsigned modules. The Linux kernel documentation covers module signing and signature enforcement mechanisms. citeturn13search1turn12search3  
This is operationally relevant to your firmware research workflow, but it is distinct from the ACPI namespace collision itself.

## Reproduction methodology you can publish

### Data collection pipeline

For a publishable, reviewer-friendly reproduction, anchor everything to standard tools:

- **Collect runtime evidence**: `journalctl -k -b` filtered to ACPI messages (you already did this).  
- **Extract ACPI tables**: `acpidump` + `acpixtract` to split tables. Tool documentation describes `acpixtract` as an ACPI table extraction utility. citeturn28search0turn28search4  
- **Disassemble AML to DSL**: `iasl -d` and (critically) use `-e` to include all needed SSDTs so externals resolve more cleanly. The `iasl` tool supports including external tables during disassembly. citeturn9search0turn31view1

A minimal, reproducible sequence (suitable for `scripts/repro.sh` in your repo) can look like this:

```bash
#!/usr/bin/env bash
set -euo pipefail

OUT="${1:-artifacts}"
mkdir -p "$OUT/tables" "$OUT/dsl" "$OUT/logs"

# 1) Capture kernel ACPI diagnostics
journalctl -k -b > "$OUT/logs/journal-kernel-boot.txt"
journalctl -k -b | grep -E 'ACPI BIOS Error|ACPI Error|AE_ALREADY_EXISTS|XHCI|RHUB|_UPC|_PLD|_DSM|PEGP' \
  > "$OUT/logs/journal-acpi-interesting.txt" || true

# 2) Dump and extract ACPI tables (requires acpica-tools)
acpidump -o "$OUT/tables/acpidump.dat"
acpixtract -a "$OUT/tables/acpidump.dat" -o "$OUT/tables"

# 3) Disassemble DSDT + all SSDTs together (better external resolution)
# Adjust list depending on filenames produced by acpixtract.
pushd "$OUT/tables" >/dev/null
iasl -e ssdt*.dat -d dsdt.dat ssdt*.dat
mv ./*.dsl ../dsl/ || true
popd >/dev/null
```

### Proof of collision in DSL

In your “analysis” section, you want to show **two things** for each collided object:

- The object is defined more than once across definition blocks (DSDT/SSDTs), or created repeatedly at runtime.
- The second definition likely triggers `AE_ALREADY_EXISTS` during load/execute.

Linux ACPI documentation provides the conceptual baseline (single namespace from DSDT+SSDT). citeturn31view1  
ACPICA’s `AE_ALREADY_EXISTS` definition provides the semantic. citeturn3search1

### Data sanitisation before publication

Be careful: ACPI table dumps can contain sensitive data. In particular, the `MSDM` ACPI table is commonly used to store an embedded Windows OEM product key on many systems. Guidance from embedded/firmware vendors notes the MSDM table stores the Windows product key. citeturn13search3  

For a public repo, you should:

- Exclude `msdm.dat` (and any derived disassembly of it) unless you have redacted it.
- Redact UUIDs, disk identifiers, hostnames, and serials from logs (your kernel command line often contains LUKS UUIDs, volume names, etc.).
- Consider publishing hashes/checksums of full raw tables and providing redacted excerpts plus diffs/patches that demonstrate collisions without exposing keys.

## Security and reliability impact assessment

### Realistic impact classes

Most `AE_ALREADY_EXISTS` ACPI errors should be treated first as **firmware correctness** problems; the interpreter is rejecting or aborting parts of AML installation or method execution. That can lead to:

- Missing `_UPC/_PLD` metadata for USB ports (affecting how ports are classified as internal/external or user-visible). The ACPI definitions of `_UPC` and `_PLD` make clear these objects influence port/device description to the OS. citeturn24search0turn22view2  
- Broken device-specific behaviour if a `_DSM` method aborts after a field-creation error, which can degrade device driver feature discovery. `_DSM` is explicitly designed for device-specific functions. citeturn27search2

### When it becomes security-relevant

To justify PSIRT/security-advisory treatment (rather than a “BIOS bug” report), you would need to demonstrate a measurable security consequence, for example:

- A port intended to be internal is reported as externally accessible (or vice versa), changing policy decisions.
- A collision causes a method to abort in a way that provokes a kernel NULL dereference or out-of-bounds access in a driver (that would then be a kernel bug, but firmware-triggered).
- A reproducible path where the firmware bug can be used to bypass a mitigation or expose otherwise restricted capabilities.

Your repo can be structured to support this escalation if you discover it, but the initial write-up should keep claims conservative and evidence-based.

## Repository blueprint and publication artefacts

### Suggested repo layout

A clean public repo structure that supports peer review and vendor triage:

```text
lenovo-acpi-ae-already-exists/
  README.md
  LICENSE
  SECURITY.md
  CHANGELOG.md

  docs/
    paper.md
    conference-writeup.md
    glossary.md
    responsible-disclosure-timeline.md

  artifacts/
    logs/
      journal-kernel-boot.redacted.txt
      journal-acpi-interesting.redacted.txt
    tables/
      acpidump.redacted.dat
      # NEVER publish msdm.dat unless you have fully redacted it
    dsl/
      dsdt.dsl
      ssdt*.dsl

  analysis/
    collisions.md
    tables-and-scope-map.md
    _dsm-usrg-reentrancy.md
    usb-port-metadata.md

  diagrams/
    namespace.dot
    namespace.svg

  scripts/
    repro.sh
    redact.py
    build-namespace-graph.py

  advisory/
    draft-ghsa.md
    lenovo-psirt-report.md
```

### Deliverables in numbered form

1. **GitHub Security Advisory draft**: a private draft (or a public write-up if no security impact is claimed) aligned to GitHub’s security advisory workflow and required fields. citeturn8search2turn8search0  
2. **Conference-style write-up** (`docs/conference-writeup.md`): abstract → background (ACPI namespace + `_UPC/_PLD/_DSM`) → methodology → results → impact discussion → remediation/disclosure timeline → appendices.  
3. **Markdown “research paper”** (`docs/paper.md`): more formal, with citations and clear reproduction steps; suitable for submission to a firmware/OSDev/security meetup.  
4. **ACPI namespace diagrams** (`diagrams/namespace.dot` + rendered SVG): a graph that highlights conflicted name paths (e.g., colouring nodes that appear in multiple tables or that trigger loader/runtime errors).  
5. **Lenovo PSIRT submission pack** (`advisory/lenovo-psirt-report.md`): concise, vendor-triage friendly report with exact firmware/table identifiers, affected OS versions, and reproduction. Lenovo publishes PSIRT contact guidance and reporting channels. citeturn30search2turn14search2  
6. **Sanitised evidence bundle** (`artifacts/`): redacted logs + lists of table signatures/checksums + minimal DSL excerpts proving collisions, with explicit redaction notes (including MSDM handling). citeturn13search3  
7. **Reproduction scripts** (`scripts/repro.sh`, `scripts/redact.py`): turn-key tooling so others can confirm the behaviour on the same platform.

## Responsible disclosure and communication

### Vendor contact and expectations

entity["company","Lenovo","pc manufacturer"] documents that its Product Security Incident Response Team (PSIRT) handles newly reported vulnerabilities and provides contact details for reporting. citeturn30search2turn14search2  
For coordinated vulnerability disclosure norms, it is common to follow a structured timeline and provide vendors time to investigate and ship fixes before full public release; multiple industry bodies publish CVD guidance and best practices. citeturn7search2turn7search1

### Practical disclosure strategy for this bug class

- Start with a **“firmware correctness issue”** framing unless you have a clear security impact.
- Attach the **minimum necessary evidence**: redacted logs, ACPI table IDs (OEMID/OEM TableID), and the smallest DSL excerpts showing duplicate definitions.
- Offer a clear **mitigation hypothesis**: BIOS update likely best; if not available, vendor can fix AML by removing duplicate objects or making `_DSM` idempotent (guard or restructure field creation).

### Note on bug bounty expectations

Lenovo publicly emphasises internal ethical hacking and PSIRT intake, but a public cash-bounty programme for firmware issues may be private or scoped; treat payout expectations as uncertain and focus first on a high-quality, reproducible report. citeturn30search2turn30search1