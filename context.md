# Project Context — Odoo.sh Masterclass

This document gives a fresh agent everything needed to continue work on this project.

---

## What this is

A two-day masterclass at Odoo Experience 2026 (Sep 22-23, Brussels).

- **Day 1** (not our concern): on-premise Odoo architecture and performance.
- **Day 2** (this project): Odoo.sh — platform overview, dev workflow, testing, CI,
  performance debugging, and upgrades. Delivered by Stanislas (sts@odoo.com).

**Audience:** Developers and QA engineers, advanced level (Python, SQL, Linux, Odoo dev).
**Format:** 30 students max, each with their own Odoo.sh project. Instructor-led demos
alternating with hands-on exercises.

The full day plan is in `/tmp/plan.md`.

---

## The repository

**GitHub:** `github.com/sts-odoo/oxp26-smartclass-upgrade`
**Local path:** `/home/odoo/src/user/`

### Branch map

| Branch | Odoo | Commit | Purpose |
|---|---|---|---|
| `origin/main` | 17.0 | `e2689ef` | **Perf exercise base** — slow field + populate factories |
| `origin/17.0` | 17.0 | `88dd31a` | Original clean module |
| `origin/17.0-failed-test` | 17.0 | `0d530de` | Module + tests + **intentional bug** |
| `origin/19.0` | 19.0 | `b820bf3` | Upgraded module for upgrade exercise |
| `origin/prod` | — | `2153429` | Initial empty commit |

### Branch state (all pushed)

All branches are pushed to GitHub. The `main` branch contains the perf exercise code.

---

## The module: `conference.session`

### Current model fields (HEAD state)

| Field | Type | Notes |
|---|---|---|
| `name` | Char (required) | Session title |
| `speaker` | Char | Speaker name — becomes Many2one in v19 |
| `duration` | Integer | Duration in **minutes** — becomes Float (hours) in v19 |
| `room` | Char | Room name |
| `notes` | Text | Free notes |
| `date` | Date | Session date |
| `room_session_count` | Integer (computed) | **Deliberately slow** — uses `search_count` in a loop (N+1) |

### The slow field — for the performance exercise

`room_session_count` fires one `search_count` SQL query per record.
With 10,000 records in the DB (~2,000 per room), loading the list view triggers
80 queries (one per visible record). Clearly visible in the profiler.

**The fix** (to show after the exercise):
```python
def _compute_room_session_count(self):
    groups = self.env['conference.session'].read_group(
        domain=[('room', 'in', self.mapped('room'))],
        fields=['room'],
        groupby=['room'],
    )
    counts = {g['room']: g['conference_session_count'] for g in groups}
    for session in self:
        session.room_session_count = counts.get(session.room, 0)
```
One query instead of N.

### Populate factories

```bash
odoo-bin populate --size=medium --models=conference.session -d $PGDATABASE --stop-after-init --no-http
```

Sizes: small=50, medium=500, large=5000.
Note: `iterate` factory drove actual count to ~10,000 for medium — fine for the demo.

### Current DB state

The DB currently has **10,005 sessions** (4 demo + 10,001 populated), spread across
5 rooms (Hall A–E) with 20 rotating speakers. This DB is ready for export from outside
the container (the instructor will handle this via Odoo.sh tools). After export, reset
the DB with:

```bash
# Empty the DB
python3 -c "
import psycopg2
conn = psycopg2.connect('')
conn.autocommit = True
cr = conn.cursor()
cr.execute(\"SELECT tablename FROM pg_tables WHERE schemaname = 'public'\")
for (t,) in cr.fetchall():
    cr.execute(f'DROP TABLE IF EXISTS public.\"{t}\" CASCADE')
cr.execute(\"SELECT relname FROM pg_class c JOIN pg_namespace n ON c.relnamespace=n.oid WHERE c.relkind='S' AND n.nspname='public'\")
for (s,) in cr.fetchall():
    cr.execute(f'DROP SEQUENCE IF EXISTS public.\"{s}\" CASCADE')
print('Done')
"
# Reinstall clean
odoo-bin -i conference_session --stop-after-init --no-http
```

---

## The three exercises

### Exercise 1 — Testing (dev branch: `17.0-failed-test`)

The `duration_in_hours` computed field divides by `100.0` instead of `60.0`.
4 out of 5 tests fail. Students must:
1. Read the red build log in the Odoo.sh UI
2. Run tests manually: `odoo-bin -u conference_session --test-tags /conference_session --stop-after-init --no-http`
3. Read `AssertionError: 0.9 != 1.5`, find the divisor in the model
4. Fix it (`100.0` → `60.0`), re-run, confirm green, push

Teaching point: `test_duration_in_hours_zero` passes even with the bug (0/anything = 0).

### Exercise 2 — Performance (staging branch, with populated DB)

The `room_session_count` field loads slowly. Students must:
1. Activate profiler (`?profile=1`) on the Sessions list
2. Read the flamegraph: many thin repeated bars → N+1
3. Open `odoo-bin shell`, reproduce and measure:
   ```python
   import time, logging
   logging.getLogger('odoo.sql_db').setLevel(logging.DEBUG)
   t = time.time()
   env['conference.session'].search([], limit=80).mapped('room_session_count')
   print(f"{time.time()-t:.2f}s")
   ```
4. Apply the `read_group` fix, measure again

### Exercise 3 — Upgrade (staging branch: `17.0` → `19.0`)

Students trigger a test upgrade via the Odoo.sh UI. They:
1. Push 19.0 code **without** migration scripts → see data loss
2. Push **with** migration scripts → data preserved

The two migration issues:
- `speaker` (Char) → `presenter_id` (Many2one res.partner): needs **post_migration.py**
- `duration` (Integer, minutes) → `duration` (Float, hours): needs **pre_migration.py**

Migration scripts are in `conference_session/migrations/19.0.1.0.0/` (not committed —
show them to students during class).

---

## What still needs to be done

| Item | Status |
|---|---|
| Push perf exercise commits to `main` | ✅ Done (`e2689ef` force-pushed to `origin/main`) |
| Export populated DB from outside container | ⬜ TODO (instructor handles) |
| Reset DB to clean state after export | ⬜ TODO |
| Student Odoo.sh projects provisioned | ⬜ TODO |

---

## Git identity

```
user.name  = Stanislas
user.email = sts@odoo.com
```

## Key commands

```bash
odoo-bin -i conference_session --stop-after-init --no-http   # fresh install
odoo-bin -u conference_session --stop-after-init --no-http   # update
odoo-bin -u conference_session --test-tags /conference_session --stop-after-init --no-http  # tests
odoo-bin populate --size=medium --models=conference.session -d $PGDATABASE --stop-after-init --no-http
odoo-bin shell --no-http
odoosh-push   # never use git push directly
echo $ODOO_BACKEND_URL
```
