# Submission — `settle-recon`

> **Send this file to your reviewer when the project is done.**
> Fill every section honestly. "Not done / partially done" is a valid answer and
> is better than an empty box. Attach raw evidence (paste output, don't summarize).

## 1. Author & repository

| | |
|---|---|
| Name | |
| Repository URL | |
| Started on | |
| Finished on | |
| Estimated hours spent | |

## 2. How to build & run

```bash
git clone <repo-url> && cd <repo>
go build ./...
go test -race ./...
go run ./cmd/settle-recon run --transactions samples/transactions.day1.jsonl \
    --fees samples/fees.yaml --outdir /tmp/out
echo "exit=$?"
```

## 3. Acceptance evidence

| # | Acceptance criterion | Evidence | Pass? |
|---|---|---|---|
| 1 | Sample run produces `expected_settlement.csv` byte-for-byte | | ☐ |
| 2 | Report equals the normative rendering byte-for-byte (D10) | | ☐ |
| 3 | Exit `3` with issues, `0` on your clean fixture | | ☐ |
| 4 | Two runs → identical files (`diff -r`) | | ☐ |
| 5 | Rounding + `max_fee` tests | | ☐ |
| 6 | Malformed line → exit `1` naming the line | | ☐ |
| 7 | build/vet/race pass; coverage ≥ 80 % | | ☐ |
| 8 | Only yaml.v3 dependency; all money `int64` | | ☐ |

## 4. Demo transcript

Paste the terminal session showing at minimum:

1. the sample run and `cat /tmp/out/settlement.csv`,
2. `cat /tmp/out/report.md`,
3. `diff -r` of two runs (empty diff),
4. a malformed-line run with its error and `echo $?`.

```
(paste here)
```

## 5. Design notes & tradeoffs

Answer briefly (a few sentences each):

1. How is the refund/reverse cap computed (per what key, in what order)?
2. Where does determinism come from — list every place you sort.
3. How are output files written so a crash cannot leave a half-written file?
4. Why are reconciliation results data and not errors?
5. What would break first if the input grew to 10 million lines?

## 6. Known limitations / not done

- 

## 7. Questions for reviewer

- 

---

## 8. Reviewer use only

| | |
|---|---|
| Reviewer | |
| Date | |
| Verdict | ☐ accepted  ☐ accepted with comments  ☐ rework needed |
| Comments | |
