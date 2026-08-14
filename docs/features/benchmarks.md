# Benchmarks

Grounding the model in Ontul's definitions, storing what analysts verify, letting
them write down the fiscal calendar — each of those is an argument that answers
get better. None of them is evidence.

Benchmarks are the evidence: questions with a known-correct answer, run on
demand, scored, and kept so today's score can be compared with last month's. A
benchmark a change breaks is worth more than a plausible explanation of why the
change should have helped.

## A case

Each case is a question a user would type, paired with **the SQL an analyst says
answers it**.

| Field | Example |
|---|---|
| Question | `revenue and order count by order status` |
| Reference statement | `SELECT o_orderstatus, total_revenue, order_count FROM ice.examples.order_sales` |
| Connection | the data connection to run against |
| Note | what this case is meant to catch |

The truth is stored as SQL, not as a number. The suite runs against a warehouse
that is still being written to, and a hard-coded total starts failing for reasons
that have nothing to do with the assistant. Running the reference statement at
comparison time asks the question actually worth asking: *does the assistant's
answer agree with the statement an analyst vouches for, right now?*

A verified answer makes an excellent first case — it is already a question with
an agreed-upon statement.

## Running the suite

**Settings → Benchmarks**, then *Run*. The screen leads with the score, how it
moved since the previous run, and the failures — a benchmark you have to dig
through is one nobody runs twice. Each failure shows why it failed and the
statement the assistant produced.

Every case is answered through **the same path a user's question takes**: worker
dispatch, the verified library, the house rules, all of it. Calling the model
directly would be simpler and would measure a product nobody uses — the parts
most likely to regress are exactly the ones a shortcut skips.

Running requires the `WRITE_PROMPT` action, since a run consumes LLM calls
against every case.

## How answers are compared

The comparator has to be fair in both directions. Too strict and every run is red
for reasons no user would notice, and a suite that is always red gets ignored.
Too lenient and it certifies wrong answers, which is worse than having no
benchmark at all.

| Difference | Verdict |
|---|---|
| Column named `revenue` instead of `total_revenue` | Agrees — names are presentation |
| An extra column in the answer | Fails — that answered a different question |
| Rows in a different order | Agrees, unless the reference statement ends in `ORDER BY` |
| `204756995.37761158` vs `204756995.37761152` | Agrees — summation order, not a wrong answer |
| `100.0` vs `101.0` | Fails — the classic wrong-filter symptom |
| `42` vs `"42"` | Agrees |
| `Shipped` vs `  shipped ` | Agrees |
| `0.0` vs `null` | Fails — an absent value is not zero |

Row order counts only when the reference statement makes an ordering claim.
"Top 5 by revenue" is a claim about order; "revenue by status" is not. An
`ORDER BY` inside a subquery shapes the computation rather than the result and is
not treated as a claim.

Numeric comparison uses the relative tolerance
`mium.benchmark.numeric.tolerance` (default `1e-9`), applied absolutely for
values below 1 where a relative tolerance would degenerate into demanding exact
equality.

## Reading a failure

A case can fail for reasons that are not the assistant's fault, and the verdict
says which:

| Reason | What it means |
|---|---|
| `row 2, column 2: expected …, got …` | A real disagreement — the interesting case |
| `expected 3 row(s), got 5` | The answer covered a different set |
| `the reference statement could not run: …` | The **case** is broken, not the assistant |
| `the assistant returned no result to compare` | No statement came back — usually an upstream timeout |
| `no data connection is configured for this case` | Setup, not accuracy |

The distinction matters: without it, an afternoon gets spent debugging the model
when the reference SQL was the thing that stopped parsing.

## History

Every run is retained with its per-case verdicts, so the history answers the
question a single score cannot: is this better than last month, and which case
started failing when. The history endpoint returns
`mium.benchmark.history.max` runs (default 50); retention itself is unbounded.

## Storage

Two NeorunBase tables:

- `mium.mium_benchmark_case` — the suite
- `mium.mium_benchmark_run` — one row per run, with the per-case verdicts stored
  as JSON. A run is read as a whole, so splitting the verdicts across rows would
  buy nothing.

## REST API

| Route | Permission | Purpose |
|---|---|---|
| `GET /api/benchmarks` | any authenticated user | The cases |
| `POST /api/benchmarks` | `WRITE_PROMPT` | Add or replace a case |
| `DELETE /api/benchmarks` | `WRITE_PROMPT` | Remove a case by id |
| `POST /api/benchmarks/run` | `WRITE_PROMPT` | Run every case; returns the score and verdicts |
| `GET /api/benchmarks/runs` | any authenticated user | Run history, newest first |

## Related

- [Grounded Answers](grounded-answers.md) — what the score is measuring
- [Verified Answers](verified-answers.md) — a good source of cases
- [Instructions](instructions.md) — conventions that change what "correct" means
- [Configuration](configuration.md) — tolerance and history bounds
