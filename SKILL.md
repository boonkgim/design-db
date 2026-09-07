---
name: db-design
description: Turn an approved PRD and its tech stack decision into a numbered data model document a coding agent can generate the schema from - every table, column, constraint and index traced to a business invariant, every modelling call argued with its alternatives, and every rule enforced at the lowest level that can express it. The document is a single self-contained HTML file carrying semantic types and named constraint predicates rather than engine DDL, with an entity sketch and a write-path diagram. Use when the user asks to design a schema, model the data, decide tables, keys and constraints, write the DDL, or answer "how should this be stored" after a PRD and a stack exist.
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

5. **Ask only what the PRD and the stack document deliberately exclude** — chiefly whether data
   already exists, and who else writes to this database. Ask inline, with a recommended default
   for each, and proceed on defaults if the user does not care. See *What to ask*. Do not open a
   questionnaire round for this.

6. **Model the tables**, applying the tests in *How to model* and the ladder in *The enforcement
   ladder*. Use `references/modelling-patterns.md` to **eliminate, never to pick** — a shape
   chosen from that file and justified afterwards is the exact failure the *Core principle*
   forbids.

   Settle two things before the rest, because they are not independent of anything: the **key
   strategy** — see *Keys and identity*, and settle it together with anything created outside
   the database, since those are one decision — and **what is append-only**. Both propagate into
   every table, and both are the most expensive things in the document to change once rows exist.

   Before writing that the store cannot express something, **write the rule out as an actual
   predicate** and check it — see *Writing rules*. Batch every capability and limit question you
   are going to need and send them out at once, the way *Context and fan-out* describes, rather
   than fetching a documentation page in the middle of an argument.

7. **Write the document.** If a data model already exists for this PRD and this run is
   correcting it, edit that file in place — see *Folder convention*. Otherwise write to the next
   free number (`NN-data-model.html`) by copying `references/data-model-template.html` and
   replacing its content. **Copy it with `cp` — do not read it into context.** It is a large
   file, most of it a stylesheet you will not change, and the copy on disk already contains all
   of it. Open it in a browser first if you want to see the idioms working; looking at it costs
   nothing, reading it costs a large fraction of the run. Its head comment is addressed to you,
   not to the reader: **delete that comment from the copy.** Number invariants `I1`, `I2` … and
   decisions `DM-1`, `DM-2` …, each with an `id`.

8. **Read it back as the builder.** Re-read it as an agent about to write the first migration
   with no other context. Every question it cannot answer — what type is that column, what
   happens on delete, is this unique, what does that null mean, in what order do these get
   created — is a gap. Then walk each PRD journey's writes through the tables, including the
   unhappy branches. A journey that cannot be executed against this schema is the cheapest bug
   you will ever find.

9. **Assert, then look.** Most defects here are machine-checkable and a script costs a fraction
   of a screenshot: body scroll width against viewport width, every `.wide` container scrolling
   inside itself, each `<text>` element against its `viewBox` and against the `<rect>` it sits
   in, every `href="#…"` resolving to an `id`, no bracketed placeholder surviving, and — specific
   to this document — every named predicate referring to a column that section 4 defines, every
   table referenced by a foreign key existing, and every `I` and `DM` reference resolving. Grep
   for `CREATE TABLE`, `pgTable`, `ALTER TABLE` and `sqliteTable` while you are there: any hit is
   the altitude slipping. Run it at desktop and at phone width. Then open it in a browser
   to judge what a script cannot: whether a diagram reads, whether a label collides with an
   arrow. Once per illustration, and once in the dark theme.

10. **Report** the file path, the table count, the invariant count and how many of them the
    database enforces, how many denormalisations were spent, and any open questions. Say the
    file opens by double-clicking it. State that the user should critique it before any
    migration is written, because every constraint in it is cheap to change now and expensive to
    change once there are rows. Do not create the database.

## Context and fan-out

The inputs to this document are a PRD and a stack document, both HTML, and both large. The
writing itself is not what overruns a run — **page content is**: a store's constraint reference,
a plan's limits page, an ORM's migration documentation, each thousands of tokens of which two
lines decide anything.

**Delegate the looking up. Never delegate the modelling.**

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

### Never fan out

- **The invariants.** They are a single consistent set, cross-referenced by every later section,
  and two agents writing them produce two lists that disagree.
- **The tables.** A schema is one artefact; foreign keys, naming and key strategy must agree
  across every table, and reconciling independently-written tables costs more than writing them.
- **The illustrations.** Geometry is laid out against the actual labels.

### Cheap wins first

- **Copy the template, never read it** — see step 7.
- **Read the PRD and the stack document as extracted text, not as HTML.**
- **Assert, then look** — see step 9.

## What to ask

Five questions, each with a default, answerable in one line. Ask them together, once.

- **Does a database already exist with real rows in it?** (No, greenfield · yes, with data that
  matters · yes, but disposable) — the single biggest determinant of how section 10 is written.
  Greenfield means migrations can be edited freely; live data means every change is additive
  first.
- **Who else writes to this database?** (Only this application · a second service · a BI tool or
  an operator with a console) — every extra writer moves rules down the enforcement ladder,
  because a rule held in one application's code is not held at all once there are two writers.
- **Is there data to import from somewhere** — a spreadsheet, an old system, a payment
  provider's history? It usually carries identifiers that must be kept and duplicates that must
  be reconciled, and both change the model.
- **Any naming or structural convention already in force** — a company standard, an ORM's
  expectations, a schema someone will read alongside this one?
- **Anything that must be deletable or exportable on request** beyond what the PRD's data
  section already says?

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
comment — a rule listed as enforced when it is not is worse than a rule nobody wrote down.

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
- **Do not modify the PRD, the stack document, the brief, or any questionnaire.** They are
  inputs.
- **Do not hide a guess.** Anything decided without evidence — a volume, a limit, a capability —
  goes in section 11, visible.
- **No JavaScript in the document, and no external asset of any kind.** If an illustration seems
  to need either, it is the wrong illustration.
