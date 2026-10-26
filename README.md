# AppDynamics Database Agent in Docker

Run the AppDynamics Database Agent in Docker and connect it to an AppDynamics Controller for database monitoring.

This repository includes Docker Compose configuration, installation notes, and troubleshooting guides for running the Database Agent in a containerized environment.

## Overview

- **Agent:** `appdynamics/db-agent:latest`
- **Documented version:** `26.7.1-6035` — verify the image version before deployment
- **Platform:** Linux/AMD64
- **Deployment:** Docker or Docker Compose
- **Controller connection:** HTTP or HTTPS
- **Database monitoring:** Configure collectors through the AppDynamics Controller UI

## Repository contents

| File | Purpose |
|---|---|
| `docker-compose.yml` | Docker Compose service definition |
| `databases-under-docker-installing.md` | Guide to running databases in Docker |
| `db-agent-docker-guide.md` | Database Agent installation and troubleshooting |
| `.env.example` | Template for Controller connection settings |

The actual `.env` files and local image archives are intentionally excluded from version control.

## Prerequisites

- Docker Engine and Docker Compose
- Network connectivity from the agent container to the AppDynamics Controller
- An AppDynamics account name and access key
- Database endpoints reachable from the agent container
- Appropriate database credentials and permissions for each collector

## Configuration

Create a local `.env` file based on the example:

```bash
cp .env.example .env
```

Configure these variables in `.env`:

```dotenv
APPDYNAMICS_CONTROLLER_HOST_NAME=your-controller-host
APPDYNAMICS_CONTROLLER_PORT=8090
APPDYNAMICS_CONTROLLER_SSL_ENABLED=false
APPDYNAMICS_AGENT_ACCOUNT_NAME=your-account-name
APPDYNAMICS_AGENT_ACCOUNT_ACCESS_KEY=your-access-key
```

Use the correct Controller hostname, port, account name, and access key for your environment. Enable SSL when using HTTPS.

**Security:** Never commit `.env` files or real access keys to a public repository. Store credentials securely and rotate any key that has been exposed.

## Run with Docker Compose

Start the Database Agent:

```bash
docker compose up -d
```

Check its status:

```bash
docker compose ps
```

Follow the logs:

```bash
docker compose logs -f dbagent
```

Stop the service:

```bash
docker compose down
```

Ensure that `docker-compose.yml` references the expected image and that the `.env` file exists before starting the container.

## Run with Docker

```bash
docker run -d \
  --name dbagent \
  --restart unless-stopped \
  --env-file .env \
  -e APPDYNAMICS_DB_AGENT_NAME=DatabaseAgent \
  appdynamics/db-agent:latest
```

Inspect the logs:

```bash
docker logs -f dbagent
```

## Network connectivity

The Database Agent must be able to reach both the Controller and each monitored database.

For example, test connectivity to a Controller using:

```bash
curl -sI http://your-controller-host:8090/controller/
```

Replace the example hostname with your actual Controller address.

For databases running in other Docker containers, configure an appropriate shared Docker network or use a reachable hostname and port. On Docker Desktop, `host.docker.internal` can be used to reach services on the host in supported configurations.

## Configure database collectors

Database collectors are configured in the AppDynamics Controller UI, generally under:

**Database Visibility → Configuration → Add Collector**

Select the database type and provide its connection details and credentials. Confirm that the Database Agent can reach the database endpoint and that the database user has the required monitoring permissions.

Additional JDBC drivers may be required for certain database types or configurations.

## Troubleshooting

### Agent container exits

```bash
docker logs dbagent
docker inspect dbagent
```

Check the required environment variables, Controller connectivity, and container configuration.

### Controller connection fails

Verify the hostname, port, SSL settings, account name, access key, and network/firewall rules.

### Database collector cannot connect

Check the database hostname, port, authentication details, permissions, and network access from the agent container.

### Inspect the running container

```bash
docker exec -it dbagent sh
```

The available shell and diagnostic commands depend on the container image.

## Large image archives

Local image archives such as `db-agent.tar.gz` can be used for offline transfer or image loading, but are not included in this Git repository.

If an image archive is required, transfer it separately using an approved artifact repository or file-transfer method.

## Security notes

- Keep `.env` and other credential files out of Git.
- Use least-privilege database accounts.
- Avoid exposing database ports or the stress/debug interfaces to untrusted networks.
-