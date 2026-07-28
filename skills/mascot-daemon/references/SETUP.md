# Setup — SQL task database and Mascot Server security

Two one-off configuration jobs stand between a default Mascot Daemon install and anything in this
skill working. Both need administrator rights. Do them in this order.

---

## 1. Move the task database to a client/server SQL engine

### Why this is required

Mascot Daemon 2.5 ships with **VistaDB** as its default database engine. VistaDB is embedded — the
whole task database is one file (`taskdb.vdb5`) opened directly by the Daemon process, with no network
listener and no way for a second process to connect. Daemon 2.4 and earlier defaulted to Microsoft
Access, which has the same problem plus worse performance and a well-known 32-bit/64-bit ODBC driver
conflict with Microsoft Office.

Neither default lets an external script connect to TaskDB, so **neither works with this skill**. You
need one of the four client/server engines Daemon supports.

Sharing a single task database between Daemon instances on several PCs requires this anyway, so many
sites have already done it.

### Choosing an engine

| Engine | Notes |
|---|---|
| **PostgreSQL** | Free, cross-platform, straightforward install. Good default choice. Python access via `psycopg2`. |
| **Microsoft SQL Server** | Free Express edition is capped at 1 CPU / 1 GB RAM / 10 GB database — ample for Daemon. The natural pick where SQL Server is already the house standard. Python access via `pyodbc`. |
| **MySQL** | Free, well understood. Python access via `mysql-connector-python`. |
| **Oracle** | Supported, but installation and ODBC configuration are substantially more involved. Only worth it if Oracle is already mandated. |

The engine may run on the Daemon PC or on a separate server reached over the LAN.

### Create an empty database

Daemon creates its own tables, but **it cannot create the database**. An administrator must create an
empty one first, and Daemon's login must have rights to create and drop tables. Once the tables exist,
you can downgrade that login to something less privileged.

**PostgreSQL**

```sql
CREATE USER mduser WITH PASSWORD 'choose-a-strong-password';
CREATE DATABASE "TaskDB" OWNER mduser;
```

To reach it from another PC: open the server's port (5432 by default) in the firewall, and add a rule
to `pg_hba.conf` under the PostgreSQL data directory permitting the connection, e.g.
`host all all 10.0.0.0/8 md5`. Scope it to your subnet — `0.0.0.0/0` exposes the database to every
host that can route to it.

**Microsoft SQL Server**

Create the database with SQL Server Management Studio, or with Transact-SQL:

```sql
CREATE DATABASE TaskDB;
CREATE LOGIN mduser WITH PASSWORD = 'choose-a-strong-password';
USE TaskDB;
CREATE USER mduser FOR LOGIN mduser;
ALTER ROLE db_owner ADD MEMBER mduser;
```

**MySQL**

```sql
CREATE USER 'mduser'@'localhost' IDENTIFIED BY 'choose-a-strong-password';
CREATE DATABASE TaskDB;
GRANT ALL ON TaskDB.* TO 'mduser'@'localhost';
```

**Oracle**

The DBA creates an empty database; Daemon logs in first as a user able to create and drop tables. It
also creates public synonyms for its tables so less privileged users can reach them afterwards.

### Install the ODBC driver

Daemon connects through ODBC, so the matching driver must be installed on the **Daemon PC**:

| Engine | Driver | Pick |
|---|---|---|
| PostgreSQL | [psqlODBC](https://www.postgresql.org/ftp/odbc/versions/msi/) | the Unicode driver |
| SQL Server | [Microsoft ODBC Driver for SQL Server](https://learn.microsoft.com/sql/connect/odbc/download-odbc-driver-for-sql-server) | the Native Client / driver with the highest version number |
| MySQL | [MySQL Connector/ODBC](https://dev.mysql.com/downloads/connector/odbc/) | the Unicode driver, highest version |
| Oracle | Oracle Instant Client — Basic **and** ODBC packages | see the Daemon help; also needs `PATH`, `TNS_ADMIN`, and a `tnsnames.ora` |

> **32-bit and 64-bit ODBC are completely separate stacks.** A 64-bit application cannot use a 32-bit
> driver or vice versa. If Daemon's ODBC connection tab reports *No drivers found*, you almost
> certainly installed the wrong architecture.

### Configure Daemon's connection

In Daemon: **Edit → Preferences → ODBC connection tab**.

1. Set **Engine** to your chosen engine. The **Driver** list populates with the drivers found.
2. **Uid** / **Pwd** (**User** / **Password** on MySQL) — the login that can create and drop tables.
3. **Database** — the database name you created.
4. **Server** — `localhost`, or the hostname of the database machine. For SQL Server Express, append
   the instance name: `HOSTNAME\SQLEXPRESS`.
5. **Additional arguments** — normally leave blank.
6. Save. Daemon tries to connect, and creates its tables on first successful connection.

If you were previously connected to a different database, **Daemon exits after saving and must be
restarted**. That is expected. On connecting to a database from an older Daemon version you will be
warned that the tables are about to be upgraded.

### Migrate existing task history

Skip this if you do not care about old tasks. Otherwise, the engine-independent route is Daemon's own
XML backup, on the same ODBC connection tab:

1. **While still connected to the old database**, back up to XML. The file lands in
   `%ALLUSERSPROFILE%\Matrix Science\Mascot Daemon\<timestamp>_TaskDB_Export.xml`.
2. Switch the engine over as above and restart Daemon.
3. Restore from the XML file.

Restoring **completely overwrites** the target database, so restore into the new empty one before
doing any real work.

If you added custom columns to any table, the restore pauses with a prompt so you can recreate them
with an external tool before the table contents are written back. Have that ready in advance.

Alternatives, if the XML route is unavailable: the **VistaDB Data Migration Wizard**
(`DataMigrationWizard.exe`, in the Daemon application directory) copies an Access or SQL Server
database into VistaDB; and the Access→SQL Server, Access→PostgreSQL and Access→MySQL paths are covered
in the Daemon help. Note the current schema is documented in `TaskDB_schema.xls` in the Daemon
application directory.

### Verify

Connect with an ordinary client and confirm the tables are there:

```bash
psql -h localhost -p 5432 -U mduser -d TaskDB -c "\dt"
```

You should see the five `mascot_daemon_*` tables. If they are missing, Daemon has not connected
successfully yet — go back to the ODBC tab.

---

## 2. Enable Mascot Server security

### Why this is required

Mascot Server 2.1 and later has a role-based security model, and it is **off** on a default install.
With security off:

- `mascot_daemon_tasks.mascot_user_id` is meaningless.
- The `msm_user_name` and `msm_group_name` parameter-set columns do nothing.
- Every `<mascot_user_*>` template tag expands to nothing — those tags are only assigned values when
  the server has security enabled.
- The task **Owner** control in the Daemon GUI stays disabled, so tasks cannot be submitted on behalf
  of another user.

Since a submitted task carries a `mascot_user_id`, security needs to be on for that field to mean
anything.

### Turn it on

Security is enabled on the Mascot Server itself, from the Security Administration pages in the Mascot
configuration interface. This is a Mascot Server administration task rather than a Daemon one — see
the Mascot Server setup and installation manual for the full procedure.

### Give Daemon an account that may submit searches

Daemon must log in to the Mascot Server as a user permitted to submit searches from Daemon. In
practice that means membership of the **power users** or **daemons** group. Ask your Mascot
administrator for a username and password if you do not administer the server yourself.

To submit tasks *on behalf of other users* — the normal arrangement in a service lab, so each customer
sees their own results and no-one else's — the account also needs the privilege to submit searches
under other user names. That is what enables the Owner drop-down in the task editor and lets you set
an arbitrary `mascot_user_id`.

### Enter the credentials in Daemon

**Edit → Preferences → Authentication tab**, in the **Mascot Security** frame.

There are three separate credential frames on that tab; do not confuse them:

| Frame | Used for |
|---|---|
| **Web Server Authentication** | The web server in front of Mascot demands a username/password |
| **Proxy Server Authentication** | An authenticated proxy sits between the Daemon PC and the server |
| **Mascot Security** | Mascot's own security module, challenge by username and password |

If web server authentication is enabled *and* Mascot security is configured to use that same
authentication channel, the Mascot Security fields may be left empty.

While you are on the Preferences dialog, the **Intranet** tab must hold the Mascot Server URL up to
and including the CGI directory, e.g. `http://your_server/mascot/cgi/`.

### Find the user IDs

`mascot_user_id` is Mascot's internal numeric user ID, not the username. Read it back from tasks
already submitted through the GUI:

```sql
SELECT DISTINCT mascot_user_id, user_name
FROM mascot_daemon_tasks
ORDER BY mascot_user_id;
```

The Mascot Server security administration pages also list users and their IDs.

### Verify

Submit one small task through the Daemon GUI with a specific owner selected, then check it landed with
the right ID:

```sql
SELECT task_uid, task_label, user_name, mascot_user_id, task_status
FROM mascot_daemon_tasks
ORDER BY task_uid DESC LIMIT 5;
```

A populated, non-zero `mascot_user_id` confirms security is on and Daemon is authenticating. Now the
rest of the skill will work.

---

## Credentials

Everything above involves at least two sets of credentials — the database login and the Mascot
security account. Keep them out of source control: put them in a `.env` file that is gitignored, or
in the platform credential store, and read them from the environment.

```
MASCOT_TASKDB_HOST=localhost
MASCOT_TASKDB_PORT=5432
MASCOT_TASKDB_NAME=TaskDB
MASCOT_TASKDB_USER=mduser
MASCOT_TASKDB_PASS=...
```

The database login used by *scripts* does not need the create/drop table rights that Daemon's own
login needed at setup time. Give it only what it needs — `SELECT`, `INSERT` and `UPDATE` on the
`mascot_daemon_*` tables is enough for everything in this skill.
