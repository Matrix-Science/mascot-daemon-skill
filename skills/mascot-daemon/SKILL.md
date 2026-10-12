---
name: mascot-daemon
description: Use when submitting, monitoring, pausing, cancelling or troubleshooting Mascot Daemon search tasks programmatically — via its SQL task database (TaskDB) or the mascot-daemon MCP tools (create_parameter_set, create_task, set_task_action). Triggers on mentions of Daemon tasks or parameter sets, batch searching (series vs parallel), task submission, DIA tasks through the Distiller import filter, re-searching an MGF, a task stuck at new or paused (MDS error codes, Daemon event log), peak list processing, or the Mascot Daemon database.
license: Apache-2.0
---

# Mascot Daemon Task Database Interface

Programmatic interface to Mascot Daemon's task database (TaskDB) for creating, submitting, and
monitoring proteomics search tasks without using the Daemon GUI.

Daemon's own engine keeps polling the same tables, so a row you insert is picked up and run exactly
as though a user had built the task in the GUI. You are not bypassing Daemon — you are driving it.

## Before you start: two configuration prerequisites

This skill talks to TaskDB over a normal SQL client connection. That is only possible if the site has
been configured for it. Check both of these before writing any code — if either is wrong, everything
below fails in confusing, silent ways.

**1. The task database must be on a client/server SQL engine.**
Daemon 2.5's default engine is **VistaDB**, an embedded single-file database
(`taskdb.vdb5`) with no network listener — nothing outside the Daemon process can connect to it.
Earlier versions defaulted to Microsoft Access, which is no better. You must move TaskDB to one of the
supported client/server engines: **PostgreSQL**, **Microsoft SQL Server**, **MySQL**, or **Oracle**.

**2. Mascot Server security must be enabled.**
The `mascot_user_id` column on a task, and every `<mascot_user_*>` template tag, are only populated
when the Mascot Server has security switched on. Daemon must also authenticate to the server as a
user in the **power users** or **daemons** group to be allowed to submit searches at all.

Both are one-off admin jobs. **`references/SETUP.md`** walks through them — engine choice, creating the
empty database, ODBC driver and connection settings, migrating existing task history without losing it,
and the Mascot security account setup.

## Safety rules

- **NEVER** modify or delete tasks that are `running` or `paused` — the Daemon engine owns those rows.
- **NEVER** write to `task_status`. It is owned by the Daemon engine. You control tasks via
  `action_request` instead.
- **NEVER** try to repair a failed task by editing its rows (resetting `mascot_task_id`,
  `next_action`, result rows). The engine detects the inconsistency and cancels the task. Create a
  fresh task instead.
- **ALWAYS** back up before bulk operations. Daemon has a built-in backup (Preferences → ODBC
  connection → backup to XML) which is engine-independent and the safest option. Engine-native dumps
  work too, e.g. `pg_dump -h HOST -p PORT -U USER TaskDB > backup.sql`.
- Test with a small sequence database (`contaminants`, `cRAP`) and a single small peak list. A
  mistake costs seconds instead of hours.

## Connecting

Read credentials from the environment rather than hardcoding them. The connection details must match
what Daemon itself uses on its ODBC connection tab.

```python
import os
import psycopg2                      # PostgreSQL
from dotenv import load_dotenv

load_dotenv()                        # MASCOT_TASKDB_* in a local .env, never committed

conn = psycopg2.connect(
    host     = os.environ.get("MASCOT_TASKDB_HOST", "localhost"),
    port     = os.environ.get("MASCOT_TASKDB_PORT", "5432"),
    database = os.environ["MASCOT_TASKDB_NAME"],
    user     = os.environ["MASCOT_TASKDB_USER"],
    password = os.environ["MASCOT_TASKDB_PASS"],
)
```

For SQL Server use `pyodbc` with the Native Client driver; for MySQL use `mysql-connector-python`. The
table and column names are identical across engines — only the driver and the parameter placeholder
style change (`%s` for psycopg2/MySQL, `?` for pyodbc).

> Check the port. A PostgreSQL instance installed alongside another Postgres — including the one that
> ships with Mascot Server — is often bumped to 5433 to avoid a clash with the default 5432. Confirm
> against Daemon's ODBC connection tab rather than assuming.

## Using the mascot-daemon MCP server

If the `mascot-daemon` MCP tools are available, use them for routine work instead of raw SQL — they
set `next_action`, file numbering and ownership columns for you. Points that have tripped callers:

- **`create_parameter_set` / `update_parameter_set` need `label`** on every call; calls without it
  fail validation. Clone rather than build from scratch:
  `create_parameter_set(label="new", base_parameter_set="known-good", overrides={"db": "...", "tol": 10})`.
  `None` in `overrides` writes NULL — which is what blank columns need (see the MDS1194 gotcha).
- **`create_task`** inserts with `action_request=''` unless `run_immediately=True`. For DIA raw files
  pass `import_filter="Mascot Distiller (DIA)"` (older builds of the tool always wrote plain
  `Mascot Distiller`; check the stored row).
- **`set_task_action`'s docstring says `Run` resumes a paused task. It does not** — see
  "Task control" below.
- After an MCP `delete_task`, one build left an `idle in transaction` session holding row locks, so
  the engine reported "failed to update Mascot Daemon table". Restarting the MCP server released it.

## The five tables

| Table | One row per | You write to it? |
|---|---|---|
| `Mascot_Daemon_Tasks` | task | Yes — insert the task, then set `action_request` |
| `Mascot_Daemon_Files` | data file in a task | Yes — insert one row per file |
| `Mascot_Daemon_Parameters` | reusable search parameter set | Yes — create/edit parameter sets |
| `Mascot_Daemon_Results` | search result | No — read only; written by Daemon |
| `Mascot_Daemon_Events` | log entry | No — read only; may be stale (see "Where the errors are") |

Full column-by-column reference, including every parameter-set field and the positional
`filter_options` string, is in **`references/SCHEMA.md`**. Read it before building an insert — several
columns have non-obvious constraints, and a wrong value usually produces a silently completed task
rather than an error.

## Task lifecycle

```
1. Ensure a parameter set exists in mascot_daemon_parameters (or create one).
2. Get the next task_UID:   SELECT COALESCE(MAX(task_uid),0)+1 FROM mascot_daemon_tasks
3. INSERT the task with task_status='new' and parameter_set = the parameter set's label.
4. INSERT one mascot_daemon_files row per data file, with next_action = NOW().
5. UPDATE mascot_daemon_tasks SET action_request='Run' WHERE task_uid=?
6. Poll task_status:
     'new' | 'running'  → still going; wait and re-poll
     'paused'           → Daemon hit a problem; read the event log CSV. It will NOT resume —
                          fix the cause, Cancel it, and submit a new task
     'completed'        → done
     'cancelled'        → aborted
7. Read mascot_daemon_results WHERE task_uid=? AND returned > '2002-01-01'
```

Daemon inserts a result row with placeholder `submitted`/`returned` = `2001-01-01 01:01:01` as soon
as it starts a file, so counting result rows overstates progress. Track per-file progress with
`mascot_daemon_files.peak_list` (set when Distiller finishes a raw file), `mascot_task_id` (set at
submission) and a real `returned` timestamp plus `result_url`.

## Submitting a task

```python
import socket
from datetime import datetime

THERMO_DEFAULT = (
    r"C:\ProgramData\Matrix Science\Mascot Distiller\processing options"
    r"\default.ThermoXcalibur.opt,4,False,False,,,1,1,0,0,1,0,True,0,Default,0"
)

def submit_daemon_task(conn, task_label, parameter_set_label, data_files,
                       user_name, mascot_user_id,
                       import_filter="Mascot Distiller",
                       filter_options=THERMO_DEFAULT):
    """Insert a Daemon task plus its data files. Returns the new task_uid.

    parameter_set_label must match an existing
    mascot_daemon_parameters.parameter_set_label, or Daemon will silently
    complete the task without searching anything.
    import_filter: "Mascot Distiller" (DDA raw), "Mascot Distiller (DIA)"
    (DIA raw) or "--- None ---" (peak lists; filter_options='').
    """
    cur = conn.cursor()
    now = datetime.now()

    cur.execute("SELECT COALESCE(MAX(task_uid), 0) + 1 FROM mascot_daemon_tasks")
    task_uid = cur.fetchone()[0]

    cur.execute("""
        INSERT INTO mascot_daemon_tasks
        (task_uid, task_label, parameter_set, import_filter, filter_options,
         schedule_type, new_files_only, start_time, monitor_filename,
         follow_up, fu_score, fu_repeat, fu_number, fu_interval,
         fu_move, fu_discard, fu_task, task_status, sub_directories,
         user_name, computer_name, merge_data_files, search_priority,
         mascot_user_id, action_request)
        VALUES (%s,%s,%s,%s,%s, 'now',1,%s,'*.*',
                0,0,0,1,'days', 1,0,'--- None ---','new',0,
                %s,%s,0,0, %s,'')
    """, (task_uid, task_label, parameter_set_label, import_filter,
          filter_options, now, user_name,
          socket.gethostname().upper(), mascot_user_id))

    for file_path in data_files:
        cur.execute("SELECT COALESCE(MAX(file_number), 0) + 1 FROM mascot_daemon_files")
        cur.execute("""
            INSERT INTO mascot_daemon_files
            (file_number, task_uid, file_name, next_action)
            VALUES (%s, %s, %s, %s)
        """, (cur.fetchone()[0], task_uid, file_path, now))

    conn.commit()
    return task_uid


def start_task(conn, task_uid):
    """Hand the task to the Daemon engine."""
    cur = conn.cursor()
    cur.execute("UPDATE mascot_daemon_tasks SET action_request='Run' WHERE task_uid=%s",
                (task_uid,))
    conn.commit()


def monitor_task(conn, task_uid, poll_interval=30):
    """Block until the task completes, is cancelled or pauses. Returns the status.

    'paused' is effectively terminal: a paused task cannot be re-run.
    """
    import time
    cur = conn.cursor()
    while True:
        cur.execute("SELECT task_status FROM mascot_daemon_tasks WHERE task_uid=%s",
                    (task_uid,))
        status = cur.fetchone()[0]
        if status in ("completed", "cancelled", "paused"):
            return status
        conn.commit()          # end the read transaction so the next poll sees fresh data
        time.sleep(poll_interval)
```

> The `conn.commit()` inside the poll loop matters on PostgreSQL. The default isolation level opens a
> transaction on first read and holds a consistent snapshot, so without it you re-read your own stale
> snapshot forever and the task appears to hang on `new`.

## Gotchas

These are the failure modes that actually bite, in rough order of how often they do.

### The Daemon Engine is not running: tasks sit at `new` forever

Tasks are executed by the **Mascot Daemon Engine** (`MascotDaemonEngine.exe`), a separate process
from the GUI (`MascotDaemon.exe`). If the engine is down, an inserted task stays at
`task_status='new'` with `action_request='Run'` indefinitely — no error, no event, nothing. Check
before submitting work:

```
tasklist | findstr /i MascotDaemonEngine        (cmd)
Get-Process MascotDaemonEngine                  (PowerShell)
tasklist | grep -i MascotDaemonEngine           (Git Bash)
```

The engine is typically a console/tray process, not a Windows service — and unrelated
Daemon-branded services (web UI / API components) can be Running while the engine itself is down,
so a services check alone proves nothing. Nor does an old last entry in `mascot_daemon_events`:
that table can stop being written while the engine runs normally. Behavioral liveness test: the
engine clears `action_request` when it picks a task up; if the column still reads `Run` after a
minute, the engine is not consuming the queue. Start Mascot Daemon (the GUI starts the engine) or
run `MascotDaemonEngine.exe` directly — or ask the user to, if it lives in their desktop session.

### A silently completed task means a bad reference

By far the most common failure. The task goes straight from `new` to `completed` in seconds, with no
results and no error. Two causes, both name-matching problems:

- `parameter_set` does not match any `parameter_set_label` in `mascot_daemon_parameters`.
- `next_action` was left at its placeholder value, so Daemon skipped every file.

Check both before looking anywhere else.

### `next_action` must be set to now

When inserting into `mascot_daemon_files`, set `next_action` to `NOW()` / `datetime.now()`. Daemon
skips files whose `next_action` still holds the placeholder timestamp `2001-01-01 01:01:01`, and
marks the task completed without processing anything. The column is `NOT NULL` with no default on
some installs; a row inserted without it can leave the task sitting at `new` with no error.

### Task control: `task_status` vs `action_request`

`task_status` is the engine's; `action_request` is yours (`Run`, `Pause`, `Cancel`). The engine
clears the field once it has acted, and may write a reason to `action_response`.

| From status | `Run` | `Pause` | `Cancel` |
|---|---|---|---|
| `new` | starts the task | holds it (now `paused`, so see below) | — |
| `running` | — | pauses after the current step | **refused** — Pause first |
| `paused` | **refused**: `action_response` = "Error 9000: Cannot run a task unless status is new" (or nothing happens) | — | cancels |

- **A paused task cannot be resumed** through TaskDB (verified for error-paused tasks on several
  hosts; don't plan on resuming one you paused yourself either). Fix the cause, `Cancel` it, and submit a new task with the same files and
  parameter set. Pausing a task whose search is already on the server throws that search's work away.
- **To stop a running search:** set `Pause`, wait for `task_status='paused'`, then `Cancel`. That
  stops Daemon only — the search keeps running on Mascot Server. Cancel it there too (database status
  page, or the Mascot Server MCP `cancel_search` with the F-number). The Daemon `mascot_task_id` is
  not an F-number; the newest `F*.msr` in `<MASCOT_HOME>/data/<YYYYMMDD>/` is. Confirm the search
  reads `cancelled` and its result file has stopped growing.

### Blank parameter-set columns must be NULL, never `''` (MDS1194)

The engine reads many parameter-set columns as .NET Booleans. An empty string (or a single space)
throws `(MDS1194) Conversion from string "" to type 'Boolean' is not valid` and pauses the task a few
seconds after "Connected to Mascot server", before any search is submitted. NULL takes a different
code path and works. Columns seen to trigger it: `errortolerant`, `percolate`, `ml_adapter_param`,
`quantitation`, `decoy`, `icat`, `crosslinking`, `overview`, `reptype`, `msm_user_name`,
`msm_group_name`, `mascot_server`, `et_classifications`. Booleans take `'0'`, `'1'` or NULL. Also
reported: `report=0` (use NULL for AUTO) and `decoy='0'` combined with `target_fdr_percent='1'`.
Safest: clone a parameter set that is known to run (`INSERT ... SELECT` or the MCP
`base_parameter_set`) and override only what differs; compare column by column against a working
set if a task pauses.

### Where the errors are: the event log CSV

Task error codes (MDSnnnn, Mascot Mnnnnn) may appear **only** in
`C:\ProgramData\Matrix Science\Mascot Daemon\Mascot_Daemon_Event_Log.csv`, not in
`mascot_daemon_events` — on current builds that table can stop receiving rows while tasks keep
running. The CSV is **UTF-16LE**: read it with `iconv -f UTF-16LE -t UTF-8` or `tr -d '\000'`, or
`Get-Content -Tail 30` in PowerShell. Columns: type, time, date, task UID, task label, data file,
message. If the table has rows for your task, use it; if not, the CSV is authoritative.

### DIA raw files need the `Mascot Distiller (DIA)` import filter

`import_filter='Mascot Distiller (DIA)'` is what makes the engine tell Distiller the data is DIA
(`-rstExperimentType`). A `.dia.opt` processing-options file under the plain `Mascot Distiller`
filter is **not** enough: the raw file is peak-picked as DDA and the search completes with no
warning — one Thermo DIA file gave 76,232 queries this way against 333,884 as DIA. Verify the first
result: `DATA_INDEPENDENT_ACQUISITION=1` in its search parameters. The `filter_options` string is
the same for both filters. (On one host the DIA filter failed within seconds with MDS1067 on every
attempt; running Distiller from the command line was the workaround.)

### Peak lists over 2 GB need command-line submission

Daemon submits over HTTP by default, and IIS caps uploads at 2 GB. A larger peak list (long DIA
runs) logs "Search submitted by HTTP" and gets a `mascot_task_id`, but the server never runs it; the
task sits `running`, later logging "Search has already been submitted". Enable Daemon's
submit-via-command-line option (only possible with Daemon on the Mascot Server host) and resubmit as
a new task.

### `MAX(id)+1` is not atomic

`SELECT MAX(task_uid)+1` races if two clients insert at once, producing a duplicate key error. The
Daemon GUI is single-threaded so this never shows up there, but automation must handle it — take the
max and insert inside one transaction with a retry loop, or `SELECT ... FOR UPDATE`.

### Remote files need UNC paths

Data files on network shares must use UNC paths (`\\SERVER\share\file.raw`), not mapped drive letters.
The Daemon engine may run under a different account with different drive mappings — and always does
when configured as a Windows service.

### `filter_options` is positional and silently forgiving

The comma-delimited string is parsed by position and must have exactly 16 fields (15 commas). A
missing or extra comma shifts every later value without raising an error, so the search runs with the
wrong settings. Count the commas before submitting. Layout is in `references/SCHEMA.md`.

### Instrument names must exist on the server

`instrument` must match an entry in the Mascot Server's `fragmentation_rules` file
(`<MASCOT_HOME>/config/fragmentation_rules`). Common valid values: `ESI-TRAP`, `ESI-QUAD-TOF`,
`MALDI-TOF-TOF`, `ETD-TRAP`. A name that is not there fails with "Instrument '%s' cannot be found in
fragmentation_rules file".

### `computer_name` should be the uppercase hostname

Use `socket.gethostname().upper()`. A mismatch can cause the engine to ignore the task.

### Why a task pauses

A task moves to `paused` when the Mascot Server rejects the search (bad parameters, missing sequence
database), Distiller fails on the raw file, a pre/post command fails with `halt=1`, or a licence check
fails — or, very often, MDS1194 from a blank column (above). Read the event log CSV for the task
first (the events table may be stale), then the Mascot Server error log. Treat the message as a lead,
not a verdict: "Decoy search not allowed with spectral library" once turned out to mean the
`quantitation` method had not actually been set. Then cancel and resubmit — the paused task will not
run again.

### Modification names follow the Unimod spec_group, sites sorted alphabetically

Mascot names a variable modification after its Unimod spec_group, joining all `Anywhere` sites in
that group **alphabetically**: there is no `Phospho (S)`, only `Phospho (ST)`. An invented name fails
the whole submission with **M00135** "Sorry, the modification '%s' is not available". `unimod.xml`
does not list sites in alphabetical order (it has `TS`, `QN`), so sort them yourself; and
`hidden="true"` does not mean unavailable (`Methyl (K)` is hidden and valid). Better: take names from
the server's own list (Mascot Server MCP `list_modifications`, or the server's modification page).

## Recipes

**`references/RECIPES.md`** covers the common query patterns and some non-obvious workflows:

- Useful queries — recent tasks, results joined to files, finding tasks by label
- Re-searching an existing MGF to skip Distiller reprocessing (saves ~10 min per iteration on
  Tribrid-sized raw files)
- Scheduling a batch in series or in parallel, and running Distiller passes in parallel
- Per-spectrum variable modifications via MGF `IT_MODS` (the parameter set must be primed)
- Reading Percolator-rescored results with `msparser`
- Confirming a finished search really is at the FDR you think it is

## Testing checklist

1. Confirm both prerequisites hold — SQL engine, and Mascot security enabled (`references/SETUP.md`).
2. Use `contaminants` or `cRAP` and one small raw file or a pre-made MGF.
3. Check the Daemon Engine process is actually running (`Get-Process MascotDaemonEngine`) — see the first gotcha.
4. After inserting, confirm the task appears in the Daemon GUI — if it does not, the insert is wrong.
5. Watch `task_status`. Stuck on `new` usually means `action_request` was never set to `Run`.
6. If it goes to `paused`, read the event log CSV (and `mascot_daemon_events`) before guessing; then
   cancel and resubmit.
7. Validate by opening the `result_url` from `mascot_daemon_results` in a browser — and for DIA,
   check `DATA_INDEPENDENT_ACQUISITION=1` and a plausible query count.
