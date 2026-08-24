# Halcyon Brain — an access-control model built to be attacked

A working implementation of access control for an AI assistant that answers
questions over records not every asker is allowed to see. It exists so the
design can be attacked rather than described. All data is synthetic.

The short version of the thesis:

> **The AI is never trusted.** Its only job is turning a question into a
> structured query. It holds no credentials, has no database access, and never
> reads anything the asker is not cleared for. Everything that actually protects
> a record is ordinary code and arithmetic underneath it.

This is the opposite of the common approach — give the model everything and
instruct it to withhold the sensitive parts. That approach is broken, and
published research ([FragFuse][fragfuse]) shows you break it by splitting a
forbidden question into innocent pieces across turns and reassembling the
answers yourself.

**Read [`FINDINGS.txt`](FINDINGS.txt) first.** It is the red-team log: every
attack run, what held, what did not, and two findings that were withdrawn after
checking them against the specification and discovering the design had
anticipated both.

**What's in this repo:** the design write-up (this file) and the full red-team
log. The implementation — schema, query gate, grammar, attack suite — is
private. File names below refer to that private code. Ask if you want a
walkthrough.

---

## What is actually enforced, and where

| Layer | Mechanism | Why there and not in the model |
| --- | --- | --- |
| Rows | PostgreSQL row-level security | The assistant cannot be talked into revealing a row that does not exist for its session |
| Query shape | Whitelist grammar, validated before execution | A jailbroken translator produces a rejected query, not an executed one |
| Aggregates | k-threshold, Laplace noise, per-compartment budget | Bounds what any *sequence* of answers can reveal, rather than trying to detect an attack |
| Retrieval | Fixed top-N, no distance threshold | The result *count* cannot depend on rows you cannot see |
| Prose | Quarantined summarizer, no tools, no database | It reads attacker-controlled text, so it is given nothing to do damage with |
| Record | Append-only audit log, enforced by privileges | A log the application could choose to skip is not evidence |

### The database decides, not the model

Permissions live in RLS policies on the table, not in application middleware.
The application connects as a **non-superuser** role — Postgres silently ignores
row-level security for superusers, so connecting as `postgres` would make every
policy in `db/schema.sql` decorative while the system still appeared to work.
`assertNotSuperuser()` in `lib/db.ts` fails loudly at startup for exactly that.

Session identity is set **per transaction**, not per session. On a pooled
connection a session-level setting outlives the request, and the next person to
borrow that connection inherits someone else's clearances.

### The tool surface is fixed at deploy time

Capability follows declaration. The `employee` table cannot be searched
semantically — not because a special case says so, but because it declares no
embedding source columns, and the grammar refuses to search a table with none.
Three lists are cross-checked at startup: a sensitive column may not appear in
lookup columns, may not be filterable, and may not be fed into an embedding.
Each is a separate channel onto the same value — read it, binary-search it
through row counts, or rank rows by proximity to it.

### Compartments are sideways

Not a ladder. In the demo the CFO outranks the security analyst and still cannot
read a security incident; an HR partner is inside HR and still cannot read
executive compensation, because that is a named list *within* the compartment.

---

## The attack suite

Twelve rounds, roughly ninety findings. Row lookups, semantic search, filtered
search, the time column, per-account walls, CSPRNG and distribution checks, and
a 1000-trial differencing attack whose measured rate is asserted against what
the noise parameters predict. The suite runs against the private implementation;
the results are all in [`FINDINGS.txt`](FINDINGS.txt).

---

## What got through

One result stands, and it is published as a **rate rather than a guarantee**:

Two noised counts whose filters differ by one slice, subtracted, recover the
true value exactly **23.1%** of the time and to within one **60.6%** of the
time. That is differential privacy behaving exactly as ε = 1 predicts. The
budget is what bounds it — ten novel questions per person per month is five
differencing pairs, and repeating a pair is cached and free.

`attack/stats.ts` does not merely measure that rate, it **derives the expected
rate from the noise parameters and asserts the measurement against it**, failing
past four standard errors. An earlier version only measured, and reported 24.0%
then 19.0% after the corpus grew — which reads as scale making the attack
harder. It does not: the true counts cancel out of the algebra, so corpus size
cannot appear in the answer. That was sampling noise, and a number with nothing
to compare it against cannot catch a bug.

## What is deliberately not claimed

- Persona is self-selected; there is no login, so impersonation is out of scope
  by construction. Every claim here is about what a **given** persona can reach.
- The attack cases are self-designed. This is not a third-party assessment.
- Deciding whether a given answer leaks is provably intractable in general, and
  denials themselves leak — which is what broke most published auditing schemes.
  So this design does not detect attacks. It **prices** them.

## Findings worth reading

Two entries in `FINDINGS.txt` are there because being wrong is part of what a
red team produces:

- **A load-bearing rule enforced only by a comment.** The aggregate path spends
  k-suppression, noise and a budget protecting salary — all worthless if the
  same value could be read off a record lookup. It could not be, for exactly one
  reason: salary was absent from a list. That was the entire protection. It is
  now a startup check with a test that has been *seen to fire*.
- **A finding whose first write-up was wrong.** It was drafted as a disclosure;
  reverting the fix to prove the regression test fired showed it was not one —
  clearance separation was happening as a side effect of an unrelated staleness
  mechanism. The real defect was different and smaller, and the write-up says so.

[fragfuse]: https://arxiv.org/pdf/2606.15609
