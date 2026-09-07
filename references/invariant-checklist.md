# Invariant checklist

A sweep of the places invariants hide, so that section 2 of the data model is complete rather
than merely plausible. Each entry is a **question to put to the PRD**, not a feature to add.
Most will come back "the PRD does not say", and that answer is the point: it is either an
assumption to record or a question to send back.

## How to use this file

Run it once, after the invariants you already know are written down, and before any table is
designed. It is a completeness check on section 2 — not a schema generator.

Two rules:

- **A category the product does not have is answered "none, because …", not left blank.** A
  product with no money, no scheduling, and no multi-tenancy is common and fine; what is not
  fine is discovering in week three that nobody asked.
- **Never add a table because this file mentions one.** Every table in the document has to
  trace to a PRD requirement. This file finds rules; the PRD authorises them.

---

## Before anything: what the store gives you for free

Settle this first, because it decides which mechanisms are available to every category below.
The store is PostgreSQL and was settled by the scaffold; this is reading its capabilities, not
re-opening the choice. Three rows are answered before you start — they are here so the answers
are stated rather than assumed — and the rest still have to be checked against the managed
instance, which is the one that can differ from the local Docker one.

| Ask                                                                             | Why it decides things                                                                                                                                                                                                    |
| ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Are there real multi-statement transactions?                                    | **Yes.** So "two tables must agree" is enforceable, and a rule that cannot be expressed has no excuse waiting for it                                                                                                     |
| What is the default isolation level, and is `serializable` available?           | **Read committed, and yes.** So a read-then-write needs a guarded single statement, an explicit lock, or a retry loop — and section 9 says which, naming the anomaly rather than the level                               |
| Are `CHECK`, `UNIQUE`, partial indexes, and deferred constraints all supported? | **All four.** A rule that lands on application code here did so by choice, and section 6 has to say whose                                                                                                                |
| Which extensions can be installed on the Neon plan?                             | Exclusion constraints need `btree_gist`; a managed provider may not allow it. Check it against the plan before designing on it, and print the source and date                                                            |
| Is there a hard row, size, or connection limit?                                 | From the Neon plan. Connections come through Hyperdrive, which pools them — say so rather than budgeting for direct ones                                                                                                 |
| Does the product need semantic search, recommendations, or dedup-by-similarity? | Then embeddings are part of the data model: `pgvector` in the same database if the plan permits it, a separate store joined by a stable id if not. Decide which row the vector belongs to before deciding where it lives |

---

## Shape: one fact, in one place

The three normal forms are a vocabulary, not a procedure — nobody diagnoses "this violates 2NF"
in the wild. Unfolded into checks you can actually run against a column list:

| Ask                                                                                                                               | What it catches                                                                                                                                               |
| --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Does any column hold a list — comma-separated values, an array standing in for a child table, or `phone_1`, `phone_2`, `phone_3`? | A child table that was never created. Filtering, counting and constraining all become impossible on a value that should have been a row                       |
| Is every row distinguishable by a real key?                                                                                       | A table with no identity, where a duplicate cannot be detected and a single row cannot be updated                                                             |
| Where the key is composite, does every non-key column depend on the **whole** key?                                                | `order_item(order_id, product_id)` storing `product_name`, which depends on the product alone. It will disagree with the product table the day one is renamed |
| Is any non-key column determined by **another non-key column**?                                                                   | `order.customer_email` beside `order.customer_id`. The email now has two homes and one of them goes stale                                                     |

**The last two are only violations if the value should track its source.** `order.customer_email`
is either a redundancy bug or a deliberate point-in-time capture, and nothing about the shape
tells you which — only the track/resist test does. So route every hit here into that test rather
than "normalising" it away: a captured email on a dispatched order is correct and must not be
replaced by a join.

## Identity and keys

| Ask                                                                                                                          | Usually enforced by                                                                                                                                                                                                                               |
| ---------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| What identifies a row to the business, as opposed to to the database?                                                        | Surrogate primary key, plus a `UNIQUE` on the natural key                                                                                                                                                                                         |
| Does this row belong to a person?                                                                                            | Then it references the generated `user` table and inherits Better Auth's key type. That is not this document's decision to make, and the mismatch with the key strategy is stated in section 3 rather than resolved by re-keying a generated file |
| Is anything created outside the database alongside this row — a stored object, a file, a provider-side record?               | If yes, the key must be generatable in the application, so the external write can go first and a failure leaves inert garbage rather than a live broken row                                                                                       |
| Does the id need to hide when the row was created?                                                                           | Time-ordered keys (UUIDv7, ULID) leak it by construction; only a random key does not                                                                                                                                                              |
| Will a human ever read, type or dictate this id?                                                                             | Then it wants a separate short public reference, not an encoding chosen for the primary key                                                                                                                                                       |
| Can that business identifier ever change? (An email, a username, a slug, an external code — the answer is almost always yes) | Never make it the primary key; `UNIQUE` on the column and let it change                                                                                                                                                                           |
| Does any identifier ever appear in a URL, an email, or an API response?                                                      | If yes, sequential keys leak volume and adjacency — decide the public identifier separately from the key                                                                                                                                          |
| Are ids generated anywhere other than inside the database?                                                                   | Client- or service-generated ids need a globally unique scheme; nothing else does                                                                                                                                                                 |
| Can two rows describe the same real thing? (Two customer rows for one person, two workshops with the same title)             | A `UNIQUE` if the business says never; a merge procedure and no constraint if it says sometimes                                                                                                                                                   |

The recurring failure is a natural key that was stable right up until the day it was not. If
you cannot name the circumstance under which a value changes, that is not evidence it is
stable — it is evidence nobody has asked.

## Money

Every product that touches money has more invariants here than its PRD states.

| Ask                                                                              | Usually enforced by                                                                                                    |
| -------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| What is the unit, and is it the same everywhere?                                 | Integer minor units, with the currency stored beside every amount — never a bare number                                |
| Is more than one currency possible, ever?                                        | A currency column beside every amount, even in a single-currency v1: adding one later means finding every amount       |
| Can an amount be negative, and what does that mean?                              | `CHECK` naming the meaning: a refund, a credit, a correction. "Negative is impossible" is also a rule and gets a check |
| Is a stored price a copy of a catalogue price, or a fact in its own right?       | If the catalogue can change, the paid price is its own fact and must be copied — this is not a denormalisation         |
| Is any balance stored, and can it disagree with the entries that produced it?    | Prefer deriving it. If stored, name what keeps it honest and how drift is detected                                     |
| Is every charge traceable to what authorised it, and every refund to its charge? | `NOT NULL` foreign keys in both directions, and append-only rows                                                       |
| Can a row that represents money ever be updated or deleted?                      | Almost never. Corrections are new rows; this is what makes reconciliation possible at all                              |
| How is rounding done, and who absorbs the remainder when a total is split?       | A rule, written down. Splitting is where cents disappear                                                               |

## Capacity and conflict

The category that produces the hardest invariants, because they are the ones that only break
under simultaneity.

| Ask                                                         | Usually enforced by                                                                                                                                        |
| ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| What is finite? Seats, slots, stock, licences, a rate limit | Something that makes over-allocation impossible: a guarded single statement, a row per unit, or an exclusion constraint                                    |
| Is the limit hard or soft?                                  | Decides whether the count may go negative. A soft cap with a signed count is often what the business actually wants, and a `CHECK (n >= 0)` would be wrong |
| Can two things overlap in time on the same resource?        | An exclusion constraint over a range, with the equality column first — the only concurrency-safe answer                                                    |
| Are there holds or reservations that expire?                | An expiry column plus a partial index the expiry job uses; and a decision about what happens when a payment lands after the hold lapsed                    |
| What does the loser of a race see?                          | A business outcome, from the PRD. If the PRD does not say, that is a question, not an assumption                                                           |
| Can the limit itself change while claims are outstanding?   | Decides whether the constraint can be a `CHECK` at all                                                                                                     |

## Time

| Ask                                                           | Usually enforced by                                                                                                                                                                                                                                                                                   |
| ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Is every instant stored with its offset?                      | `timestamptz` everywhere; a bare local timestamp cannot answer "was this before the deadline"                                                                                                                                                                                                         |
| Is there a business timezone distinct from the user's?        | Store the instant, store the timezone name, derive the local rendering. Never store a local time and hope                                                                                                                                                                                             |
| Which pairs of times must be ordered?                         | `CHECK (ends_at > starts_at)` — cheap, and catches a whole class of bug                                                                                                                                                                                                                               |
| Is any period half-open, and is that consistent?              | Pick `[start, end)` everywhere. Mixing conventions is where double-counting at midnight comes from                                                                                                                                                                                                    |
| Does anything expire, and is expiry a state or a computation? | Prefer computing it from a timestamp; a stored `expired` flag needs a job and can be wrong                                                                                                                                                                                                            |
| Is a date a date, or an instant?                              | A birthday is a `date`. A deadline is an instant. Storing the first as an instant creates a timezone bug that appears once a year                                                                                                                                                                     |
| Does anything read `updated_at`?                              | If a sync cursor, an incremental export or a cache invalidation reads it, it needs a trigger — application code misses backfills, second services and console edits, and a missed update there loses data silently. If only a UI reads it, application-maintained is fine and the tolerance is stated |

## State and lifecycle

| Ask                                           | Usually enforced by                                                                                                              |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| What are the states, exhaustively?            | An enumeration — and the exhaustive list is the deliverable, not the type                                                        |
| Which transitions are legal?                  | Often only application code. If so, say so in the enforcement map rather than implying the database checks it                    |
| Is any transition irreversible?               | A timestamp column set once and never cleared expresses this better than a boolean                                               |
| Does a state imply another column is present? | `CHECK (state <> 'confirmed' OR paid_at IS NOT NULL)` — a conditional not-null, which is the most under-used constraint there is |
| Can a row move backwards?                     | If never, the state is derivable from timestamps and may not need a column at all                                                |

## Relationships and cardinality

| Ask                                                                                        | Usually enforced by                                                                                                                                                                                                                                                                                                |
| ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| For each reference: is it required or optional, and what does the null mean?               | `NOT NULL` plus a stated meaning for every nullable foreign key                                                                                                                                                                                                                                                    |
| What happens to children when a parent goes away?                                          | An explicit `ON DELETE` on every foreign key — `RESTRICT` by default, `CASCADE` only where the child genuinely has no independent existence                                                                                                                                                                        |
| Is a "many to many" really that, or does the link carry its own facts?                     | If the link has attributes — a quantity, a price, a role, a date — it is an entity, not a join table                                                                                                                                                                                                               |
| Is any reference both required and unique — a forced 1:1?                                  | Only correct for a pure extension row. Otherwise an optional reference on the many side, which behaves the same today and does not forbid a second child tomorrow                                                                                                                                                  |
| Can the parent ever have a second child — a retake, a revision, a repeat, a re-submission? | If yes it is 1:N now, whatever the first write path does                                                                                                                                                                                                                                                           |
| Was a join table reached for to model a 1:N?                                               | If one side has at most one owner, an optional reference models it and the join table is machinery with no purpose                                                                                                                                                                                                 |
| Is there exactly one "primary" child?                                                      | A partial unique index (`WHERE is_primary`) rather than a flag nobody enforces                                                                                                                                                                                                                                     |
| Does an entity have several types with different fields?                                   | Separate tables per type. A single table with a `type` column and a drift of nullable type-specific columns cannot express "this field is required _for this type_", so nothing is enforced for anyone. Exceptions: the types share all fields, the types are user-defined at runtime, or the row count is trivial |
| Can a row reference something in a different tenant, customer, or account?                 | A composite foreign key that carries the tenant, or a check. This is the classic cross-tenant leak                                                                                                                                                                                                                 |

## Uniqueness

| Ask                                                | Usually enforced by                                                                                                                                          |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| What must be unique, and under what conditions?    | A partial unique index when uniqueness only applies to live rows — "one active subscription per customer" is not the same as "one subscription per customer" |
| Is uniqueness case- or whitespace-sensitive?       | A unique index on a normalised expression, or the case-insensitive type. Two accounts differing only in capitalisation is a support ticket                   |
| Does a soft-delete flag break a unique constraint? | It always does. This is the most common reason a blanket soft delete fails, and it is a reason to reconsider the soft delete, not to drop the constraint     |

## Text, enumerations and units

| Ask                                                                                                            | Usually enforced by                                                                                                                                                                                   |
| -------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Does any length limit come from the business, or was it invented?                                              | A `CHECK` for a real limit; nothing at all for an invented one                                                                                                                                        |
| Can this column be empty string, and does that differ from null?                                               | Almost never should both be legal — a check that the trimmed length is non-zero, and let null carry "absent"                                                                                          |
| Does any value stand in for "no data" — a `0`, an empty string, a sentinel date like 1970-01-01 or 9999-12-31? | Null, always. A sentinel is a real value to every sum, average, sort and comparison in the system, so "not measured" silently becomes "measured as zero" and nobody finds out until a report is wrong |
| Does every quantity carry its unit, in the column name or in a column?                                         | `duration_minutes`, not `duration`. A unitless number is a future incident                                                                                                                            |
| Is this set of values in the customer's vocabulary or only in an engineer's?                                   | Customer-facing values want a lookup table with labels and ordering; internal states are fine as a plain enumeration                                                                                  |
| Is any free-text field actually structured?                                                                    | If it is ever filtered or grouped on, it is a column, not prose                                                                                                                                       |

## History, audit and deletion

| Ask                                                                         | Usually enforced by                                                                                               |
| --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Which questions about the past will someone eventually ask?                 | Decides whether history is recorded at all. History never recorded cannot be reconstructed                        |
| For each table: does "delete" mean gone, hidden, or superseded?             | A per-table answer. A blanket `is_deleted` column across the schema is a decision nobody made                     |
| Is there a legal or contractual retention period, or a deletion obligation? | Both, usually in tension: keep the financial record, destroy the personal data. That is a tombstone, not a delete |
| Can a row be restored after removal, and to what state?                     | If yes, it was never a delete — it is a state                                                                     |
| Who changed this, and does anyone need to know?                             | An audit trail is a separate table; putting `updated_by` on a row records only the most recent answer             |

## Visibility and tenancy

| Ask                                                                   | Usually enforced by                                                                                                                                                                                                                                       |
| --------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Can one customer's data ever be returned to another?                  | A tenant column on every table that has one, and a policy that makes forgetting it impossible rather than merely discouraged                                                                                                                              |
| **Who owns a resource — a user, or an account that users belong to?** | Unless the product is single-user forever, ownership belongs to an account, with users joined to it and carrying a role. Adding the account now costs one table and one column; retrofitting it after launch rewrites every ownership check and every row |
| Can a row reference something owned by a different tenant?            | A composite key carrying the tenant, or a check. This is the classic cross-tenant leak, and it is written by an ordinary-looking join                                                                                                                     |
| What can the operator see and change?                                 | From the PRD. It decides whether "admin" is a role in the data or a separate surface                                                                                                                                                                      |
| Is anything hidden rather than absent — a draft, an unpublished item? | A state and a partial index, not a flag consulted by convention                                                                                                                                                                                           |

## External systems

| Ask                                                                             | Usually enforced by                                                                                                                                                                                                                                                                        |
| ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Which external identifiers must we keep forever?                                | A column, `UNIQUE`, never reused — a charge id, a subscription id, a message id                                                                                                                                                                                                            |
| Can the same external message arrive twice?                                     | It can. A unique constraint on `(source, external_id)` and an insert that does nothing on conflict                                                                                                                                                                                         |
| Can we tell a duplicate from a legitimate repeat?                               | If the business allows two identical operations, the idempotency key must come from the caller, not be derived from the contents                                                                                                                                                           |
| What is the external service's own replay window?                               | It decides how long the deduplication rows are kept — not a round number picked for tidiness                                                                                                                                                                                               |
| Is anything written to an external system inside a database transaction?        | It must not be. That is a section 9 question and it is a correctness one                                                                                                                                                                                                                   |
| When a row and an external object are created together, which is written first? | Whichever ordering leaves inert garbage on failure. An orphaned object nothing references is swept later; an orphaned row is live, appears in listings, and points at nothing. Writing the external thing first requires the id to exist first — an identity decision, not an ordering one |
| If the second write never happens, what finds the leftovers?                    | A sweep, and the predicate it runs on. "We would notice" is not a mechanism                                                                                                                                                                                                                |

---

## Cross-cutting checks

Run these at the end, over the whole model.

- **Every invariant in section 2 appears in the enforcement map.** A rule with no mechanism is
  a rule that is not enforced, and writing it down without a mechanism is worse than not
  writing it, because the document then implies it holds.
- **Every nullable column has a stated meaning.** Sweep the DDL for `NULL` and check each one
  against section 4. This finds more real bugs than any other check here.
- **Every foreign key has an explicit `ON DELETE`.** The default is not a decision.
- **Every index names a query.** An index nobody reads is a write tax that outlives everyone
  who remembers it.
- **Every denormalised value has a reconciliation.** Or a written tolerance for being wrong.
  There is no third option.
- **Every stored value that could be computed has been through the track/resist test.** Track →
  derive on read. Resist → capture, and capture only the part that cannot be recomputed, with
  the input and a version tag beside it.
- **No sentinel values.** Sweep for `0`, `''`, and any suspiciously round date used to mean
  "none".
- **Every table has `created_at`.** The one column that is always wanted later and cannot be
  backfilled. Every `updated_at` has a stated mechanism and a stated reader.
- **No engine-specific syntax outside the enforcement map.** Types are semantic, constraints are
  named predicates. `CREATE TABLE` in a design document means a second schema now exists.
- **No table is named for a technology or a layer.** `bookings`, not `booking_records`,
  `booking_data`, or `tbl_booking`.
- **Read the PRD's journeys against the schema, one at a time.** Walk each journey's writes
  through the tables. A journey that cannot be executed, or that needs a column nobody
  designed, is the cheapest bug you will ever find.
- **Read the PRD's unhappy paths the same way.** Cancellation, refund, expiry, and failure are
  where models are thin, because the happy path is the part everyone thought about.
