# Server Monitoring

Minimal Docker Compose stack for monitoring servers with Prometheus, Grafana, Node Exporter, cAdvisor, and an example application (`doozle`).

Quick start
1. (Optional) Set environment variables in a `.env` file (ports, admin passwords).
2. Run:

```bash
docker compose up -d
```

Access services
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000
- Node Exporter: port 9100
- cAdvisor: http://localhost:1081
- Doozle (example): http://localhost:8082

This repository contains the `docker-compose.yml` and `prometheus.yml` for the stack.
