<img width="1365" height="2048" alt="image" src="https://github.com/user-attachments/assets/0dc3d7f6-2a64-40b4-b367-4adf61530643" />

=======
# ACPI Namespace Collision Research

Research repository documenting reproducible ACPI namespace collisions observed
on a Lenovo platform.

The firmware exposes duplicate ACPI namespace objects which cause
ACPICA interpreter errors (AE_ALREADY_EXISTS) and abort certain hardware
methods during boot.

Current classification: firmware correctness defect with possible security relevance.

No privilege escalation or memory corruption has been demonstrated.

Contents:
- ACPI dumps
- DSL disassembly
- kernel logs
- reproduction steps
- analysis timeline

Researcher: Grant Scott Turner
>>>>>>> 1ef496e (Clean ignore rules and update ACPI disclosure artifacts)
