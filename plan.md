# Odoo.sh Masterclass — Day 2 Plan

> **Context:** Second day of a two-day masterclass (Odoo Experience 2026, Sep 22-23).
> Day 1 covered on-premise architecture and performance.
> Each student has their own Odoo.sh project pre-configured with branches from the repo.
> Repo: github.com/sts-odoo/oxp26-smartclass-upgrade

---

## Morning

### 09:00 – 09:15 · Intro & bridge from day 1 *(15 min, talk)*

Map everything from yesterday (workers, nginx, postgres, backups) to "who manages it on
Odoo.sh." Sets the framing for the day.

---

### 09:15 – 10:00 · Platform overview & branches *(45 min, demo)*

- The three branch types and what each is for
- What a build is: container + DB + code, rebuilt on every push
- Tour of the UI: builds, logs, settings

**🏋 Exercise — 10 min (embedded)**
> In your project: find the build log for your main branch. What version is running? How long
> did the last build take? Where are the daily backups listed?

Simple navigation, no risk — gets everyone oriented on their own project.

---

### 10:00 – 10:45 · The development workflow *(45 min, demo)*

- Dev branch lifecycle: push → rebuild → test
- Shell access & online editor — when to use which
- `requirements.txt` for Python dependencies (persistent across rebuilds)
- What survives a rebuild, what doesn't

**🏋 Exercise — 15 min (embedded)**
> On your dev branch: add a field to the Sessions form view, push, wait for the rebuild,
> verify it appears.

Teaches the core push → rebuild loop. First time students touch their own project.

---

### 10:45 – 11:00 · Break

---

### 11:00 – 11:45 · Tests & CI on Odoo.sh *(45 min, demo + exercise)*

- How tests run automatically on every push (show the red build badge)
- How to read the test output in the build log
- Running tests manually in the shell

**🏋 Exercise — 25 min**

The `conference_session` module on the `17.0-failed-test` branch has a deliberate bug:
`duration_in_hours` divides by 100 instead of 60. The build is red. Students must:

1. Read the failure in the build log
2. Run the tests themselves in the shell:
   ```bash
   odoo-bin -u conference_session --test-tags /conference_session --stop-after-init --no-http
   ```
3. Read the assertion error (`0.9 != 1.5`) and find the bug in the model
4. Fix it (`/ 100.0` → `/ 60.0`), run tests again, confirm green, push

**Debrief:** Why does `test_duration_in_hours_zero` pass even with the bug?
(Because 0 / anything = 0 — good moment to discuss test coverage.)

---

### 11:45 – 12:30 · Staging & the path to production *(45 min, demo)*

- Staging as a copy of production — why it matters
- The merge flow: dev → staging → production
- What Odoo.sh validates automatically vs what is your responsibility
- Backups: daily automatic, one-click restore to staging

---

### 12:30 – 14:00 · Lunch

---

## Afternoon

### 14:00 – 14:30 · Production operations *(30 min, demo)*

Connect explicitly to day 1:

| Day 1 topic | On Odoo.sh |
|---|---|
| Workers & longpolling | Configured in project settings |
| Nginx / SSL | Fully managed, custom domains supported |
| PostgreSQL tuning | Managed; you can read slow query logs |
| Filestore | Managed, included in backups |
| Backups | Daily automatic, downloadable, one-click restore |

- Reading logs: `odoo.log`, build log, error alerts
- What to do when production is down: the restore + staging workflow

---

### 14:30 – 15:30 · Performance debugging *(60 min)*

**Setup — before the exercise starts (students do this themselves, ~3 min):**

Students must populate their dev database first. The module ships with only 4 demo
records — not enough to see the slowness. They run this once in the shell:

```bash
odoo-bin populate --size=large --models=conference.session \
  -d $PGDATABASE --stop-after-init --no-http
```

This inserts **65,000 sessions** (~13,000 per room). Takes about 20 seconds.
The data persists in the dev DB until the next rebuild — students must not push
any commit during the exercise or the DB resets.

> **Why this works on dev:** `populate` only inserts records into the existing DB.
> It does not rebuild anything. The data is safe as long as no push triggers a rebuild.

Expected result after populate: loading the Sessions list takes **2–3 seconds**.
That is the starting point for the flamegraph exercise.

---

**Teach — 15 min: how to read a flamegraph**

Two patterns to recognise:
- Wide flat bar → one slow function
- Many thin repeated bars → N+1 (same query fired once per record)

Show one pre-recorded example before students open their own profiler.

Connect to day 1: "Yesterday we looked at slow queries in pg_activity. The flamegraph
shows you *which Python code* triggered those queries."

---

**🏋 Exercise 1 — 20 min: "Find it"**

> 1. Go to Conference → Sessions in your dev build
> 2. Enable debug mode (add `?debug=1` to the URL)
> 3. Add `?profile=1` to activate the profiler
> 4. Reload the page — it will be slow
> 5. Open the profiler viewer, read the flamegraph
>
> Deliverable: write down *what* is slow and *why*, before looking at the code.

What they will see: `_compute_room_session_count` repeated 80 times (one per visible
record), each with a thin `search_count` SQL bar underneath it.

---

**🏋 Exercise 2 — 20 min: "Confirm and fix it in the shell"**

Open `odoo-bin shell` and follow these steps:

```python
import time

# Step 1 — time one page (what the list view loads)
records = env['conference.session'].search([], limit=80)
t = time.time()
_ = records.mapped('room_session_count')
print(f"Slow: {time.time() - t:.2f}s")   # expect ~1-2s

# Step 2 — see every query firing
import logging
logging.getLogger('odoo.sql_db').setLevel(logging.DEBUG)
_ = records.mapped('room_session_count')  # watch the flood of identical SELECT count(*)

# Step 3 — the fix: one read_group instead of N search_counts
groups = env['conference.session'].read_group(
    domain=[('room', 'in', records.mapped('room'))],
    fields=['room'],
    groupby=['room'],
)
counts = {g['room']: g['conference_session_count'] for g in groups}
t = time.time()
_ = {s: counts.get(s.room, 0) for s in records}
print(f"Fast: {time.time() - t:.4f}s")   # expect <0.01s
```

**Debrief:** The fix is not about the number of records — it is about the number of
*round-trips to the database*. 80 queries × 14ms each = ~1.1s. One GROUP BY = 18ms.
Speedup: ~60×. This is the most common performance bug in Odoo custom code.

---

### 15:30 – 15:45 · Break

---

### 15:45 – 16:45 · Upgrade on Odoo.sh *(60 min)*

**Teach — 15 min:**
- The three actors: Upgrade Platform, Odoo.sh, your repo
- What "upgrade mode" is on a staging branch
- The upgrade.log (`~/logs/upgrade.log`) and how to read it
- pre_migration.py vs post_migration.py — why the split exists

**🏋 Exercise — 40 min: "Run the upgrade"**

Students already have a 19.0 branch. Walk through:
1. Trigger the test upgrade from the Odoo.sh UI
2. While waiting — read a pre-loaded upgrade.log example together
3. Push the 19.0 code **without** migration scripts → observe the damage in the DB
   (speaker names lost, duration values wrong)
4. Push **with** migration scripts → verify data preserved and correctly transformed

The two migration issues in the module:
- `speaker` (Char) → `presenter_id` (Many2one res.partner): handled by **post_migration.py**
- `duration` (Integer, minutes) → `duration` (Float, hours): handled by **pre_migration.py**

**Debrief — 5 min:**
- Why pre vs post matters (ORM has/hasn't run yet)
- What the platform does vs what you own

---

### 16:45 – 17:15 · Wrap-up & Q&A *(30 min)*

- What Odoo.sh still does not do for you (code quality, test coverage, downtime planning)
- Resources: docs, release notes, upgrade-util repo
- Open floor

---

## Branch map for the class

| Branch | Content | Used for |
|---|---|---|
| `main` | Module + slow `room_session_count` + populate factories | Performance exercise |
| `17.0-failed-test` | Module + tests + `/ 100.0` bug | Test & CI exercise |
| `19.0` | Upgraded module + migration scripts | Upgrade exercise |

---

## Pre-class checklist

| Item | Status |
|---|---|
| `conference_session` module with 4 demo sessions | ✅ Done |
| `duration_in_hours` computed field + 5 tests | ✅ Done |
| Bug committed (`/ 100.0` instead of `/ 60.0`) on `17.0-failed-test` | ✅ Done |
| `room_session_count` slow computed field | ✅ Done |
| `_populate_factories` with `large=65000` | ✅ Done |
| 19.0 branch with `pre_migration.py` + `post_migration.py` | ✅ Done |
| Push perf exercise commits to `main` branch | ✅ Done |
| Student Odoo.sh projects provisioned | ⬜ TODO |
