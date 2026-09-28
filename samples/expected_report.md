# Expected report — facts and exact layout

The layout of `report.md` is up to you. Every fact below (ids and numbers) must
appear in it. A reference layout follows at the end.

## Required facts

| Fact | Value |
|---|---|
| Input lines | 11 |
| Unique transactions | 10 |
| Settled | 4 purchases, 1 reverse, 1 refund |
| Settled purchases | 1350000 |
| Settled reverses | 200000 |
| Settled refunds | 20000 |
| Fees | 17450 |
| Net | 1112550 |
| Declined | 1 — `txn-1007` |
| Timeout | 1 — `txn-1008` |
| Duplicates | `txn-1003` (identical, counted once) |
| Conflicting duplicates | none |
| Missing originals | `txn-1009` -> `txn-9999` |
| Amount mismatches | `txn-1010`: reverse 99999 > original `txn-1002` amount 50000 |
| Exit code | 3 (issues exist) |

Per-transaction fees (purchases only):

| Transaction | Terminal | Amount | Fee |
|---|---|---|---|
| `txn-1001` | T-001 | 100000 | 600 |
| `txn-1002` | T-001 | 50000 | 450 |
| `txn-1003` | T-002 | 200000 | 3400 |
| `txn-1004` | T-002 | 1000000 | 13000 |

## Normative rendering — `report.md` must equal this

```markdown
# Settlement report — 2026-01-05

## Totals
- input lines: 11
- unique transactions: 10
- settled: 4 purchases, 1 reverse, 1 refund
- settled amounts: purchases 1350000, reverses 200000, refunds 20000
- fees: 17450
- net: 1112550

## Non-settled
- declined: 1 (txn-1007)
- timeout: 1 (txn-1008)

## Issues
### Duplicates
- txn-1003 (identical, counted once)
### Conflicting duplicates
- none
### Missing originals
- txn-1009 -> txn-9999
### Amount mismatches
- txn-1010: reverse 99999 > original txn-1002 amount 50000

## Fee schedule
| terminal | percentage_bps | fixed_fee | max_fee |
| --- | --- | --- | --- |
| T-001 | 30 | 300 | - |
| T-002 | 120 | 1000 | 50000 |
```
