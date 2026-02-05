# Monitoring Agent

This directory contains the shared monitoring agent configuration for deployment to all VPS nodes in the All Things Linux infrastructure.

## Components
- **Grafana Alloy**: Telemetry collector (Logs & Metrics).
- **Node Exporter**: Host hardware and OS metrics.
- **cAdvisor**: Docker container metrics.

## Deployment Instructions

1.  **Copy this directory** to the target server (e.g., `/opt/monitoring`).
   ```bash
   scp -r metrics/agent user@target-server:/opt/monitoring
   ```

2.  **Create an `.env` file** in the directory:
   ```bash
   HOSTNAME=target-server-name
   # Optional: If auth is enabled on the central stack
   # BASIC_AUTH_USER=
   # BASIC_AUTH_PASS=
   ```

3.  **Start the stack**:
   ```bash
   docker compose up -d
   ```

## Configuration
- `config.alloy` scans for `node-exporter:9100` and `cadvisor:8080`.
- It collects all Docker container logs via the unix socket.
- Data is pushed to:
  - Metrics: `https://metrics.atl.services/api/v1/push`
  - Logs: `https://metrics.atl.services/loki/api/v1/push`
