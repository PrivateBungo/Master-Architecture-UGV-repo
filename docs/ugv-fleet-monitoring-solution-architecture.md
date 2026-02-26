# Monitoring Solution Architecture: Fluent Bit + Grafana Cloud

This document defines the technical implementation of the monitoring requirements established in [ugv-fleet-monitoring-architecture-draft.md](./ugv-fleet-monitoring-architecture-draft.md). 

## 1) Solution Overview

The solution uses a **Hybrid Edge-SaaS** model. Data is collected, filtered, and buffered locally on the UGV before being securely transmitted to a central operations plane.

- **Onboard Agent:** [Fluent Bit](https://fluentbit.io/) (C-based, high performance, low footprint).
- **Security Boundary:** [Tailscale](https://tailscale.com/) (Encrypted tunnel for all telemetry traffic).
- **Central Ops Platform:** [Grafana Cloud](https://grafana.com/products/cloud/) (Managed Loki for logs, Prometheus for metrics).

## 2) Components and Data Flow

1.  **Ingestion:** Fluent Bit sits on the Debian host and "sucks up" JSON logs from Docker containers and hardware stats from the Linux kernel.
2.  **Edge Pre-processing:** Fluent Bit filters logs by severity. `ERROR` and `WARNING` logs are tagged for high-priority shipping.
3.  **Buffering:** If the UGV is offline, Fluent Bit writes data to a dedicated partition on the SD card/SSD (up to a 24-hour buffer).
4.  **Transport:** Once the Tailnet is active, Fluent Bit uses the `loki` and `prometheus_remote_write` output plugins to ship data to Grafana Cloud.
5.  **Visualization:** The operator accesses a single Grafana Dashboard to view fleet health, correlate metric spikes with log errors, and receive alerts.

---

## 3) Implementation Roadmap (Work to be Done)

### 3.1 Host-Infra Repo (`Host-infra-for-UGV-controller`)
The host infrastructure owns the installation and lifecycle of the monitoring binary.

**Tasks:**
- **Ansible Role:** Create a `monitoring` role that adds the Fluent Bit APT repository and installs the `fluent-bit` package.
- **Systemd Service:** Configure Fluent Bit to run as a host-level systemd service.
- **Environment Override:** Create a systemd drop-in (`/etc/systemd/system/fluent-bit.service.d/override.conf`) that loads the robot's specific `.env` file (containing Grafana API keys) into the Fluent Bit process.
- **Symlink Logic:** Create a symlink from the local configuration repo to `/etc/fluent-bit/fluent-bit.conf` to allow for GitOps-driven config updates.

### 3.2 Local Environment Var Repo (`UGV-local-environment-variables`)
This repo owns the "personality" of the monitoring agent—what it collects and where it sends it.

**Tasks:**
- **Configuration File:** Add a `monitoring/fluent-bit.conf` file.
- **Inputs:** Define native inputs for `cpu`, `mem`, `thermal` (CPU temp), and `netif`.
- **Docker Tail:** Configure the `tail` input to read from `/var/lib/docker/containers/*/*.log`.
- **Outputs:** Configure the `loki` and `prometheus_remote_write` plugins using environment variables (e.g., `${GRAFANA_API_KEY}`) for endpoints and authentication.

### 3.3 Docker Repo (`Dockers-for-onboard-UGV-Controller`)
Containerized services must provide machine-readable logs to enable edge prioritization.

**Tasks:**
- **JSON Logging:** Update the `setup_logging()` function in Python services (`webrtc-service`, `uart-bridge`) to use a JSON formatter (e.g., `python-json-logger`).
- **Standard Criticality:** Ensure all log calls use standard levels (`logger.error`, `logger.warning`, `logger.info`).
- **Metadata:** Include the `service_name` in every log record to allow easy filtering in the central plane.

### 3.4 Central Ops Plane (Grafana Cloud)
This is the "home" for the fleet's data.

**Tasks:**
- **Data Sources:** Provision a Grafana Cloud API key with `MetricsPublisher` and `LogsPublisher` permissions.
- **Dashboards:** Use the **Grafana Assistant (AI)** to build an operations dashboard that highlights:
    - UGV Online/Offline status (Heartbeat).
    - Camera failure alerts (Log-based).
    - Thermal warnings (Metric-based).
    - Container crash-loop detection.
- **Alerting:** Define central alerting rules to notify the operator via Discord/Telegram if a critical mission failure is detected.

---

## 4) How logs "Find their way home"

1.  **Generation:** A Python service (e.g., `uart-bridge`) hits a failure and logs `{"level": "ERROR", "msg": "UART Timeout"}`.
2.  **Storage:** Docker writes this JSON line to a file on the Debian host.
3.  **Collection:** Fluent Bit (host-side) sees the new line, parses the JSON, and tags it as `critical_log`.
4.  **Buffering:** If the link is dead, Fluent Bit stores the `critical_log` in `/var/log/fluent-bit/buffer`.
5.  **Uplink:** When the Tailscale tunnel is up, Fluent Bit identifies the `critical_log` tag and ships it to the Loki endpoint first.
6.  **Ingestion:** Grafana Cloud Loki indexes the log with the UGV's hostname.
7.  **Alert:** An alert rule in Grafana Cloud triggers because an "ERROR" was received, and the operator sees the failure in the UI within seconds.