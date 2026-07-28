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

**Why did this task pause?** Always the first query after a task stops unexpectedly.

```sql
SELECT date_time, event_type, message
FROM mascot_daemon_events
WHERE task_number = %s
ORDER BY event_number DESC
LIMIT 30;
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

## Distiller passes run in parallel across tasks

Submitting several `import_filter='Mascot Distiller'` tasks in quick succession runs them **in
parallel**, not serially — each task gets its own Distiller process.

Per-task wall time is unchanged whether you submit one task or several (around 10 minutes for a
100k-query Ascend raw file), so throughput scales close to linearly with task count up to the
machine's CPU count. Batch work should therefore be split across several tasks rather than piled into
one task with many files.

The tradeoff: one task with many files and `merge_data_files=1` produces a single merged search and a
single result row, which is what you want for fractionated samples. Separate tasks give parallelism
but separate results.

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
