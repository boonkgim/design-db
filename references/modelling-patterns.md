# Modelling patterns

Shapes that hold together, with **what selects each** and **what disqualifies it**.

## How to use this file, and how not to

**This is an elimination aid, not a menu.** Its job is to stop you designing something whose
failure mode is already known, and to stop you rejecting a shape that would have worked. It is
not a list of recommendations, and a model picked from here and justified afterwards is exactly
the failure the skill's *Core principle* forbids: the invariants decide, not taste, and not a
table.

Use it in one direction only — you have the invariants from section 2, and you are asking which
shapes can still express them. If you find yourself reading it before the invariants are
written down, close it.

Two consequences, both binding:

- **The "disqualifies" line is the load-bearing one.** It is what this file is for. "Selects
  it" is context so the elimination makes sense, not an endorsement.
- **Every shape still has to be argued in the document from the invariants**, with its
  alternatives named. "It is the standard pattern" is not a justification and must never appear
  in a data model document.

**The DDL here is PostgreSQL-flavoured and is illustrative.** Every mechanism — exclusion
constraints, partial indexes, deferred constraints, generated columns — must be checked against
the store and plan the tech stack document actually chose before a decision rests on it. A
mechanism that is not available does not eliminate the pattern; it moves the enforcement up a
level, and section 6 has to say so.

---

## Something finite: seats, slots, stock, licences

### A — A counter on the parent, changed by one guarded statement

```sql
UPDATE runs SET seats_taken = seats_taken + :n
 WHERE id = :id AND seats_taken + :n <= capacity;
-- zero rows updated means "it just filled"
```

**Selects it:** the units are interchangeable — nobody cares *which* seat; the count is read on
every page and the read must be cheap; the claim must be one statement so there is no
read-then-write to lock around.

**Disqualifies it:** units are individually identifiable or assignable (seat 4A, licence key,
serial number); anything needs to reference one unit; the count is not the only thing consumed.

**Costs:** the counter is a denormalisation and can drift. It needs a reconciliation, and it
needs the constraint written so an intentionally soft cap is not rejected by a `CHECK`.

### B — A row per unit, claimed by update

**Selects it:** units are distinguishable, assignable, or must be referenced individually; you
want the invariant to be structural — a seat cannot be sold twice because there is one row and
it has one owner; the number of units is small.

**Disqualifies it:** thousands of units per parent with a hot "how many left" read; units are
created and destroyed constantly; the business genuinely counts rather than allocates.

**Costs:** counting the free ones is a scan or another index; bulk claims lock many rows.

### C — An exclusion constraint over a range

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE bookings ADD CONSTRAINT bookings_no_overlap
  EXCLUDE USING gist (resource_id WITH =, tstzrange(starts_at, ends_at) WITH &&)
  WHERE (state <> 'cancelled');
```

**Selects it:** the conflict is *overlap*, not count — one resource, one period, one claimant.
This is the only concurrency-safe answer to that question, and it is enforced by the engine
rather than by a query that races.

**Disqualifies it:** the store has no exclusion constraints or the plan forbids the extension;
overlap is legal and only the total matters; the "resource" is not a single column.

**Costs:** the index is a GiST index, larger and slower to write than a B-tree; the partial
`WHERE` clause has to be kept in step with the state machine.

---

## An amount that changes: balances, credit, stock levels

### D — Append-only ledger, balance derived

**Selects it:** the business must be able to answer *why* the balance is what it is; money,
credit, or anything auditable; corrections must be visible rather than silent.

**Disqualifies it:** the balance is read constantly and the ledger is long enough that summing
it is a real cost — and you have measured that, rather than assumed it.

**Costs:** every read is an aggregate. Mitigate with a covering index before reaching for a
stored balance; a stored balance is a different pattern with different failure modes, not a
tuning knob.

**Non-negotiable if chosen:** entries are never updated and never deleted. A reversal is a new
entry with the opposite sign, referencing the one it reverses. The moment a ledger row can be
edited, it stops being able to explain itself, which was the only reason to have one.

### E — Stored balance, guarded

**Selects it:** the balance is the hot read; history is genuinely not required; the update is
expressible as one guarded statement.

**Disqualifies it:** any audit, dispute, or reconciliation requirement — which in practice
means any money. If someone will ever ask "how did it get to this number", this shape cannot
answer.

**Costs:** it is a denormalisation and needs a reconciliation. It also silently loses the
question that produced it.

### F — Ledger plus periodic snapshot

**Selects it:** D was right but the ledger has grown past the point where summing it is
acceptable, and you can point at the measurement.

**Disqualifies it:** you have not measured. This is an optimisation with a whole new class of
bug — a snapshot that is wrong is worse than a slow query — and it is almost never needed at
the volumes a v1 PRD describes.

---

## A value that changes over time

### G — Mutable row, no history

**Selects it:** nobody will ever ask what it was before. This is a legitimate and common
answer, and it must be *stated*, because history not recorded cannot be recovered.

**Disqualifies it:** anything financial, contractual, or disputable; anything the operator can
change on a customer's behalf.

### H — Mutable row plus a history table

**Selects it:** the current value is read constantly and history is read rarely — which is the
usual shape; the questions about the past are known and few.

**Disqualifies it:** the history must be the source of truth rather than a record of it; the
two can disagree, and here they can.

**Costs:** two writes, and a trigger or a discipline that keeps them together.

### I — Append-only events, current state derived

**Selects it:** the sequence of changes *is* the domain — a ledger, a state machine whose
history is auditable, a workflow where "how it got here" is a business question.

**Disqualifies it:** ordinary CRUD. This shape makes simple reads expensive and simple
constraints hard to express, and it is chosen far more often than it is needed.

### J — Validity ranges on the row

**Selects it:** the business genuinely reasons in periods — a price that applies from a date, a
rate valid for a term, an assignment with a start and end.

**Disqualifies it:** you only want an audit trail. Then it is H, which is much simpler.

**Costs:** every query needs the period predicate, and forgetting it returns superseded rows.
Pair it with an exclusion constraint so two rows cannot be valid at once.

---

## Removal

### K — Actually delete the row

**Selects it:** the row has no downstream references and no retention obligation; a draft, a
cart, an expired token, a deduplication record past its window.

**Disqualifies it:** anything references it; anything must be recoverable; anything financial.

### L — A state column

**Selects it:** "deleted" is really a business state — cancelled, archived, withdrawn — with
its own rules about what may still happen to it. This is usually what a soft delete was trying
to be, said honestly.

**Disqualifies it:** nothing, usually. Prefer this over a boolean flag: the state has a name in
the business already, and the flag throws it away.

**Costs:** every query needs the predicate. Partial indexes on the live rows make that cheap
and make forgetting it visible in the plan.

### M — A tombstone: keep the row, destroy the contents

**Selects it:** a deletion obligation for personal data collides with a retention obligation
for the record — the usual case. The foreign keys survive; the personal data does not.

**Disqualifies it:** the obligation is to remove the record itself, not the data in it.

**Costs:** every column that is cleared must be nullable or have a defined tombstone value, and
unique constraints on cleared columns must be partial or they will collide on the second one.

### N — Move it to an archive table

**Selects it:** the live table's size is a real problem and archived rows are read almost never.

**Disqualifies it:** archived rows are still referenced by live rows; the schema then has to be
kept in two places, which it will not be.

### The blanket `is_deleted` column

**Never as a default.** Not because hiding rows is wrong, but because applying one mechanism to
every table means no table got a decision. It breaks unique constraints, it makes every query
wrong by omission, and it accumulates rows nobody can explain. Decide per table: K, L, M, or N.

---

## A fixed set of values

### O — `CHECK (col IN (…))`

**Selects it:** an internal set, small, changing rarely, with no labels or ordering needed.
Changing it is one `ALTER`, and ordinary string operators still work.

**Disqualifies it:** the values need display labels, translations, or an order; other tables
need to reference one.

### P — A native enumerated type

**Selects it:** the same set is used in several columns or in function signatures, and you want
the engine to reject a typo at parse time rather than at insert time.

**Disqualifies it:** values are removed or renamed with any frequency; the set is
customer-facing and product will want labels; the store does not have the type.

**Costs:** string functions do not apply without a cast, and removing a value is awkward.

### Q — A lookup table

**Selects it:** the values appear in a customer's vocabulary — a dropdown, an API response, an
invoice — or carry anything beyond their name: a label, an order, a validity period, a rate.

**Disqualifies it:** an internal state machine of four values. Then it is a join on every query
for nothing.

**The boundary, stated once:** whether the value lives in the customer's vocabulary or only in
an engineer's. Apply it consistently and say so in section 3, so the next table does not get a
different answer.

---

## "This already happened"

### R — A unique constraint on the operation itself

```sql
ALTER TABLE webhook_events
  ADD CONSTRAINT webhook_events_once UNIQUE (source, external_id);
-- INSERT … ON CONFLICT DO NOTHING
```

**Selects it:** the repeat carries an identifier the sender guarantees is stable — a webhook
event id, a message id, a caller-supplied idempotency key. This is the whole mechanism: the
insert and the work happen in one transaction, so a duplicate cannot half-apply.

**Disqualifies it:** the operation legitimately happens twice with identical contents and no
distinguishing identifier. Then the key must come from the caller, and if the caller cannot
supply one, the operation is not idempotent and section 9 has to say what that costs.

**The failure to avoid:** checking for the key, then doing the work, then recording it. The gap
between the check and the record is a race that lets both attempts through. The constraint has
to do the excluding, not a preceding `SELECT`.

### S — An idempotency key table with recovery points

**Selects it:** one logical operation spans more than one external call, and a retry must
resume rather than restart — a payment that charges, then provisions, then emails.

**Disqualifies it:** a single external call. Then R is enough, and this is a great deal of
machinery for it.

**Costs:** every phase boundary is a commit, and the phases must be written so that resuming
from any of them is correct.

**What it requires of the external service:** an idempotency key of its own, derived
deterministically from our stored state, so that a retried call is the same call. Without that,
an indeterminate failure — a timeout — cannot be retried safely, and the model has to record
the uncertainty rather than guess.

### T — An outbox table

**Selects it:** something outside must happen if and only if the transaction commits — an
email, a webhook, a queue message. The row is written in the same transaction and a worker
sends it afterwards.

**Disqualifies it:** the external call's *result* is needed to complete the operation. That is
S, not T.

**Costs:** at-least-once delivery, so the receiving side needs R.

---

## Variable attributes

### U — Columns

**Selects it:** the attributes are known, and any of them might be filtered, sorted, grouped,
or constrained. This is the default and needs no defence.

**Disqualifies it:** genuinely open-ended attributes defined by users at runtime.

### V — A JSON column beside the columns

**Selects it:** sparse or caller-defined attributes that are read as a unit and never
constrained; a raw external response kept alongside the normalised fields that were extracted
from it.

**Disqualifies it:** anything filtered or aggregated on regularly — the planner has no column
statistics for it and will guess; anything that needs a foreign key, a unique constraint, or a
`NOT NULL`.

**Costs:** no constraints, no referential integrity, and a schema that is now enforced in
application code. Every field that graduates to being queried should graduate to a column.

### W — Entity-attribute-value

**Selects it:** almost nothing, now. It is listed so it can be rejected on the record.

**Disqualifies it:** every query with two predicates becomes two self-joins, and ten becomes
ten. V does the same job with one column and better ergonomics; U does it properly when the
attributes are actually known.
