
# AppDynamics Database Agent in Docker

## Image summary

| Item       | Value                                                                             |
| ---------- | --------------------------------------------------------------------------------- |
| Image      | `appdynamics/db-agent:latest`                                                     |
| Version    | `26.7.1-6035` (built 2026-10-08)                                                  |
| Platform   | linux/amd64, ~360 MB, 10 layers                                                   |
## Image internals
- No `.env` file and no Controller values baked in.
- `conf/controller-info.xml` is empty (placeholders only). Everything is configured through environment variables.
- Other useful paths inside `/opt/appdynamics/db-agent`:
  - `README-to-run-dbagent-in-docker.md` – official Docker run notes
  - `start-appdynamics-database-monitoring` – entrypoint (identical to `start-dbagent-dockerfile-entrypoint.sh`)
  - `k8s-deploy/` – sample Kubernetes manifests
  - `lib/` – JDBC drivers (extra drivers, e.g. Oracle/Sybase, must be mounted here)
  - `conf/logging/` – logging config

## How the entrypoint works

The script converts env vars into `-D` JVM properties, prints a masked copy (access key replaced with `****`) to the container logs, then runs:

```sh
java ${DB_AGENT_PROPERTIES} -jar ./db-agent.jar
```

Default JVM flags for Java 17:
`--add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.security=ALL-UNNAMED`

## Environment variables

| Env var                                | JVM property                         | Purpose                               |
| -------------------------------------- | ------------------------------------ | ------------------------------------- |
| `APPDYNAMICS_CONTROLLER_HOST_NAME`     | `appdynamics.controller.hostName`    | Controller host/IP                    |
| `APPDYNAMICS_CONTROLLER_PORT`          | `appdynamics.controller.port`        | 8090 HTTP / 8181 HTTPS (443 for SaaS) |
| `APPDYNAMICS_CONTROLLER_SSL_ENABLED`   | `appdynamics.controller.ssl.enabled` | `true` / `false`                      |
| `APPDYNAMICS_AGENT_ACCOUNT_NAME`       | `appdynamics.agent.accountName`      | Account name                          |
| `APPDYNAMICS_AGENT_ACCOUNT_ACCESS_KEY` | `appdynamics.agent.accountAccessKey` | Access key                            |
| `APPDYNAMICS_DB_AGENT_NAME`            | `dbagent.name`                       | Agent name shown in the Controller    |
| `APPDYNAMICS_AGENT_UNIQUE_HOST_ID`     | `appdynamics.agent.uniqueHostId`     | Unique host ID (optional)             |
| `APPDYNAMICS_DB_PROPERTIES`            | (appended as-is)                     | Extra `-D` flags, space separated     |

> [!warning]
> `controller-info.xml` mentions `CONTROLLER_HOST_NAME`, but the entrypoint only reads `APPDYNAMICS_CONTROLLER_HOST_NAME`. Use the latter.

## `.env` file

```dotenv
APPDYNAMICS_CONTROLLER_HOST_NAME=192.168.1.143
APPDYNAMICS_CONTROLLER_PORT=8090
APPDYNAMICS_CONTROLLER_SSL_ENABLED=false
APPDYNAMICS_AGENT_ACCOUNT_NAME=customer1
APPDYNAMICS_AGENT_ACCOUNT_ACCESS_KEY=<ACCESS_KEY>
```

- No quotes around values when used with `docker run --env-file`.
- `chmod 600 .env` and keep it out of git.
- Rotate the access key if it was ever pasted somewhere shared.

## Run

### docker run

```bash
docker run -d --name dbagent \
  --restart unless-stopped \
  --env-file .env \
  -e APPDYNAMICS_DB_AGENT_NAME="DatabaseAgent" \
  -e APPDYNAMICS_DB_PROPERTIES="-Ddbagent.telemetry.enabled=true" \
  appdynamics/db-agent:latest
```

### docker compose

```yaml
services:
  dbagent:
    image: appdynamics/db-agent:latest
    container_name: dbagent
    restart: unless-stopped
    env_file: .env
    environment:
      APPDYNAMICS_DB_AGENT_NAME: "DatabaseAgent"
      APPDYNAMICS_DB_PROPERTIES: "-Ddbagent.telemetry.enabled=true"
```

### Pre-flight connectivity check

```bash
curl -sI http://192.168.1.143:8090/controller/ | head -1
```

## Debugging

### Keep a container alive to explore it

```bash
docker run -d --name dbagent-debug \
  --entrypoint /bin/busybox \
  appdynamics/db-agent:latest \
  sleep infinity
# fallback if busybox doesn't accept "infinity":
#   ... tail -f /dev/null

docker exec -it dbagent-debug /bin/busybox sh

docker rm -f dbagent-debug
```

Start the agent manually inside that container:

```sh
sh /opt/appdynamics/db-agent/start-appdynamics-database-monitoring
```

### Inspect files without running the agent

```bash
docker run --rm --entrypoint /bin/busybox appdynamics/db-agent:latest \
  ls -la /opt/appdynamics/db-agent

docker run --rm --entrypoint /bin/busybox appdynamics/db-agent:latest \
  cat /opt/appdynamics/db-agent/conf/controller-info.xml
```

### Logs

```bash
docker logs -f dbagent
```

Expected at startup: `MASKED_DB_AGENT_PROPERTIES=...` (key masked), then agent connection logs. If it exits, the cause is usually missing env vars or the Controller being unreachable.

## Monitoring databases

- Databases are added in the Controller UI: **Database Visibility → Configuration → Add Collector**, not via container env vars.
- The container must be able to reach each DB host and port. For a DB on the Docker host use `host.docker.internal` or `network_mode: host`.
- JDBC drivers for PostgreSQL, MySQL and SQL Server are normally bundled. For Oracle/Sybase mount the driver into `lib/`:

```yaml
    volumes:
      - ./drivers/ojdbc.jar:/opt/appdynamics/db-agent/lib/ojdbc.jar:ro
```
