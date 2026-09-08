---
name: design-db
description: Turn an approved PRD into a numbered data model document a schema is later generated from - every table, column, constraint and index traced to a business invariant, every modelling call argued with its alternatives, and every rule enforced at the lowest level that can express it. The document is a single self-contained Markdown file carrying semantic types and named constraint predicates rather than DDL or ORM code, readable by a human in any editor and by an agent reading only its text. Use when the user asks to design a schema, model the data, decide tables, keys and constraints, or answer "how should this be stored" once a PRD exists. It writes a document and nothing else - writing the schema from it, and deciding when, are separate steps.
license: MIT
---

# PRD to data model

Turn an approved PRD into a **data model document**: the invariants written as things that must
never be true, the tables that hold them, and — for each invariant — the exact mechanism that
makes it impossible rather than merely unlikely.

**The store is a fact, not a decision made here.** If your project has already settled one —
the engine, its version, where it runs, the ORM or migration tool that owns the schema file —
state it and move on. What is worth checking rather than recalling is narrower: which
extensions the plan permits, what it includes in storage, whether the ORM's builder can
declare a given mechanism. If nothing has been decided yet, record the store as an assumption
in section 11 rather than choosing one here — that decision belongs to whoever owns the stack.

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
it runs on** — that is a decision made elsewhere, if it has been made at all. If you find
yourself deciding a business rule, that is a PRD gap; if you find yourself preferring a
different store, that is a question for whoever owns the stack. Both go to section 11 and get
sent back.

## Folder convention

The data model lives in the PRD's folder, at the next free number:

```
docs/<dated-folder>/
  01-brief.md                    <- working notes, not read
  02-create-prd-questions.md     <- working notes, not read
  04-prd.md                      <- the input, and the only one
  05-design-db-questions.md      <- written only if step 4 finds a blocking gap; see below
  06-data-model.md               <- next free number
```

The folder is the PRD's own. Follow your project's own docs convention if it has one — never
drop the data model into a folder some other tool already owns and diffs against, where it
would read as drift. Questions this skill writes carry its own name —
`design-db-questions.md`, never bare `questions.md` — because the same folder may already
hold a questionnaire from whatever wrote the PRD; the prefix says which skill is asking.

**A new number is for new material. A correction goes back into the document it corrects.**

- **Additive** — a questionnaire after a brief, a PRD after the questionnaires, a data model
  after the PRD. Each is new material that does not invalidate what came before. Take the next
  free number; never overwrite or renumber one.
- **Corrective** — the same model, revisited: an invariant was misread, a constraint turned out
  unenforceable on the plan you're running against, the user pushed back and was right. **Edit the data model in
  place.** Do not write `06-data-model.md` beside `05`.

Correcting in place is not a loss of history — the previous revision is one `git log -p` away.
Leaving the superseded document on disk costs real harm: it is a confident specification telling
an agent to build the schema you just decided against, with nothing saying which one wins.

**The three passes work in `.cache/`, not in the PRD's folder.** The run's working files live in
a folder named after the final document — `06-data-model.md` gets `.cache/06-data-model/` — and
`.cache` is gitignored, so a half-written model never sits in `docs/` looking authoritative:

```
.cache/06-data-model/
  01-first-pass.md   <- the generate pass writes this
  02-critique.md     <- the critique pass writes this; findings only, never the document
  03-final.md        <- the fix pass writes this, and it is what gets copied
```

Only `03-final.md` is copied to `docs/<dated-folder>/06-data-model.md`. The three working files
are scratch: a later run for the same document overwrites them, and nothing downstream may read
them. If a run is corrected later, the folder is reused and the numbering restarts at `01`.

What a corrected document must carry:

- The superseded shape **named as a live alternative**, present tense, in _Alternatives
  rejected_ and the blast-radius table. A reversal that hides what it beat reads as fashion.
- Nothing else. **Do not narrate the revision.** No "this was revised", no pointer to the old
  one. The test is that a corrected document reads **as though it had been written once,
  correctly**.

## Steps

1. **Read the PRD, and nothing else in the folder.** It is self-contained by construction: the
   brief and the questionnaires beside it are the working notes that produced it, and reading
   them can only reintroduce a rule the PRD deliberately dropped or a version of one it
   superseded. If the PRD does not say it, it is a **PRD gap** — section 11, not a table.

   If your project already has a schema file, read it as the other input — it is usually short.
   Most projects are not greenfield: tables generated by an auth library, a payments
   integration keyed on a provider's own event id because that conflict _is_ its idempotency
   mechanism, or seed data left over from a scaffold are not yours to redesign or duplicate. A
   model that ignores them invents a second version of something that already exists.

   If there is no PRD, stop and say so. This skill turns an approved PRD into a data model; it
   does not write the PRD.

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

   And from the repo, if a schema already exists: anything belonging to a person may already
   reference a generated `user` table and inherit its key type — settled by a file nobody
   hand-edits, not a decision this document gets to make twice.

3. **Write the invariants down before you draw a single table.** As negative statements about
   state — never as sequences of steps; the procedural form is a journey and already belongs to
   the PRD. This is the step that gets skipped, and skipping it produces a schema that stores the
   nouns and enforces nothing.

4. **Sweep `references/invariant-checklist.md`** for what the PRD did not say, and sort every
   "the PRD is silent" finding into one of two piles — see _Business-rule gate_ for which
   categories belong in the blocking pile, and what to do when one does. A rule that is
   genuinely absent (not merely unaddressed) gets stated positively in section 2's
   _deliberately not invariants_ list, or the next person will add a constraint for it.

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

7. **Open the run folder and seed the first pass.** Decide the final name — the next free number
   in the PRD's folder, or the existing document's name if this run is correcting one — and
   create `.cache/<that name without its extension>/`. Then put the starting file in place as
   `01-first-pass.md`: a copy of `references/data-model-template.md` for a new model, or a copy
   of the existing document for a correction. This is yours to do, not an agent's — the three
   passes then each have exactly one file to open and one to write.

8. **Generate.** One `general-purpose` agent models every table and writes the whole document in
   a single context, applying the tests in _How to model_ and the ladder in _The enforcement
   ladder_, using `references/modelling-patterns.md` to **eliminate, never to pick** — a shape
   chosen from that file and justified afterwards is the exact failure the _Core principle_
   forbids. It overwrites `01-first-pass.md` in place and writes nothing else. Hand it what
   _Generate, critique, fix_ lists. The template's head comment is a note to the writer, not part
   of the document, and gets deleted. Invariants are labelled `I1`, `I2` … and decisions `DM-1`,
   `DM-2` …

9. **Critique.** A second `general-purpose` agent reads `01-first-pass.md` against the checklist
   in _Generate, critique, fix_ and writes `02-critique.md` — **findings only, and it may not
   edit the document.** Separating the reading from the writing is the whole point: an agent
   asked to review its own draft defends it. Give it the same invariant set and settled context
   the generate pass had, so it can check traceability rather than taste. If it finds nothing,
   `02-critique.md` says so and step 10 becomes a copy.

10. **Fix.** A third `general-purpose` agent reads `01-first-pass.md` and `02-critique.md` and
    writes the corrected document as `03-final.md`. It applies the findings and nothing else — it
    does not re-draft, does not re-argue a decision the critique did not raise, and does not
    resolve a finding that re-opens the PRD or the stack. Those become section 11 questions, per
    _Generate, critique, fix_.

11. **Copy it into place.** Read `03-final.md` yourself and run the mechanical gate: no bracketed
    placeholder survived, no `CREATE TABLE`, `ALTER TABLE`, `pgTable` or `drizzle` anywhere in
    the text, every table renders and is five columns or fewer, every contents anchor matches its
    heading. Then copy it to `docs/<dated-folder>/<name>.md`. **The copy is the only write into
    `docs/`** — nothing before this step touches the PRD's folder.

12. **Report** the file path, the table count, the invariant count and how many the database
    enforces, how many denormalisations were spent, and any open question that section 11 carries.
    Say the file opens in any text editor or markdown viewer. State that the user should critique
    it before any migration is written, because every constraint in it is cheap to change now and
    expensive to change once there are rows.

    Then stop. **Do not touch the schema file, do not generate or run a migration, do not create
    the database, and do not commit.** Once the user has approved the document, building it is a
    separate step, one vertical slice at a time through schema → api → web, with section 10's
    order deciding which slice can come first.

## Business-rule gate

Not every "the PRD is silent" finding from step 4 deserves the same answer. Two kinds exist,
and only one is allowed to become a silent assumption.

- **Anything the store, a convention, or the repo can settle is never blocking.** Settle it from
  evidence per _What to settle before modelling_ and move on — that is context, not a business
  decision.
- **A gap in a load-bearing business category is blocking, no matter how confident the obvious
  guess feels.** Five categories: **money** (who owes what, when, and what happens on failure),
  **cancellation, refund, or reversal** (can an action be undone, by whom, until when),
  **concurrency or capacity** (what happens when two things compete for one resource),
  **retention or deletion** (what must survive, what must not, on whose request), and
  **ownership or delegation** (who may act on whose behalf — see _Ownership and acting on
  behalf_). A wrong guess in one of these is not a modelling mistake to correct later; it is a
  wrong product, shipped under a data model that now defends it.

**A PRD produced by a skill with its own readiness gate — one that already forces these
decisions explicit before it hands off — will rarely trip this.** A PRD from anywhere else,
including one a person wrote directly, might trip it on the first sweep. That is the gate doing
its job, not a false positive, and it is exactly the case this section exists for: this skill
has no way to know how the PRD in front of it was produced, and treating every PRD as
pre-vetted would silently model over whichever gap a less rigorous source left behind.

**When any blocking category has a genuine gap, stop before drafting.** Write
`NN-design-db-questions.md` at the next free number: one short, closing question per blocking
gap, each naming the category, quoting the PRD passage that comes closest without deciding it,
and offering a recommended default the user can accept by writing "yes". Then stop and tell the
user which file to fill in. **Do not draft a first pass, and do not silently pick the
recommended default to avoid the stop** — a first pass built on an unconfirmed guess invites
approval of that guess instead of the decision, the same failure a PRD's own readiness gate
exists to prevent.

**When the user returns with answers, treat the file as evidence, not as an amendment to the
PRD.** Fold each answer into the invariant it settles and cite the questions file in section 11
exactly as any other assumption cites its evidence — this skill still writes nothing but the
data model. If an answer changes what the PRD itself should say going forward, tell the user so
rather than letting the data model quietly diverge from a PRD nobody updated.

**This gate runs once per document, not once per round.** Correcting an existing data model
re-sweeps the checklist against what changed, not against everything again — never re-ask a
blocking question the PRD or an earlier questions file already answered.

## Context and fan-out

The input to this document is a PRD, and it is not what overruns a run — **page content is**:
the engine's constraint reference, the hosting plan's limits page, the ORM's migration
documentation, each thousands of tokens of which two lines decide anything.

**Delegate the looking up. Never delegate the invariants.**

Send these as `general-purpose` subagents, in one message so they run concurrently. Each writes
to `<scratchpad>/findings-<topic>.md` and returns a compact summary — facts only.

| Delegate                                                                                                                                                                                                                                                            | Because                                                                                                                                                                                     |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Store capability checks** — which extensions the plan permits, whether `EXCLUDE`, partial indexes and deferred constraints are available, the identity/UUID generation options in the major version you run | _Writing rules_ requires these be tested rather than recalled. A local development database is rarely the same major version as production; do not assume they match |
| **Hosting plan limits** — storage included, row or connection ceilings, backup and point-in-time recovery window                                                                                                                                                       | Section 8 prints them. Which plan this project is on is itself often an assumption — say so in §11 rather than inventing a tier                                                             |
| **ORM expressibility** — whether the mechanisms this model rests on can be declared in the ORM's table builder, or need SQL appended to the generated migration by hand                                                                                                       | A rule the ORM's builder cannot spell is still enforceable; what changes is who writes the SQL, and that belongs in section 6                                                              |

Tell each agent the invariant it is serving and the exact question — "Does the free plan allow
`CREATE EXTENSION btree_gist`?" comes back usable; "research the hosting plan" comes back as a
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
- **The three document passes are a pipeline, not a fan-out.** Generate, critique and fix run one
  after another over one file, because each needs what the last one produced. Only the lookups
  above run concurrently.

## Generate, critique, fix

Three `general-purpose` agents, in order, over the run folder from step 7. Each opens named
files and writes one named file; none of them touches `docs/`, and none of them commits.

| Pass         | Reads                                | Writes              |
| ------------ | ------------------------------------ | ------------------- |
| **Generate** | the PRD, the existing schema file if one exists, `01-first-pass.md` (the template or the document being corrected) | `01-first-pass.md`  |
| **Critique** | `01-first-pass.md`                   | `02-critique.md`    |
| **Fix**      | `01-first-pass.md`, `02-critique.md` | `03-final.md`       |

The **generate** agent gets, explicitly:

| Hand over                                                                 | Because                                                                    |
| ------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| The numbered invariant set from step 3, final                             | It models against them. It does not get to invent them                    |
| The checklist sweep from step 4 — assumptions taken, gaps found           | Otherwise it re-derives them, differently                                 |
| The settled context from step 5, with the evidence that settled each one  | §11 prints both                                                            |
| The key strategy and the append-only decision from step 6                 | They reach into every table                                               |
| The store, its version and plan, and every findings file from the lookups | With source URLs and dates — the document prints them                     |
| The path to `01-first-pass.md`, already seeded                           | See step 7                                                                 |

It may not invent or amend an invariant, re-decide the product or the store, or write DDL.
Anything it finds missing comes back as a question and lands in §11 — a drafting agent that
quietly resolves a PRD gap has made a product decision nobody reviewed.

**The critique agent gets that same table** — it cannot check traceability against an invariant
set it has not been given — plus the path to `01-first-pass.md` and to the `02-critique.md` it
writes. It reports findings and nothing else: one entry per finding, each
naming the file's own label (`I7`, `DM-3`, `### booking_seat`), what is wrong, and what would
make it right. **It may not edit the document**, and it may not manufacture a finding to justify
its own existence — "nothing on the list fails" is a complete and expected answer.

**This is the list it checks against**, and it is what six critics would otherwise each own a
slice of, collapsed into one pass because a document is drafted once and should be right once,
not negotiated into shape over rounds:

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

**The fix agent gets `01-first-pass.md`, `02-critique.md`, and that same table again** — it must
not re-derive an invariant to apply a finding. It writes `03-final.md`: the first pass with
the findings applied, and nothing else changed. A finding it disagrees with is answered in
`03-final.md`'s section 11, not silently dropped; a finding that re-opens the PRD or the stack is
not a defect in this document and becomes a §11 question rather than an edit. It may not
re-draft a section the critique did not raise, and where `02-critique.md` reports nothing, it
copies the first pass through unchanged.

**The result must read as though it had been written once, correctly.** No pass may leave a
trace of itself in the document — no "revised", no "addressed", no note about what the critique
found. The critique is a file in `.cache/`, and that is the only place it exists.

## What to settle before modelling

**This skill does not open with context questions.** Five things below are settled by looking,
not asked. A question whose answer is on disk spends the user's attention confirming your own
reading; a question whose answer would not change a single column buys nothing. This is
distinct from _Business-rule gate_, which does stop the run — the difference is that nothing
here is a business decision the PRD was supposed to have already made.

State each in section 11 as an assumption with the evidence that settled it, and raise it as a
question only under the condition in the third column:

| Settle                                                                                                                                                                                                                  | Where you look                                                                                                                                                                                                                                                      | Raise it only if                                                                                       |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| **Does a database already exist with rows that matter** — decides how section 10 is written | the project's migrations folder and whether any deployed instance has real data in it. A local development database never counts: it is thrown away | production has rows and you cannot tell whether any of them matter |
| **Every path that writes to this database** | the project's own services and their connection paths — an API layer, a migration tool run from a developer machine, an auth library's own writes | never; if a PRD implies a writer that isn't already accounted for, note it in §11 |
| **Data to import** — a spreadsheet, an old system, a payment provider's history | the PRD's scope, data and phases sections | the PRD names a predecessor system but not what comes across |
| **Naming or structural convention in force** | the existing schema file, if one exists: its casing, its export style, what any generated tables already do to both | the existing tables contradict each other |
| **Whether one person ever acts on another's rows** — decides whether ownership is a user column or an account with grants | The PRD's journeys and its data section on who may see what; an operator journey that writes to customer-owned rows is the tell | the PRD shows one person acting for another and never says who may |
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

### Ownership and acting on behalf

**Rows belong to an account; users are granted access to accounts.** A user is who signs in; an
account is what things belong to. The checklist settles the first half — ownership goes on an
account rather than on a user unless the product is single-user forever. This is the second half:
**who may act on that account is a table, not a property of the account row.**

It carries a role, so by _Is this one thing or two?_ it is an entity with a business name rather
than an anonymous join table: `account_grant (account_id, user_id, role, granted_at,
revoked_at)`, unique on `(account_id, user_id)` so one user cannot hold two contradictory roles
on one account. Granting, revoking and expiring access are then rows and updates — never a
deploy, and never a column added to whatever the person is being given access to.

**The PRD does not have to ask for delegated access for this to be the right shape.** An operator
issuing a refund for a customer, a second person on a business account, and support acting for
someone are all the same table. One table now against rewriting every ownership column, every
query filter and every authorization check later is not a close call — but if the PRD is silent
on who else may act, that is a section 11 question, not a role vocabulary invented here.

- **The API passes the account in and checks it against this table.** An account-scoped field
  should take the account as an explicit argument, never inferred from the session, precisely so
  the check has somewhere to happen; that rule holds only if there is a table to check it
  against.
- **State which rung the reading rule lands on rather than assuming one.** The grant's own
  integrity is _Structure_ — foreign keys and the unique pair. "A row is visible only to a viewer
  holding a grant on its account" is a different rule, and it lands on _Application_ unless
  row-level security carries it. RLS needs a per-request identity set inside the transaction,
  which against a pooled connection is a deliberate decision rather than a free rung. Take it, or
  record the Application landing with its one-line justification; do not let section 6 claim a
  rung nothing implements.
- **Never copy the role onto the rows it governs.** A `role` or `is_owner` on an order is a
  denormalisation of something revocable, and it disagrees with the grant the day it is revoked.

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
predicate and check it against the current documentation for the engine and version you're
actually running. If it turns out expressible, you have moved a rule up two rungs for the cost
of one line.

**A mechanism your ORM cannot spell has not fallen off the ladder.** Where its table builder
cannot declare it, the migration tool still generates the migration file and a line is added to
it by hand — that is usually permitted, and is a decision for whoever writes the schema, not a
reason to enforce the rule anywhere else. Say in section 6 which of the two writes it.

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
- **Never guess past a blocking business-rule gap.** Money, cancellation, concurrency,
  retention, and ownership are not this skill's decisions to invent a default for silently. See
  _Business-rule gate_: stop and ask before drafting, don't record a guess and move on.
- **Never re-decide the stack.** The store, its ORM and its migration tool, if already settled,
  are a fact for this document, not a choice made here. If the model needs something they cannot
  do, that is a note in section 11, not a substitution made here.
- **Do not touch the schema file, run a migration, create the database, or commit.** The
  document is the whole deliverable; turning it into a schema, and when, are separate steps.
- **Stay engine-neutral in the design, engine-specific only in the enforcement map.** Which
  extension provides a mechanism, and whether the ORM can declare it or the migration needs a
  hand-written line, belongs in section 6, not inside a type annotation.
- **No table without a trace, no invariant without a mechanism.** If the PRD is silent on
  something the schema seems to need, record it as an assumption rather than inventing a
  requirement. Every row in section 2 is answered in section 6, even when the answer is
  "application code, because …".
- **One generate, one critique, one fix — and no more.** Three passes over one file, then the
  copy. A second critique means something upstream was wrong — a PRD gap or a stack question —
  and that goes to section 11, not into a fourth attempt at the same document.
- **Nothing is written into `docs/` until step 11.** The three passes work in
  `.cache/<name>/`; a document that is still being argued with must not sit in the PRD's folder,
  where the next agent would read it as settled.
- **Write nothing but the data model.** The PRD is an input, and the brief and questionnaires
  are its working notes — none of them are edited here.
- **Do not hide a guess.** Anything decided without evidence — a volume, a limit, a capability —
  goes in section 11, visible.
