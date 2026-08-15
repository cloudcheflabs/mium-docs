# Grounded Answers

Mium does not maintain a semantic layer of its own. Ontul already curates
metrics, dimensions, join paths, synonyms and a `DRAFT` / `CERTIFIED` /
`DEPRECATED` lifecycle, and its `SemanticRewriter` expands a metric reference
into its expression at query time. Mium reads that surface over Ontul's REST API
— under Ontul's own IAM — and aims the model at it.

This is the accuracy argument for the coupling, and it is worth stating plainly:
a small on-prem model asked to assemble a correct star-schema query will
sometimes get it wrong, and the failure is invisible because wrong SQL still
returns a number. Asked instead to pick two names from a curated list, it is
reliable, and everything that makes the answer correct — the aggregate, the
join, the `GROUP BY` — is written by an engine that cannot get it wrong.

## What the model is aimed at

```sql
SELECT o_orderstatus, total_revenue FROM ice.examples.order_sales
```

No join, no `SUM(...)`, no `GROUP BY`. `total_revenue` is a metric on the
semantic view `ice.examples.order_sales`; Ontul rewrites the reference into its
expression and generates the grouping over the remaining selected columns. The
system prompt says so explicitly, and says not to write the aggregate or join
the underlying tables.

A table with no semantic view is still visible in the raw table listing, but the
definitions come first in the prompt and the model is told to prefer them. That
makes governance the path of least resistance rather than an optional extra.

## What goes into the prompt

Assembled per turn by `SemanticContextBuilder`, selected by relevance to the
question being asked, with `CERTIFIED` views ahead of `DRAFT` ones. Anything
dropped for size is announced in the prompt rather than silently cut — a model
that believes it has seen the whole catalogue will invent the parts it is
missing.

**Semantic views** — fully-qualified name, status, description, metrics (with
descriptions and synonyms) and dimensions.

**Metrics matching the question** — resolved through Ontul's
`semantic-metrics/search`, so "revenue" reaches `total_revenue` without the user
knowing the metric's name.

**Retrievers** — offered *instead of* SQL for questions shaped like similarity,
semantic search, graph traversal or ranking. The prompt tells the model to call
one by name with its declared arguments rather than trying to express the
question as analytic SQL.

**Ontology** — typed business entities and their relationships, each carrying
Ontul's certification verdict. The physical column is shown first and the
ontology property name second:

```
ontology.sales.Customer  -> ice.examples.customers  [CERTIFIED]
  properties:
    - c_name  (ontology property: name): the customer's registered name  [PII]
  relationships:
    - Customer places Order  (join on c_custkey = o_custkey)  one-to-many  [CERTIFIED]
```

Both details are load-bearing. Property names exist only in the ontology API, so
a model shown `name` will write `SELECT name` and get a "column not found"
error; and relationships carry their join keys so two entities are related the
way the ontology says, not the way the model guesses.

## Certification: what Mium reads, and what it ignores

Every Ontul definition carries two status fields, and only one of them is safe to
build a trust indicator on.

| Field | What it is |
|---|---|
| `status` | What a person declared: `DRAFT`, `CERTIFIED` or `DEPRECATED` |
| `effectiveStatus` | What Ontul derives, and the only field Mium reads |

`effectiveStatus` answers the question `status` cannot: *is that declaration
still true?* When a definition is certified, Ontul records a fingerprint of what
it meant at that moment — read source, property-to-column mapping, primary key,
join keys, SQL template. If any of that changes afterwards, the derived status
becomes `STALE` while the declared one stays `CERTIFIED`. Editing a description,
a title, a synonym or a tag does not break the signature: a certification that
evaporated whenever someone improved the documentation would teach everybody to
stop improving the documentation.

The derived status also rolls up. An object type is never more trusted than the
read source behind it, a link type never more than either end it joins, an action
type never more than the object type it writes to. A `CERTIFIED` entity sitting on
a draft view is, in effect, draft — and Ontul says so, so Mium does no graph
walking of its own.

| Value | How Mium treats it |
|---|---|
| `CERTIFIED` | Signed, unchanged since, and its dependencies are certified too |
| `STALE` | **Not** certified. Signed once, and the definition has changed since |
| `DRAFT` | Not certified — never was, or demoted by something it depends on |
| `DEPRECATED` | Not certified, and retired |

When a verdict is not `CERTIFIED`, Ontul supplies a `certificationNote` saying
why, naming the dependency where the fault lies. Mium carries that note through
to the tooltip: *"the definition changed after it was certified"* and *"endpoint
ontology.sales.Order is DEPRECATED"* call for completely different responses, and
only the server knows which applies.

## Provenance

Every answer carries what it was built from, resolved from the SQL that actually
ran rather than from what the model claimed to do. `SemanticProvenanceResolver`
matches the executed statement against the registry on identifier boundaries and
reports:

- whether the answer is **grounded** — it referenced a governed asset at all
- the **worst `effectiveStatus`** among the assets it used, which is what the
  answer as a whole is worth
- which semantic views, ontology entities and metrics were used
- for anything that fell short, Ontul's reason

Certification is an *all-of* claim. A join across a certified view and a draft
one is a draft answer; reporting it as certified because one half qualifies would
let the ungoverned half borrow the other's credibility, which is the confusion
this indicator exists to prevent.

Ontology entities are matched through their **read source**, because a statement
says `FROM ice.examples.customers` and never `FROM ontology.sales.Customer`. Link
types are matched separately rather than inferred from their endpoints — a link
is its own definition with its own signature and can be less trusted than either
end — and are only attributed when both ends and both join columns appear.

The UI renders all of this beside the answer, including the unflattering verdict:
an answer written against raw tables says *Not from a certified definition*, and
a stale one says *Certification out of date*, naming the definition and, on
hover, Ontul's reason. A trust signal that cannot fail is decoration.

The re-ranking outcome is reported the same way — *Re-ranked* when Ontul's
second-stage model ran, and *Ranking not re-scored* with the reason when it did
not.

## When something goes wrong

Storage failures, permission denials, timeouts and LLM faults are translated
into a sentence a user can act on before they reach the screen
(`UserFacingError`). A stack trace or a raw upstream message is the assistant
failing twice: once at the task, once at explaining itself.

Messages that name a missing column or table pass through unchanged — those are
the ones a user can actually fix. Transient failures (a reset connection, no
live upstream, a `502`) are retried once before anything is reported at all;
authorization, parse and not-found errors are never retried, because the second
attempt fails identically and only costs the user time.

## Related

- [Verified Answers](verified-answers.md) — recording what an analyst confirms
- [Instructions](instructions.md) — the conventions no definition can hold
- [Benchmarks](benchmarks.md) — whether any of it worked
- [Tool Integration](tool-integration.md) — the Ontul surfaces this reads
