# Project 5 — `settle-recon`

**Build a settlement and reconciliation CLI: ingest a day of transactions, apply
per-terminal fee schedules, produce a settlement file, and report duplicates,
missing originals and amount mismatches — with byte-identical output on re-runs.**

| | |
|---|---|
| Difficulty | Easy–Medium |
| Suggested time | 1 week |
| Dependencies | `gopkg.in/yaml.v3` only, everything else stdlib |
| Language | Go |

---

## 1. Why this project

Settlement is where money math and data quality meet: if the same day is
processed twice, the totals must be identical; if a refund references a
transaction that does not exist, a human must be told instead of the number
being quietly dropped. The real switch does this in `settlementsvc`. Here you
build the offline, deterministic core — file in, files out, no database.

## 2. Scope

**In scope**

- One CLI: `run --transactions f.jsonl --fees fees.yaml --outdir out/`.
- Input validation, deduplication, reconciliation checks, fee computation,
  settlement CSV, report Markdown.
- Deterministic, idempotent output.
- Unit tests, golden-file tests over `samples/`.

**Out of scope**

- HTTP, databases, message queues, multi-day windows.
- Real Shaparak file formats; your own documented format is enough.
- Multi-currency handling: this project is single-currency; there is no
  `currency` field anywhere.

## 3. Specification (the contract)

### 3.1 Input: transactions JSONL

One JSON object per line:

```json
{"txn_id":"txn-1001","terminal":"T-001","type":"purchase","amount":100000,
 "status":"approved","original_txn_id":null,
 "timestamp":"2026-01-05T10:00:00+03:30"}
```

| Field | Rule |
|---|---|
| `txn_id` | non-empty; duplicates handled per §3.2 |
| `terminal` | non-empty |
| `type` | `purchase`, `reverse` or `refund` |
| `amount` | integer > 0, minor units |
| `status` | `approved`, `declined` or `timeout` |
| `original_txn_id` | required for `reverse`/`refund`, must be `null` for `purchase` |
| `timestamp` | RFC 3339 |

A malformed line fails the whole run with exit code `1` (do not partially
process; the input is either trustworthy or it is not).

### 3.2 Reconciliation rules

- **Only `approved` transactions settle.** `declined` and `timeout` are counted
  and listed in the report but contribute nothing.
- **Duplicates** — the same `txn_id` appearing more than once. The **first**
  line for a `txn_id` is canonical (first-wins):
  - later copies with equal parsed content → counted once, reported under
    `duplicates`,
  - later copies that differ from the canonical → reported under
    `conflicting_duplicates`; the canonical line still settles normally.
- **Originals** — an approved `reverse` or `refund` must reference an approved
  **purchase** as its original, on the same terminal. Violations are reported
  under `missing_originals` and do not settle.
- **Amounts** — refunds and reverses may be partial; a refund does **not**
  require the original to be reversed first. The sum of settled
  refunds+reverses for a transaction must not exceed the original amount.
  Violations are reported under `amount_mismatches`; the offending line does not
  settle (earlier accepted partials stay). Refunds and reverses are terminal:
  they can never be referenced as originals.
- Issues never stop the run: outputs are still written, and the exit code is `3`
  when at least one issue was found (`0` when clean, `1` bad input,
  `2` internal bug).

### 3.3 Fees (`fees.yaml`)

```yaml
defaults:
  percentage_bps: 50        # 0.50 %
  fixed_fee: 500            # minor units per purchase
terminals:
  T-001:
    percentage_bps: 30
    fixed_fee: 300
  T-002:
    percentage_bps: 120
    fixed_fee: 1000
    max_fee: 50000          # cap per transaction, optional
```

- Fee applies to **purchases only** in this project; refunds and reverses carry
  no fee (documented simplification).
- `fee = round_half_up(amount × percentage_bps / 10000) + fixed_fee`, capped at
  `max_fee` when `max_fee > 0`. Use integer arithmetic only:
  `bpsFee = (amount*percentage_bps + 5000) / 10000`. Example:
  `amount=10500, bps=50` → `52.5` → `53`.
- Terminal override wins over `defaults`; unknown terminal uses `defaults`.
- Negative `percentage_bps`, `fixed_fee` or `max_fee` is invalid config → exit `1`.

### 3.4 Outputs

**`out/settlement.csv`** — one row per terminal, sorted by terminal:

```
terminal,purchase_count,purchase_amount,reverse_count,reverse_amount,refund_count,refund_amount,fee_total,net_amount
T-001,2,150000,0,0,1,20000,1050,128950
T-002,2,1200000,1,200000,0,0,16400,983600
```

`net_amount = purchase_amount − reverse_amount − refund_amount − fee_total`.

**`out/report.md`** — follows the layout pinned in `DECISIONS.md` D10, which
contains the required sections, ordering and formats:

- run totals: input lines, unique transactions, settled counts by type, settled
  amounts, fees, net total;
- non-settled: counts of `declined` and `timeout` with txn ids;
- issues: `duplicates`, `conflicting_duplicates`, `missing_originals`
  (with the missing id), `amount_mismatches` (with both amounts);
- the fee schedule used (by terminal).

### 3.5 Determinism

- Re-running the same inputs produces byte-identical `settlement.csv` and
  `report.md`.
- No run timestamps, no map-iteration-order output, stable sorting everywhere.
- Output files are written atomically (temp file + rename) so a failed run does
  not leave half-written results.

## 4. Milestones

1. **Models + parsing** — JSONL reader, validation, strict error reporting.
2. **Fees** — YAML load, override lookup, integer fee math + table tests
   including the rounding example.
3. **Dedup + reconciliation** — duplicates, conflicting duplicates, originals,
   partial refund cap.
4. **Aggregation** — per-terminal sums, deterministic sorting.
5. **Outputs** — CSV writer, report generator, atomic file writes.
6. **Golden tests** — run on `samples/transactions.day1.jsonl` and compare with
   `samples/expected_settlement.csv`; assert every fact in
   `samples/expected_report.md`.
7. **Idempotency test** — run twice into two directories, `diff -r` must be empty.
8. **Finish** — README, coverage, `SUBMISSION.md`.

## 5. Acceptance criteria

1. Running on the sample input produces exactly
   `samples/expected_settlement.csv` (byte-for-byte).
2. `report.md` matches the normative rendering in
   `samples/expected_report.md` (pinned by `DECISIONS.md` D10) byte-for-byte:
   duplicate `txn-1003`, missing original `txn-9999`, amount mismatch
   `txn-1010` 99999 > 50000, declined `txn-1007`, timeout `txn-1008`.
3. Exit code is `3` for the sample (issues exist) and `0` for a clean fixture
   you create.
4. Two runs into different directories produce identical files (`diff -r`).
5. Fee rounding test includes `amount=10500, bps=50 → 53` and a `max_fee` cap
   case.
6. Malformed input exits `1` with a message naming the line number.
7. `go build`, `go vet`, `go test -race ./...` pass; coverage ≥ 80 %.
8. Only `gopkg.in/yaml.v3` in `go.mod`; all money is `int64`.

## 6. Constraints & conventions

- No floating point anywhere: parse nothing into `float64`, format nothing with
  `%f`. If a percentage cannot be represented as basis points, don't support it.
- The pipeline is functions over slices + one `Run` orchestrator; no global
  state, no `init()` magic.
- `report.md` generation is deterministic: sort every list before printing.
- Errors carry context (`file:line: what`); reconciliation issues are *data*,
  not errors.

## 7. What you will learn (Go vs. Java habits)

| Topic | Java habit | Go reality |
|---|---|---|
| Money | `BigDecimal`, careful scale | `int64` minor units; rounding is your explicit formula |
| CSV | OpenCSV/BOM | `encoding/csv` from the stdlib |
| YAML config | Jackson/SnakeYAML annotations | struct tags + one small dependency |
| Collections | `HashMap`, `TreeMap` | maps are unordered by design; sort explicitly when outputting |
| Files | `Files.write` | create temp + `os.Rename` for atomicity |
| CLI | Apache Commons CLI | `flag` package, subcommands by hand |

## 8. Stretch goals

- Multi-day input with carry-over of unsettled partials.
- Fee reversal on refunds (undo the original fee proportionally).
- A second input format (CSV) behind the same reader interface.

## 9. Files in this folder

| File | Purpose |
|---|---|
| `DECISIONS.md` | Normative decision log — every open design point is pinned there |
| `samples/transactions.day1.jsonl` | 11 lines: 4 purchases, 1 partial refund, 1 reverse, 1 declined, 1 timeout, 1 duplicate, 1 missing original, 1 amount mismatch |
| `samples/fees.yaml` | Fee schedule with per-terminal override and a `max_fee` cap |
| `samples/expected_settlement.csv` | Exact expected CSV (byte-for-byte) |
| `samples/expected_report.md` | Required report facts + the normative rendering (byte-for-byte) |

## 10. Getting started


```bash
mkdir settle-recon && cd settle-recon
go mod init example.com/settle-recon
go get gopkg.in/yaml.v3
mkdir -p cmd/settle-recon internal/{ingest,fees,recon,report}
# copy PROJECT.md and samples/ from this folder into your repo
go run ./cmd/settle-recon run --transactions samples/transactions.day1.jsonl \
    --fees samples/fees.yaml --outdir /tmp/out
```

Parse the whole file first and print a summary of counts before writing any
aggregation code; it is much easier to debug a parser than a total.
