# setup-agent

Self-hosted [Hermes Agent](https://hermes-agent.nousresearch.com/) gateway running in Docker, with read-only access to the host Docker API through a socket proxy.

## Services

| Service               | Image                              | Role                                                       |
| --------------------- | ---------------------------------- | ---------------------------------------------------------- |
| `hermes`              | `nousresearch/hermes-agent:latest` | The agent: CLI/TUI plus messaging gateways (Telegram, Discord, Slack, …) |
| `docker-socket-proxy` | `tecnativa/docker-socket-proxy:latest` | Read-only view of the host Docker API (`POST`, `AUTH`, `SECRETS` disabled) |

## Requirements

- Docker with Compose v2 (`docker compose`)

## Quick start

```bash
HERMES_UID=$(id -u) HERMES_GID=$(id -g) docker compose up -d
docker compose exec -u hermes hermes hermes setup gateway
```

`HERMES_UID`/`HERMES_GID` remap the container's internal `hermes` user to your host user, so files written under `./data` stay readable outside the container. Skip them if you do not care about host-side file ownership.

The setup wizard writes config and API keys to `./data` (mounted at `/opt/data` inside the container). It runs once; the gateway keeps running afterwards.

For the full wizard (provider, model, tools, personality) instead of just the messaging section:

```bash
docker compose exec -u hermes hermes hermes setup
```

## Commands

| Command                                             | Purpose                                        |
| --------------------------------------------------- | ---------------------------------------------- |
| `docker compose up -d`                              | Start both services                            |
| `docker compose logs -f hermes`                     | Follow gateway logs                            |
| `docker compose exec -u hermes hermes hermes`       | Interactive CLI chat                           |
| `docker compose exec -u hermes hermes hermes doctor` | Diagnose config and dependency issues         |
| `docker compose exec -u hermes hermes hermes status` | Agent, auth, and platform status              |
| `docker compose pull && docker compose up -d`        | Update the image (data preserved)              |
| `docker compose down`                               | Stop                                           |

## Docker access

Hermes reaches the host Docker daemon through `docker-socket-proxy` (`DOCKER_HOST=tcp://docker-socket-proxy:2375`), not a bind-mounted socket. The proxy allows read calls only (`CONTAINERS`, `IMAGES`, `INFO`, `NETWORKS`, `VOLUMES`, `SYSTEM`); `POST`, `AUTH`, and `SECRETS` are disabled, so the agent cannot create or delete containers or read secrets.

Known upstream limitation: `hermes doctor` reports "docker daemon not running" when the daemon is reached over `DOCKER_HOST` instead of a local socket. Verify the backend functionally instead:

```bash
docker compose exec -T hermes docker version
```

## Data

`./data` is the single source of truth for all Hermes state and is mounted at `/opt/data`:

| Path         | Contents                                        |
| ------------ | ----------------------------------------------- |
| `.env`       | API keys and secrets                            |
| `config.yaml`| Hermes configuration                            |
| `sessions/`  | Conversation history                            |
| `memories/`  | Persistent memory                               |
| `skills/`    | Installed skills                                |
| `cron/`      | Scheduled jobs                                  |
| `logs/`      | Runtime logs                                    |

`data/` is gitignored — it holds secrets. Back it up before image upgrades; the image itself is stateless.

## Configuration

- **Gateway API**: published on `127.0.0.1:8642` only. It needs `API_SERVER_ENABLED=true` plus `API_SERVER_KEY` (min 8 chars) in `./data/.env` before it answers.
- **Resources**: 4 GB RAM, 2 CPUs.
- **Security**: `no-new-privileges` on the gateway; the socket proxy allows read calls only; the published port is loopback-bound.

## Agent skills

`.agents/skills/` holds project-scoped agent skills (e.g. `ub-mode`), following the [agentskills.io](https://agentskills.io) layout.

## Links

- Hermes docs: https://hermes-agent.nousresearch.com/docs/
- Source: https://github.com/NousResearch/hermes-agent
