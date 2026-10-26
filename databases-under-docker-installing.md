# Databases Under Docker — AppDynamics Monitoring Lab

This project starts three independent database services on one Docker host:

- MySQL 8.4
- PostgreSQL 16
- MongoDB 7.0

The databases are intended for AppDynamics Database Agent connection and monitoring tests. Their ports are published on the host IP configured in `.env`, so Database Agents on other machines can connect remotely.

> **Network security:** Do not expose database ports to the public internet. Restrict access in the host/network firewall to the IP addresses of the approved Database Agent hosts. Use a private network or VPN where possible.

## 1. Files

```text
db-under-docker/
├── docker-compose.yml
├── .env.example
└── README.md
```

## 2. Requirements

- Docker Engine and the Docker Compose plugin.
- Network routing between the database VM and the Database Agent hosts.
- Firewall rules that allow approved Agent IPs to reach the database ports.

**Local file-access note:** On this host, the Docker executable was found at `/snap/bin/docker`, and earlier Docker commands could not open files under `/opt`. To avoid repeating that issue, keep this project under the user's home directory (for example, `~/db-under-docker`) rather than `/opt`. If file-access errors continue, check the Docker Snap confinement and its interfaces.

## 3. Configure `.env`

From the project directory, copy the example file:

```bash
cp .env.example .env
chmod 600 .env
```

Edit `.env` and set `DB_HOST_IP` to the actual IP address of this VM that the Database Agent hosts can reach. Replace all `CHANGE_ME_...` values with strong, unique passwords.

Example only:

```dotenv
DB_HOST_IP=192.168.1.150
MYSQL_ROOT_PASSWORD=CHANGE_ME_MYSQL_ROOT_PASSWORD
POSTGRES_ROOT_PASSWORD=CHANGE_ME_POSTGRES_ROOT_PASSWORD
MONGO_ROOT_USERNAME=mongoadmin
MONGO_ROOT_PASSWORD=CHANGE_ME_MONGO_ROOT_PASSWORD
```

The IP above is an example. Do not use it unless it is the real, reachable address of your VM. Keep `.env` private and do not commit it to a public repository.

## 4. Start all three databases at once

Run these commands from the directory containing `docker-compose.yml` and `.env`:

```bash
docker compose config
docker compose pull
docker compose up -d
docker compose ps
```

- `docker compose config` validates the Compose configuration and resolves values from `.env`.
- `docker compose pull` downloads the images.
- `docker compose up -d` starts all three database containers in the background.
- `docker compose ps` shows the container status.

The Compose file creates a shared Docker network named `db-monitoring` and persistent volumes for each database. The container resource limits are 768 MB memory and 1 CPU per database; actual consumption depends on workload.

## 5. Published ports and container names

| Database | Container | Host port | Container port | Compose database |
|---|---|---:|---:|---|
| MySQL 8.4 | `dbmon-mysql` | `3306` | `3306` | `appdmon` |
| PostgreSQL 16 | `dbmon-postgres` | `5432` | `5432` | `appdmon` |
| MongoDB 7.0 | `dbmon-mongo` | `27017` | `27017` | `admin` for authentication |

The ports bind to `DB_HOST_IP`, not to every host interface. Use the VM's reachable IP in the AppDynamics Collector configuration. Ensure that the selected ports are not already in use on the host.

## 6. MySQL monitoring user rules

The following MySQL rules are included **exactly as supplied**. Use the rule that matches your MySQL version. The Compose service uses MySQL 8.4, so use the **MySQL 8+** rule for that service.

### For MySQL before 8

```sql
GRANT SELECT,PROCESS,SHOW DATABASES on *.* to 'DBMon_Agent_User'@'host' identified by 'password'; GRANT REPLICATION CLIENT ON *.* to 'DBMon_Agent_User'@'host'; FLUSH privileges;
```

### For MySQL 8+

```sql
CREATE USER 'DBMon_Agent_User'@'%' identified by 'Aa123456'; GRANT SELECT,PROCESS,SHOW DATABASES on *.* to 'DBMon_Agent_User'@'%' WITH GRANT OPTION; GRANT REPLICATION CLIENT ON *.* to 'DBMon_Agent_User'@'%' WITH GRANT OPTION; FLUSH privileges;
```

For the Compose MySQL container, open the MySQL client as root:

```bash
docker compose exec mysql mysql -uroot -p
```

Enter the `MYSQL_ROOT_PASSWORD` value from `.env`, then run the MySQL 8+ statements above. The supplied username, password, grants, and `WITH GRANT OPTION` are preserved as requested. Note that `WITH GRANT OPTION` permits this monitoring user to grant its privileges to other users; keep that in mind before using the supplied rule on a production system.

The pre-8 rule contains the literal placeholders `'host'` and `'password'`; it is provided unchanged and is intended only for the corresponding older MySQL version.

## 7. MongoDB monitoring user rule

The following MongoDB rule is included **exactly as supplied**:

```javascript
use admin db.createUser({ user: "tanujaAdmin", pwd: "tanuja123", roles: [ { role: "clusterMonitor", db: "admin" }, { role: "read", db: "admin" }, ] })
```

Connect to MongoDB as the root user created from `.env`:

```bash
docker compose exec mongo mongosh --username mongoadmin --password --authenticationDatabase admin
```

When prompted, enter the value of `MONGO_ROOT_PASSWORD` from `.env`. In the MongoDB shell, run the supplied rule. If your `MONGO_ROOT_USERNAME` in `.env` is not `mongoadmin`, use that configured username in the command.

The supplied rule creates `tanujaAdmin` in the `admin` database with the `clusterMonitor` and `read` roles on `admin`. Its username and password are preserved exactly as requested. Replace these example credentials with strong credentials before allowing access from other machines.

## 8. PostgreSQL monitoring rules

The following PostgreSQL rules are included **exactly as supplied**:

```sql
CREATE FUNCTION get_sa() RETURNS SETOF pg_stat_activity LANGUAGE sql AS $$ SELECT * FROM pg_catalog.pg_stat_activity; $$ VOLATILE SECURITY DEFINER; CREATE VIEW pg_stat_activity_allusers AS SELECT * FROM get_sa(); GRANT SELECT ON pg_stat_activity_allusers TO public;
CREATE FUNCTION get_querystats() RETURNS SETOF pg_stat_statements LANGUAGE sql AS $$ SELECT * FROM pg_stat_statements; $$ VOLATILE SECURITY DEFINER; CREATE VIEW pg_stat_statements_allusers AS SELECT * FROM get_querystats(); GRANT SELECT ON pg_stat_statements_allusers TO public;
```

The Compose PostgreSQL service starts with `shared_preload_libraries=pg_stat_statements`. Connect to PostgreSQL:

```bash
docker compose exec postgres psql -U postgres -d appdmon
```

Enter the `POSTGRES_ROOT_PASSWORD` value from `.env` if prompted. Before running the supplied rules, create the extension:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

Create a login role for the Database Agent (replace the example password with a strong password of your choice):

```sql
CREATE ROLE appdmonitor LOGIN PASSWORD 'CHANGE_ME_POSTGRES_MONITOR_PASSWORD';
GRANT CONNECT ON DATABASE appdmon TO appdmonitor;
```

Then run both supplied SQL statements above in the same `appdmon` database. The statements create the functions and views and grant `SELECT` on the views to `public`, as supplied. Granting to `public` allows every database role to query those views; review that access before using this setup beyond an isolated lab.

## 9. Verify database startup

```bash
docker compose ps
docker compose logs --tail=50 mysql
docker compose logs --tail=50 postgres
docker compose logs --tail=50 mongo
```

Wait for each database to finish initialization before creating monitoring users or configuring collectors.

## 10. Test remote TCP connectivity

From each Database Agent host, run the relevant tests (replace `DB_HOST_IP` with the actual IP):

```bash
nc -vz DB_HOST_IP 3306
nc -vz DB_HOST_IP 5432
nc -vz DB_HOST_IP 27017
```

A successful TCP test verifies that the port can be reached; it does not verify database credentials. If a test fails, check the database container status, the VM's host firewall, any network firewall, routing, and whether the IP in `.env` is correct.

## 11. AppDynamics Collector connection details

| Field | MySQL | PostgreSQL | MongoDB |
|---|---|---|---|
| Host | VM IP (`DB_HOST_IP`) | VM IP (`DB_HOST_IP`) | VM IP (`DB_HOST_IP`) |
| Port | `3306` | `5432` | `27017` |
| Monitoring user from supplied rules | `DBMon_Agent_User` | `appdmonitor` (created in section 8) | `tanujaAdmin` |
| Database | `appdmon` | `appdmon` | Authentication database: `admin` |
| Password | As set in the MySQL rule | As set when creating `appdmonitor` | `tanuja123` from the supplied rule |

Use the settings required by the installed AppDynamics Database Agent/Collector version. For MongoDB, set the authentication database to `admin`.

## 12. Stop and remove containers

Stop the containers but preserve their database data:

```bash
docker compose down
```

Start them again later with:

```bash
docker compose up -d
```

**Warning:** This command removes the named database volumes and permanently deletes the data stored in them:

```bash
docker compose down -v
```

Do not use `-v` unless you intentionally want to delete the databases' persistent data.
