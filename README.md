# Backend Engineering Lab

A local, Docker-based development environment for practicing backend engineering:
relational and non-relational databases, a message broker, a reverse proxy, and a
full observability stack (metrics, logs and traces) wired together and ready to run
with a single command.

Built as part of my preparation for backend engineering roles, with a focus on the
infrastructure pieces that show up in real production systems.

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
   client ────────▶  │   nginx    │  :80
                     └─────┬──────┘
                           │ proxy_pass
                           ▼
                   ┌────────────────┐
                   │  order-api     │  ← not included yet, see "Known limitations"
                   │  (your app)    │
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
| nginx     | http://localhost:80                  | —                             |
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

docker compose up -d      # start everything in the background
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

## Project structure

```
.
├── compose/
│   ├── docker-compose.yml   # all services
│   ├── .env.example         # copy to .env, never commit the real one
│   ├── nginx/nginx.conf     # reverse proxy config
│   ├── tempo/tempo.yaml     # Tempo config (OTLP receiver, local storage)
│   └── alloy-config.alloy   # Alloy pipeline: docker logs → Loki
├── LICENSE
└── README.md
```

## Known limitations

- `nginx` proxies to an `order-api` upstream on port 3000 that **is not part of
  this stack**. It's a placeholder for whatever backend service you build on top
  of this environment — add it to `docker-compose.yml`, attach it to the
  `backend_lab` network, and name it `order-api` (or edit `compose/nginx/nginx.conf`
  to match your service name).
- All credentials default to `admin/admin` for convenience. This is meant for
  **local development only** — never reuse these values outside your machine.

## Roadmap

- [ ] Add a sample API service (`order-api`) wired to Postgres, Redis, Mongo and RabbitMQ
- [ ] Provision Grafana datasources/dashboards automatically (Loki + Tempo)
- [ ] Add a `Makefile` with shortcuts (`make up`, `make down`, `make logs`)

## License

MIT — see [LICENSE](LICENSE).
