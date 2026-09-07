---
name: db-design
description: Turn an approved PRD and its tech stack decision into a numbered data model document a coding agent can generate the schema from - every table, column, constraint and index traced to a business invariant, every modelling call argued with its alternatives, and every rule enforced at the lowest level that can express it. The document is a single self-contained HTML file carrying semantic types and named constraint predicates rather than engine DDL, with an entity sketch and a write-path diagram. It is drafted in one pass, then attacked by parallel critics and repaired between rounds until every rule passes. Use when the user asks to design a schema, model the data, decide tables, keys and constraints, write the DDL, or answer "how should this be stored" after a PRD and a stack exist.
---

# PRD and stack to data model

Turn an approved PRD and the stack that was chosen for it into a **data model document**: the
invariants written as things that must never be true, the tables that hold them, and — for each
invariant — the exact mechanism that makes it impossible rather than merely unlikely.

## Core principle

A schema is not a place to put the data. It is **the set of states the product is allowed to be
in**, and everything else is a client of it: the application, the next coding agent, a
migration, a background job, an operator with a SQL console at 2am. The value of the document is
the reasoning about what must never be true — the column list is the residue.

Three rules govern every choice:

1. **The PRD's rules decide, not the query patterns.** Every table, column and constraint
   traces to an invariant, a journey, or a stated retention obligation. Model what is true;
   tune for what is read afterwards, and only with a measurement in hand. A schema shaped by
   guessed query patterns is a schema shaped by nothing.
2. **Enforce at the lowest level that can express the rule.** Structure beats a type, a type
   beats a constraint, a constraint beats a transaction, and all of them beat application code —
   because the database is the only layer every writer passes through. Push each rule down until
   it stops fitting, then record where it stopped. See *The enforcement ladder*.
3. **Prefer making a bad state unrepresentable to detecting it.** A balance that is always a sum
   cannot disagree with its entries. A column that does not exist cannot be wrong. This is the
   only move in the document that removes a class of bug instead of guarding against it, and it
   is almost always available at design time and never available later.

The document answers **what is true**. It never re-opens **what** — that is the PRD — or **what
it runs on** — that is the stack document. If you find yourself deciding a business rule, that
is a PRD gap; if you find yourself preferring a different store, that is a stack question. Both
go to section 11 and get sent back.

## Folder convention

The data model lives in the PRD's folder, at the next free number:

```
docs/<dated-folder>/
  01-brief.md
  02-questions.md
  04-prd.html
  05-tech-stack.html
  06-data-model.html   <- next free number
```

**A new number is for new material. A correction goes back into the document it corrects.**

- **Additive** — a PRD after a brief, a stack after a PRD, a data model after a stack. Each is
  new material that does not invalidate what came before, and the earlier file is still true and
  still read. These take the next free number. Never overwrite or renumber one.
- **Corrective** — the same model, revisited. An invariant was misread, a constraint turned out
  to be unenforceable on the chosen plan, the user pushed back and was right. **Edit the data
  model in place.** Do not write `07-data-model.html` beside `06`.

Correcting in place is not a loss of history in a git repository — the previous revision is one
`git log -p` away, with the prompt that caused the change in the commit message. Leaving the
superseded document on disk costs real harm: it is a confident, well-argued specification
telling an agent to build the schema you just decided against, and nothing inside either file
says which one wins.

What a corrected document must carry, so the deletion is safe — and note what is *not* on this
list:

- The superseded shape **named as a live alternative**, in present tense, in *Alternatives
  rejected* and in the blast-radius ladder. A reversal that hides what it beat reads as fashion.
- Nothing else. **Do not narrate the revision.** No "this was revised", no "the previous version
  stored", no pointer to the old revision. A data model is read far more often than it is
  edited, and every reader after the first is paying attention to an editing event they were not
  present for.

The test is that a corrected document should read **as though it had been written once,
correctly**.

**The data model is HTML**, for the same reasons the PRD and the stack document are, plus one of
its own: it argues in column tables and named predicates, which need to be shown exactly — in a
monospaced block, with their names intact — rather than paraphrased into a sentence. It is one file that opens by
double-clicking. Questionnaires in the folder stay markdown.

## Steps

1. **Read the whole folder in number order** — brief, every questionnaire, the PRD, the stack
   document. The PRD and the stack document are HTML; read them as extracted text and work from
   their numbered sections, because the markup is most of the file and carries no decisions.

   The PRD is the source of truth for what must be true. The stack document tells you the store,
   its major version, the plan, and what that plan can actually do — which decides which
   enforcement mechanisms exist at all.

   If there is no PRD, stop and say so — run `brief-to-prd` first. Do not design a schema from a
   brief. If there is no stack document, say so and offer to run `prd-to-stack`: the store
   decides section 6, and a model designed against an unnamed store enforces nothing in
   particular. If the user wants to proceed anyway, name the store and version as an assumption
   in section 11 and flag every mechanism that depends on it.

2. **Extract the invariants.** Go through the PRD section by section and write down every line
   that constrains state, keeping the reference:

   | PRD section | What to pull out |
   |---|---|
   | 6 — Journeys | The writes each journey performs, and — more useful — the failure branches, where models are thinnest |
   | 8 — Business rules | Money, time, capacity, concurrency. This is the bulk of section 2 |
   | 9 — Data | The entities, what is retained, what is exportable, who may see what |
   | 10 — Constraints | Compliance, residency, what may never be stored at all |
   | 11 — Acceptance criteria | Already written as checkable statements; several are invariants in all but name |
   | 13 — Phases | The order things must exist in, which decides migration order |

   And from the stack document: the store and version, the plan's limits, and the transaction
   shape its store decision committed to. If the stack document argued that the hardest rule
   collapses into one statement, that argument is now yours to honour or to reopen in section 11.

3. **Write the invariants down before you draw a single table.** As negative statements about
   state — "no two bookings may…", "no payment may exist without…" — never as sequences of
   steps. The negative form is what becomes a constraint; the procedural form is a journey and
   already belongs to the PRD. This is the step that gets skipped, and skipping it produces a
   schema that stores the nouns and enforces nothing.

4. **Sweep `references/invariant-checklist.md`** for what the PRD did not say. Most entries will
   come back "the PRD is silent", and that is the point: each is either a blocking question or a
   recorded assumption. A rule that is genuinely absent gets stated positively in section 2's
   *deliberately not invariants* list — an omitted rule reads as an oversight, and the next
   person will add a constraint for it.

5. **Settle the context from evidence, not from questions.** Everything in *What to settle before
   modelling* is answerable by looking at the repository and the two documents; look, then record
   each answer as a stated assumption rather than spending a question on it. Do not open a
   questionnaire round, and do not open with inline questions either.

6. **Settle what propagates, and test what is about to be claimed.** Two decisions are not
   independent of anything, and both are yours before a single table exists: the **key
   strategy** — see *Keys and identity*, and settle it together with anything created outside
   the database, since those are one decision — and **what is append-only**. Both reach into
   every table, and both are the most expensive things in the document to change once rows
   exist. Everything downstream inherits them; nothing downstream may re-open them.

   Before anyone writes that the store cannot express something, **write the rule out as an
   actual predicate** and check it — see *Writing rules*. Batch every capability and limit
   question the model is going to need and send them out at once, the way *Context and fan-out*
   describes, rather than fetching a documentation page in the middle of an argument.

7. **Hand the tables and the writing to one agent.** It models every table and writes the whole
   document in a single context, applying the tests in *How to model* and the ladder in *The
   enforcement ladder*, and using `references/modelling-patterns.md` to **eliminate, never to
   pick** — a shape chosen from that file and justified afterwards is the exact failure the
   *Core principle* forbids. What it is given, what it may not do, and where it writes are in
   *Draft, critique, revise*.

   If a data model already exists for this PRD and this run is correcting it, it edits that file
   in place — see *Folder convention*. Otherwise it copies `references/data-model-template.html`
   to the next free number (`NN-data-model.html`) and replaces the content. **Copy it with
   `cp` — nobody reads it into context**, not you and not the agent: it is a large file, most of
   it a stylesheet nothing will change, and the copy on disk already contains all of it. Opening
   it in a browser to see the idioms working costs nothing; reading it costs a large fraction of
   the run. Its head comment is addressed to whoever writes the document, not to the reader:
   **delete that comment from the copy.** Invariants are numbered `I1`, `I2` … and decisions
   `DM-1`, `DM-2` …, each with an `id`.

8. **Critique and revise until it passes.** Concurrent critics, each owning one way the document
   can be wrong, then one reviser applying what you triaged, then round again — until a whole
   round comes back clean or the third round ends. *Draft, critique, revise* has the groups, the
   verdict format, the triage rule and the stopping rule. This is where the document becomes
   correct; the draft is a first attempt, not a deliverable.

9. **Look at what a script cannot judge.** The assertions have already run at the top of every
   round — see *Draft, critique, revise* — so what is left is the part no script has an opinion
   about: whether a diagram reads, whether a label collides with an arrow, whether the dark
   theme holds. Open it in a browser, once per illustration and once in the dark theme, at
   desktop and at phone width. Anything wrong here goes back to the reviser as one more blocking
   finding — geometry, not argument — and does not restart a round.

10. **Report** the file path, the table count, the invariant count and how many of them the
    database enforces, how many denormalisations were spent, **how many rounds it took and what
    each one changed**, and any open questions — including every finding that survived the last
    round, which is now a question and not a defect you buried. Say the file opens by
    double-clicking it. State that the user should critique it before any migration is written,
    because every constraint in it is cheap to change now and expensive to change once there are
    rows — the rounds sharpened it against the rules, not against their business. Do not create
    the database.

## Context and fan-out

The inputs to this document are a PRD and a stack document, both HTML, and both large. The
writing itself is not what overruns a run — **page content is**: a store's constraint reference,
a plan's limits page, an ORM's migration documentation, each thousands of tokens of which two
lines decide anything.

**Delegate the looking up, the writing and the judging. Never delegate the invariants.**

### Fan out

Send these as `general-purpose` subagents, in one message so they run concurrently. Each writes
to `<scratchpad>/findings-<topic>.md` and returns a compact summary — facts only.

| Delegate | Because |
|---|---|
| **Store capability checks** — one agent for all of them: which extensions the chosen plan permits, whether exclusion constraints and partial indexes are available, what the identity and UUID generation options are in that major version | *Writing rules* requires these be tested rather than recalled, and testing one means reading a reference page |
| **Plan limits** — storage included, row or connection ceilings, backup and point-in-time recovery window | Section 8 prints them, and they belong to the plan the stack document actually bought |
| **ORM and migration-tool conventions** — what the tool named in the stack document expects of table and column names, and which key types it handles natively | One page, two paragraphs of which matter, and a naming convention the tool fights costs a rename on every table |

Tell each agent the invariant it is serving and the exact question. "Does Neon's free plan allow
`CREATE EXTENSION btree_gist`?" comes back usable; "research Neon" comes back as a brochure.
**Require every answer with its source URL and the date checked** — the document prints both.

### Delegate whole, or not at all

- **The invariants stay with you.** They are a single consistent set, cross-referenced by every
  later section, and two agents writing them produce two lists that disagree. The same goes for
  the key strategy and the append-only decision, which reach into every table.
- **The tables go to one agent — all of them, or none.** A schema is one artefact: foreign keys,
  naming and key strategy must agree across every table, and reconciling independently-written
  tables costs more than writing them. One agent writing every table is not a fan-out. Six agents
  writing six tables is, and it is the failure this rule was always about.
- **The illustrations go with the tables.** Geometry is laid out against the actual labels.
- **Critique fans out; writing never does.** Six readers disagreeing about a finished document is
  the entire point — see *Draft, critique, revise*. Six writers disagreeing about an unfinished
  one is a reconciliation job you will end up doing by hand.

### Cheap wins first

- **Copy the template, never read it** — see step 7.
- **Read the PRD and the stack document as extracted text, not as HTML.**
- **Assert before you critique, and look last** — see *Draft, critique, revise*. A defect a script
  can name should never cost a critic a finding.

## Draft, critique, revise

A data model is judged better than it is written. A first pass has to commit to a key strategy, a
cardinality, and a rung for every rule all at once, and what survives it is rarely carelessness —
it is the places where committing to one thing quietly cost another, invisible to the person who
did the committing. So the document is written once, attacked from several directions at once by
readers who did not write it, and repaired between attacks. **Reflection is not the polish step
here; it is where the correctness comes from.**

One round:

```
draft ──▶ assertions + six critics (concurrent) ──▶ you triage ──▶ one reviser ──▶ round again
```

**You never edit the document.** You settle the invariants, you triage the findings, and you
decide when it stops. Writing belongs to the drafter and the reviser, judging to the critics.
Picking up the pen yourself collapses the loop back into one head, which is the thing it exists to
prevent.

### The draft

One `general-purpose` agent, writing straight to `docs/<folder>/NN-data-model.html` — the real
destination, not a scratch copy. There is no promotion step, every round after the first is an
in-place edit, and `git diff` shows exactly what each round did.

Hand it, explicitly:

| Hand over | Because |
|---|---|
| The numbered invariant set from step 3, final | It models against them. It does not get to invent them |
| The checklist sweep from step 4 — assumptions taken, gaps found | Otherwise it re-derives them, differently |
| The settled context from step 5, with the evidence that settled each one | §11 prints both |
| The key strategy and the append-only decision from step 6 | They reach into every table; this is not a choice two agents can each make |
| The store, its version and plan, and every findings file from the lookups | With source URLs and dates — the document prints them |
| The destination path, already copied from the template | See step 7 |

And it may not invent or amend an invariant, re-decide the product or the store, or write DDL.
Anything it finds missing comes back as a question and lands in §11: a drafting agent that quietly
resolves a PRD gap has made a product decision nobody reviewed.

### The critics

Send them in one message so they run concurrently. Each reads the document — only the document,
plus its own brief — and owns exactly one way it can be wrong. The groups are disjoint on purpose:
two critics filing the same finding is attention spent twice, and a domain no critic owns is a
domain nobody read.

| Critic | Owns | Fails it on |
|---|---|---|
| **Trace** | §2 ↔ §4 ↔ §6, in both directions | A table tracing to no invariant, an invariant with no mechanism, a requirement invented to justify a table, a guess that never reached §11 |
| **Ladder** | Which rung each rule landed on | A rule one `OR` away from a `CHECK` sitting in application code, an untested "cannot express", a foreign key with no explicit `ON DELETE`, a nullable column with no stated meaning, a constraint named `check2` |
| **Identity and shape** | Keys, cardinality, what is one thing and what is two | A forced 1:1 that only the first write path justifies, a cardinality true at creation and false in a year, a bad state that could have been made unrepresentable, a key strategy that forbids the safe write order |
| **The expensive domains** | Money, time, capacity, concurrency, state | A bare amount, a local timestamp, a read-then-write with no guard, a stored balance with nothing keeping it honest, a state machine listed as enforced when nothing enforces it, an outside call inside a transaction |
| **Budget and growth** | The denormalisation ledger, §8, §10 | A fourth denormalisation, an entry with no reconciliation, a resisting value miscounted as one, storage arithmetic that does not show its work, a migration order that creates a child before its parent |
| **The builder** | Whether the thing can be built from | Any question the first migration would have to ask and the document cannot answer, and any PRD journey — especially a failure branch — that cannot be executed against these tables |

The builder is the one critic that also gets the PRD, for the journeys. The rest get the document
alone, because a critic holding the PRD starts reviewing the PRD.

Tell every critic the same three things: **quote the rule, point at the `id`, say what to do.**

- **A finding names the rule it breaks** — from this skill or from
  `references/invariant-checklist.md` — **and the `id` where it lives.** No rule, no finding.
- **Two severities.** *Blocking* means a rule is broken. *Note* means the critic would have done
  it differently. Only blocking findings drive another round; notes are reported once and dropped.
- **"Consider adding" is not a finding.** More material is not a fix. A critic may demand
  something absent only where this skill requires it to be present.
- **PASS is a real verdict**, and by the second round it is the expected one for most groups. A
  critic that feels obliged to produce findings will produce bad ones, and every bad one costs a
  revision.

Each writes `<scratchpad>/critique-r<N>-<group>.md` and returns only its verdict and its blocking
findings, numbered.

### The assertions

Run these yourself at the top of every round, before the critics go out. Machine-checkable defects
are cheap to catch and expensive to critique, and a broken cross-reference distracts all six at
once. Desktop width and phone width:

- body scroll width against viewport width, and every `.wide` container scrolling inside itself
- each `<text>` element against its `viewBox` and against the `<rect>` it sits in
- every `href="#…"` resolving to an `id`, and every `I` and `DM` reference resolving
- no bracketed placeholder surviving
- every named predicate referring to a column §4 defines, and every table a foreign key names
  existing
- `CREATE TABLE`, `ALTER TABLE`, `pgTable`, `sqliteTable` — any hit is the altitude slipping

Failures go to the reviser as blocking findings, alongside the critiques.

### Triage, then revise

The findings arrive as six independent opinions and have to leave as one ordered instruction list.
That is your job, and it is the step that keeps the loop from eating itself:

- **Drop anything that re-opens the PRD or the stack.** A critic that wants a different business
  rule has found a PRD gap; a critic that wants a different store has found a stack question. Both
  go to §11 as questions. Neither is a defect in this document, and neither is fixable here.
- **Resolve contradictions before handing them down.** Two critics can want opposite things —
  most often the ladder wanting a constraint that the expensive-domains critic wants deferred. You
  decide, in one line, against the invariant. A reviser asked to arbitrate a design conflict will
  split the difference and satisfy neither rule.
- **Carry the rebuttals forward.** A finding the reviser refused, with a reason you accepted, is
  settled. If it returns in the same form next round, drop it: a critic re-filing a rebutted
  finding is not new information, and honouring it is how a loop stops converging.

Then one `general-purpose` reviser, given the document, the triaged list, and nothing left to
decide. It edits in place, and it may refuse a finding in one line where the finding is wrong — a
reviser that obeys everything turns critique into growth. It reports what it changed, what it
refused and why, and the file size before and after.

**The revision leaves no trace of itself.** No "revised", no "previously", no note about what a
critic said. The test in *Folder convention* is the test here too: the document must read as
though it had been written once, correctly.

Write the triaged list and the rebuttals to `<scratchpad>/round-<N>.md`. It is what the next round
is given, and it is what you report from.

### When it stops

**Every critic returns PASS and the assertions are clean.** That is the pass condition, and
nothing weaker is.

From round two, run **the whole set again**, not only the groups whose findings were acted on.
Fixes travel: a constraint added for the ladder changes the enforcement map, which changes
traceability; a table split for shape changes the migration order. A critic re-reading a document
it has already passed is the cheapest agent in this skill.

**Three rounds, and no more.** A blocking finding that has survived two revisions is almost never
a defect still sitting in the document — it is a disagreement about the PRD or about the store,
wearing the costume of one. Stop, put each survivor into §11 as an open question in the user's
terms, and say in the report that you stopped at the cap and what is outstanding. A loop that runs
until it agrees with itself has proved only that it can.

Watch the length across rounds; it is the tell. A document that grows every round is being
negotiated rather than corrected, and if the third draft is materially longer than the first with
no blocking finding that demanded new material, the critics have quietly started writing it.

## What to settle before modelling

**This skill does not open with questions.** Five things must be settled and every one of them is
settled by looking. A question whose answer is on disk is not diligence — it spends the user's
attention confirming your own reading, and offering a recommended default gives away that you had
already read it. A question whose answer would not change a single column is worse: it buys
nothing at all.

State each in section 11 as an assumption with the evidence that settled it — "greenfield: the
repository contains `docs/` only, no migrations and no schema" — and raise it as a question only
under the condition in the third column:

| Settle | Where you look | Raise it only if |
|---|---|---|
| **Does a database already exist with rows that matter** — the single biggest determinant of how section 10 is written; greenfield means migrations can be edited freely, live data means every change is additive first | migrations or schema files in the repository, a connection string in an environment file, whether the stack document provisioned an instance or merely chose one | there is a live instance and you cannot tell whether anything in it matters |
| **Every path that writes to this database** | the stack document, which decided the services, the migration tool and the connection paths — that is a stack decision and it is already made | the stack document names no write paths at all, which is a gap in `prd-to-stack`; name it as one rather than papering over it with a question |
| **Data to import** — a spreadsheet, an old system, a payment provider's history; it carries identifiers that must be kept and duplicates that must be reconciled | the PRD's scope, data and phases sections, which is where a migration would have been scoped | the PRD names a predecessor system but not what comes across |
| **Naming or structural convention in force** | the stack document's ORM and its native conventions, plus any existing schema in the repository | the stack document names no ORM, or the repository's existing tables contradict it |
| **Anything deletable or exportable on request** | the PRD's data and constraints sections | the PRD is silent *and* it stores personal data — then it is a blocking question, not a preference |

**Do not ask how many writers there are in order to decide where a rule lives.** The ladder does
not consult that answer — see *The enforcement ladder*. Counting writers can only ever license
holding a rule in application code, which is the outcome this document exists to avoid.

Do not ask which tables they want, or whether to use UUIDs. Those are the decisions they came
here to have made.

## How to model

Apply these tests to every table and every column, in this order. The first failure ends the
question — there is no point tuning a column that should not exist.

- **Does it trace to an invariant or a journey?** Name it. No trace, no table. A table added
  because the domain "obviously has one" is how a schema acquires entities nothing writes to.
- **Is this one thing or two?** An entity has independent identity and an independent lifecycle.
  If a group of columns is written together, becomes null together, and means nothing on its
  own, it is one thing. If a "join table" carries a quantity, a price, a role or a date, it is
  an entity and deserves a name from the business.
- **Is the cardinality the one that holds over the lifetime, or only at the first write?** Two
  rows created together today are not 1:1 forever. Ask three questions before placing any
  reference: can the child outlive the parent (→ optional reference, `SET NULL`); can the parent
  ever have a second child — a retake, a revision, a repeat purchase, a re-submission (→ it is
  1:N and the reference goes on the many side); will the two ever be created independently, one
  anonymously and attached later (→ decouple them). See *The forced 1:1* below.
- **Does anything outside the database get created with this row?** A file, an object in blob
  storage, a record at a payment provider, a search-index document. If so, the id strategy and
  the write order are one decision, not two — see *Ordering writes against the outside world*.
- **Can the bad state be made unrepresentable?** Before adding a check, ask whether the column
  that could be wrong needs to exist. A balance that is always the sum of a ledger, a status
  derivable from timestamps, a "primary" flag that a foreign key on the other side would
  replace — each is a constraint you never have to write, test, or reconcile.
- **What is the lowest level that can enforce it?** Walk it down the ladder below and record
  where it stopped.
- **What does a null mean in this column?** If there is no answer, the column is `NOT NULL`. A
  nullable column with no stated meaning is where an agent will put whatever it has.
- **What happens when the parent goes away?** Every foreign key gets an explicit `ON DELETE`.
  The default is not a decision, and `CASCADE` is only right when the child has no independent
  existence — a booking is not deleted because a customer was.
- **What breaks with two writers at once?** Read-then-write is the shape to look for. Most
  collapse into one guarded statement, and the ones that do not need a named lock order in
  section 9.
- **What does this cost to change once there are rows?** Spend deliberation in proportion to
  that, not to how interesting the choice is. A key strategy and an append-only decision are
  months; an index is minutes; a column name is a migration.
- **Is it the shape whose failure modes are already documented?** Where two shapes both express
  the invariant, take the ordinary one. Not because it is better, but because its ways of
  breaking have been discovered by other people, more of the corpus an agent will build with
  assumes it, and the next maintainer has seen it before. A clever schema is a schema only its
  author can debug. The unusual shape wins only when it expresses an invariant the ordinary one
  cannot — elegance is not an invariant.

### Keys and identity

Settle this before any table, because it propagates into every reference and is the most
expensive thing in the document to reverse.

**Default to a key the application can generate: UUIDv7, or ULID where a codebase already uses
it.** Both are 128 bits with a leading millisecond timestamp, both sort chronologically, and
both leak creation time identically. The reason to prefer them over a database sequence is not
that sequences are slow — they are not, and see the correction below — it is that **knowing the
id before the write buys you the choice of write order**, which is what the next subsection is
about, and that a client-generated id makes a retried create deduplicate on the primary key
without a separate idempotency table.

Between the two: **UUIDv7 for a new schema, ULID where one is already established.** v7 is
RFC 9562, stores in a native 16-byte `uuid` column, and is generated in-engine from PostgreSQL
18; ULID has no native type, so it costs 26 bytes as text in every reference and every index
that carries one. ULID's real advantage is its encoding — shorter, case-insensitive, and it
omits the characters people confuse — which pays only when a human reads, types or dictates
the id. If that is the requirement, the better answer is usually a separate short public
reference beside the key, not an encoding chosen for the key itself.

**When creation time must not leak, the fallback is UUIDv4, not v7.** v7 leaks exactly as ULID
does. This is the one case where a random key is correct, and it costs index locality to get
there.

**A database sequence is still right** when nothing is generated outside the database, nothing
is exposed, and the smallest possible key matters — a bigint is 8 bytes against 16 or 26, in
every foreign key column and every index. Take it deliberately, not by default.

Two arguments against sequences that are commonly made and are wrong, so nobody rebuilds a
decision on them: **you do not pay an extra round-trip** — `INSERT … RETURNING` yields the id
with the write, and a parent-child batch can be threaded with a CTE; the real limit is that you
cannot know the id *before* the write. And **sequence contention is not the bottleneck people
think** — `nextval()` takes no row lock, does not roll back, and caches per session. The real
contention under heavy concurrent insert is the **right edge of the index**, where every
monotonic key lands on the same leaf page — and time-ordered keys do not fix that, because
UUIDv7 and ULID append at the right edge too. They remove generation contention, not insertion
contention.

### The forced 1:1

A **required, unique reference is a forced 1:1**: it forbids the parent from existing without
the child *and* the child from ever having siblings, welding two lifecycles together. It is
correct only when the child is a pure extension of exactly one parent, permanently — a profile
row hanging off a user. It is wrong far more often than it is written, because the first write
path really does create one of each, and the constraint records that accident as a law.

When unsure, take **an optional reference on the many side**. It behaves identically today —
each submit still makes one of each — and it keeps three futures open for nothing: retakes,
created-anonymously-then-attached, and records arriving from another source. The asymmetry is
the whole argument: choosing 1:N up front is free, and retrofitting it onto a shipped forced
1:1 is a data migration.

### Ordering writes against the outside world

When a row is created alongside something outside the database — an object in blob storage, a
file, a provider-side record, an index document — the two writes can fail independently and one
of the two orderings leaves much worse wreckage.

**Order them so that a failure leaves inert garbage rather than visible-but-broken state.** An
orphaned storage object is inert: nothing references it, and a sweep for keys with no row
reclaims it. An orphaned row is live and wrong: it appears in listings, queries return it, and
it points at something that is not there.

That means writing the external thing first and the row second, which is only possible if the
id exists before either write — which is what *Keys and identity* buys. With a
database-generated key the order is forced the other way, and the compensation is a delete
against a row that has already been committed and may already have been read.

The rule composes with §9's transaction boundary rather than competing with it: nothing outside
the database may sit inside an open transaction, so the recovery is always a compensating
action, and the design question is only which direction's leftovers are cheaper to clean up.
Where neither is cheap, write the row first in a pending state, do the external write, then mark
it ready — and sweep the rows that never got there.

## The enforcement ladder

For every invariant, start at the top and take the first level that can express it. Then write
down which level you landed on, because section 6 is that list and the *Application* rows in it
are the document's risk register.

**Assume more than one writer — as a threat model, not as an architecture.** Routing every write
through one application is good discipline and worth recommending. It is also a convention, held
by everyone remembering it, and the schema is what makes the invariant true whether or not it
holds. So do not establish the writer count first and then choose a rung: the ladder is the same
either way, and the assumption costs nothing when the app really is alone while being the only
one that survives a second writer arriving without a schema change. There is almost always a
second writer already — a migration tool backfilling a column is one, and so is the console
session that fixes the row nobody could fix through the app. The count matters at exactly one
rung, the bottom one, and only once a rule has already fallen there.

**This is not "logic in the database".** The top rungs are declarative predicates over state,
checked as part of a write already happening; a trigger is execution logic, which is why it sits
one rung off the bottom and is named a last resort. Keeping procedures out of the database and
keeping invariants in it are the same position, not competing ones.

| Level | Mechanism | Holds against |
|---|---|---|
| **Structure** | The state cannot be written down — the column does not exist, or a foreign key makes it impossible | Everything, forever |
| **Type and domain** | `timestamptz`, integer minor units, `NOT NULL` | Every writer |
| **Constraint** | `CHECK`, `UNIQUE`, `FOREIGN KEY`, `EXCLUDE` | Every writer |
| **Index** | Partial unique index — "one *active* subscription per customer" | Every writer |
| **Transaction** | Two tables that must agree, committed together | Every writer that uses one |
| **Trigger** | Last resort inside the database | Every writer, at the cost of action at a distance |
| **Application** | A rule held in code | Only the writers that remember it |

Three things this ladder is for:

**Enforcement has a runtime cost, and it is not the one people assume.** Most rungs are close to
free: a `CHECK` is a predicate over columns already in memory on a row already being written, and
`UNIQUE` rides an index the read path almost certainly wanted anyway. The application-side
alternative to uniqueness is a read followed by a write — more database work than the constraint,
two round trips instead of one, and still wrong in the gap between them. Moving those rules out
of the schema does not spare the database; it gives it more to do and stops it being right.

Three mechanisms do cost enough to argue about, and the argument belongs in section 6 next to the
rule, not in a general policy against constraints:

- **Foreign keys** — an index probe on the parent and a `FOR KEY SHARE` lock on the parent row.
  Real contention when many children point at one hot parent; unmeasurable otherwise.
- **`EXCLUDE`** — needs GiST, and is heavier than the b-tree constraints above it.
- **Triggers** — actual execution on the write path, on top of the action at a distance that
  already makes them a last resort.

If write volume is the reason a rule leaves the schema, say so with the number from the PRD's
scale figures next to it. "Constraints are slow" as a standing principle is how a schema ends up
enforcing nothing at a write rate that never needed the concession.

**Conditional not-null is the most under-used rung.** "A confirmed booking must have a payment"
is not a `NOT NULL` and it is not application logic — it is
`CHECK (state <> 'confirmed' OR paid_at IS NOT NULL)`. A surprising share of rules that get
written off as "needs application logic" are one `OR` away from being a constraint.

**"The database cannot express this" must be tested, not recalled.** It is the most dangerous
sentence in the document, because it moves a rule to the bottom rung permanently and reads as
authoritative. Write the rule out as an actual predicate over columns, then check it against the
current documentation for the version in the stack document. If it turns out to be expressible,
you have not lost an argument — you have moved a rule up two rungs for the cost of one line.

**Landing on *Application* is allowed, and must be justified in one line.** "A four-state
transition table needs a trigger, and a trigger is harder to reason about than the invariant it
protects" is a real answer. Silence is not, and neither is putting it in the table without
comment — a rule listed as enforced when it is not is worse than a rule nobody wrote down. This
is the one place the writer count is worth knowing, and the point at which to ask: with a second
writer, *Application* is not a rung at all, and the row says the rule is unenforced rather than
enforced elsewhere. Write it that way — the risk register is only useful if it is honest about
which rules nothing holds.

## Denormalisation budget

Every stored value that duplicates something derivable is a value that can drift, and each one
needs a mechanism that keeps it honest or a written tolerance for being wrong. Allow **at most
two or three in the whole schema**, and spend them only where a measured query made it
necessary — not where one might.

Keep the count explicit as a short ledger: what is stored, what it is derivable from, what it
bought, and what keeps it honest. Every entry also needs a reconciliation in section 8: the
check that detects drift, how often it runs, and what it does when it finds some. A
denormalisation with no reconciliation is a value that will silently be wrong.

**Decide by direction, not by derivability.** "Could this be computed?" is the wrong test,
because the answer is almost always yes and it settles nothing. Ask instead whether the value
must **track** its source or **resist** it:

- **Track** — it should follow the source as the source changes. Age tracks today's date; a
  displayed total tracks its line items; a run's remaining seats track its bookings. **Derive on
  read.** Storing it creates drift with no upside.
- **Resist** — it must stay frozen as it was when the event happened, even after the source
  moves. The price on a paid booking resists a later price change; the address on a dispatched
  order resists the customer editing their address.

**What does not cost a budget entry.** A value that resists is not a denormalisation at all. The
price paid on a booking is not a copy of the workshop's current price — it is a *different fact*
that happened to have the same value once, and capturing it is the only way to keep it true
after the catalogue changes. The same goes for a name on an invoice or an address at time of
shipping. This is a category distinction, not an exception to a rule: framing it as an exception
invites an argument about where the exception ends, and there is no such boundary to police.
Counting these inflates the ledger, and an inflated ledger gets spent defending them instead of
the one or two places a real trade was made.

**When you do capture a computed verdict, capture the least that cannot be recomputed.** Most
fields of a computed result re-derive for free and must not be stored: anything derivable by a
stable, version-independent function of stored inputs (a sum, an average, a percentage), and
anything that is a static lookup off a column you already store (a label or category keyed off a
stored status). What is left — the part that would come out differently if the deriving logic
were re-run after it was tuned — is the only candidate. Store that, plus the immutable input,
plus a version tag.

**Better still: when the computation is fixed logic over tunable parameters, capture the
parameters, not the result.** The parameters in force are an *input*; the verdict is an
*output*. Store a version key that resolves to parameters held in version-controlled config and
recompute on read — the row stays free of derived values, and a captured *result* silently
disagrees with a fresh recompute once the parameters move, whereas captured *parameters* always
reconcile. Capture the output only once the *logic itself* can change past results. See
*Captured parameters, verdict recomputed* in `references/modelling-patterns.md` for what this
requires — chiefly that published versions are immutable.

**Never justify a captured value by "it saves maintaining code."** It does not — the code that
renders or interprets the stored shape still has to exist and stay compatible with it. The only
thing a capture buys is freedom from keeping every historical version of the logic executable.
A stored value is justified by freezing a version-sensitive verdict, and by nothing else.

## Writing rules

**Semantic types and named predicates — never DDL, never ORM code.** A schema file is generated
downstream from this document; if the document also carries `CREATE TABLE` blocks, there are two
schemas and they will disagree. But the altitude cut runs between *syntax* and *rule*, not
between "code" and "prose", and putting it in the wrong place throws away the thing this document
exists to produce:

| Belongs downstream | Belongs here |
|---|---|
| `text` vs `varchar`, the ORM's builder call, index and migration syntax | The semantic type: `ULID PK`, `timestamptz`, `integer minor units + currency`, `FK→run (SET NULL)` |
| How a constraint is spelled on a given engine | The constraint's **name** and its **predicate** |
| Which extension provides a mechanism | That the rule must be enforced, and at which rung |

**The test: if changing it changes which states are legal, it is design and stays. If it changes
only how the same states are stored or spelled, it is implementation and goes.** A predicate
written down as prose — "make sure bookings don't overlap" — has been deleted, not abstracted,
and the codegen step will re-derive it, usually as an application check that races.

So a table is a column table plus a block of named predicates:

```
runs_end_after_start      ends_at > starts_at
runs_capacity_positive    capacity > 0
webhook_events_once       unique (source, external_id)
bookings_confirmed_paid   state ≠ 'confirmed' OR paid_at is not null
bookings_no_overlap       no two rows with the same resource_id may have overlapping
                          [starts_at, ends_at) where state ≠ 'cancelled'
```

The last one is prose because the mechanism that enforces it is engine-specific while the rule
is not. That split belongs in the enforcement map: the invariant is design, the mechanism is a
note about the store the stack document chose, the syntax is implementation.

**Name every constraint, and treat the name as design.** The name is what appears in the error
a user eventually sees and in the log line someone has to act on. `bookings_no_overlap` says
what was violated; `bookings_check2` sends the reader to go and find out. That is not a detail
the code-generation step should be inventing.

**Value sets are named, not resolved.** Write `enum{draft, live, cancelled}` in the column
table. Whether that becomes a native enumerated type, a check constraint, or a lookup table is a
decision with its own argument and its own section — settling it inside a type annotation hides
it.

**Constraints for implementation.** A few rules cannot be expressed as schema and are reliably
got wrong by whoever writes the code from this document. Record them once, in the section they
belong to, rather than trusting them to be remembered: a date-only value parsed from a string
must be constructed explicitly rather than through a parser that assumes UTC midnight, or it
lands a day early for half the world; an API returns a full instant, never a datetime truncated
to a date, because truncation discards the offset and produces the same off-by-one; and any
value stored as `integer minor units` is formatted at the edge, never rounded in transit.

**Every decision carries its alternatives.** A modelling decision with no rejected option is not
a decision; it is a habit wearing a costume. Name the two real contenders and say in one line
each why they lost — against an invariant, not against vibes.

**A rejection on capability must be tested, not recalled.** Rejections that are safe from
knowledge: what a type means, what normalisation costs, which shapes are common. Rejections that
need checking: anything of the form "cannot", "does not support", "has no", and anything about a
specific version or a specific managed plan. The first kind is a fact about design; the second
is a claim about a surface that moves.

**Every nullable column states what its null means.** "Not yet paid", "never cancelled", "no
note given" are three different facts, and the column that carries them is the one place a
reader can find out which.

**And nothing stands in for null.** A `0` meaning "not measured", an empty string meaning "not
given", a sentinel date meaning "never" — each one is a value that arithmetic, sorting and
aggregation will silently treat as real. Zero measured and not measured are different facts and
must be different values. The two rules are opposite failures of the same decision: use null,
and say what it means.

**`created_at` on every table; `updated_at` only with a trigger.** A creation timestamp
defaulted by the engine holds against every writer, including a migration and a console session,
so it is free and it is the one column that is always wanted later and cannot be backfilled. An
`updated_at` maintained by application code is correct only for writes that go through that
code, which excludes backfills, a second service, and anyone with a SQL prompt. If it drives a
sync cursor, an incremental export, or cache invalidation, it needs a `BEFORE UPDATE` trigger,
because a missed update there is a correctness bug that surfaces as missing data much later. If
it only renders "last modified" in a UI, application-maintained is fine and the tolerance is
written down. What is not acceptable is the column existing with nobody having chosen which of
those it is.

**Non-choices are stated positively.** "No history table on `customers`: the PRD asks no
question about a previous email address, and adding one later cannot recover what was not
recorded — revisit if support ever needs it." Silence is what an agent fills in on its own
initiative.

**Volume figures are dated and sourced.** Row counts and storage estimates in section 8 come
from the PRD's success metric, not from imagination, and they say which. A storage estimate that
decides whether a plan is adequate must show its arithmetic — bytes per row times rows per
year — because that is the only part a reader can check.

**Traceability both ways.** Each table and constraint references the invariant that forced it;
each invariant in section 2 is visibly answered in section 6. In HTML this is literal: the
invariant table's last column links to the mechanism, so an unenforced invariant is visible on
the page rather than only in your head.

## HTML output

The document is **one file that opens by double-clicking it**. Nothing else may be needed to
read it — not a server, not a build step, not an internet connection, not a font download.

- **Self-contained.** All CSS in a single `<style>` block; take the template's and leave it
  alone. No CDN links, no external stylesheets, no web fonts, no images, **no JavaScript**.
  System font stacks only — a serif for reading, a sans for labels and tables, a mono for
  predicates, column names and identifiers.
- **The same house style as the PRD and the stack document** — literally the same stylesheet,
  not a resemblance. The three are read together and should look like they came from one hand.
  If you change something here, change `references/prd-template.html` in `brief-to-prd` and
  `references/stack-template.html` in `prd-to-stack` to match, or the family drifts apart one
  document at a time. The shared stylesheet carries a few components this document does not
  use; leave them alone.
- **The text sits on a bordered card.** `main` is the only panel: no nested cards, no callout
  boxes, no colour-coded blocks, no shadows.
- **Semantic structure.** One `<h1>`, one `<section>` per numbered section with an `id`, `<h2>`
  for section headings, `<h3>` per table and per decision, `<h4>` for the labelled blocks inside
  a decision. Real `<table>` for every table.
- **Invariants and decisions are addressable.** `id="i1"` on every invariant row, `id="dm-1"` on
  every decision heading, `id="t-bookings"` on every table heading. The document cross-references
  itself constantly and a later prompt needs to point at one decision precisely.
- **Predicate blocks go in `<pre><code>` and must be escaped.** `<`, `>` and `&` are everywhere
  in a constraint predicate. A raw `<` silently eats the rest of the block — which is exactly the
  failure this document exists to prevent, committed in the document itself. Prefer the typed
  symbols `≤ ≥ ≠` in predicates: they need no escaping, and they read as rules rather than as
  code, which is the altitude this document is written at.
- **Wide tables scroll inside themselves.** Wrap them in a container with `overflow-x: auto` so
  the page body never scrolls sideways on a phone.
- **Works in both themes.** Palette as custom properties on `:root`, overridden inside
  `@media (prefers-color-scheme: dark)`. Never hard-code a colour — this is also what makes the
  diagrams theme-aware, since custom properties cascade into inline SVG.
- **Prints cleanly.** Keep the template's `@media print` block. Data models get printed and
  argued over.

HTML is the presentation. It does not license a longer or more decorated document, and it does
not change what a table has to contain: a type, a constraint, an invariant it serves, and a
stated meaning for every null.

## Illustrations

Three of this document's questions are structural or quantitative — *what holds what*, *where
should I argue*, and *does this fit the plan we bought* — and prose answers all three slowly.

**The test: name the question the picture answers.** If the caption can only say what the
picture is ("Entity diagram"), it has not earned its place; if it can say what the reader should
take from it, it has. Never draw what the adjacent table already says.

| Illustration | The question it answers | Form |
|---|---|---|
| **Entity sketch** (§1, required) | What holds what, and where do the invariants live? | Inline SVG |
| **Blast-radius ladder** (§5, required) | Where should I spend my critique? | CSS bars |
| **Storage against the plan** (§8, required) | Does this fit what the stack document bought? | CSS bars |
| **Denormalisation pips** (§1 and §5) | How much drift is in here? | CSS, inline |
| **Migration chain** (§10) | What must exist before what? | CSS list |
| **The write path** (§9, required if anything is contended) | Where is the transaction boundary, and what is outside it? | Inline SVG |

That last one is the most valuable diagram in the document and the most rationed: draw exactly
one, for the single flow carrying the most risk — the money path, the capacity claim, the token
exchange. It must show where the transaction commits and where the call to something outside our
control sits, because those two facts are what make the flow correct and neither is visible in
prose. A picture of a generic write teaches nobody anything.

Everything else stays a table. Columns, constraints, indexes and retention rules are comparisons
across named things, and a table beats a picture at that every time. **An entity sketch is not
an exhaustive ERD** — if it needs every column, it has stopped answering a question and started
duplicating section 4.

### Which library

**None — hand-written inline SVG and CSS.** A diagram library needs JavaScript, which breaks
"opens by double-clicking, offline, forever"; rendered-at-runtime diagrams theme badly, print
badly, and change appearance when the library version moves. The template has working examples
of every idiom — copy them rather than inventing them.

### SVG that does not come out broken

- **`viewBox` plus `width: 100%`, never fixed pixel dimensions.** Coordinates are design units.
- **SVG text does not wrap.** Every line is its own `<text>` or `<tspan>` with its own `y`.
  Budget roughly 0.55 × font-size per character: about 22 characters at 13px in a 170-unit box.
  A third line means the label is too long.
- **Set vertical position with `y`, not `dominant-baseline`.** Baseline handling differs between
  browsers and print engines.
- **Colour comes from CSS classes, never from a `fill` attribute.** That is what makes it work
  in dark mode and in print without a second copy.
- **One arrowhead `<marker>` in `<defs>`, reused.** Two — one muted, one accent — is enough;
  emphasise exactly one path, the one carrying the hardest invariant.
- **Always `role="img"` with a `<title>` and `<desc>`, and always a `<figcaption>`.** The
  description says what is drawn; the caption says what to conclude.

## Constraints

- **Never re-decide the product.** No new features, no changed business rules, no altered scope.
  If a PRD rule cannot be expressed as an invariant, that usually means it has not been
  decided — record it in section 11 and let the user amend the PRD.
- **Never re-decide the stack.** The store, its version and its plan are settled. If the model
  needs something the store cannot do, that is a note back to the stack document, not a
  substitution made here.
- **Do not create the database, run a migration, or write application or ORM code.** The
  document is the whole deliverable, and a schema file is generated from it downstream. Two
  artefacts describing one schema drift, so this one carries semantic types and named predicates
  and never `CREATE TABLE`, never a builder call, never migration syntax.
- **Stay engine-neutral in the design, engine-specific only in the enforcement map.** Which
  extension provides a mechanism, and what it is called, is a note about the store the stack
  document chose — it belongs beside the invariant it serves, not inside a type annotation.
- **No table without a trace.** If the PRD is silent on something the schema seems to need, say
  the choice was made on defaults or record it as an assumption — do not invent a requirement to
  justify a more interesting model.
- **No invariant without a mechanism.** Every row in section 2 is answered in section 6, even
  when the answer is "application code, because …".
- **The loop is bounded; the document is not allowed to be.** Three critique rounds at most, and
  a round that adds material no blocking finding demanded has gone wrong. Findings that outlive
  the cap become open questions in section 11 — never silent omissions.
- **Do not modify the PRD, the stack document, the brief, or any questionnaire.** They are
  inputs.
- **Do not hide a guess.** Anything decided without evidence — a volume, a limit, a capability —
  goes in section 11, visible.
- **No JavaScript in the document, and no external asset of any kind.** If an illustration seems
  to need either, it is the wrong illustration.
