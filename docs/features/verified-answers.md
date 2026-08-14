# Verified Answers

Grounding the model in Ontul's definitions fixes what it can aim at. It does not
capture the judgement that "revenue this quarter" means the fiscal quarter, or
that this particular question has to exclude internal accounts, or that the
right join for it runs through the returns table. That knowledge sits in an
analyst's head until something records it — and without somewhere to put it, the
assistant is exactly as accurate on day 200 as on day one.

The verified library is that place.

```
user asks  ->  the answer is wrong  ->  the user reports it
           ->  an analyst corrects the SQL and verifies it
           ->  the next person asking gets the verified answer, with attribution
```

## For the person who spotted it

Under every answer is **Was this right?** with a *Not right* button. It takes one
click; the note is optional, because the question and the statement that produced
it are already recorded and are most of what an analyst needs.

Reporting is open to every user and needs no permission. The person who notices
that a number is wrong is almost never the person who can correct it, and
charging them for the complaint loses the signal entirely.

## For the analyst

**Settings → Verified Answers** is the review queue.

- **Needs review** — reported answers, plus any verified answer a user has since
  disputed. The SQL is editable in place; correct it and press *Verify*.
- **Verified** — the working library, showing who verified each pair, when, and
  how many times it has been served.
- **Retired** — superseded pairs, kept for history and never reused.

Verifying requires the `WRITE_PROMPT` action, because it changes the answer
everybody else receives. A curator who is looking at a correct answer in the
Analyze workspace can also verify it directly from there, without visiting this
screen.

## What happens on the next ask

The master resolves the question against the library before dispatching the turn.
On a hit, the verified statement is substituted for whatever the model would have
generated, and the answer carries **Verified by _name_** in its trust bar.

Resolution happens on the master rather than in the agent loop for a structural
reason: the loop runs on workers, which do not connect to NeorunBase. The
statement travels to the worker on the `EXECUTE_AGENT` request, and the local
fallback path uses the same value, so a single-node deployment answers the way a
fleet does.

## Matching

Retrieval is by **normalised question text**, not by embedding. Case, surrounding
punctuation and repeated whitespace are noise; word order and wording are not.

| These are the same question | These are not |
|---|---|
| `Revenue by order status?` | `revenue by order status` vs `revenue by order priority` |
| `  Revenue   by  order status  ` | `customers who ordered` vs `customers who never ordered` |
| `revenue by order status.` | `revenue last quarter` vs `revenue this quarter` |

This is deliberately narrow for a first version. An exact or near-exact repeat is
both the common case and the one where reuse is unambiguously right; matching a
verified statement onto a merely similar question would answer something the
analyst never verified, which is precisely the harm this feature exists to
prevent. The embedding store is already wired and can widen the match later, once
there is evidence about which near-misses are safe.

## Disputing a verified answer

If a user reports an answer that is already verified, the pair is **marked as
disputed and returned to the review queue while it keeps serving**.

It is not retired. One complaint undoing an analyst's judgement would hand any
user a way to switch off an answer for everyone. Nor is the complaint merely
appended to a note and forgotten: the pair people keep flagging is exactly the
one worth rereading, and before this it was the one nobody saw.

## Storage

A NeorunBase table, `mium.mium_verified_query`, beside the memory store.

| Column | Meaning |
|---|---|
| `question`, `question_key` | The original text and its normalised lookup key |
| `sql_text` | The statement an analyst verified |
| `status` | `REPORTED` / `VERIFIED` / `RETIRED` |
| `connection_id` | Which data connection the pair belongs to |
| `verified_by`, `verified_at` | Attribution, shown in the trust bar |
| `challenged_at`, `challenged_by` | Set when a user disputes a verified pair |
| `use_count` | Times served in place of a generated query |

`use_count` is maintained read-then-write rather than with `SET use_count =
use_count + 1`: NeorunBase's parser keeps only literal values from a `SET`
clause, so the arithmetic form is accepted and then writes nothing. Two
simultaneous hits on one pair can lose an increment, which is acceptable for a
counter that only shows which entries earn their place.

## REST API

| Route | Permission | Purpose |
|---|---|---|
| `POST /api/verified/report` | any authenticated user | Report a wrong answer, or dispute a verified one |
| `POST /api/verified/verify` | `WRITE_PROMPT` | Store or replace the verified statement for a question |
| `GET /api/verified?status=` | any authenticated user | List a queue. `REPORTED` also returns disputed verified pairs |
| `DELETE /api/verified` | `WRITE_PROMPT` | Remove a pair by id |

Row counts are bounded by `mium.verified.list.max` — see
[Configuration](configuration.md).

## Related

- [Grounded Answers](grounded-answers.md) — where the model aims before any of this
- [Instructions](instructions.md) — conventions that apply to every question
- [Benchmarks](benchmarks.md) — measuring whether the library helped
- [IAM](iam.md) — the `WRITE_PROMPT` action
