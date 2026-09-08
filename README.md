# design-db

**A schema is not a place to put the data — it's the set of states your product is allowed
to be in. This turns an approved PRD into that decision, before it gets made by accident
inside whichever feature slice reaches the database first.**

An [agent skill](https://agentskills.io) for Claude Code, Codex, and any other AI coding
agent that reads `SKILL.md`. Hand it an approved PRD and it writes a numbered data model
document: every table, column, constraint and index traced to a business invariant, every
modelling call argued against its alternatives, and every rule pushed down to the lowest
level of the database that can actually make it impossible. It writes semantic types and
named predicates, never `CREATE TABLE` or ORM code — the document is a specification a
schema is generated from later, by you or another skill, as a separate step.

This repo's own [commit history](https://github.com/boonkgim/design-db/commits/main) *is*
the skill's design record: every commit that shaped it carries the real prompt that drove
the change, in order, from the first version through the project it was adopted into. Read
it before you install anything.

## Why you would want this

A coding agent asked to "add bookings" will pick a key strategy, a cardinality, and a
delete policy on the spot, inside whatever file it's editing — usually correctly for the
one case in front of it, and silently wrong for the one that shows up in three months. The
fix is not more careful prompting; it's a document those decisions have to pass through
before any code exists.

- **The database is the only layer every writer passes through.** Structure beats a type, a
  type beats a constraint, a constraint beats a transaction, and all of them beat
  application code, because a migration, a background job, and an operator with a console
  at 2am all skip the app. The skill pushes every invariant down an enforcement ladder and
  records exactly where it stopped — the rows that land on "application code" become the
  document's risk register, not a silent gap.
- **Traceable both ways.** Every table and constraint names the invariant that forced it;
  every invariant is answered somewhere, even when the honest answer is "application code,
  because …". Nothing is modelled from a guessed query pattern.
- **Drafted once, checked once, not negotiated over rounds.** A generate pass writes the
  whole document in one context; a critique pass reads it against a fixed checklist and may
  only report findings, never edit; a fix pass applies them. Three files, one pipeline, and
  the working files never sit in your project's docs folder looking authoritative.
- **A denormalisation budget, not a denormalisation ban.** At most two or three stored
  values that duplicate something derivable, each with a stated reconciliation — and a
  track-versus-resist test that tells you which ones don't even count.
- **Keys, ownership and write-ordering are settled once, deliberately.** UUIDv7 or ULID by
  default and why, an account-and-grant shape for anything more than one person can act on,
  and a rule for which order to write a row and an external object in so a failure leaves
  garbage instead of a live broken row.
- **A PRD that's thin on a business decision gets a question, not a guess.** Most gaps a PRD
  leaves are settled from evidence and recorded as an assumption, no interruption. But money,
  cancellation, concurrency, retention, and ownership are load-bearing: a gap in one of those
  stops the run with a short written question before any table gets drafted, instead of
  quietly modelling on top of whichever default felt plausible.

## Folder convention

The data model lives in the PRD's folder, at the next free number, with its working passes
kept out of it entirely:

```
docs/<dated-folder>/
  01-brief.md                  <- not read
  02-create-prd-questions.md   <- not read
  04-prd.md                    <- the input, and the only one
  05-design-db-questions.md    <- written only if a business-rule gap blocks the run
  06-data-model.md             <- next free number; this skill's output

.cache/06-data-model/          <- gitignored; deleted or overwritten on the next run
  01-first-pass.md             <- the generate pass writes this
  02-critique.md                <- the critique pass writes this — findings only
  03-final.md                   <- the fix pass writes this; the only file copied to docs/
```

Follow your own project's docs convention if it has one. Nothing is written into `docs/`
until the final copy — a document still being argued with must not sit where the next
agent would read it as settled.

## Install

Paste this to your agent:

```
install the skill at https://github.com/boonkgim/design-db
```

It clones the repo and puts `SKILL.md` and `references/` where your tool looks for skills.
To update it later, ask the same way, or `git pull` in the clone.

<details>
<summary>By hand</summary>

```bash
git clone https://github.com/boonkgim/design-db.git

# Claude Code
ln -s "$PWD/design-db" ~/.claude/skills/design-db

# Codex
ln -s "$PWD/design-db" ~/.agents/skills/design-db
```

Symlink into a project's `.claude/skills/` instead to scope it to one repo. Other tools
read skills from their own location, and some take an upload; check yours.

</details>

A skill is instructions your agent will follow, so read `SKILL.md` before installing this
or any other. It is one file, plus three reference files it reads or copies from.

## Works with

`SKILL.md` follows the [Agent Skills](https://agentskills.io) open standard, so it loads
directly in any agent that reads the format — **Claude Code**, from `~/.claude/skills/`,
**OpenAI Codex**, from `~/.agents/skills/`, and any other tool with its own skills
directory. Where a tool does not read `SKILL.md` natively, paste it into the session or
drop it into the rules file that tool already reads, such as `AGENTS.md`. Nothing in it is
tool-specific — the whole skill is prose and markdown, illustrated in PostgreSQL.

## Usage

Have an approved PRD saved somewhere under `docs/<dated-folder>/`. Tools that support
invoking a skill by name take `/design-db` directly; otherwise just ask for a data model,
schema, or "how should this be stored" once the PRD exists.

The skill reads the PRD — and the project's existing schema file, if one exists — extracts
every invariant, and sweeps for what the PRD didn't say. If that sweep finds a gap in money,
cancellation, concurrency, retention, or ownership, it stops and writes
`NN-design-db-questions.md` instead of guessing; fill that in and ask again. Otherwise it
settles the key strategy and what's append-only, delegates the store's capabilities and
limits to concurrent lookups, then runs the generate/critique/fix pipeline over one run
folder. It never touches the schema file, never runs a migration, never creates the
database, and never commits. Once you've approved the document, turning it into a schema is
a separate step.

If this is useful, a ⭐ helps other people find it.

## When not to use this

- **There's no approved PRD yet.** This skill turns a PRD into a data model; it doesn't
  write the PRD, and reading a brief instead just reintroduces rules the PRD deliberately
  dropped or superseded.
- **The change doesn't touch an invariant.** Adding a column with no new business rule
  behind it doesn't need this pipeline — write the migration.
- **You need the schema file itself, not the document it's generated from.** This skill is
  deliberately upstream of that: it writes semantic types and named predicates so a human
  and a coding agent can review a decision before it becomes irreversible SQL, not the SQL.

## Author

Built by **Khur Boon Kgim** at [boonkgim.com](https://boonkgim.com), where I write about
practical AI for builders: AI agents, coding workflows, and shipping software.

## License

MIT. See [LICENSE](LICENSE).
