# Recipes

Query patterns and workflows that come up repeatedly when driving Daemon from scripts.

SQL below is written for PostgreSQL. On SQL Server swap `LIMIT n` for `TOP n`, and on all engines
adjust the parameter placeholder to suit the driver (`%s` for psycopg2 and MySQL, `?` for pyodbc).

---

## Useful queries

**Recent tasks and their status**

```sql
SELECT task_uid, task_label, task_status, schedule_type, start_time
FROM mascot_daemon_tasks
ORDER BY task_uid DESC
LIMIT 20;
```

**Anything currently in flight**

```sql
SELECT task_uid, task_label, task_status, action_request
FROM mascot_daemon_tasks
WHERE task_status IN ('new', 'running', 'paused');
```

**Results for one task, joined to the file each came from**

```sql
SELECT r.result_number, r.accession, r.hit_score, r.seq_database,
       r.submitted, r.returned, f.file_name, r.result_url
FROM mascot_daemon_results r
JOIN mascot_daemon_files f
  ON  r.task_uid      = f.task_uid
  AND r.result_number = f.result_number
WHERE r.task_uid = %s;
```

**Find tasks by label**

```sql
SELECT task_uid, task_label, task_status
FROM mascot_daemon_tasks
WHERE task_label ILIKE %s;          -- '%keyword%'
```

**Why did this task pause?** Always the first check after a task stops unexpectedly. On current
builds the error is often only in the UTF-16 event log CSV, so read that first:

```bash
iconv -f UTF-16LE -t UTF-8 "/c/ProgramData/Matrix Science/Mascot Daemon/Mascot_Daemon_Event_Log.csv"   | grep ",<task_uid>," | tail -20
```

then the table, if it is still being written (`event_type` is stored capitalised: `Event`, `Warning`,
`Error`):

```sql
SELECT date_time, event_type, message
FROM mascot_daemon_events
WHERE task_number = %s
ORDER BY event_number DESC
LIMIT 30;
```

**Real progress of a task** — ignore placeholder result rows (`returned = 2001-01-01 01:01:01`):

```sql
SELECT f.file_name, f.peak_list IS NOT NULL AS peak_picked, f.mascot_task_id,
       r.returned, r.result_url
FROM mascot_daemon_files f
LEFT JOIN mascot_daemon_results r ON r.result_number = f.result_number
WHERE f.task_uid = %s
ORDER BY f.file_number;
```

**Full picture of one task**

```sql
SELECT t.task_uid, t.task_label, t.task_status,
       f.file_name, r.accession, r.hit_score
FROM mascot_daemon_tasks t
LEFT JOIN mascot_daemon_files   f ON t.task_uid = f.task_uid
LEFT JOIN mascot_daemon_results r ON t.task_uid = r.task_uid
WHERE t.task_uid = %s;
```

**Which parameter sets exist?** Worth running before every submission, since a `parameter_set` that
matches nothing makes the task complete silently.

```sql
SELECT parameter_set_label, db, cle, instrument, percolate, last_modified
FROM mascot_daemon_parameters
ORDER BY last_modified DESC;
```

---

## Re-search an existing MGF, skipping Distiller

When iterating on search parameters — toggling Percolator, changing instrument type, adjusting
tolerances — there is no reason to re-run Distiller peak picking over the raw file each time. Point
the task at the MGF Distiller already produced.

Set `import_filter='--- None ---'` and `filter_options=''`, and give
`mascot_daemon_files.file_name` the path to the existing MGF:

```python
cur.execute("""
    INSERT INTO mascot_daemon_tasks
    (task_uid, task_label, parameter_set, import_filter, filter_options,
     schedule_type, new_files_only, start_time, monitor_filename,
     follow_up, fu_score, fu_repeat, fu_number, fu_interval,
     fu_move, fu_discard, fu_task, task_status, sub_directories,
     user_name, computer_name, merge_data_files, search_priority,
     mascot_user_id, action_request)
    VALUES (%s,%s,%s,'--- None ---','',
            'now',1,%s,'*.*',
            0,0,0,1,'days', 1,0,'--- None ---','new',0,
            %s,%s,0,0, %s,'')
""", (task_uid, label, paramset, now, user, computer, mascot_user_id))

cur.execute("""
    INSERT INTO mascot_daemon_files (file_number, task_uid, file_name, next_action)
    VALUES (%s,%s,%s,%s)
""", (file_num, task_uid, mgf_path, now))
```

Daemon caches the MGF it generated for each task under

```
C:\ProgramData\Matrix Science\Mascot Daemon\MGF\<task_uid> <task_label>\
```

but rather than construct that path, read it from the row that produced it:

```sql
SELECT file_name, peak_list
FROM mascot_daemon_files
WHERE task_uid = %s AND peak_list IS NOT NULL;
```

On Tribrid-sized raw files this saves roughly ten minutes per iteration.

---

## Scheduling a batch: series or parallel

Mascot Server runs searches **in parallel, not in a queue** (up to `MaxConcurrentSearches` in
`mascot.dat`, default 10), and every Daemon task set to `Run` is sent to it straight away. That is
deliberate — see Matrix Science's
["Don't get stuck in a queue"](https://www.matrixscience.com/blog/dont-get-stuck-in-a-queue.html):
concurrent searches share the processors fairly, so a short search submitted beside a long one still
comes back quickly. With N searches running, each takes roughly N times as long as it would alone.

| Batch | Run it | Why |
|---|---|---|
| Many **similar-sized** searches (fractions, replicate runs) | **in series** | Total time is the same (files × search time), but results arrive one by one, so the first can be checked while the rest run. In parallel none finish until all finish together. |
| One large search plus several small ones | in parallel | The small ones are not stuck behind the large one — the case the parallel design is for |
| Low-urgency work on a shared server | in parallel at low `search_priority` | It gets processor time only when the server is otherwise idle |

Heavy searches (no-enzyme, top-down, large databases) also compete for memory: 26 no-enzyme
top-down searches submitted as 26 tasks ran ~10 at once, throughput fell from ~150 to ~63
spectra/min, and the workstation ran short of memory. **Ask the user before firing a large batch of
heavy searches at once.**

**How to run a batch in series:**

- **One task, many files, `merge_data_files=0`.** Daemon submits a task's files strictly one at a
  time and returns one result per file (observed: 12 files, each submitted only after the previous
  returned). Simplest option.
- **Chain tasks** with `schedule_type='after'` and `start_after='<uid> <label>'` of the previous
  task, when you want one task (and one clean run time) per file.

Do **not** pause tasks and release them later with `Run` — a paused task cannot be re-run.

## Distiller passes run in parallel across tasks

Submitting several `import_filter='Mascot Distiller'` tasks in quick succession runs them **in
parallel**, not serially — each task gets its own Distiller process.

Per-task wall time is unchanged whether you submit one task or several (around 10 minutes for a
100k-query Ascend raw file), so throughput scales close to linearly with task count up to the
machine's CPU count. Batch work should therefore be split across several tasks rather than piled into
one task with many files.

The tradeoff: one task with many files and `merge_data_files=1` produces a single merged search and a
single result row, which is what you want for fractionated samples. Separate tasks give parallel
peak picking, but their searches then also run in parallel on the server — see the scheduling table
above before splitting a batch of heavy searches.

---

## Per-spectrum variable modifications via MGF `IT_MODS`

An MGF `BEGIN IONS` block may carry its own `IT_MODS=` line: variable modifications for **that query
only**. Use it to offer an unusual modification to the few spectra that need it without inflating the
identity threshold for every other query (top-down, or a per-spectrum detector proposing mods).

Separators differ, silently: MGF `IT_MODS=` is **comma**-separated; `mascot_daemon_parameters.it_mods`
is **`@`**-separated.

**Prime the parameter set, or per-query `IT_MODS` is inert.** Per-query mods are *appended* to the
parameter set's `it_mods`:

1. With an empty parameter-set `it_mods`, searches complete and score normally while placing **zero**
   modifications on every query — nothing in the result or log says so.
2. A name in both lists fails the whole search with "Modification name collision or same
   modification %s is used in two groups".

So seed the parameter set's `it_mods` with one "primer" modification that no block names (e.g.
`Oxidation (M)` when no block uses it). Primed correctly, placement is exactly the block's own set.
Verify by reading back per query: `resfile.getInputQuery(q).getLocalVarModName()` — an empty string
means the mechanism was inert.

**Cost.** It works at scale but is far more expensive per unit of search work than the same mods
given globally (20–68× in one measurement), because cost tracks the number of *distinct* `IT_MODS`
strings. Sort mod names within each line (`Acetyl (K),Trimethyl (K)` and `Trimethyl (K),Acetyl (K)`
count as two) and keep the number of distinct combinations down. The sensitivity gain can be worth it
— on one file 8 global mods gave fewer IDs (746) than no variable mods (908), and the per-query
equivalent gave 1,091 — but compare against an error-tolerant search first.

---

## Reading Percolator-rescored results

After a task whose parameter set has `percolate='1'` and, say,
`ml_adapter_param="MS2Rescore.ms2pip_model=HCD2019"` completes, the Mascot Server writes
`.target.pop`, `.decoy.pop` and `.pip` files into

```
<MASCOT_HOME>/data/cache/YYYY/MM/<server_hash>/
```

Reading these from Python is **not** just a matter of setting `MSPEPSUM_PERCOLATOR`. The cache files
have to be staged into a temporary directory alongside the results file first. See the
[`mascot-parser` skill](https://github.com/Matrix-Science/mascot-parser-skill), recipe 20 — "Read
Percolator-rescored results: stage cache files in a tmp dir" — for the full pattern.

---

## Confirm a search is at the FDR you think it is

Run this before drawing any conclusion from PSM counts. A search submitted without FDR control, or one
where `setTargetFDR` was silently ignored, produces counts that look fine and mean something else.

```python
import os
import msparser

os.chdir(r"C:\inetpub\mascot\cgi")     # msparser resolves config paths relative to cwd
M = msparser.ms_mascotresults

resfile = msparser.ms_mascotresfilebase.createResfile(result_path)
df = msparser.ms_datfile(r"C:\inetpub\mascot\config\mascot.dat")
mo = df.getMascotOptions()

rp = msparser.ms_mascotresults_params()
resfile.get_ms_mascotresults_params(mo, rp)
rp.setTargetFDR(0.01)
ps = msparser.ms_peptidesummary(resfile, rp)

# returns [ok, achieved_FDR, sigLevel, n_target, n_decoy]
ident = ps.getThresholdForFDRAboveIdentity(0.01, M.DS_COUNT_PSM, M.DS_IDENTITY)
print(f"Identity: {ident[3]} target / {ident[4]} decoy at {100 * ident[1]:.2f}% FDR")
```

If `achieved_FDR` is far from the 0.01 you asked for, stop and investigate before trusting the numbers.

The `result_url` in `mascot_daemon_results` identifies the search; the corresponding `.dat` file path
on the server can be derived from it, or taken from `mascot_task_id`.

---

## Sequence databases for testing

Small, fast databases that keep the edit–submit–check loop short:

| Database | Contents |
|---|---|
| `contaminants` | Common laboratory contaminants |
| `cRAP` / `cRAP_Uniprot` | Contaminant Repository for Affinity Purification |
| `Biognosys_iRT` | iRT standard peptides |

List what is actually installed with `ls "C:\inetpub\mascot\sequence\"`, or read it from the Mascot
Server database status page — the name in the `db` column of a parameter set must match exactly.
