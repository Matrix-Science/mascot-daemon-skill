# Mascot Daemon Skill

**Version:** 1.1.1 &nbsp;|&nbsp; **License:** Apache-2.0

An Agent Skill that teaches AI coding assistants how to drive **Mascot Daemon** programmatically
through its task database (TaskDB) — creating, submitting, and monitoring proteomics search tasks
without the GUI.

The skill is just markdown — `skills/mascot-daemon/SKILL.md` plus reference files. It contains no
Matrix Science software and redistributes nothing.

> Mascot Daemon is a **Windows** application, so this skill targets a Windows environment.

## What it covers

- The task lifecycle end to end: create a parameter set, insert a task, attach data files, hand it to
  the Daemon engine, poll for completion, read results back
- The full TaskDB schema — all five `Mascot_Daemon_*` tables, every parameter-set column, and the
  positional `filter_options` string the Distiller import filter is configured with
- The failure modes that actually bite: tasks that complete silently having done nothing, the
  `next_action` placeholder timestamp, `task_status` vs `action_request` ownership, the `MAX(id)+1`
  race, UNC path requirements, instrument names that must match the server, MDS1194 from `''` where
  Daemon wants `NULL`, paused tasks that cannot be resumed, the UTF-16 event log CSV, and DIA raw
  files that need the `Mascot Distiller (DIA)` import filter
- Task control: what `Run`, `Pause` and `Cancel` do from each status, and stopping a search that is
  already running on Mascot Server; notes for the mascot-daemon MCP tools
- Recipes — re-searching an existing MGF to skip Distiller reprocessing, running a batch in series
  or in parallel, parallelising Distiller passes across tasks, per-spectrum MGF `IT_MODS`, reading Percolator-rescored results, and checking a finished search really is
  at the FDR you assumed

Daemon's engine keeps polling the same tables, so a task inserted by a script is picked up and run
exactly as though a user had built it in the GUI. The skill drives Daemon rather than bypassing it.

## Prerequisites

Beyond a working Mascot Daemon and Mascot Server, **two configuration changes are required** before
anything in this skill will work. Both are one-off administrator jobs, and
[`skills/mascot-daemon/references/SETUP.md`](skills/mascot-daemon/references/SETUP.md) walks through
both in full.

**1. The task database must be moved to a client/server SQL engine.**

Mascot Daemon 2.5 defaults to **VistaDB**, an embedded single-file database with no network listener —
nothing outside the Daemon process can connect to it. Daemon 2.4 and earlier defaulted to Microsoft
Access, which is no better. Move TaskDB to **PostgreSQL**, **Microsoft SQL Server**, **MySQL**, or
**Oracle** (Preferences → ODBC connection tab). Daemon's built-in XML backup/restore migrates existing
task history between engines without loss.

Sites already sharing one task database across several Daemon instances will have done this.

**2. Mascot Server security must be enabled.**

The `mascot_user_id` column and every `<mascot_user_*>` template tag are only populated when the
Mascot Server has security switched on. Daemon must also authenticate as a user in the **power users**
or **daemons** group to be permitted to submit searches, with credentials entered in Preferences →
Authentication → Mascot Security.

More on Mascot Daemon: **https://www.matrixscience.com/daemon.html**

## Install — Claude Code

Pick one location depending on whether you want the skill globally or per-project.

**Globally (all projects):**
```powershell
git clone https://github.com/Matrix-Science/mascot-daemon-skill.git
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills" | Out-Null
Copy-Item -Recurse mascot-daemon-skill\skills\mascot-daemon "$env:USERPROFILE\.claude\skills\"
```

**Per project** (run from your project's root):
```powershell
New-Item -ItemType Directory -Force .claude\skills | Out-Null
Copy-Item -Recurse "C:\path\to\mascot-daemon-skill\skills\mascot-daemon" .claude\skills\
```

If you'd rather not use the command line, clone or download the repo in File Explorer and copy the
`skills\mascot-daemon` folder into either:
- `%USERPROFILE%\.claude\skills\` (global), or
- `<your-project>\.claude\skills\` (per-project)

### After install

Restart Claude Code (or start a new session). The skill auto-loads when you mention Daemon tasks,
batch searching, task submission, peak list processing, or the Mascot Daemon database. You can also
force it with `/skill mascot-daemon`.

## Install — other AI coding tools

The skill format follows Anthropic's [Agent Skills](https://www.anthropic.com/news/skills) spec, which
several tools now support:

| Tool | Supported | Install location |
|---|---|---|
| **Claude Code** (CLI / desktop / IDE plugins) | yes | `%USERPROFILE%\.claude\skills\` or `.claude\skills\` (project) |
| **Claude API / Agent SDK** | yes | Pass via the Skills API or bundle with your agent |

For tools that don't yet support the skill format, the SKILL.md is just a long-form prompt:

1. **Paste it in** — open `skills/mascot-daemon/SKILL.md` and paste the contents into the chat or
   system prompt.
2. **Reference it as context** — most assistants let you @-mention or attach files. Point at
   `skills/mascot-daemon/SKILL.md` and the relevant `references/*.md`.

## Using the skill

Once installed, just ask in plain language. Examples:

- *"Submit these 40 raw files to Daemon as a batch using the PXD028735 parameter set."*
- *"Task 214 is paused — find out why."*
- *"Re-run that search with Percolator on, but reuse the MGF instead of re-running Distiller."*
- *"Show me every task from the last week that finished with no results."*
- *"Set up a parameter set for tryptic SwissProt searches at 10 ppm with a decoy database."*

## Repo layout

```
skills/mascot-daemon/
  SKILL.md                          # main skill entry point — prerequisites, connection,
                                    #   task lifecycle, submission code, gotchas
  references/
    SETUP.md                        # the two prerequisites in full: choosing and creating a
                                    #   SQL task database, ODBC drivers and connection settings,
                                    #   migrating existing history, Mascot Server security
    SCHEMA.md                       # all five TaskDB tables column by column, the positional
                                    #   filter_options string, template tags, legacy .par format
    RECIPES.md                      # query patterns, MGF reuse, parallel Distiller passes,
                                    #   Percolator results, FDR sanity check
```

## Credentials

The skill reads database and Mascot credentials from environment variables and never hardcodes them.
Keep them in a gitignored `.env` file or the platform credential store. The database login used by
scripts needs only `SELECT`, `INSERT` and `UPDATE` on the `mascot_daemon_*` tables — not the
create/drop rights Daemon itself needed when it first built them.

## Related

- [`Matrix-Science/mascot-parser-skill`](https://github.com/Matrix-Science/mascot-parser-skill) —
  reads Mascot `.dat` results files with `msparser`. The natural next step once a Daemon task has
  produced results.
- [`Matrix-Science/mascot-mdro-skill`](https://github.com/Matrix-Science/mascot-mdro-skill) — drives
  the Mascot Distiller SDK for raw data access and peak list generation, covering what Daemon's
  Distiller import filter does for you here.
- [`Matrix-Science/mascot-distiller-skill`](https://github.com/Matrix-Science/mascot-distiller-skill) —
  builds custom Mascot Distiller quantitative reports.

## License

The skill content in this repository (the markdown under `skills/` and the documentation at the repo
root) is licensed under the **Apache License, Version 2.0** (Apache-2.0). See [LICENSE](LICENSE) for
the full text.

Mascot Daemon, Mascot Server and Mascot Distiller are **not** covered by this license — they are
proprietary software licensed separately by Matrix Science. See
https://www.matrixscience.com/daemon.html for licensing.
