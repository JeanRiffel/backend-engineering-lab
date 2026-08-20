# Backend Engineering Lab

A reusable, Docker-based local development infrastructure: relational and
non-relational databases, a message broker, a reverse proxy, and a full
observability stack (metrics, logs and traces) — wired together and ready to
run with a single command, so you don't have to rebuild this setup for every
new backend project.

Built as part of my preparation for backend engineering roles, with a focus on
the infrastructure pieces that show up in real production systems.

## Stack

| Category      | Technology                         |
|----------------|-------------------------------------|
| Relational DB  | PostgreSQL 17                       |
| Document DB    | MongoDB 8                           |
| Cache          | Redis 7                             |
| Message broker | RabbitMQ 3 (management UI included) |
| Reverse proxy  | nginx                               |
| Metrics/dashboards | Grafana                         |
| Logs           | Grafana Loki + Alloy (log shipping) |
| Traces         | Grafana Tempo (OTLP)                |

## Architecture

```
                     ┌────────────┐
   client ────────▶  │   nginx    │  :80   (optional, see "Adding your own service")
                     └─────┬──────┘
                           │ proxy_pass
                           ▼
                   ┌────────────────┐
                   │   your app     │  ← not included — this repo is just the infra
                   └───────┬────────┘
              ┌────────────┼─────────────┬───────────────┐
              ▼            ▼              ▼               ▼
         ┌─────────┐  ┌────────┐   ┌──────────┐   ┌──────────────┐
         │Postgres │  │ Redis  │   │  Mongo   │   │  RabbitMQ    │
         │ :5432   │  │ :6379  │   │ :27017   │   │ :5672/:15672 │
         └─────────┘  └────────┘   └──────────┘   └──────────────┘

  Observability (independent of the app path above):

  containers ──logs──▶ Alloy ──▶ Loki ──┐
  app/services ──traces (OTLP)──▶ Tempo ─┼──▶ Grafana :3300
                                          ┘
```

## Services & default ports

| Service   | URL / Port                          | Default credentials         |
|-----------|--------------------------------------|------------------------------|
| Postgres  | `localhost:5432`                     | `admin` / `admin`            |
| Redis     | `localhost:6379`                     | —                             |
| MongoDB   | `localhost:27017`                    | `admin` / `admin`            |
| RabbitMQ  | `localhost:5672` (AMQP)              | `admin` / `admin`            |
| RabbitMQ management | http://localhost:15672      | `admin` / `admin`            |
| nginx (opt-in) | http://localhost:80             | —                             |
| Grafana   | http://localhost:3300                | `admin` / `admin`            |
| Loki      | `localhost:3100`                     | —                             |
| Tempo     | `localhost:3200` (HTTP), `4317`/`4318` (OTLP) | —                    |
| Alloy UI  | http://localhost:12345               | —                             |

Credentials come from `compose/.env` (see below) — the values above are just the
defaults used if you don't create one.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and Docker Compose v2 (`docker compose`, not `docker-compose`).

## Getting started

```bash
git clone https://github.com/<your-user>/backend-engineering-lab.git
cd backend-engineering-lab/compose

cp .env.example .env      # adjust credentials if you want

docker compose up -d      # start the infra (databases, broker, observability)
docker compose ps         # check status / health
```

Stop and remove containers (keeping data volumes):

```bash
docker compose down
```

Stop and wipe all data (fresh start):

```bash
docker compose down -v
```

## Adding your own service

`nginx` is not started by default — it's an example reverse-proxy config, not
a fixed dependency of the infra. It's meant to front whichever app you plug
into this environment:

1. Add your service to `docker-compose.yml`, attached to the `backend_lab`
   network (so it can reach Postgres, Redis, Mongo and RabbitMQ by container
   name — e.g. `postgres:5432`).
2. Rename the `api` upstream in `compose/nginx/nginx.conf` to match your
   service's container name (defaults to `api:3000`).
3. Start it together with nginx: `docker compose --profile app up -d`.

Point your app's OpenTelemetry exporter at Tempo (`tempo:4317` from inside the
network) and its `docker` log driver already flows into Loki via Alloy — no
extra config needed for logs.

## Project structure

```
.
├── compose/
│   ├── docker-compose.yml   # all services
│   ├── .env.example         # copy to .env, never commit the real one
│   ├── nginx/nginx.conf     # reverse proxy config (opt-in, see above)
│   ├── tempo/tempo.yaml     # Tempo config (OTLP receiver, local storage)
│   └── alloy-config.alloy   # Alloy pipeline: docker logs → Loki
├── LICENSE
└── README.md
```

## Known limitations

- All credentials default to `admin/admin` for convenience. This is meant for
  **local development only** — never reuse these values outside your machine.

## Roadmap

- [ ] Provision Grafana datasources/dashboards automatically (Loki + Tempo)
- [ ] Add a `Makefile` with shortcuts (`make up`, `make down`, `make logs`)
- [ ] Add an example app service showing end-to-end wiring (DB + broker + traces)

## License

MIT — see [LICENSE](LICENSE).
