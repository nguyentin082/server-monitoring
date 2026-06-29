# 🖥️ Server Monitoring Stack

> **Observability stack đầy đủ** chạy trên Docker Compose — metrics, logs, visualization trong một lệnh.

---

## 📐 Kiến trúc tổng thể

```
┌─────────────────────────────────────────────────────────────────────┐
│                          VISUALIZATION                              │
│                                                                     │
│                    ┌─────────────────────┐                          │
│                    │      Grafana         │  :3000                  │
│                    │  dashboards + logs   │                          │
│                    └──────────┬──────────┘                          │
│                               │                                     │
└───────────────────────────────┼─────────────────────────────────────┘
                                │
          ┌─────────────────────┼──────────────────────┐
          │                                            │
          ▼                                            ▼
┌─────────────────────┐                    ┌─────────────────────┐
│      METRICS        │                    │       LOGS          │
│                     │                    │                     │
│  Prometheus  :9090  │                    │    Loki  :3100      │
│  (TSDB store)       │                    │  (log aggregation)  │
│                     │                    │                     │
└──────────┬──────────┘                    └──────────┬──────────┘
           │ scrape                                    │ push
    ┌──────┴──────┐                         ┌─────────┴──────────┐
    │             │                         │                    │
    ▼             ▼                         ▼                    ▼
┌────────┐  ┌──────────┐            ┌────────────────┐  ┌──────────────┐
│  Node  │  │ cAdvisor │            │   Docker logs  │  │  /var/log    │
│Exporter│  │  :1081   │            │ (all container)│  │  + journal   │
│  :9100 │  │          │            └────────────────┘  └──────────────┘
│        │  │          │                       ▲                ▲
│ host   │  │container │                       └────────────────┘
│metrics │  │ metrics  │                            Promtail
└────────┘  └──────────┘                        (log shipper)
```

---

## 🔄 Data Flow chi tiết

```
HOST SYSTEM
│
├── CPU / RAM / Disk / Network
│       └── node-exporter (:9100) ──────────────────────► Prometheus
│
├── Docker containers (stats)
│       └── cAdvisor (:1081) ────────────────────────────► Prometheus
│
│                                            Prometheus ──► Grafana
│
├── Docker container logs
│       └──────────────┐
├── /var/log/*.log      ├── Promtail ──────────────────────► Loki ──► Grafana
└── systemd journal ───┘
```

---

## 📦 Services

| Service | Image | Port | Vai trò |
|---------|-------|------|---------|
| **Grafana** | `grafana/grafana:latest` | `3000` | Dashboard & log query UI |
| **Prometheus** | `prom/prometheus:latest` | `9090` | Thu thập & lưu metrics (TSDB) |
| **Loki** | `grafana/loki:3.5.0` | `3100` | Thu thập & lưu logs |
| **Promtail** | `grafana/promtail:3.5.0` | — | Ship logs → Loki |
| **Node Exporter** | `prom/node-exporter:latest` | `9100` | Metrics hệ thống host |
| **cAdvisor** | `gcr.io/cadvisor/cadvisor:latest` | `1081` | Metrics Docker containers |
| **Dozzle** | `amir20/dozzle:latest` | `1000` | Xem Docker logs realtime |

---

## 🗄️ Nguồn dữ liệu Grafana

| Datasource | Type | URL nội bộ | Default |
|------------|------|-----------|---------|
| **Prometheus** | `prometheus` | `http://prometheus:9090` | ✅ |
| **Loki** | `loki` | `http://loki:3100` | — |

> Cả hai datasource được **auto-provisioned** — không cần cấu hình thủ công sau khi khởi động.

---

## 📊 Prometheus Targets

| Job | Target | Metrics |
|-----|--------|---------|
| `node` | `node-exporter:9100` | CPU, RAM, disk, network, filesystem |
| `cadvisor` | `cadvisor:8080` | Container CPU/RAM/net/IO |
| `prometheus` | `prometheus:9090` | Prometheus self-metrics |

---

## 📝 Promtail — Log Sources

| Job | Nguồn | Labels |
|-----|-------|--------|
| `docker` | Docker socket (service discovery) | `container`, `image`, `compose_service`, `compose_project` |
| `journal` | systemd journal | `unit`, `hostname` |
| `varlogs` | `/var/log/*.log` | `job=varlogs`, `host` |

> Promtail tự động detect **tất cả Docker containers** đang chạy qua Docker Service Discovery — không cần cấu hình từng container.

---

## 🗂️ Cấu trúc thư mục

```
server-monitoring/
├── docker-compose.yml              # Định nghĩa toàn bộ stack
├── prometheus.yml                  # Scrape configs cho Prometheus
├── loki-config.yml                 # Cấu hình Loki (storage, retention)
├── promtail-config.yml             # Cấu hình Promtail (scrape jobs)
├── .env                            # Biến môi trường (ports, passwords)
├── .env.example                    # Template .env
└── grafana/
    └── provisioning/
        └── datasources/
            └── datasource.yml      # Auto-provision Prometheus + Loki
```

---

## ⚡ Quick Start

```bash
# 1. Clone và vào thư mục
git clone <repo-url> && cd server-monitoring

# 2. (Tùy chọn) Tùy chỉnh cấu hình
cp .env.example .env
# Chỉnh sửa ports, password trong .env

# 3. Khởi động toàn bộ stack
docker compose up -d

# 4. Dừng toàn bộ stack
docker compose down

# 5. Kiểm tra trạng thái
docker compose ps
```

---

## 🕵️ Cài đặt Agent (Remote Server)

Để cài đặt agent thu thập metrics và logs trên một server khác (chỉ chạy **Node Exporter**, **cAdvisor**, và **Promtail**):

```bash
# 1. Clone thư mục dự án lên remote server
git clone <repo-url> && cd server-monitoring

# 2. Tạo file cấu hình từ template agent
cp .env.agent.example .env

# 3. Cập nhật các biến môi trường cần thiết
# Mở file .env và điền IP/Domain của máy chủ chính vào LOKI_URL, và thiết lập SERVER_HOSTNAME
nano .env

# 4. Khởi động các agent services
docker compose -f docker-compose.agent.yml up -d

# 5. Kiểm tra trạng thái
docker compose -f docker-compose.agent.yml ps

#6. Dừng các agent services
docker compose -f docker-compose.agent.yml down
```
