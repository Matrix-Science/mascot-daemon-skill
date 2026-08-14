---
name: mascot-daemon
description: Use when submitting, monitoring, or managing Mascot Daemon search tasks programmatically via its SQL task database (TaskDB). Triggers on mentions of Daemon tasks, batch searching, task submission, peak list processing, or the Mascot Daemon database.
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

## The five tables

| Table | One row per | You write to it? |
|---|---|---|
| `Mascot_Daemon_Tasks` | task | Yes — insert the task, then set `action_request` |
| `Mascot_Daemon_Files` | data file in a task | Yes — insert one row per file |
| `Mascot_Daemon_Parameters` | reusable search parameter set | Yes — create/edit parameter sets |
| `Mascot_Daemon_Results` | search result | No — read only; written by Daemon |
| `Mascot_Daemon_Events` | log entry | No — read only; written by Daemon |

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
     'paused'           → Daemon hit a problem; check the event log
     'completed'        → done
     'cancelled'        → aborted
7. Read mascot_daemon_results WHERE task_uid=?
```

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
    """Block until the task reaches a terminal state. Returns the final status."""
    import time
    cur = conn.cursor()
    while True:
        cur.execute("SELECT task_status FROM mascot_daemon_tasks WHERE task_uid=%s",
                    (task_uid,))
        status = cur.fetchone()[0]
        if status in ("completed", "cancelled"):
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
tasklist | findstr /i MascotDaemonEngine
```

The engine is typically a console/tray process, not a Windows service — and unrelated
Daemon-branded services (web UI / API components) can be Running while the engine itself is down,
so a services check alone proves nothing. Behavioral liveness test: the engine clears
`action_request` when it picks a task up; if the column still reads `Run` after a minute, the
engine is not consuming the queue. Start Mascot Daemon (the GUI starts the engine) or run
`MascotDaemonEngine.exe` directly.

### A silently completed task means a bad reference

By far the most common failure. The task goes straight from `new` to `completed` in seconds, with no
results and no error. Two causes, both name-matching problems:

- `parameter_set` does not match any `parameter_set_label` in `mascot_daemon_parameters`.
- `next_action` was left at its placeholder value, so Daemon skipped every file.

Check both before looking anywhere else.

### `next_action` must be set to now

When inserting into `mascot_daemon_files`, set `next_action` to `NOW()` / `datetime.now()`. Daemon
skips files whose `next_action` still holds the placeholder timestamp `2001-01-01 01:01:01`, and
marks the task completed without processing anything.

### `task_status` vs `action_request`

`task_status` is the engine's; `action_request` is yours. Set `action_request='Run'` to start a task —
the engine clears the field once it has acted. Setting it on a `paused` task resumes that task.

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
`MALDI-TOF-TOF`, `ETD-TRAP`.

### `computer_name` should be the uppercase hostname

Use `socket.gethostname().upper()`. A mismatch can cause the engine to ignore the task.

### Why a task pauses

A task moves to `paused` when the Mascot Server rejects the search (bad parameters, missing sequence
database), Distiller fails on the raw file, a pre/post command fails with `halt=1`, or a licence check
fails. Query `mascot_daemon_events` for the task first, then the Mascot Server error log.

## Recipes

**`references/RECIPES.md`** covers the common query patterns and some non-obvious workflows:

- Useful queries — recent tasks, results joined to files, finding tasks by label
- Re-searching an existing MGF to skip Distiller reprocessing (saves ~10 min per iteration on
  Tribrid-sized raw files)
- Running Distiller passes in parallel across tasks
- Reading Percolator-rescored results with `msparser`
- Confirming a finished search really is at the FDR you think it is

## Testing checklist

1. Confirm both prerequisites hold — SQL engine, and Mascot security enabled (`references/SETUP.md`).
2. Use `contaminants` or `cRAP` and one small raw file or a pre-made MGF.
3. Check the Daemon Engine process is actually running (`tasklist | findstr /i MascotDaemonEngine`) — see the first gotcha.
4. After inserting, confirm the task appears in the Daemon GUI — if it does not, the insert is wrong.
5. Watch `task_status`. Stuck on `new` usually means `action_request` was never set to `Run`.
6. If it goes to `paused`, read `mascot_daemon_events` before guessing.
7. Validate by opening the `result_url` from `mascot_daemon_results` in a browser.
