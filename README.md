# Databasus--PostgreSQL-Backups

# Databasus: PostgreSQL Backup Tool – Notes

## 1. What this is
- Databasus is a web tool (runs as a Docker container) that backs up databases.
- Supports PostgreSQL, MySQL, MariaDB, MongoDB.
- This test: back up a PostgreSQL 17 database (`bench_db`) to (a) the local disk and (b) an S3 bucket.

## 2. Test environment
| Item | Value |
|---|---|
| Server | AWS EC2, Debian 13 (trixie) |
| PostgreSQL | v17 |
| Docker | Already installed (Portainer also running) |
| Databasus UI | `http://<EC2-PUBLIC-IP>:4005` |
| Install folder | `/opt/databasus` |
| Backup data folder | `/opt/databasus/databasus-data/backups` |

## 3. Install (one command)
```bash
sudo apt-get install -y curl && \
sudo curl -sSL https://raw.githubusercontent.com/databasus/databasus/refs/heads/main/install-databasus.sh | sudo bash
```
- Creates `/opt/databasus`, writes `docker-compose.yml`, and starts the container `databasus` (port 4005).
- To start it again later:
```bash
cd /opt/databasus && docker compose up -d
docker ps        # container should show "healthy"
```

**Before opening the UI, check:**
- EC2 security group allows TCP **4005** only from your trusted IP.
- Docker is running.
- Enough disk space for backups.

## 4. First login
- First visit = sign-up page. Whatever user you create becomes the **admin**.
- Then create a workspace (here: "My DB workspace").
![First-time sign-up page](images/step01.png)
*First-time sign-up page*

![Workspace creation (1)](images/step02.png)
*Workspace creation (1)*

![Workspace creation (2)](images/step03.png)
*Workspace creation (2)*


## 5. Set up a backup (UI steps)
1. **Add database** → give a name (e.g. `EC2 PG`) → pick **PostgreSQL**.
2. **Backup type**:
   - **Logical** – for DBs under ~50 GB, simple setup. *(Used here.)*
   - **Physical** – for DBs over 50 GB, enables point-in-time recovery, needs extra setup.
3. **Connection details**: host (private IP `172.31.16.57`), port `5432`, user, password, DB name (`bench_db`), SSL mode (`Disable` here).
4. **Read-only user prompt**: the tool offers to create a read-only DB user for backups.
   - Recommended: it can only read data, so backups can't change or corrupt anything (least privilege).
   - The current `postgres` user is a superuser, which is why it warns.
5. **Schedule & options**:
   - Interval: Daily, time of day (e.g. 2:00 PM)
   - Storage: pick from dropdown (local or S3)
   - Encryption: "Encrypt backup files" option
   - Retention: keep last N time (e.g. 3 months)
   - Notifications: on backup success / on backup failed
6. Click **Test connection** → **Save**.
7. Use **Make backup right now** for a manual backup. Each backup row has actions: delete, verify, upload, download.

**Screenshots for these steps**

![Step 2: choose Logical backup](images/step04.png)
*Step 2: choose Logical backup*

![Step 3: connection details (password blurred)](images/step05.png)
*Step 3: connection details (password blurred)*

![Step 4: read-only user prompt](images/step06.png)
*Step 4: read-only user prompt*

![Step 5: schedule, encryption, retention, notifications](images/step07.png)
*Step 5: schedule, encryption, retention, notifications*

Storage dropdown, then add/test the storage:

![Adding Local storage – note the warning: local disk can run out of space; S3 recommended](images/step08.png)
*Adding Local storage – note the warning: local disk can run out of space; S3 recommended*

![Test connection and save](images/step09.png)
*Test connection and save*

![Backups tab: 0.67 MB backup for a 149.7 MB database](images/step10.png)
*Backups tab: 0.67 MB backup for a 149.7 MB database*

## 6. Storage option A: Local file system
- Backups sit in `/opt/databasus/databasus-data/backups`.
- Each backup = 2 files:
  - the backup file (~701 KB in the test)
  - a small `.metadata` file (174 bytes)
- File name pattern: `<DBName>-<date>-<time>-<uuid>`.
- Files are owned by user `999`.

## 7. Storage option B: Amazon S3
**Prerequisites**
- An S3 bucket already created
- IAM access key + secret key with access to that bucket

**Steps**
1. Add storage → Type = **S3**.
2. Fill: name, bucket (`pg-ec2-backup`), region (`us-east-1`), access key, secret key. Endpoint is optional (for S3-compatible services like Cloudflare R2).
3. **Test connection** → save → select it as the storage of your database backup.
4. Run a backup, then check the bucket: it contains the backup (685.3 KB) + its `.metadata` file (174 B).

**Screenshots for S3**

![Add a second database entry (EC2 PG to S3)](images/step11.png)
*Add a second database entry (EC2 PG to S3)*

![Choose Logical again](images/step12.png)
*Choose Logical again*

![Connection details – this time using the auto-created read-only user `databasus-...`](images/step13.png)
*Connection details – this time using the auto-created read-only user `databasus-...`*

![Add S3 storage](images/step14.png)
*Add S3 storage*

![Backup done, stored in S3](images/step15.png)
*Backup done, stored in S3*

![Backup file + metadata visible in the S3 bucket](images/step16.png)
*Backup file + metadata visible in the S3 bucket*

## 8. Findings from the test
- **Size:** the UI shows a **0.67 MB** backup for a **149.7 MB** database (about 220x smaller). Databasus backup was **smaller** than a `pg_dump` plain-format backup of the same DB (reason: it is compressed/binary, not plain SQL).
- **Limitation:** no option to produce a **plain SQL format** dump. Output is always the tool's own dump format.
  - You restore through the tool (or `pg_restore`-style tools), not by reading the file.
- Same backup was 701 KB on local disk vs 685 KB in S3; small difference, not explained in the source.

## 9. Things to watch out for (my notes, not in the source doc)
- **Password in original doc:** your Word file shows a DB password in clear text. I blurred it here. Rotate that password and don't share the original doc.
- **Use a read-only user** rather than the `postgres` superuser.
- **Port 4005 on `0.0.0.0`:** the login page is open to the internet unless the security group restricts it. Use IP restriction or HTTPS reverse proxy.
- **Local backups live on the same server** as the database. If the EC2 dies, you lose both. Prefer S3 (or both).
- **Encryption keys:** if you enable encryption, store the key safely; without it the backups can't be restored.
- **Test a restore.** A backup is not proven until you've restored it. The doc only tests taking backups.
- **Size comparison is thin:** one test, one DB. Also compare backup time, restore time and `pg_dump -Fc` (custom format, which is also compressed) for a fair result.
- Over 50 GB, plan for the **Physical** backup type.

## 10. Quick summary
- Install: 1 command → UI on port 4005
- Add DB (Logical) → choose storage → schedule/retention → test → save
- Storage: local disk or S3 both work
- Good: easy UI, smaller backups, scheduling, retention, alerts
- Bad: no plain SQL format; restore/verification not covered in the doc
