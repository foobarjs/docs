[← Back to docs](../README.md)

---
title: Database workflow
layout: default
nav_order: 30
---

# Database workflow (Explanation)

This is an **Explanation** doc — you're here to build a mental model, not
to look up a command. For the command reference, see
[migrations.md](./migrations.md). For a step-by-step build, see the
[tutorial](../tutorial.md).

The framework has two modes for keeping your database schema in sync
with your models. Understanding which mode you're in — and when the
switch happens — is the single most important thing to internalize
before you ship anything.

---

## The two modes

**Mode A: auto-sync (dev-loop mode).** On boot, the framework compares
your current models to the database and applies any additive changes
directly. Concretely, this calls MikroORM's
`schema.update({ dropTables: false })` — new tables and new columns
show up; nothing is ever dropped. This is what makes `foobar serve --dev`
feel magical: change a model, save, the watcher restarts, and the column
is there.

**Mode B: migrations-only.** Boot does nothing to the schema. You promote
schema changes by writing (or generating) files under
`database/migrations/`, then applying them with `foobar db:migrate`. This
is the mode you want the moment a schema change starts to matter to
anyone besides you.

## What triggers the switch

The framework picks the mode automatically:

- `database/migrations/` contains **zero** `.js` files → **Mode A**
  (auto-sync)
- `database/migrations/` contains **one or more** `.js` files → **Mode B**
  (migrations-only)

Source of truth: `framework/src/orm/Db.js` — `hasMigrations` counts
`.js` files, `autoSyncEnabled = !hasMigrations`. A `.gitkeep` or other
non-`.js` file does not count.

You can also override explicitly in `config/database.js`:

```js
export default {
  // ...
  autoSync: false,  // force Mode B even without migration files
  // autoSync: true // force Mode A even with migration files (rare)
}
```

**The one command that flips you into Mode B: `foobar db:make`.** It
writes a `.js` file into `database/migrations/`. From the next boot on,
auto-sync will not fire; you own the schema promotion story from that
point.

## The snapshot file

The framework maintains one artifact alongside your migrations:
`database/.foobar-schema.json`. This is a serialized copy of the schema
the framework thinks your models describe.

It's used by:

- `foobar db:sync` and `foobar db:make` — to compute the diff between
  "what the DB currently has" (snapshot) and "what your models now
  describe" (current). That diff becomes either the applied change
  (sync) or the generated migration body (make).
- `foobar db:migrate` — on successful apply, refreshes the snapshot from
  current models so the next diff starts from a clean base.

**Commit `database/.foobar-schema.json` to git.** Without it, `db:make`
run on a fresh checkout has no baseline to diff from, and you'll get a
migration proposing to create tables that already exist.

`foobar new` writes an initial snapshot for you.

## Safety guarantees

The framework refuses to shoot you in the foot without you explicitly
saying "yes shoot me."

**`db:sync` refuses on destructive changes** without `--force`
([framework/src/cli/commands/db.js:178-199](../../foobarjs/framework/src/cli/commands/db.js:178)).
Destructive means: dropping a table, dropping a column, changing a
column's type, or dropping an index. The refusal prints the specific
ops that would happen and points you at `foobar db:make` as the safe
route. `--force` skips this guard; use only when you know exactly
what's being dropped.

**`db:sync` refuses in production**, period, unless you pass
`--force` ([db.js:201-206](../../foobarjs/framework/src/cli/commands/db.js:201)).
The message tells you to use `foobar db:migrate` instead. Don't override
this — production schema changes should always go through a reviewable,
versioned migration file, not a diff-and-hope invocation.

**Auto-sync never drops.** `schema.update({ dropTables: false })` in
`Db.js:110` — even if you remove a table from your models, auto-sync
will leave the DB table alone. To actually drop, use `db:sync` with
`--force` (dev) or write an explicit migration (anywhere).

**`db:make` comments out destructive ops** in the generated migration
unless you pass `--allow-destructive`. The file is still generated; you
review it, uncomment intentionally, and apply. Fail-safe by default.

**`db:reset` and `db:fresh` do drop everything**. They exist for the
"I know exactly what I'm doing" case (e.g. a local dev DB you want
scorched). Neither prompts. Both refuse to run implicit magic — you
called them explicitly.

## Solo workflow

While you're the only person touching the schema, Mode A is fine.

```
# Iterate freely
edit app/models/link.model.js
# save; watcher restarts; schema.update({dropTables:false}) applies additively
```

`foobar db:sync` is only needed when auto-sync's additive-only behavior
isn't enough — you want to *drop* something during dev. `db:sync --force`
does the drop and updates the snapshot.

When you're ready to check in your work — or when the schema is about
to matter to anyone else (teammate, CI, staging, prod) — flip to
Mode B:

```
foobar db:make --diff --name initial_schema
# reviews the generated file, commits everything under database/
```

From here on, `foobar db:migrate` is how the schema advances, everywhere.

## Multi-team workflow

Once anyone besides you needs the schema to match, use Mode B
exclusively. The story is straightforward if everyone follows the same
protocol.

### What lives in git

- `database/migrations/*.js` — every migration file, in order.
- `database/.foobar-schema.json` — the snapshot the next `db:make` will
  diff against.
- **Not** `database/foobar.db` (or whatever your dev SQLite file is
  called). Add it to `.gitignore` if the scaffold didn't.

### The "I pulled a branch" recipe

Teammate merges a branch that adds a `role` column. You pull `main`.
Your local DB doesn't have the column yet.

```
git pull
foobar db:migrate           # applies every pending file, in order
```

That's the whole flow. `db:migrate` reads the `_foobar_migrations`
tracking table, computes pending (files on disk minus files already
applied), and applies them in filename order under one new batch
number.

If you had a scratch DB you don't care about, `foobar db:fresh` is
often faster: drop everything, replay all migrations, re-seed.

### Two teammates each add a column on separate branches

Alice adds `role` to `User`. Bob adds `avatar_url` to `User`. Both use
`foobar db:make --diff --name add_role` / `add_avatar_url`. Both
commit. Both open PRs.

If Alice's PR merges first:

- Bob rebases his branch on `main`. He now has Alice's migration file
  he didn't have before.
- Bob runs `foobar db:migrate` — Alice's migration applies to Bob's
  local DB, adding the `role` column.
- Bob's own migration is still pending; it also applies, adding
  `avatar_url`.
- Bob's snapshot file may have merge conflicts. Resolve by regenerating:
  delete `database/.foobar-schema.json`, then run `foobar db:make
  --diff --name refresh_snapshot` — the framework will produce a "no
  changes needed" outcome and rewrite the snapshot to match current
  models. Commit.

If both PRs somehow merge without either being rebased on the other,
the order of migrations on `main` is determined by filename (which the
framework prefixes with a timestamp). Whoever has the earlier timestamp
runs first; the other runs second. Both apply cleanly because they touch
different columns.

**When they touch the same column** — Alice adds `role: string`, Bob
adds `role: number` — the conflict is real. Neither timestamp order
matters; the second migration will fail at the DB level (SQLite doesn't
care about type, but Postgres/MySQL will reject the second `ALTER
COLUMN` for a column that's already there with a different type). Talk
to Alice, pick one, write a follow-up migration to fix.

### PR review checklist for migrations

- [ ] The migration is auto-generated but was **read** by the author.
  `db:make` emits comments; blindly-committed generated files are how
  bad schema changes get merged.
- [ ] If the change is destructive, `--allow-destructive` was used and
  the reviewer agrees the drop is intentional.
- [ ] The `down()` method is thought through. If the change is
  irreversible (data loss, dropping a NOT NULL column), the migration
  should say so in a comment. Don't fake a `down()` that pretends it
  reversed something it didn't.
- [ ] Data changes (`UPDATE`, `INSERT`, backfills) are in a separate
  migration file from structural ones. Rollback semantics get subtle
  when the two are mixed.
- [ ] `database/.foobar-schema.json` was regenerated (the diff should
  show it changing, otherwise the migration wasn't actually derived
  from a real model change).

### Rolling back on a shared branch

Only roll back migrations that haven't been merged and applied by
anyone else. Once a migration is on `main` and CI/staging has applied
it, don't rewrite history — write a **forward migration that undoes
it**:

```
foobar db:make --name revert_add_role
# hand-write the reverse in up(); leave down() empty
```

`foobar db:rollback` rolls back the most recent **batch** (all
migrations applied together under the last `db:migrate` invocation),
in reverse order. Use it locally to undo something you just applied.
Don't use it on `main` after other people have pulled.

## CI and deployment

**In CI**: `foobar db:migrate` against a fresh database. Every test run
should exercise the migration path — that's the only way you catch
"works on my machine" migrations. Never use `db:sync` in CI; it hides
migration bugs.

**In staging / production**: `foobar db:migrate` as a deploy step,
before booting the app. If a migration fails, boot should not proceed;
the deploy should abort. `db:sync` refuses to run with
`NODE_ENV=production` for this reason.

If a migration fails partway through (e.g. it created two columns then
choked on the third), the framework doesn't wrap the batch in a
transaction — SQLite/MySQL don't support transactional DDL uniformly,
and Postgres does but you'd have to opt in. Your migration file is
responsible for its own atomicity concerns. In practice: keep each
migration small and each `up()` idempotent-ish.

## Common surprises

**"I removed a field from my model but the column is still in the DB."**
That's Mode A being safe — auto-sync doesn't drop. In dev, either
`foobar db:sync --force` (fast) or `foobar db:make --allow-destructive
--name drop_stale_column` (proper). Either updates the snapshot.

**"My teammate's boot doesn't pick up my model change."** Check
`database/migrations/` on their branch. If there are files there, Mode
B is in effect and they need to run `foobar db:migrate` after pulling.
If your model change hasn't been captured as a migration file, they
have no way to sync — do `foobar db:make --diff`, commit, and reroute
the review through the same PR flow.

**"`db:make` says 'no changes' but I know I changed a model."** Most
common causes: the model file wasn't autodiscovered (check the
filename convention — `app/models/foo.model.js`), or the snapshot
already reflects your change (someone ran `db:sync` and committed the
snapshot without generating a migration — regenerate now to catch up).

**"The tracking table is polluting my DB dumps."** The
`_foobar_migrations` table lives in your app DB. Exclude it from
dumps by name if you're generating anonymized fixtures.

## See also

- [Migrations reference](./migrations.md) — every command, every flag
- [Configuration](../configuration.md#database) — `autoSync`,
  `autoIndexForeignKeys`, connection options
- [Tutorial](../tutorial.md) — Mode A in practice on a fresh scaffold
