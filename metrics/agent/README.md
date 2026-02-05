# ATL Monitoring Agent

Shared monitoring agent configuration for deploying to remote VPS nodes. This agent collects host metrics, container metrics, and Docker logs, then forwards them to the central metrics stack.

## Components

- **[Grafana Alloy](https://grafana.com/docs/alloy/latest/)**: Unified telemetry collector (logs + metrics)
- **[Node Exporter](https://github.com/prometheus/node_exporter)**: Host-level metrics (CPU, memory, disk, network)
- **[cAdvisor](https://github.com/google/cadvisor)**: Container-level metrics (CPU throttling, memory, OOM kills)

## What It Collects

### Metrics (forwarded to Mimir)
- **Host Metrics**: CPU, memory, disk, network, load average from Node Exporter
- **Container Metrics**: Per-container resource usage, throttling, OOM events from cAdvisor

### Logs (forwarded to Loki)
- **Docker Logs**: All container logs with automatic labeling (`container`, `instance`, `job`)

## Deployment

### 1. Copy Agent to VPS

```bash
scp -r agent/ user@vps:/opt/monitoring/
cd /opt/monitoring/agent
```

### 2. Configure Environment Variables

Create a `.env` file with the following variables:

```bash
# Required: Unique hostname identifier for this VPS
HOSTNAME=atl-chat

# Required: Central Loki endpoint (Tailscale IP recommended)
LOKI_URL=http://100.64.2.0:3100/loki/api/v1/push

# Required: Central Mimir endpoint (Tailscale IP recommended)
REMOTE_WRITE_URL=http://100.64.2.0:8080/api/v1/push

# Optional: Basic auth credentials (if central stack requires authentication)
# BASIC_AUTH_USER=your-username
# BASIC_AUTH_PASS=your-password
```

**Environment Variable Reference:**

| Variable | Required | Description | Example |
|----------|----------|-------------|----------|
| `HOSTNAME` | ✅ | Unique identifier for this host | `atl-chat`, `atl-network` |
| `LOKI_URL` | ✅ | Central Loki push endpoint | `http://100.64.2.0:3100/loki/api/v1/push` |
| `REMOTE_WRITE_URL` | ✅ | Central Mimir remote write endpoint | `http://100.64.2.0:8080/api/v1/push` |
| `BASIC_AUTH_USER` | ❌ | Basic auth username (if needed) | `metrics-agent` |
| `BASIC_AUTH_PASS` | ❌ | Basic auth password (if needed) | `your-secure-password` |

### 3. Deploy the Stack

```bash
docker compose up -d
```

### 4. Verify Deployment

```bash
# Check container status
docker compose ps

# View Alloy logs
docker compose logs -f alloy

# Check metrics are being scraped
curl http://localhost:9100/metrics  # Node Exporter
curl http://localhost:8080/metrics  # cAdvisor
```

### 5. Access Live Debugging UI

Alloy includes a built-in web UI for real-time debugging and monitoring:

```bash
# Access the Alloy UI (from the VPS or via SSH tunnel)
http://localhost:12345
```

**Features:**
- Real-time component status and health
- Live data streaming from each pipeline stage
- Configuration validation and syntax checking
- Metrics about Alloy's own performance

> **Note**: The live debugging UI is enabled by default in our configuration. To access it remotely, use SSH port forwarding:
> ```bash
> ssh -L 12345:localhost:12345 user@vps-hostname
> ```

## Troubleshooting

### Logs Not Appearing in Loki

1. **Check Alloy logs**: `docker compose logs alloy`
2. **Verify LOKI_URL**: Ensure the central Loki endpoint is reachable
3. **Test connectivity**: `curl -v $LOKI_URL`
4. **Check firewall**: Ensure port 3100 is accessible from this VPS

### Metrics Not Appearing in Mimir

1. **Check Alloy logs**: `docker compose logs alloy`
2. **Verify REMOTE_WRITE_URL**: Ensure the central Mimir endpoint is reachable
3. **Test connectivity**: `curl -v $REMOTE_WRITE_URL`
4. **Check firewall**: Ensure port 8080 is accessible from this VPS

### Alloy Configuration Errors

1. **Validate config syntax**:
   ```bash
   docker compose exec alloy alloy fmt config.alloy
   ```
2. **Check for missing environment variables**:
   ```bash
   docker compose config
   ```

### Container Won't Start

1. **Check Docker socket permissions**: Ensure `/var/run/docker.sock` is accessible
   ```bash
   # The Alloy container needs read access to the Docker socket
   ls -la /var/run/docker.sock
   # Should show: srw-rw---- 1 root docker
   
   # If running Alloy as non-root, ensure the user is in the docker group
   # OR run Alloy as root (as configured in our compose.yaml)
   ```
2. **Review compose logs**: `docker compose logs`
3. **Verify .env file**: Ensure all required variables are set

> **Security Note**: Our `compose.yaml` runs Alloy as `root` (UID 0) to access the Docker socket. This is required for `loki.source.docker` to read container logs. In production, ensure the VPS is properly secured and Alloy only has access to necessary resources.

## Architecture

```
┌─────────────────────────────────────────┐
│           Remote VPS Host               │
│                                         │
│  ┌──────────────────────────────────┐  │
│  │   Docker Containers              │  │
│  │   (app, db, etc.)                │  │
│  └──────────┬───────────────────────┘  │
│             │ logs                     │
│             ▼                          │
│  ┌──────────────────────────────────┐  │
│  │   Grafana Alloy                  │  │
│  │   - Collects Docker logs         │  │
│  │   - Scrapes Node Exporter        │  │
│  │   - Scrapes cAdvisor             │  │
│  └──────────┬───────────────────────┘  │
│             │                          │
│  ┌──────────┴───────────────────────┐  │
│  │   Node Exporter   │   cAdvisor   │  │
│  │   (host metrics)  │   (container)│  │
│  └──────────────────────────────────┘  │
└─────────────┬───────────────────────────┘
              │ Tailscale VPN
              ▼
┌─────────────────────────────────────────┐
│      Central Metrics Stack              │
│      (atl.services)                     │
│                                         │
│  ┌──────────────┐   ┌──────────────┐   │
│  │    Loki      │   │    Mimir     │   │
│  │   (logs)     │   │  (metrics)   │  │
│  └──────────────┘   └──────────────┘   │
│           │                 │           │
│           └────────┬────────┘           │
│                    ▼                    │
│           ┌──────────────┐              │
│           │   Grafana    │              │
│           │ (dashboards) │              │
│           └──────────────┘              │
└─────────────────────────────────────────┘
```

## References

- [Grafana Alloy Documentation](https://grafana.com/docs/alloy/latest/)
- [Collect Prometheus Metrics](https://grafana.com/docs/alloy/latest/collect/prometheus-metrics/)
- [Collect Logs with Loki](https://grafana.com/docs/alloy/latest/reference/components/loki/)
- [Node Exporter Guide](https://prometheus.io/docs/guides/node-exporter/)
- [cAdvisor Documentation](https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md)

## Learning Resources

If you're new to Grafana Alloy or want to learn more about the components used in this configuration, check out these official tutorials:

### Getting Started
- [Send Logs to Loki](https://grafana.com/docs/alloy/latest/tutorials/send-logs-to-loki/) - Learn the basics of log collection
- [First Components and Standard Library](https://grafana.com/docs/alloy/latest/tutorials/first-components-and-stdlib/) - Understand component basics
- [Logs and Relabeling Basics](https://grafana.com/docs/alloy/latest/tutorials/logs-and-relabeling-basics/) - Master relabeling techniques

### Monitoring Examples
- [Monitor Docker Containers](https://grafana.com/docs/alloy/latest/monitor/monitor-docker-containers/) - Docker metrics and logs collection
- [Monitor Linux Servers](https://grafana.com/docs/alloy/latest/monitor/monitor-linux/) - Host-level monitoring with Node Exporter
- [Monitor Logs from Files](https://grafana.com/docs/alloy/latest/monitor/monitor-logs-from-file/) - File-based log collection

### Advanced Topics
- [Process Logs](https://grafana.com/docs/alloy/latest/tutorials/processing-logs/) - Advanced log processing and filtering
- [Monitor Structured Logs](https://grafana.com/docs/alloy/latest/monitor/monitor-structured-logs/) - Working with JSON logs

## Configuration
- `config.alloy` scans for `node-exporter:9100` and `cadvisor:8080`.
- It collects all Docker container logs via the unix socket.
- Data is pushed to:
  - Metrics: `https://metrics.atl.services/api/v1/push`
  - Logs: `https://metrics.atl.services/loki/api/v1/push`
