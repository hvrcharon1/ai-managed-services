# Integrating Codex with a Locally Running Oracle Database 23ai on Windows

This guide explains how to connect Codex-assisted workflows to an Oracle Database 23ai instance that is running locally on a Windows machine.

> Scope: local development only (Windows host + local Oracle 23ai listener).

## 1) Prerequisites

- Oracle Database 23ai installed and running on Windows.
- Oracle listener enabled (default port is typically `1521`).
- A database user with the required schema privileges.
- Codex running in an environment that can reach your Windows host and Oracle listener.
- One of the following client options:
  - Python (`oracledb`), or
  - Node.js (`oracledb`), or
  - SQL*Plus / SQLcl for quick connection checks.

## 2) Confirm Oracle service and listener on Windows

Open **PowerShell as Administrator** and verify Oracle services:

```powershell
Get-Service | Where-Object { $_.Name -like "Oracle*" }
```

Validate listener status:

```powershell
lsnrctl status
```

You should see an endpoint similar to:

- `HOST = localhost`
- `PORT = 1521`
- `SERVICE_NAME = FREEPDB1` (example)

If listener is not running:

```powershell
lsnrctl start
```

## 3) Identify a working connection string

For Oracle 23ai Free defaults, a typical connect descriptor is:

```text
localhost:1521/FREEPDB1
```

If Codex runs in WSL or a container and cannot use `localhost` directly, use the Windows host IP instead:

```powershell
ipconfig
```

Then substitute, for example:

```text
192.168.1.25:1521/FREEPDB1
```

## 4) Create and validate a dedicated application user (recommended)

Connect as SYSDBA and create a least-privileged user for Codex-assisted tasks:

```sql
CREATE USER codex_app IDENTIFIED BY "StrongPasswordHere";
GRANT CREATE SESSION TO codex_app;
GRANT CONNECT, RESOURCE TO codex_app;
ALTER USER codex_app QUOTA UNLIMITED ON USERS;
```

Then test login:

```sql
CONNECT codex_app/StrongPasswordHere@localhost:1521/FREEPDB1
SELECT banner_full FROM v$version;
```

## 5) Configure Codex-side environment variables

Store credentials in environment variables instead of hardcoding.

### PowerShell (session scope)

```powershell
$env:DB_HOST = "localhost"
$env:DB_PORT = "1521"
$env:DB_SERVICE = "FREEPDB1"
$env:DB_USER = "codex_app"
$env:DB_PASSWORD = "StrongPasswordHere"
```

### Recommended DSN format

```text
$env:DB_DSN = "localhost:1521/FREEPDB1"
```

## 6) Example integration patterns Codex can use

### Option A: Python (`oracledb` thin mode)

Install dependency:

```powershell
pip install oracledb
```

Minimal connectivity script:

```python
import os
import oracledb

conn = oracledb.connect(
    user=os.environ["DB_USER"],
    password=os.environ["DB_PASSWORD"],
    dsn=os.environ.get("DB_DSN", f"{os.environ['DB_HOST']}:{os.environ['DB_PORT']}/{os.environ['DB_SERVICE']}")
)

with conn.cursor() as cur:
    cur.execute("select 'Oracle 23ai connection OK' from dual")
    print(cur.fetchone()[0])

conn.close()
```

### Option B: Node.js (`oracledb`)

Install dependency:

```powershell
npm install oracledb
```

Minimal connectivity script:

```javascript
const oracledb = require('oracledb');

async function run() {
  const connection = await oracledb.getConnection({
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    connectString: process.env.DB_DSN || `${process.env.DB_HOST}:${process.env.DB_PORT}/${process.env.DB_SERVICE}`
  });

  const result = await connection.execute(`select 'Oracle 23ai connection OK' as MSG from dual`);
  console.log(result.rows[0][0]);
  await connection.close();
}

run().catch(err => {
  console.error(err);
  process.exit(1);
});
```

## 7) Common network cases on Windows

- **Codex process on same Windows host**: use `localhost:1521/SERVICE_NAME`.
- **Codex process in WSL2**: try Windows host IP from `ipconfig`; ensure Windows Firewall allows inbound TCP 1521.
- **Codex process in Docker**: use `host.docker.internal:1521/SERVICE_NAME` (if supported) or host IP.

## 8) Firewall checklist

If connections timeout or are refused:

1. Open **Windows Defender Firewall with Advanced Security**.
2. Add inbound rule for TCP `1521`.
3. Restrict scope to local/private network profiles where possible.

PowerShell example:

```powershell
New-NetFirewallRule -DisplayName "Oracle 1521 Inbound" -Direction Inbound -Protocol TCP -LocalPort 1521 -Action Allow
```

## 9) Security best practices

- Do not commit real DB credentials into source control.
- Use a dedicated low-privilege user for Codex workflows.
- Rotate passwords regularly.
- Prefer local-only listener exposure for development.
- Mask secrets in logs and terminal output.

## 10) Quick troubleshooting

- **ORA-12541 (no listener)**: start listener (`lsnrctl start`) and verify port/service.
- **ORA-01017 (invalid username/password)**: check credential case, quoting, and target PDB.
- **ORA-12154 / connection resolution issues**: use full EZConnect string (`host:port/service`).
- **Timeout/refused**: verify firewall rule and host/IP routing between Codex runtime and Windows host.

## 11) Suggested workflow with Codex

1. Keep SQL migrations and schema scripts in the repo.
2. Ask Codex to generate migration SQL and validation queries.
3. Execute scripts with your chosen client.
4. Ask Codex to analyze query plans and optimize indexes based on `EXPLAIN PLAN` output.

This keeps Codex focused on code and SQL generation while your local Oracle 23ai instance remains the execution backend.
