# TaskDB schema reference

The five tables Mascot Daemon creates in its task database. Table and column names are identical
across all supported database engines; only the SQL types vary in detail.

The authoritative schema for your installed version is `TaskDB_schema.xls` in the Daemon application
directory. Older databases have fewer tables and columns, but existing names do not change between
versions.

> As a rule you should not edit these tables by hand. This reference exists so that automation can
> insert *new* tasks correctly — not so you can rewrite Daemon's own bookkeeping.

---

## Mascot_Daemon_Tasks

One row per task. This is the row you insert to create work.

| Column | Type | Notes |
|---|---|---|
| `task_UID` | integer PK | `SELECT COALESCE(MAX(task_uid),0)+1 FROM mascot_daemon_tasks` for the next ID. Not atomic — see the race condition note in SKILL.md. |
| `task_label` | varchar(255) | Human-readable label |
| `parameter_set` | varchar(255) | The **label name** from `mascot_daemon_parameters`. In legacy file-based mode this was a full path to a `.par` file. A name that matches nothing means the task completes silently having done nothing. |
| `import_filter` | varchar(255) | `"Mascot Distiller"` (DDA raw), `"Mascot Distiller (DIA)"` (DIA raw — required for DIA mode; a `.dia.opt` alone is not enough), or `"--- None ---"` (peak lists) |
| `filter_options` | varchar(255) | Comma-delimited Distiller options string — see below |
| `schedule_type` | varchar(255) | `"now"`, `"timed"`, `"monitor"`, `"follower"`, `"after"` |
| `new_files_only` | smallint | Boolean: in monitor mode, ignore files that already exist |
| `start_time` | timestamp | When to begin; use the current time for `"now"` tasks |
| `monitor_directory` | varchar(255) | Directory watched by monitor tasks |
| `monitor_filename` | varchar(255) | Wildcard pattern, typically `"*.*"` |
| `follow_up` | smallint | 0 = none, 1 = if score less than, 2 = if probability > 1 in |
| `fu_score` | integer | Threshold for follow-up |
| `fu_repeat` | smallint | Boolean |
| `fu_number` | integer | Number of repeat intervals to wait |
| `fu_interval` | varchar(255) | `"minutes"`, `"hours"`, `"days"`, `"weeks"` |
| `fu_move` | smallint | Boolean: move the data file to another task |
| `fu_discard` | smallint | Boolean: discard results below the threshold |
| `fu_task` | varchar(255) | Follower task UID + label, or `"--- None ---"` |
| `task_status` | varchar(255) | `"new"`, `"running"`, `"paused"`, `"completed"`, `"cancelled"` — **written by the Daemon engine only** |
| `sub_directories` | smallint | Boolean: monitor subdirectories too |
| `user_name` | varchar(255) | Task owner username |
| `computer_name` | varchar(255) | Task owner computer name, uppercase |
| `merge_data_files` | smallint | Boolean: merge all files into a single search |
| `search_priority` | smallint | −20 to 0 (nice value) |
| `mascot_user_id` | integer | Mascot security user ID — only meaningful with server security enabled |
| `action_request` | varchar(255) | **Your control channel.** `"Run"` (only from `new`), `"Pause"`, `"Cancel"` (from `paused`; ignored while running) |
| `action_response` | varchar(255) | Response from the engine, e.g. `Error 9000: Cannot run a task unless status is new` |
| `pre_task_command` | text | Shell command run before the task |
| `pre_task_wait` | smallint | Boolean: wait for the command to finish |
| `pre_task_halt` | smallint | Boolean: halt the task if the command errors |
| `pre_search_command` | text | Shell command before each search |
| `post_search_command` | text | Shell command after each search |
| `post_task_command` | text | Shell command after the task completes |
| `filter_fields` | text | XML for generic import filters |
| `export_fields` | text | Auto-export command line |
| `start_after` | varchar(255) | For `schedule_type='after'`: the predecessor's UID and label, space-separated (`"142 My earlier task"`) |

---

## Mascot_Daemon_Files

One row per data file in a task.

| Column | Type | Notes |
|---|---|---|
| `file_number` | integer PK | `SELECT COALESCE(MAX(file_number),0)+1 FROM mascot_daemon_files` |
| `task_UID` | integer FK | Parent task |
| `file_name` | varchar(255) | Full local or UNC path to the data file. Network files **must** be UNC, not a mapped drive letter |
| `next_action` | timestamp | **Set to `NOW()` when inserting.** Files still holding the placeholder `2001-01-01 01:01:01` are skipped, and the task completes without processing anything. Possibly `NOT NULL` with no default (unverified) — always set it |
| `file_comment` | varchar(255) | Search title with tags substituted |
| `file_size` | integer | File size when last checked |
| `size_checked` | timestamp | When the size was last checked |
| `intermediate` | text | MIME string for a follow-up search subset |
| `sample_number` | integer | For multi-sample files, e.g. Sciex `.wiff` |
| `mascot_task_id` | varchar(255) | Mascot Server search ID (set by Daemon) |
| `peak_list` | varchar(255) | Path to the cached MGF peak list (set by Daemon) |
| `result_number` | integer | FK into the results table (set by Daemon) |

---

## Mascot_Daemon_Results

One row per search result. Read-only — written by Daemon. A row is inserted with
`submitted`/`returned` = `2001-01-01 01:01:01`, empty `accession` and no `result_url` as soon as a
file is started; only a real `returned` timestamp means the search came back.

| Column | Type | Notes |
|---|---|---|
| `result_number` | integer PK | Assigned by Daemon |
| `task_UID` | integer FK | Parent task |
| `accession` | varchar(255) | Top hit accession |
| `hit_mass` | integer | Top hit protein mass |
| `hit_score` | integer | Top hit Mascot score |
| `title_text` | varchar(255) | Top hit description |
| `seq_database` | varchar(255) | Database(s) searched, comma-separated |
| `input_data_file` | varchar(255) | Source data file path |
| `result_url` | varchar(255) | URL of the result report |
| `submitted` | timestamp | When the search was submitted |
| `returned` | timestamp | When the result came back |
| `distiller_project` | varchar(255) | Path to the Distiller project file |
| `mascot_task_id` | varchar(255) | Mascot Server search ID |

> If the Mascot Server moves to new hardware with a new URL, every stored `result_url` breaks. This is
> the one documented case where editing Daemon's own rows is legitimate — a bulk `UPDATE` of the host
> portion of `result_url` repairs the links.

---

## Mascot_Daemon_Parameters

One row per reusable parameter set. **This is the preferred way to store search parameters** —
`mascot_daemon_tasks.parameter_set` references `parameter_set_label` by name.

| Column | Type | Notes |
|---|---|---|
| `parameter_set_label` | varchar(255) PK | Unique name, referenced by tasks |
| `com` | varchar(255) | Search title template; supports the tags listed below |
| `seg` | real | Segment size — use `NULL`, not `''`, if unused |
| `tol` | real | Peptide mass tolerance |
| `tolu` | varchar(255) | `"ppm"`, `"Da"`, `"%"` |
| `itol` | real | Fragment ion tolerance |
| `itolu` | varchar(255) | Fragment tolerance units |
| `pfa` | smallint | Missed cleavages |
| `db` | text | Sequence database name(s), comma-separated |
| `mods` | text | Fixed modifications (empty string if none) |
| `it_mods` | text | Variable modifications, `@`-separated: `"Oxidation (M)@Acetyl (Protein N-term)"` |
| `mass` | varchar(255) | `"Monoisotopic"` or `"Average"` |
| `cle` | varchar(255) | Enzyme name |
| `search` | varchar(255) | `"MIS"` (MS/MS), `"PMF"`, `"SQ"` |
| `username` | varchar(255) | Typically the template `<mascot_user_full_name>` |
| `useremail` | varchar(255) | Typically the template `<mascot_user_email>` |
| `charge` | varchar(255) | e.g. `"2+ and 3+"` |
| `report` | integer | Report top N hits. Use `NULL` for AUTO — `0` has been seen to trigger MDS1194 |
| `overview` | varchar(255) | Overview report setting |
| `format` | varchar(255) | `"Mascot generic"` |
| `precursor` | real | Precursor charge — use `NULL`, not `''`, if unused |
| `taxonomy` | varchar(255) | Taxonomy filter, e.g. `"All entries"`. Must be the exact `Title:` string from `<MASCOT_HOME>/config/taxonomy`, leading dots included (`. . . . . . Saccharomyces Cerevisiae (baker's yeast)`) |
| `reptype` | varchar(255) | Report type |
| `icat` | varchar(255) | ICAT setting |
| `instrument` | varchar(255) | Must match an entry in the server's `fragmentation_rules` |
| `quantitation` | varchar(255) | Quantitation method name |
| `crosslinking` | varchar(255) | Cross-linking setting |
| `pep_isotope_error` | smallint | Peptide isotope error setting |
| `decoy` | varchar(255) | `"1"` to enable a decoy search |
| `errortolerant` | varchar(255) | Error tolerant search setting |
| `msm_user_name` | varchar(255) | Mascot security user — needs server security enabled |
| `msm_group_name` | varchar(255) | Mascot security group — needs server security enabled |
| `mascot_server` | varchar(255) | Server URL; empty means the default |
| `last_modified` | timestamp | When the set was last changed |
| `target_fdr_percent` | varchar(255) | Target FDR percentage, e.g. `"1"` |
| `et_classifications` | text | Error tolerant classifications, **comma**-separated (`Post-translational,Co-translational,...`). Pipe-separated values match nothing: the ET pass runs with no modifications and no warning |
| `percolate` | varchar(255) | `"1"` to enable Percolator |
| `ml_adapter_param` | text | ML adapter parameters, e.g. `"MS2Rescore.ms2pip_model=HCD2019"` |

**`seg` and `precursor` are `real`.** Passing an empty string instead of `NULL` is a type error on
most engines.

**Leave unused columns `NULL`, never `''`.** Some text columns are read by the engine as Booleans;
`''` or `' '` can pause the task with MDS1194 (see SKILL.md for which are confirmed). Booleans take `'0'`, `'1'` or `NULL`.

**ML rescoring needs the decoy search.** Keep `decoy='1'` with `percolate='1'`; for TMT data the
MS2PIP model is `TMT`, not `HCD2019`.

### Instrument names

`instrument` must match an entry in `<MASCOT_HOME>/config/fragmentation_rules`, viewable from the
Mascot Server configuration pages. Common valid values: `ESI-TRAP`, `ESI-QUAD-TOF`, `MALDI-TOF-TOF`,
`ETD-TRAP`.

Some entries carry an `Option_PERCOLATE 0` flag, which the documentation implies disables Percolator
for that instrument. In practice the server treats it as advisory — a Daemon search combining such an
instrument with `percolate='1'` runs and produces a Percolator-rescored result.

### Template tags

Substituted at search time. They work in the `com` field and in `.par` files.

| Tag | Expands to |
|---|---|
| `<datafilename>` | Data file name |
| `<taskname>` | Task label |
| `<parameters>` | Parameter set name |
| `<localhost>` | Computer name |
| `<mascot_user_id>` | Mascot security user ID |
| `<mascot_user_name>` | Mascot security username, e.g. `jdoe` |
| `<mascot_user_full_name>` | Full name, e.g. `John Doe` |
| `<mascot_user_email>` | Email address |

The `<mascot_user_*>` tags are **only assigned values if the Mascot Server has security enabled** —
see `SETUP.md`.

---

## Mascot_Daemon_Events

Daemon's event log table. Read-only. On current builds it can stop receiving rows while tasks keep
running; the live log is then only `C:\ProgramData\Matrix Science\Mascot Daemon\Mascot_Daemon_Event_Log.csv`
(UTF-16LE; columns type, time, date, task UID, task label, data file, message).

| Column | Type | Notes |
|---|---|---|
| `event_number` | integer PK | Auto-increment |
| `user_name` | varchar(255) | Task owner |
| `event_type` | varchar(255) | Stored capitalised: `"Event"`, `"Warning"`, `"Error"` — compare case-insensitively |
| `date_time` | timestamp | When it happened |
| `task_number` | integer | Related task UID |
| `message` | varchar(255) | Event message text |

---

## The `filter_options` string

When `import_filter='Mascot Distiller'`, `filter_options` is a comma-delimited string of **exactly 16
positional fields** (15 commas). It is parsed by position, and a miscount shifts every later value
without raising an error.

```
{opt_path},{data_format},{merge_samples},{centroided},{range_start},{range_end},
{range_units},{ms_output},{msms_output},{peak_list_format},{save_project},
{quantitate},{intensity_type},{output_instrument},{instrument_name},{scan_type}
```

| Index | Field | Values |
|---|---|---|
| 0 | Processing options path | Full path to the `.opt` file |
| 1 | Data format enum | 4 = Thermo Xcalibur, 0 = Agilent, etc. |
| 2 | Merge multiple samples | `True` / `False` |
| 3 | Centroided | `True` / `False` |
| 4 | Range start | double, or empty |
| 5 | Range end | double, or empty |
| 6 | Range units | 0 = seconds, 1 = minutes, 2 = scan number |
| 7 | MS output format | 0 = m/z, 1 = MH+, 2 = Mr |
| 8 | MS/MS output format | 0 = m/z, 1 = MH+, 2 = Mr |
| 9 | Peak list format | 0 = MGF, 1 = comprehensive, 2 = mzData |
| 10 | Save Distiller project | 0 = no, 1 = yes |
| 11 | Quantitate hits | 0 = all, −1 = none, or an `N-M` range |
| 12 | Intensity type | `True` = area, `False` = S/N |
| 13 | Output instrument | 0 = no, 1 = yes (then 14 and 15 are used) |
| 14 | Instrument name | String, e.g. `"Default"` |
| 15 | Scan type | Enum |

The same string format is used with `Mascot Distiller (DIA)`.

**Default Thermo string:**

```
C:\ProgramData\Matrix Science\Mascot Distiller\processing options\default.ThermoXcalibur.opt,4,False,False,,,1,1,0,0,1,0,True,0,Default,0
```

Note fields 4 and 5 are empty, giving the `,,,` run in the middle. That is correct, not a typo.

List the available processing options with:

```powershell
ls "C:\ProgramData\Matrix Science\Mascot Distiller\processing options\"
```

When `import_filter='--- None ---'`, set `filter_options` to an empty string.

---

## Parameter files (`.par`) — legacy mode

Parameter files live in `C:\ProgramData\Matrix Science\Mascot Daemon\parameters\` and use a simple
`KEY=value` format mirroring the Mascot CGI search parameters. **Storing parameters in
`mascot_daemon_parameters` is preferred**; the files remain useful as a reference for the key names.

```
COM=File name <datafilename>, <taskname> (<parameters>), submitted from Daemon on <localhost>
TOL=100
TOLU=ppm
ITOL=100
ITOLU=ppm
PFA=1
DB=SwissProt
MODS=Carbamidomethyl (C)
MASS=Monoisotopic
CLE=Trypsin
SEARCH=MIS
CHARGE=2+ and 3+
REPORT=AUTO
FORMAT=Mascot generic
IT_MODS=Oxidation (M)
INSTRUMENT=ESI-TRAP
```
