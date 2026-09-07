---
name: db-design
description: Turn an approved nanostore PRD into a numbered data model document the schema is later generated from - every table, column, constraint and index traced to a business invariant, every modelling call argued with its alternatives, and every rule enforced at the lowest level that can express it. The document is a single self-contained Markdown file carrying semantic types and named constraint predicates rather than DDL or Drizzle code, readable by a human in any editor and by an agent reading only its text. Use when the user asks to design a schema, model the data, decide tables, keys and constraints, or answer "how should this be stored" once a PRD exists. It writes a document and nothing else - `code-db` writes schema.ts, and the `feature` skill decides when.
---

# PRD to data model

Turn an approved PRD into a **data model document**: the invariants written as things that must
never be true, the tables that hold them, and — for each invariant — the exact mechanism that
makes it impossible rather than merely unlikely.

**The store is not a question here.** The scaffold settled it: PostgreSQL 17, Docker locally on
port 5434 and Neon in production, reached from the Workers through Hyperdrive, with Drizzle
owning `packages/db/src/schema.ts` and drizzle-kit generating the migrations. What that leaves
open is narrow and worth checking rather than recalling — which extensions the Neon plan
permits, what it includes in storage — and the `code-db` skill holds the mechanics.

## Core principle

A schema is not a place to put the data. It is **the set of states the product is allowed to be
in**, and everything else — the application, the next coding agent, a migration, a background
job, an operator with a SQL console at 2am — is a client of it. The value of the document is the
reasoning about what must never be true; the column list is the residue.

Three rules govern every choice:

1. **The PRD's rules decide, not the query patterns.** Every table, column and constraint traces
   to an invariant, a journey, or a stated retention obligation. Model what is true; tune for
   what is read afterwards, and only with a measurement in hand.
2. **Enforce at the lowest level that can express the rule.** Structure beats a type, a type
   beats a constraint, a constraint beats a transaction, and all of them beat application code —
   because the database is the only layer every writer passes through. See _The enforcement
   ladder_.
3. **Prefer making a bad state unrepresentable to detecting it.** A balance that is always a sum
   cannot disagree with its entries. A column that does not exist cannot be wrong. This is the
   only move that removes a class of bug instead of guarding against it, and it is almost always
   available at design time and never available later.

The document answers **what is true**. It never re-opens **what** — that is the PRD — or **what
it runs on** — that was settled when the scaffold was built. If you find yourself deciding a
business rule, that is a PRD gap; if you find yourself preferring a different store, that is a
question for the scaffold. Both go to section 11 and get sent back.

## Folder convention

The data model lives in the PRD's folder, at the next free number:

```
docs/<dated-folder>/
  01-brief.md
  02-questions.md
  04-prd.md
  05-data-model.md   <- next free number
```

The folder is the PRD's own — a dated one under `docs/`, never `docs/2026-08-08-setup`, which
`scripts/docs-check.mjs` owns and would read a dropped-in file as drift.

**A new number is for new material. A correction goes back into the document it corrects.**

- **Additive** — a questionnaire after a brief, a PRD after the questionnaires, a data model
  after the PRD. Each is new material that does not invalidate what came before. Take the next
  free number; never overwrite or renumber one.
- **Corrective** — the same model, revisited: an invariant was misread, a constraint turned out
  unenforceable on the Neon plan, the user pushed back and was right. **Edit the data model in
  place.** Do not write `06-data-model.md` beside `05`.

Correcting in place is not a loss of history — the previous revision is one `git log -p` away.
Leaving the superseded document on disk costs real harm: it is a confident specification telling
an agent to build the schema you just decided against, with nothing saying which one wins.

What a corrected document must carry:

- The superseded shape **named as a live alternative**, present tense, in _Alternatives
  rejected_ and the blast-radius table. A reversal that hides what it beat reads as fashion.
- Nothing else. **Do not narrate the revision.** No "this was revised", no pointer to the old
  one. The test is that a corrected document reads **as though it had been written once,
  correctly**.

## Steps

1. **Read the whole folder in number order** — brief, every questionnaire, the PRD — plus
   **`packages/db/src/schema.ts`**, which is short and is the other input. This repo is not
   greenfield: the Better Auth tables generated into `src/auth-schema.ts` are not yours to
   redesign, `stripe_event` is keyed on Stripe's event id because that conflict _is_ the
   webhook's idempotency mechanism, and `items` is scaffold demo data. A model that ignores them
   invents a second `user` table.

   If there is no PRD, stop and say so — run `brief-to-prd` first.

2. **Extract the invariants.** Go through the PRD section by section and write down every line
   that constrains state, keeping the reference:

   | PRD section              | What to pull out                                                                                      |
   | ------------------------ | ------------------------------------------------------------------------------------------------------ |
   | 6 — Journeys             | The writes each journey performs, and the failure branches, where models are thinnest                  |
   | 8 — Business rules       | Money, time, capacity, concurrency — the bulk of section 2                                              |
   | 9 — Data                 | The entities, what is retained, what is exportable, who may see what                                   |
   | 10 — Constraints         | Compliance, residency, what may never be stored at all                                                 |
   | 11 — Acceptance criteria | Already checkable statements; several are invariants in all but name                                    |
   | 13 — Phases              | The order things must exist in, which decides migration order                                          |

   And from the repo: anything belonging to a person references the generated `user` table and
   inherits its key type — a third of the key strategy already settled by a file nobody hand-edits.

3. **Write the invariants down before you draw a single table.** As negative statements about
   state — never as sequences of steps; the procedural form is a journey and already belongs to
   the PRD. This is the step that gets skipped, and skipping it produces a schema that stores the
   nouns and enforces nothing.

4. **Sweep `references/invariant-checklist.md`** for what the PRD did not say. Most entries come
   back "the PRD is silent" — each becomes a blocking question or a recorded assumption. A rule
   that is genuinely absent gets stated positively in section 2's _deliberately not invariants_
   list, or the next person will add a constraint for it.

5. **Settle the context from evidence, not from questions.** Everything in _What to settle before
   modelling_ is answerable by looking at the repository and the PRD; record each answer as a
   stated assumption. Do not open a questionnaire round, and do not open with inline questions.

6. **Settle what propagates, and test what is about to be claimed.** The **key strategy** (see
   _Keys and identity_, settled together with anything created outside the database) and **what
   is append-only** both reach into every table and are the most expensive things in the document
   to change once rows exist. Before anyone writes that the store cannot express something,
   **write the rule out as an actual predicate and check it** — see _Writing rules_. Batch every
   capability and limit question the model needs and send them out at once — see _Context and
   fan-out_ — rather than fetching a documentation page mid-argument.

7. **Draft the whole document in one pass.** One `general-purpose` agent models every table and
   writes the whole document in a single context, applying the tests in _How to model_ and the
   ladder in _The enforcement ladder_, using `references/modelling-patterns.md` to **eliminate,
   never to pick** — a shape chosen from that file and justified afterwards is the exact failure
   the _Core principle_ forbids. Hand it what _Draft and check_ lists.

   If a data model already exists for this PRD and this run is correcting it, it edits that file
   in place. Otherwise it copies `references/data-model-template.md` to the next free number
   (`NN-data-model.md`) and replaces the content — its head comment is a note to the writer, not
   part of the document, and gets deleted from the copy. Invariants are labelled `I1`, `I2` …
   and decisions `DM-1`, `DM-2` …

8. **Check it once, yourself.** Read the draft against _Draft and check_'s list. If something is
   genuinely wrong, send one batch of fixes to a single revision pass by the same agent — not a
   panel, not a second round. If nothing on the list fails, it is done.

9. **Report** the file path, the table count, the invariant count and how many the database
   enforces, how many denormalisations were spent, and any open question that section 11 carries.
   Say the file opens in any text editor or markdown viewer. State that the user should critique
   it before any migration is written, because every constraint in it is cheap to change now and
   expensive to change once there are rows.

   Then stop. **Do not touch `schema.ts`, do not run drizzle-kit, do not create the database, and
   do not commit.** Once the user has approved the document, it is built the way everything else
   here is built: from the `feature` skill, one vertical slice at a time through schema → api →
   web, with section 10's order deciding which slice can come first.

## Context and fan-out

The input to this document is a PRD, and it is not what overruns a run — **page content is**:
PostgreSQL's constraint reference, Neon's limits page, Drizzle's migration documentation, each
thousands of tokens of which two lines decide anything.

**Delegate the looking up. Never delegate the invariants.**

Send these as `general-purpose` subagents, in one message so they run concurrently. Each writes
to `<scratchpad>/findings-<topic>.md` and returns a compact summary — facts only.

| Delegate                                                                                                                                                                                                                                                            | Because                                                                                                                                                                                     |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Neon capability checks** — which extensions the plan permits, whether `EXCLUDE`, partial indexes and deferred constraints are available, the identity/UUID generation options in the major version Neon runs | _Writing rules_ requires these be tested rather than recalled. Local Docker is Postgres 17; do not assume production matches it             |
| **Neon plan limits** — storage included, row or connection ceilings, backup and point-in-time recovery window                                                                                                                                                       | Section 8 prints them. Which plan this project is on is itself often an assumption — say so in §11 rather than inventing a tier                                                             |
| **Drizzle expressibility** — whether the mechanisms this model rests on can be declared in `pgTable`, or need SQL appended to the generated migration by hand                                                                                                       | A rule Drizzle's builder cannot spell is still enforceable; what changes is who writes the SQL, and that belongs in section 6                                                              |

Tell each agent the invariant it is serving and the exact question — "Does Neon's free plan
allow `CREATE EXTENSION btree_gist`?" comes back usable; "research Neon" comes back as a
brochure. **Require every answer with its source URL and the date checked** — the document
prints both.

- **The invariants, the key strategy, and the append-only decision stay with you.** They are a
  single consistent set that everything else cross-references; two agents deciding any of them
  independently produce two answers that disagree.
- **The tables go to one agent — all of them, or none.** A schema is one artefact: foreign keys,
  naming and key strategy must agree across every table. One agent writing every table is not a
  fan-out; six agents writing six tables is, and reconciling them afterwards costs more than
  writing them once.
- **Copy the template, don't second-guess it.** Copy `references/data-model-template.md` and
  replace its content; its structure has already made the layout decisions.

## Draft and check

The draft agent gets, explicitly:

| Hand over                                                                 | Because                                                                    |
| ------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| The numbered invariant set from step 3, final                             | It models against them. It does not get to invent them                    |
| The checklist sweep from step 4 — assumptions taken, gaps found           | Otherwise it re-derives them, differently                                 |
| The settled context from step 5, with the evidence that settled each one  | §11 prints both                                                            |
| The key strategy and the append-only decision from step 6                 | They reach into every table                                               |
| The store, its version and plan, and every findings file from the lookups | With source URLs and dates — the document prints them                     |
| The destination path, already copied from the template                   | See step 7                                                                 |

It may not invent or amend an invariant, re-decide the product or the store, or write DDL.
Anything it finds missing comes back as a question and lands in §11 — a drafting agent that
quietly resolves a PRD gap has made a product decision nobody reviewed.

**Then check it yourself, once, against this list** — it is what six critics would otherwise
each own a slice of, collapsed into one pass because a document is drafted once and should be
right once, not negotiated into shape over rounds:

- **Traceability, both ways.** Every table and constraint names the invariant that forced it;
  every invariant in section 2 is answered in section 6, even where the answer is "application
  code, because …".
- **The enforcement ladder was actually climbed.** No rule sitting in application code that is
  one `OR` away from a `CHECK`; no "cannot be expressed" that was recalled rather than tested;
  every foreign key has an explicit `ON DELETE`; no nullable column without a stated meaning.
- **Identity and shape hold up.** No forced 1:1 that only the first write path justifies, no
  cardinality that is true today and false in a year, no bad state left representable that could
  have been made impossible.
- **The expensive domains are honest.** No bare money amount, no local timestamp, no
  read-then-write without a guard, no stored balance without a reconciliation, no outside call
  sitting inside a transaction.
- **The budget was spent, not inflated.** At most two or three denormalisations, each with a
  reconciliation; nothing counted that resists its source rather than tracking it.
- **It is buildable.** No question the first migration would have to ask that the document
  cannot answer, and no journey — especially a failure branch — that cannot be executed against
  these tables.
- **The altitude held.** No `CREATE TABLE`, `ALTER TABLE`, `pgTable`, or `drizzle` anywhere in
  the text — a schema file is generated from this document, not written beside it.

If a finding re-opens the PRD or the stack, it is not a defect in this document — put it in §11
as a question instead of sending it back for revision. If the check turns up nothing on this
list, the document is done; do not manufacture a finding to justify a second pass.

## What to settle before modelling

**This skill does not open with questions.** Five things are settled by looking, not asked. A
question whose answer is on disk spends the user's attention confirming your own reading; a
question whose answer would not change a single column buys nothing.

State each in section 11 as an assumption with the evidence that settled it, and raise it as a
question only under the condition in the third column:

| Settle                                                                                                                                                                                                                  | Where you look                                                                                                                                                                                                                                                      | Raise it only if                                                                                       |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| **Does a database already exist with rows that matter** — decides how section 10 is written | `packages/db/migrations/` and whether the Neon instance has been deployed against. The local Docker database never counts: it is thrown away | production has rows and you cannot tell whether any of them matter |
| **Every path that writes to this database** | already decided: `apps/graphql` through Hyperdrive, drizzle-kit from a developer machine, Better Auth's own writes. `apps/web` never connects | never; if a PRD implies a fourth writer, note it in §11 |
| **Data to import** — a spreadsheet, an old system, a payment provider's history | the PRD's scope, data and phases sections | the PRD names a predecessor system but not what comes across |
| **Naming or structural convention in force** | `schema.ts`: snake_case tables/columns, camelCase exports, what Better Auth's tables already do to both | the existing tables contradict each other |
| **Anything deletable or exportable on request** | the PRD's data and constraints sections | the PRD is silent _and_ it stores personal data |

**Do not ask how many writers there are** to decide where a rule lives — the ladder does not
consult that answer. Do not ask which tables they want, or whether to use UUIDs; those are the
decisions they came here to have made.

## How to model

Apply these tests to every table and every column, in order. The first failure ends the
question — there is no point tuning a column that should not exist.

- **Does it trace to an invariant or a journey?** No trace, no table.
- **Is this one thing or two?** An entity has independent identity and lifecycle. If a "join
  table" carries a quantity, a price, a role or a date, it is an entity and deserves a business
  name.
- **Is the cardinality the one that holds over the lifetime, or only at the first write?** Two
  rows created together today are not 1:1 forever. Can the child outlive the parent (→ optional
  reference, `SET NULL`); can the parent ever have a second child — a retake, a revision, a
  repeat purchase (→ it is 1:N, reference on the many side); will the two ever be created
  independently (→ decouple them). See _The forced 1:1_.
- **Does the name say what the rows are when read cold?** No ERD, no neighbouring tables, no
  product context — a constraint violation in a log, a migration file, a `\dt` listing are all
  places relationships are invisible. `run` becomes `workshop_run` ("run of what?"); `seat`
  becomes `booking_seat`. A prefix earns its place only by answering the open question a bare
  noun left. Where two concepts could both claim one word, **qualify both** — letting the earlier
  one keep the bare noun makes the pair read as a subtype relationship that does not exist.
- **Does anything outside the database get created with this row?** A file, a blob object, a
  payment-provider record. If so, the id strategy and the write order are one decision — see
  _Ordering writes against the outside world_.
- **Can the bad state be made unrepresentable?** A balance that is always a ledger sum, a status
  derivable from timestamps, a "primary" flag a foreign key would replace — each is a constraint
  never written, tested, or reconciled.
- **What is the lowest level that can enforce it?** Walk it down the ladder and record where it
  stopped.
- **What does a null mean in this column?** No answer means `NOT NULL`.
- **What happens when the parent goes away?** Every foreign key gets an explicit `ON DELETE`.
  `CASCADE` is only right when the child has no independent existence.
- **What breaks with two writers at once?** Read-then-write is the shape to look for. Most
  collapse into one guarded statement; the rest need a named lock order in section 9.
- **What does this cost to change once there are rows?** A key strategy is months; an index is
  minutes; a column name is a migration.
- **Is it the shape whose failure modes are already documented?** Where two shapes both express
  the invariant, take the ordinary one — its ways of breaking are known, and the next maintainer
  has seen it before. The unusual shape wins only when it expresses an invariant the ordinary one
  cannot.

### Keys and identity

Settle this before any table — it propagates into every reference and is the most expensive
thing in the document to reverse.

**Default to a key the application can generate: UUIDv7, or ULID where a codebase already uses
it.** Both are 128 bits with a leading millisecond timestamp, sort chronologically, and leak
creation time identically. The reason to prefer them over a database sequence is not that
sequences are slow — they are not — it is that **knowing the id before the write buys you the
choice of write order** (see below), and a client-generated id makes a retried create deduplicate
on the primary key without a separate idempotency table.

Between the two: **UUIDv7 for a new schema, ULID where one is already established.** v7 is RFC
9562 and stores in a native 16-byte `uuid` column, generated in the application on this
PostgreSQL version. ULID has no native type, so it costs 26 bytes as text everywhere it appears;
its real advantage is an encoding that a human can read, type or dictate — if that is the
requirement, a separate short public reference beside the key is usually the better answer than
choosing the key's encoding for it.

**When creation time must not leak, the fallback is UUIDv4, not v7** — v7 leaks exactly as ULID
does. This is the one case a random key is correct, at the cost of index locality.

**A database sequence is still right** when nothing is generated outside the database, nothing
is exposed, and the smallest key matters — a bigint is 8 bytes against 16 or 26. Take it
deliberately, not by default.

Two arguments against sequences that are wrong, so nobody rebuilds a decision on them: **no
extra round-trip** — `INSERT … RETURNING` yields the id with the write. And **sequence
contention is not the bottleneck people think** — `nextval()` takes no row lock and caches per
session; the real contention under heavy insert is the **right edge of the index**, which
time-ordered keys hit too. They remove generation contention, not insertion contention.

### The forced 1:1

A **required, unique reference is a forced 1:1**: it forbids the parent existing without the
child _and_ the child ever having siblings. It is correct only when the child is a pure
extension of exactly one parent, permanently — a profile row hanging off a user. It is wrong far
more often than it is written, because the first write path really does create one of each, and
the constraint records that accident as a law.

When unsure, take **an optional reference on the many side**. It behaves identically today and
keeps three futures open for nothing: retakes, created-anonymously-then-attached, and records
arriving from another source. Choosing 1:N up front is free; retrofitting it onto a shipped
forced 1:1 is a data migration.

### Ordering writes against the outside world

When a row is created alongside something outside the database — blob storage, a file, a
provider-side record — the two writes can fail independently, and one ordering leaves much worse
wreckage.

**Order them so a failure leaves inert garbage rather than visible-but-broken state.** An
orphaned storage object is inert and swept later; an orphaned row is live, appears in listings,
and points at nothing. That means writing the external thing first and the row second, which
needs the id to exist before either write — what _Keys and identity_ buys. A database-generated
key forces the order the other way, and the compensation is a delete against a row that may
already have been read.

Nothing outside the database may sit inside an open transaction, so the recovery is always a
compensating action. Where neither direction is cheap, write the row first in a pending state, do
the external write, then mark it ready — and sweep the rows that never got there.

## The enforcement ladder

For every invariant, start at the top and take the first level that can express it, then record
which one it landed on — section 6 is that list, and the _Application_ rows are the document's
risk register.

**Assume more than one writer — as a threat model, not as an architecture.** Routing every write
through one application is good discipline, but it is a convention, held by everyone remembering
it; the schema is what makes the invariant true whether or not it holds. There is almost always a
second writer already — a migration backfilling a column, a console session fixing a row nobody
could fix through the app.

| Level               | Mechanism                                                                                          | Holds against                                     |
| ------------------- | ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| **Structure**       | The state cannot be written down — the column does not exist, or a foreign key makes it impossible | Everything, forever                               |
| **Type and domain** | `timestamptz`, integer minor units, `NOT NULL`                                                       | Every writer                                      |
| **Constraint**      | `CHECK`, `UNIQUE`, `FOREIGN KEY`, `EXCLUDE`                                                           | Every writer                                      |
| **Index**           | Partial unique index — "one _active_ subscription per customer"                                     | Every writer                                      |
| **Transaction**     | Two tables that must agree, committed together                                                       | Every writer that uses one                        |
| **Trigger**         | Last resort inside the database                                                                      | Every writer, at the cost of action at a distance |
| **Application**     | A rule held in code                                                                                   | Only the writers that remember it                 |

**Enforcement has a runtime cost, and it is not the one people assume.** Most rungs are close to
free: a `CHECK` is a predicate over a row already being written, and `UNIQUE` rides an index the
read path likely wanted anyway. The application-side alternative to uniqueness is a
read-then-write — more work, not less, and still wrong in the gap between the two calls. Three
mechanisms do cost enough to argue about, in section 6 next to the rule: **foreign keys** (a lock
on the parent row — real contention only with many children on one hot parent), **`EXCLUDE`**
(needs GiST), and **triggers** (actual execution on the write path). If write volume is the
reason a rule leaves the schema, cite the number from the PRD next to it.

**Conditional not-null is the most under-used rung.** "A confirmed booking must have a payment"
is `CHECK (state <> 'confirmed' OR paid_at IS NOT NULL)`, not application logic.

**"The database cannot express this" must be tested, not recalled.** Write the rule as an actual
predicate and check it against the current PostgreSQL documentation for the version Neon runs.
If it turns out expressible, you have moved a rule up two rungs for the cost of one line.

**A mechanism Drizzle cannot spell has not fallen off the ladder.** Where `pgTable` cannot
declare it, drizzle-kit still generates the migration and a line is added to that SQL file by
hand — which `code-db` already permits. Say in section 6 which of the two writes it.

**Landing on _Application_ is allowed, and must be justified in one line.** "A four-state
transition table needs a trigger, and a trigger is harder to reason about than the invariant it
protects" is a real answer; silence is not, and neither is listing it as enforced when it isn't.

## Denormalisation budget

Every stored value that duplicates something derivable is a value that can drift, and each needs
a mechanism that keeps it honest. Allow **at most two or three in the whole schema**, spent only
where a measured query made it necessary.

Keep the count as a short ledger — what is stored, what it is derivable from, what it bought, and
what keeps it honest — with a reconciliation for each in section 8.

**Decide by direction, not by derivability.** "Could this be computed?" is almost always yes and
settles nothing. Ask whether the value must **track** its source or **resist** it:

- **Track** — it should follow the source as the source changes: a displayed total, a run's
  remaining seats. **Derive on read.**
- **Resist** — it must stay frozen as it was when the event happened: the price on a paid
  booking, the address on a dispatched order. This is not a denormalisation at all — it is a
  different fact that happened to match its source once — and does not cost a budget entry.

**When you do capture a computed verdict, capture the least that cannot be recomputed.** A sum,
an average, a lookup off a stored column all re-derive for free and must not be stored. What
would come out differently if the deriving logic were re-run after tuning is the only candidate.

**Better still, when the computation is fixed logic over tunable parameters, capture the
parameters, not the result** — a version key resolving to version-controlled config, recomputed
on read. A captured result silently disagrees with a fresh recompute once parameters move;
captured parameters always reconcile. Capture the output only once the logic itself can change
past results, and only if published versions are immutable — see _Captured parameters, verdict
recomputed_ in `references/modelling-patterns.md`.

**Never justify a captured value by "it saves maintaining code."** It does not — the code that
renders the stored shape still has to exist. A stored value is justified only by freezing a
version-sensitive verdict.

## Writing rules

**Semantic types and named predicates — never DDL, never ORM code.** A schema file is generated
downstream from this document; if the document also carries `CREATE TABLE` blocks, there are two
schemas and they will disagree. The cut runs between _syntax_ and _rule_:

| Belongs downstream                                                      | Belongs here                                                                                       |
| --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `text` vs `varchar`, the ORM's builder call, index and migration syntax     | The semantic type: `ULID PK`, `timestamptz`, `integer minor units + currency`, `FK→run (SET NULL)` |
| How a constraint is spelled on a given engine                              | The constraint's **name** and its **predicate**                                                    |
| Which extension provides a mechanism                                       | That the rule must be enforced, and at which rung                                                  |

**The test: if changing it changes which states are legal, it stays. If it changes only how the
same states are stored or spelled, it goes.** A predicate written down as prose — "make sure
bookings don't overlap" — has been deleted, not abstracted.

So a table is a column table plus a fenced block of named predicates:

```
runs_end_after_start      ends_at > starts_at
runs_capacity_positive    capacity > 0
webhook_events_once       unique (source, external_id)
bookings_confirmed_paid   state != 'confirmed' OR paid_at is not null
bookings_no_overlap       no two rows with the same resource_id may have overlapping
                          [starts_at, ends_at) where state != 'cancelled'
```

The last one is prose because the mechanism is engine-specific while the rule is not — that
split belongs in the enforcement map.

**Name every constraint, and treat the name as design.** `bookings_no_overlap` says what was
violated; `bookings_check2` sends the reader to find out.

**Value sets are named, not resolved.** Write `enum{draft, live, cancelled}` in the column
table; whether it becomes a native enum, a check, or a lookup table is its own decision.

**A rejection on capability must be tested, not recalled.** Facts about design (what a type
means, what normalisation costs) are safe from knowledge. Anything of the form "cannot", "does
not support", or about a specific version or managed plan needs checking.

**Every nullable column states what its null means, and nothing stands in for null.** "Not yet
paid", "never cancelled", "no note given" are different facts; a `0` for "not measured" or a
sentinel date for "never" is a value that arithmetic and sorting will silently treat as real.

**`created_at` on every table; `updated_at` only with a trigger.** A creation timestamp
defaulted by the engine holds against every writer, including a migration and a console session.
`updated_at` maintained by application code is correct only for writes that go through that code
— if it drives a sync cursor or cache invalidation it needs a `BEFORE UPDATE` trigger; if it only
renders "last modified" in a UI, application-maintained is fine and the tolerance is stated.

**Non-choices are stated positively.** "No history table on `customers`: the PRD asks no
question about a previous email address, and adding one later cannot recover what was not
recorded." Silence is what an agent fills in on its own initiative.

**Volume figures are dated and sourced.** Row counts and storage estimates in section 8 come
from the PRD's success metric and show their arithmetic — bytes per row times rows per year.

**Traceability both ways.** Each table and constraint names the invariant that forced it; each
invariant in section 2 is visibly answered in section 6.

## Markdown output

The document is **one file that opens in any text editor or markdown viewer**. Nothing else may
be needed to read it — not a server, not a build step, not an internet connection.

- **Self-contained.** No images, no embedded scripts, no links to external files.
- **Standard markdown only.** One `#` title, `##` per numbered section, `###` per table and per
  decision, `####` for the labelled blocks inside a decision. Real markdown tables throughout.
- **Invariants and decisions are addressable by a bold inline label** — `**I1**`, `### DM-1 —
  …`, `### [runs]` — since markdown has no reliable cross-viewer anchor. A mention of `I1` or
  `DM-3` elsewhere is plain prose that still reads correctly whether or not the viewer links it.
- **A contents list at the top**, one line per section, linking to the heading's generated
  anchor. Match the heading text exactly, since the anchor derives from it.
- **Predicate blocks in fenced code blocks**, backtick-fenced rather than indented, using `!=`,
  `<=`, `>=` freely — nothing needs escaping there.
- **Keep tables to five columns or fewer.** A markdown table does not scroll; a wide one wraps in
  a terminal or narrow pane and stops being readable in exactly the tool this is read in most.
- **No diagrams.** What an ERD or a lifecycle diagram would draw, a table says instead: the
  lifecycle of the central object is an event → rows-written table (section 1), the load-bearing
  modelling calls are a rejected → chosen comparison table (section 5), and every relationship
  already appears as a foreign key in section 4. A reader scans a table faster than they read a
  picture's caption, and the document's second reader — a coding agent — never renders one
  anyway.
- **Two readers, one file.** A human reads this in an editor; a coding agent generating the
  schema reads only its text. Nothing may exist only in something that must be rendered to be
  seen — section 4's column tables and section 6's enforcement map are the authoritative,
  complete, text-only statement of the model.

Markdown is the presentation. It does not license a longer or more decorated document, and it
does not change what a table has to contain: a type, a constraint, an invariant it serves, and a
stated meaning for every null.

## Constraints

- **Never re-decide the product.** No new features, no changed business rules, no altered scope.
  If a PRD rule cannot be expressed as an invariant, it usually has not been decided — record it
  in section 11 and let the user amend the PRD.
- **Never re-decide the stack.** PostgreSQL, Neon, Hyperdrive and Drizzle were settled when the
  scaffold was built. If the model needs something they cannot do, that is a note in section 11,
  not a substitution made here.
- **Do not touch `packages/db`, run drizzle-kit, create the database, or commit.** The document
  is the whole deliverable; `schema.ts` is written from it later, by `code-db`, inside a vertical
  slice the `feature` skill owns.
- **Stay engine-neutral in the design, engine-specific only in the enforcement map.** Which
  extension provides a mechanism, and whether Drizzle can declare it or the migration needs a
  hand-written line, belongs in section 6, not inside a type annotation.
- **No table without a trace, no invariant without a mechanism.** If the PRD is silent on
  something the schema seems to need, record it as an assumption rather than inventing a
  requirement. Every row in section 2 is answered in section 6, even when the answer is
  "application code, because …".
- **One draft, one check, one revision at most.** A second full pass means something upstream
  was wrong — a PRD gap or a stack question — and that goes to section 11, not into a third
  attempt at the same document.
- **Do not modify the PRD, the brief, any questionnaire, or `docs/2026-08-08-setup`.** They are
  inputs, and the last one is the scaffold's plan of record.
- **Do not hide a guess.** Anything decided without evidence — a volume, a limit, a capability —
  goes in section 11, visible.
