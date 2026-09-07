<!--
DATA MODEL TEMPLATE — copy this file to the PRD's folder as NN-data-model.md and replace the
content. Everything in [square brackets] is a placeholder; the prose around it is guidance for
whoever writes the document and is meant to be overwritten too. It is filled with a plausible
booking-and-payments example so the idioms are visible working — replace all of it.

DELETE THIS COMMENT in the copy. It is a note to the writer, not part of the document.

ALTITUDE. This document is a specification a schema is generated from, not the schema. It
carries SEMANTIC TYPES (ULID PK, timestamptz, integer minor units + currency, FK->run SET NULL)
and NAMED PREDICATES — never CREATE TABLE, never an ORM builder call, never migration syntax.
packages/db/src/schema.ts is written from this document by the code-db skill, one vertical slice
at a time, and two artefacts describing one schema drift. The cut is between syntax and rule: if
changing it changes which states are legal it stays here; if it changes only how the same states
are spelled, it belongs downstream. A predicate paraphrased into prose ("make sure bookings don't
overlap") has been deleted, not abstracted.

Structure: `##` per numbered section, `###` per table and per decision. Invariants are addressed
as a bold inline label at the start of their row — `**I1**` — and decisions as `### DM-1 — …`.
Markdown has no reliable cross-file or cross-viewer anchor, so a mention of `I1` or `DM-3`
elsewhere in the text is plain prose, not a link — write it so it reads fine either way.

Predicate blocks go in fenced code blocks with backticks, not indentation, so they render the
same in every viewer. `!=`, `<=`, `>=` need no escaping there — use them freely.

Keep every table to five columns or fewer. A wider table does not scroll in a terminal or a
narrow editor pane; it wraps, and a wrapped table is unreadable in exactly the tool this
document is most often read in.

Before reporting: skim the rendered file once. Check that no bracketed placeholder survived,
that every table renders as a table (a misaligned `|` is the usual cause), and that nothing
in it depends on being executed — a coding agent reads this file as text, never as a rendered
page.
-->

# Data model: [Product name]

Owner: [name] · Date: [YYYY-MM-DD] · PRD: [NN-prd.md] · Status: Draft

## Contents

1. [Headline](#1-headline)
2. [Invariants](#2-invariants)
3. [Conventions](#3-conventions)
4. [Tables](#4-tables)
5. [Decisions](#5-decisions)
6. [Enforcement map](#6-enforcement-map)
7. [Access paths and indexes](#7-access-paths-and-indexes)
8. [Growth, history and retention](#8-growth-history-and-retention)
9. [Concurrency and transaction boundaries](#9-concurrency-and-transaction-boundaries)
10. [Build order and seed](#10-build-order-and-seed)
11. [Assumptions and open questions](#11-assumptions-and-open-questions)

## 1. Headline

**[n] new tables on PostgreSQL 17 · keys are [uuidv7 / ulid / bigint identity] · money is
[integer minor units] · [what is append-only]**

[Two or three sentences: the shape of the model, what the central table is, and the one
structural choice that everything else follows from. Name the thing a reader would otherwise
get wrong — that the balance is derived rather than stored, that the seat is a counter rather
than a row, that history lives in a separate table. Do not list every table here; section 4
does that.]

**Life of a [booking]** — the events the business would name, left to right, and the rows each
one writes. This is not the write path in section 9, which is one transaction seen for
concurrency; this is the whole story seen for sense, and it is what lets a reader ask "why does
cancelling touch four tables?" or "why is there no row between browsing and paying?" before
reading a single column.

| Event      | Rows written                                       |
| ---------- | --------------------------------------------------- |
| [browses]  | [no row]                                             |
| [holds]    | [+booking (held); run.seats_taken +n]                |
| [pays]     | [+payment; +webhook event; booking → paid]           |
| [attends]  | [no row]                                             |
| [cancels]  | [booking → cancelled; run.seats_taken −n]            |
| [refunded] | [+credit entry; payment kept, never edited]          |

[One line on what to take from it — e.g. nothing is written until the hold, so an abandoned
browse leaves nothing to sweep, and a refund adds a row rather than editing the payment, which
is what keeps the money reconstructable.]

**Shape at a glance** — what holds what, and where the invariants live:

| Table            | References     | Cardinality | Note                                |
| ----------------- | -------------- | ----------- | ------------------------------------ |
| [workshop]         | —              | —           | [catalogue, no dates]                |
| [run]              | workshop       | 1:n         | [the contended row]                  |
| [booking]          | run, customer  | n:1 each    | [the only row that claims capacity]  |
| [payment]          | booking        | 0..1:1      | [append-only]                        |
| [credit_entry]     | booking        | 1:n         | [ledger, never updated]              |
| [webhook_event]    | —              | —           | [proves an external message applied once] |

[One line on the relationship carrying the hardest invariant — e.g. every rule about capacity
lives on the booking→run edge, because a booking is the only row that consumes a seat.]

Denormalisation spent: **[n] of 3** — see section 5. Storage at the PRD's volume:
**[n] MB**; at 10×: **[n] MB**, against the **[n] GB** the Neon plan includes — see section 8.

## 2. Invariants

What must never be true, taken from the PRD's business rules and journeys. This is the spine
of the document: every table, constraint and index below exists to serve one of these, and
every one is answered in section 6 by a named mechanism. Write each as a **negative statement
about state** — "no two bookings may…", "no payment may exist without…" — never as a sequence
of steps; a rule phrased as behaviour is a journey, and journeys belong to the PRD.

| # | Must never be true | From | Scope | Enforced by |
| - | ------------------- | ---- | ------ | ------------ |
| **I1** | [e.g. a run has more confirmed seats than its capacity, other than by an honoured late payment] | PRD §8, FR-12 | One row, contended | DM-3 |
| **I2** | [e.g. a payment exists with no booking, or a confirmed booking with no payment and no credit] | PRD §10 | Two tables, one transaction | DM-4 |
| **I3** | [e.g. a credit balance that does not equal the sum of the entries that created it] | PRD §8 | Derived value | DM-5 |
| **I4** | [e.g. the same external message applied twice] | PRD FR-48 | One row, idempotent | DM-6 |
| **I5** | [e.g. two customers hold the same [unique thing] at the same time] | PRD §8 | Across rows | DM-3 |
| **I6** | [e.g. a booking in a state its history does not justify] | PRD §6 | One row, over time | DM-7 |

**Deliberately not invariants.** State these positively — an omitted rule reads as an oversight,
and the next person to touch the schema will add a constraint for it.

- [e.g. Capacity is a soft cap: a payment authorised after its hold lapsed is honoured, so the
  remaining-seat count is signed and −1 is a legal state, not a bug. PRD §8.]
- [e.g. Nothing requires an email address to be unique across customers, because the PRD allows
  a person to book twice with different addresses.]

## 3. Conventions

Decided once here so section 4 can be read without re-explaining them, and so an agent adding a
table later does not invent a second convention beside the first. Each is a real choice with a
real alternative; the close calls are argued in section 5 rather than settled by this table.

| Convention | Decided | Because |
| ----------- | -------- | -------- |
| Names | [snake_case; tables plural; columns singular; `id` for the key; `<table>_id` for a reference] | [folded to lowercase by the engine anyway] |
| Name specificity | A table name answers **what its rows are when read cold** — no ERD, no neighbouring tables. Qualified only where the bare noun leaves a question open | [name the tables the test moved and the question it left open — e.g. `run` → `workshop_run`, "run of what?"] |
| Qualify siblings together | Where two concepts could both claim one word, **both** are qualified | [or say no such pair exists in this schema] |
| Primary keys | [UUIDv7 / ULID, generated in the application] | [against the test in DM-1; never both in one schema] |
| Public identifiers | [opaque, separate from the key / the key itself] | [whether a row's id may appear in a URL or API response] |
| Timestamps | `timestamptz`, stored UTC; `created_at` on every table | [a bare local timestamp discards the offset the PRD's timezone rule needs] |
| `updated_at` | [on which tables, maintained by trigger / application] | [trigger if anything reads it to decide what changed; application-maintained if only a screen renders it] |
| Money | [integer minor units, with the currency alongside] | [exact arithmetic; float cannot represent a cent] |
| Text | `text` with a `CHECK` where a real limit exists; never `char(n)` | [an arbitrary `varchar(n)` is a production error deferred] |
| Enumerations | [lookup table / native enum / check constraint — one rule, and when each applies] | [see DM-2] |
| Deletion | [what is actually deleted, superseded by a new row, or never removed] | [no blanket `is_deleted` flag — see DM-8] |
| Nullability | Required by default; a nullable column states its meaning in section 4. Nothing stands in for null | [a null with no stated meaning is where an agent puts whatever it has] |

**Every column is `NOT NULL` unless this document says what its null means.** "Not yet paid",
"never cancelled", "no note given" are three different facts, and the cheapest rule here is
simply asking which one a null would be.

## 4. Tables

One subsection per table, in dependency order — a table appears after everything it references.
Each carries a column table and a block of named predicates. Annotate only the columns carrying
a rule, a null meaning, or a unit; a column whose name and type say everything needs no note.

**Tables that already exist are context, not content.** `packages/db/src/schema.ts` is not
empty: [name what is there and what this document does with each — the Better Auth tables
(`user`, `session`, `account`, `verification`) are generated and are not redesigned here, so a
reference to a person is a foreign key to `user` and inherits its key type; `stripe_event` is
keyed on Stripe's own event id, which is its idempotency mechanism; `items` is scaffold demo
data]. Each appears below only where this model references or changes it.

**Types are semantic, not physical.** `ULID PK`, `timestamptz`, `integer minor units + currency`,
`FK→runs (RESTRICT)`, `enum{…}`, `text`, `text, max 200`. Whether `enum{…}` becomes a native
type, a check, or a lookup table is a decision with its own argument in section 5 — settling it
inside a type annotation hides it.

### [runs]

[One sentence: what one row is, in the business's own words. A reader who knows the business
and not the schema should be able to stop here.]

| Column | Type | Null | Meaning, unit, or what its null means | Serves |
| ------- | ---- | ---- | --------------------------------------- | ------- |
| `id` | ULID PK | no | [generated in the application — see DM-1] | |
| `workshop_id` | FK→workshops (RESTRICT) | no | [a workshop with runs cannot be removed] | |
| `starts_at` | timestamptz | no | | |
| `ends_at` | timestamptz | no | [stored rather than derived, because the PRD lets the operator change one run's length] | PRD FR-[n] |
| `capacity` | integer | no | [seats offered] | |
| `seats_taken` | integer | no | [confirmed and held seats; denormalised, see DM-3; may exceed `capacity` by design] | I1 |
| `cancelled_at` | timestamptz | **yes** | [null means the run is live; set once, never cleared] | I6 |
| `created_at` | timestamptz, default now | no | [engine-defaulted, holds for every writer] | |

Constraints:

```
runs_end_after_start      ends_at > starts_at
runs_capacity_positive    capacity > 0
runs_seats_not_negative   seats_taken >= 0
    [note: deliberately NOT seats_taken <= capacity — see I1; a soft cap may be exceeded]
```

### [bookings]

[Repeat the shape for every table. The central table deserves the most annotation; a lookup
table deserves two lines and no ceremony.]

Constraints:

```
bookings_confirmed_paid  state != 'confirmed' OR paid_at is not null
bookings_seats_positive  seats > 0
bookings_no_overlap      no two rows with the same resource_id may have overlapping
                         [starts_at, ends_at) where state != 'cancelled'
```

[The last one is prose because the mechanism is engine-specific while the rule is not — section
6 names what enforces it, and what happens if the store lacks it. The second is the conditional
not-null: the most under-used constraint there is.]

## 5. Decisions

The modelling calls that could honestly have gone the other way. A table that follows directly
from the PRD needs no entry here — this section is only for the places where a competent person
would have chosen differently, and it is where a reader should spend their disagreement.

**Rejected, and what replaced it** — the load-bearing calls only, four to six rows. A reviewer's
real question is "why not the obvious thing?"; this table answers it in five seconds.

| Rejected | Chosen | The one consequence that decided it |
| --------- | ------ | -------------------------------------- |
| [a row per seat] | [a counter on the run] | [seats are interchangeable and the count is read on every page] |
| [a balance column] | [append-only entries, summed] | [money has to explain itself; a stored balance cannot say how it got there] |
| [one table, a type column] | [a table per type] | [nullable-for-everyone means no type's rule can be expressed] |
| [a required unique FK] | [an optional FK on the many side] | [identical today, and does not forbid the second child a retake creates — DM-9] |

**Blast radius** — what each decision costs to reverse once rows exist, so a reader knows where
to spend their critique:

| Decision | Cost to reverse |
| --------- | ----------------- |
| DM-1 [Key strategy] | Highest — touches every foreign key in the schema |
| DM-5 [Ledger vs balance] | Highest — history that was never recorded cannot be rebuilt |
| DM-8 [History and deletion] | High |
| DM-3 [Capacity representation] | Moderate |
| DM-2 [Enumerations] | Low |

### DM-1 — [Primary keys: UUIDv7, generated in the application]

**Invariant or driver.** [Which invariant or PRD requirement forced this. If none did, say so —
it probably belongs in section 3, not here.]

**Decision.** [The choice, stated as it will be written in the DDL.]

**Why.** [Two or three sentences, argued against the alternatives below.]

**Alternatives rejected.**

- **[ULID]** — [near-equivalent; loses on storage (26 bytes as text vs 16), wins only where a
  human reads or types the id, which does/does not apply here].
- **[A database sequence]** — [smallest key at 8 bytes; loses because [what is created outside
  the database alongside a row]. Not because of round-trips or contention — see _Keys and
  identity_ in SKILL.md for why those two objections don't hold].
- **[UUIDv4]** — [the only key that doesn't disclose creation time; rejected generally because
  random keys scatter inserts, and nothing here needs the opacity].
- **[A natural key]** — [what makes the business value mutable, and what breaks when it changes].

**Consequences.** [What this buys, and the specific bad thing it costs.]

**Reversibility.** [What it would take to undo after launch.]

### DM-2 — [Enumerations: lookup tables for customer-facing values]

[Same five-heading shape. Keep going for every real decision; delete any heading set that
followed straight from the PRD without an argument.]

### DM-3 — [Capacity: a counter on the run, not a row per seat]

[Serves I1 and I5. Name the alternative and why it lost, or take it.]

### DM-4 — [Booking and payment written in one transaction]

[Serves I2. Name what is inside the transaction and what cannot be.]

### DM-5 — [Credit as an append-only ledger, not a mutable balance]

[Serves I3. Usually the highest-consequence decision in the document.]

### DM-6 — [External messages applied once, by unique constraint]

[Serves I4. Which column is unique, and what the second arrival does.]

### DM-7 — [State machine: a column plus what checks the transitions]

[Serves I6. If transitions are only checked in application code, this decision says so and
accepts it — see section 6.]

### DM-8 — [History and deletion]

[There is no blanket soft-delete flag. Say per table what deletion means: a status change, a row
in a history table, a tombstone that keeps references valid, or an actual delete. If any table
keeps rows that carry a unique value, say how uniqueness avoids colliding with the dead ones —
partial index over live rows, or the column cleared on removal.]

### DM-9 — [Cardinality: which relationships are 1:n rather than 1:1]

[The relationships where the first write path creates exactly one of each, and the model
deliberately does not say so. Name what each optional-FK choice keeps open — retakes,
created-anonymously-then-attached, records from another source.]

### Denormalisation ledger

Every stored value that duplicates something derivable is a value that can drift. Allow at most
two or three in the whole schema, spent only where a measured query made it necessary. Decide by
direction, not by derivability: a value that **tracks** its source is derived on read; a value
that **resists** its source — frozen as it was when the event happened — is captured, and is not
a denormalisation at all.

| Stored | Derivable from | Bought | Kept honest by |
| ------- | ---------------- | ------- | ----------------- |
| [`runs.seats_taken`] | [count of live bookings] | [the seat claim becomes one guarded statement instead of a count under a lock] | [the same statement that changes it, plus the nightly reconciliation in section 8] |

**What does not count.** A value that resists its source — the price paid on a booking, an
address at time of shipping — is a different fact that happened to match its source once, not a
copy of it. Do not enter these here; counting them inflates the ledger and it gets spent
defending them instead of the one or two places a real trade was made.

## 6. Enforcement map

Every invariant from section 2, and the exact mechanism that makes it impossible rather than
merely unlikely. **This is the one section where engine-specific names belong** — everywhere
else the document is neutral about the store, but "enforced by a constraint" is not checkable
and "an exclusion constraint over a range, needing an extension this plan permits" is.

| # | Mechanism | Level | What still gets through |
| - | ---------- | ------ | -------------------------- |
| I1 | [the guarded claim in section 9 — one statement whose predicate carries the limit, returning zero rows when it fails] | Statement predicate | [nothing but the honoured overage the PRD asks for] |
| I2 | [one transaction, plus required references in both directions] | Transaction | [a crash between the commit and the external call — handled in section 9] |
| I3 | [not stored at all: the balance is a sum over the ledger] | Structural — unrepresentable | Nothing |
| I4 | `webhook_events_once` — [a unique constraint with an insert that does nothing on conflict] | Constraint | Nothing |
| I5 | `bookings_no_overlap` — [an exclusion constraint over a time range, needing `btree_gist`; if unavailable this rule moves up to a transaction and a lock] | Constraint | [nothing while the extension is available] |
| I6 | [application code — the transition table is not enforced by the database] | **Application** | [any direct UPDATE, including a migration or console session; accepted because [reason]] |

**Read the "Application" rows as the risk register for this document.** Each is a rule that
holds only while every writer remembers it. Either push it down a level or say in one line why
it cannot go lower.

## 7. Access paths and indexes

Indexes follow queries, and the queries are the PRD's journeys. Every index below names the read
it serves; an index that names no read does not get created.

| Query | From | Index | Why this shape |
| ------ | ---- | ------ | ---------------- |
| [the catalogue: live runs, soonest first] | PRD §6.1 | `[(workshop_id, starts_at) WHERE cancelled_at IS NULL]` | [equality column first, range second; partial because most rows are live] |
| [one customer's bookings, newest first] | PRD §6.4 | [`(customer_id, created_at desc)`] | |

## 8. Growth, history and retention

How big this gets, what is kept, and what the PRD obliges us to hand back or destroy. Retention
is a business rule, not an operations detail: section 2's invariants are only true for as long
as the rows that support them exist.

| | At launch | At [metric] volume | At 10×, after 3 years |
| - | ---------- | -------------------- | ------------------------ |
| Storage | [n] MB | [n] MB | [n] MB |

Against the **[n] GB** the [named] Neon plan includes. [What to conclude — which table
dominates, and whether retention is a cost question or purely a compliance one at this volume.]

| Table | Mutability | Kept for | On the customer's request |
| ------ | ----------- | --------- | ---------------------------- |
| [payments] | [append-only — corrections are new rows] | [indefinitely; PRD §9] | [exported, never deleted — reason] |
| [credit_entries] | [append-only] | [indefinitely] | [exported] |
| [customers] | [mutable] | [indefinitely] | [name and email overwritten with a tombstone; row survives so bookings keep their FK] |
| [webhook_events] | [append-only] | [n days — the external service's own replay window] | [n/a] |

**History.** [Which tables record how they got to their current state, and how — a separate
history table, an append-only ledger, or nothing at all. "Nothing at all" is legitimate and must
be written down.]

**Reconciliation.** [For every denormalised value in section 5: the check that detects drift,
how often it runs, and what it does when it finds some.]

## 9. Concurrency and transaction boundaries

The rules that only break under simultaneity, and exactly what protects them. Name the contended
rows, the order every writer takes locks in, and where calls to systems we do not control sit —
because those cannot be rolled back and so cannot be inside a transaction.

**The write path** — [name the single flow carrying the most risk, e.g. pay for a booking]:

1. [begin transaction]
2. [claim the seat — one guarded UPDATE, zero rows means sold out]
3. [insert the payment]
4. [confirm the booking]
5. [commit]
6. [outside the transaction: call the payment provider / record the webhook — repeat-safe on a
   unique constraint, so a retry is a no-op]

[One line on what to conclude — e.g. nothing that cannot be rolled back happens while the
transaction is open, so a crash anywhere leaves either a held seat that expires on its own or a
payment the next attempt can still apply.]

| Contended thing | Who competes | Protected by | Loser sees |
| ----------------- | -------------- | -------------- | ------------ |
| [`runs.seats_taken`] | [two customers claiming the last seat] | [a single `UPDATE … WHERE seats_taken + :n <= capacity`, returning zero rows on failure — no read-then-write] | [zero rows updated, and the PRD's "this run just filled" screen] |
| [a held seat expiring mid-authorisation] | [the scheduled job and the webhook] | [the webhook re-claims rather than assuming the hold survived; capacity is a soft cap so it always succeeds] | [nothing — the overage is honoured by design] |

**Isolation level.** [The level everything runs at, and which specific rule needs more. Name the
anomaly, not the level: "two concurrent claims could both read the same seat count" is
checkable; "we use serializable to be safe" is not.]

**Lock order.** [The one order in which writers take locks, so two paths cannot deadlock — or
say that nothing ever holds two locks, which is the better answer.]

**Repeat safety.** [For every write triggered by something outside our control — webhook, retry,
overlapping scheduled job — the constraint that makes the second attempt a no-op.]

**Write order against the outside world.** [For every row created alongside something outside
the database, which is written first, and what the failure of the second leaves behind. Order so
a failure leaves *inert* garbage — an unreferenced storage object, not an orphaned live row.]

## 10. Build order and seed

What gets created in what order, and what must already exist for the PRD's build phases to be
verifiable. This is an order, not a script: the schema is built one vertical slice at a time
through `schema.ts`, and drizzle-kit generates each slice's SQL from the diff.

1. **[Foundation]** — [extensions, enumerations, the tables nothing references: catalogue,
   customers]. Verifiable when: [the catalogue page lists seeded rows]. Needs DM-1, DM-2.
2. **[The contended tables]** — [runs and bookings, with the capacity constraints from I1 and
   I5]. Needs DM-3.
3. **[Money]** — [payments, credit ledger, webhook events]. Needs DM-5, DM-6.

**Seed data.** [What must exist in every environment for the application to work at all —
lookup rows, the operator account, a currency — distinct from demo data, which a migration must
never create.]

**Changing this later.** [The policy in three lines: before production data exists, migrations
may be edited freely except any already applied anywhere, local included; after, every change is
additive first — add, backfill, move readers, remove in a separate release. Name any operation
that takes a heavy lock on PostgreSQL 17.]

## 11. Assumptions and open questions

**Assumptions** — proceeding on these unless corrected.

- [assumption, e.g. volumes in section 8 are the PRD's success metric, not a forecast] — affects DM-3

**Open questions** — need an answer before the affected decision is safe.

- [question] — blocks DM-5

**Back to the PRD** — rules the modelling exposed as underspecified. A business rule that cannot
be written as an invariant is usually a rule that was never decided, not one that is hard to
store. Record it; do not decide it here.

- [e.g. the PRD does not say whether a credit expires, which decides whether the ledger needs a
  validity period] — PRD §8

**Back to the scaffold** — anything this model needs that the store, the plan, or Drizzle does
not give, or that changes a cost. Not a decision to make here: the stack was settled when the
scaffold was built.

- [e.g. requires the `btree_gist` extension, which the Neon plan does/does not allow — checked
  [date], [source]]
- [e.g. the constraint is not expressible through Drizzle's builder, so the generated migration
  is extended by hand before it is applied — a rung in section 6, not a downgrade of the rule]

This document is proposed, not agreed. Critique it before anything is built — every constraint
here is cheap to change now and expensive to change once there are rows.
