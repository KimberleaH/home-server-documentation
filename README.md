> ⚠️ This documentation is being reorganized and expanded. Check back soon for updates!

# Home Server Setup

![Home Server Diagram](/assets/system-diagram.png)

---

## Quick Overview
Ubuntu home server with docker + docker compose and Portainer as container visualization. Runs Pi-Hole for local DNS filtering (ad-blocking), serves as a private media archive, and more services planned for the near future. The system is being monitored by Node Exporter, Prometheus, and Grafana.

## Documentation
- The monolith of a document is being reorganized. Check back later!

## Config Files
- [docker-compose.yml](/config/docker-compose.yml)
- [prometheus.yml](/config/prometheus.yml)

---

## System Components & Rationale

### PC Specs
- CPU: Intel Core i5-12600K
- Memory: G.SKILL Ripjaws V Series 16GB (2 x 8GB) DDR4 3200
- Mobo: MSI PRO B760M-P LGA 1700 mATX DDR4 (no onboard wifi/bluetooth)
- Boot: Samsung 990 Evo Plus 1TB
- HHDs: x2 Seagate 4TB IronWolf 5400 rpm SATA 3 NAS HDD
- PSU: Corsair RM750x Gold Certified
- CPU Cooler: Peerless Assassin 120mm
- Case: Asus AP201 Mesh mATX
- Misc: Cat 6 Ethernet cable, be quiet! 120mm case fans

### Storage
- **RAID1 Array** (`/raid1/archive/`): Data redundancy.
- **NVMe Boot Drive**: Fast OS and Docker Compose startup; separates system files from media storage for better performance and easier maintenance.

### Backup Strategy
Manual periodic backups to external HDD and SSD.

### Orchestration & Management
- **Docker + Docker Compose** (`home/docker-lab`): Containerizes services (JellyFin, Pi-Hole, Prometheus, Grafana, Node Exporter) for easy deployment, isolation, and scaling.
- **Portainer**: Web UI for managing Docker containers, viewing logs, and monitoring resource usage without touching the command line.

### Containers
- Portainer
- Prometheus
- Grafana
- Node Exporter
- JellyFin
- Pi-Hole

### Media & Network Services
- **JellyFin**: Self-hosted media server for private streaming on local TV. Selected over Plex because it's open source and to avoid 3rd party dependencies.
- **Pi-Hole**: Ad-blocking DNS server. Filters ads and trackers for all devices on local network.

### Monitoring Stack
- **Node Exporter**: Exposes host-level metrics for Prometheus to scrape.
- **Prometheus**: Collects system metrics (CPU, memory, disk, etc.) from the host via Node Exporter.
- **Grafana**: Visualizes Prometheus metrics in dashboards. Lets you see at a glance how your server is performing and catch issues early.

![Grafana Prometheus Dashboard](/assets/Grafana_PrometheusDashboard.png)

Monitoring
- CPU usage, load, and temp
- NVMe temp and disk I/O
- Memory usage