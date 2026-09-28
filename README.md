# Port reporting queries so they keep their original meaning

A harbor-format benchmark task, authored for the AfterQuery Experts programme as data
annotation for AI evaluation. Task slug `afterquery/port-report-queries` — **Software / Databases**.

An AI agent gets a container, this brief and nothing else. It works alone for
4 hours. Then a sealed verifier it never sees scores the result `0` or `1`.

## What it asks for

We've moved the nightly reporting off MySQL onto PostgreSQL 15. The schema went across fine, and someone had already put the query set through a transpiler, so everything in `/app/queries` is valid PostgreSQL - all twenty-four parse, run against the new schema, and come back with rows. Nothing errors. Nothing looks broken.

The full brief is in [`instruction.md`](instruction.md).

| | |
| --- | --- |
| **Deliverable** | `/app/ported` |
| **Agent budget** | 4 hours (14400s) |
| **Expert estimate** | 7 hours |
| **Machine** | 2 CPU, 4096 MB RAM, 0 GPU |
| **Verifier budget** | 1800s |
| **Tags** | `sql`, `databases`, `postgresql`, `mysql`, `migration` |

## Why it is hard

The syntax half of the migration is already done. Every one of the twenty-four queries in /app/queries parses, runs against the shipped PostgreSQL schema and returns rows without error, because they are the output of a mechanical transpiler rather than a hand conversion. Nothing looks broken.

<details>
<summary>Full note</summary>

Nineteen of them are nevertheless wrong, because the transpiler translated the syntax and not the semantics, and the two engines disagree about what the same expression means. The work is finding which disagreement bites which query. Integer division yields a four-decimal DECIMAL on the source engine and a truncated integer on PostgreSQL. CONCAT returns NULL if any argument is NULL on the source, while PostgreSQL's concat() skips NULLs and only the || operator propagates them. The source collation is case-insensitive, so tier = 'gold' matches 'Gold'. Ascending order puts NULLs first there and last here. Division by zero is NULL there and an error here. || is logical OR there and string concatenation here, which silently turns a predicate into a comparison against 'overduetrue' and matches nothing. A CHAR column ignores trailing spaces in comparison; the varchar it became does not. Comparing a string column against a number coerces on the source and raises here. None of these is obscure and none is a trick; all of them are documented behaviour of two widely used engines, and an engineer who has done this migration knows most of them. Knowing them is not the task. Working out which of twenty-four queries each one actually changes, across joins, aggregates and subqueries, and repairing the semantics without disturbing the rows that were already right, is the task. What makes it resist a shortcut is that the environment ships an oracle which is not sufficient. Alongside the data the agent gets the exact output every query produced on the source engine for THAT data, so the obvious approach is to run each query, diff, repair whatever disagrees, and stop when the diff is clean. Grading is done on a second dataset with the same schema that the agent never sees, built so every divergence sits on a boundary. Measured on this bundle: the transpiled set as shipped reproduces 10 of 24 results on the visible dataset and 5 of 24 on the hidden one. A set repaired until the visible diff is completely clean reaches 24 of 24 on the visible dataset and only 19 of 24 on the hidden one. Five queries are wrong in ways the given data never exposes, and the bar is 22, so iterating against the oracle is not enough and the semantics have to be reasoned about directly. The source engine is not installed. An agent is free to install one and property test against it, which is a legitimate and fairly expensive route to the same answer, and is roughly what the alternative costs a person.

</details>

## How it is verified

The verifier runs in its own container and shares no state with the agent. It boots its own PostgreSQL, loads each dataset itself, and reads only the .sql files left in /app/ported. Nothing it grades against comes from the submission, and the queries are run with a statement timeout so a pathological one cannot stall the run.

<details>
<summary>Full note</summary>

Ground truth is the output the twenty-four original queries actually produced on a real MySQL-family engine, captured once against each dataset and stored under tests/truth. It is not hand-written and not derived from the PostgreSQL reference, so an error in the reference port cannot quietly become the expected answer. Two independent captures were compared and were byte-identical, which matters because an expected result that is not reproducible would make a query unsolvable however good the submission is. Ordering is fully determined: every query carries a total ORDER BY with a unique tie-breaker, every GROUP_CONCAT an explicit ORDER BY, and the two constructs whose source-engine result is genuinely implementation-chosen - a bare non-aggregated GROUP BY, and grouping on a case-insensitively collated column, which folds 'gold', 'Gold' and 'GOLD' into one group and then picks one of them as the label - are not used anywhere in the set. Both datasets are graded and at least 22 of 24 queries must reproduce their original result on each. The visible dataset is the one shipped to the agent together with its expected output, so a query rewritten to fit the hidden data fails there; the hidden dataset shares the schema but puts every divergence on a boundary, so a query repaired only until the visible diff went clean fails here. Requiring both also closes the obvious cheat: a query that returns literal rows can satisfy the dataset whose answers were published and cannot satisfy the one whose answers it has never seen. Comparison is on the full result set, ordered, with no tolerance. Cells are normalised as numbers where they parse as numbers, so 2.5000 and 2.5 count as equal and a correct port is not required to reproduce another engine's trailing zeros; anything that is not a number is compared exactly, and NULL is distinguished from the empty string. A reward of 0 is written before anything runs, so a crash anywhere scores zero rather than passing.

</details>

<details>
<summary>Reference approach (spoiler)</summary>

Read each query, decide which cross-engine differences its expressions are exposed to, and rewrite it so the result matches what the original produced. The reference port in solution/ported hand-translates nineteen queries and leaves five exactly as the transpiler emitted them, because those five are already correct; a blanket rewrite breaks them. Integer division becomes round(a::numeric / b, 4), which both casts and matches the source's four digits of scale - casting alone leaves a numeric division running to twenty digits. Division that can meet a zero denominator uses NULLIF to return NULL instead of raising. CONCAT becomes ||. Case-insensitive comparisons become lower() or ILIKE. Ascending sorts that can see a NULL take NULLS FIRST. DAYOFWEEK becomes EXTRACT(DOW) + 1. The aggregate ORDER BY that the transpiler dropped goes back inside string_agg. DATEDIFF against a DATETIME reduces the left side to a date, because subtracting a date from a timestamp yields an interval rather than a day count. A string column compared against a number reproduces the source's leading numeric prefix coercion rather than casting, which would raise on the first non-numeric row. || used as a boolean operator becomes OR. The column the transpiler mapped from TINYINT(1) to boolean is deliberately only ever 0 or 1 in the data. A TINYINT(1) holding 2 would be a sharper divergence, but every non-zero maps to true, so the 2 does not survive into the PostgreSQL data and no query could recover it - that would be an unsolvable query rather than a hard one.

</details>

## Layout

```
instruction.md          the brief - the only thing the agent is told
task.toml               artifacts, timeouts, resources, metadata
environment/            Dockerfile and data for the agent's container
solution/solve.sh       reference solution; scores 1 by construction
tests/                  verifier image, grading logic and ground truth
cheat/                  a deliberate cheating attempt, never executed
```

## Running it

With [harbor](https://github.com/laude-institute/harbor) installed and Docker running:

```bash
uv tool install harbor

harbor run -p . -a oracle -e docker   # reference solution - must score 1
harbor run -p . -a nop    -e docker   # nothing happens    - must score 0
```

Those two runs are the calibration the bundle has to satisfy: the oracle run substitutes
`solution/solve.sh` for the agent and must score `1`, the nop run does nothing and must
score `0`. Anything in between means the verifier is measuring the wrong thing.

## Reading this

`tests/` and `solution/` hold the answers. To try the task honestly, read only
`instruction.md` and `environment/`.

---

Part of a set of 38 tasks indexed at
[afterquery-data-annotation](https://github.com/helexdev1004/afterquery-data-annotation).
