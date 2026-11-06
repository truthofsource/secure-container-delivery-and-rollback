# Source map and deliberate adaptations

The selected implementation was reviewed November 6, 2026. Hashes identify private source inputs without publishing that private repository or its history. Example identities and labels preserve comparisons, not live authorization.

| Public file | Original implementation area | Adaptation |
| --- | --- | --- |
| [reference/rollback_release.py](../reference/rollback_release.py) | `scripts/rollback_release.py` | Actual offline functions; omit repository-file hash adapter and unused native-reviewer branch; historical schema-168 contract retained |
| [reference/test_rollback_release.py](../reference/test_rollback_release.py) | `scripts/test_rollback_release.py` | Retain 12 original offline tests; omit native-reviewer, live operations, repo inventory and SQL-helper integration cases |

The retained source tests replace one private ledger-file lookup with an explicit fictional 145-entry gap fixture. Schema 168 is a historical contract, not the current production ledger. Publishing/CLI adapters and actual release inventory are not included.

[Machine source hashes](source-map.json) · [Validation and limits](VALIDATION.md)

Related Terraform/reference provenance is recorded separately in the machine map. Parameterized/portable roots retain their own integration boundaries; they are not identical copies of the full deployed infrastructure.
