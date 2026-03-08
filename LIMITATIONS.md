## Limitations

1. No exploit primitive has been demonstrated.
2. No direct privilege escalation or arbitrary code execution path has been proven.
3. Secure Boot blocked loading of an unsigned `chipsec.ko` module, preventing certain low-level checks without MOK enrollment or local module signing.
4. Some impact statements remain inferential until validated by additional OS-level instrumentation or vendor confirmation.
