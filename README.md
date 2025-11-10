# Monitoring Setup — Prometheus & Grafana

This repository contains configuration and deployment files for setting up **Prometheus** and **Grafana** using **Docker** and **Docker Compose**.

The setup provides **real-time monitoring and visualization** for your microservices or Kubernetes workloads.

---

## Components Overview

### Prometheus
- Collects metrics from configured targets (applications, exporters, etc.)
- Stores metrics in a time-series database
- Exposes metrics via the `/metrics` endpoint

### Grafana
- Connects to Prometheus as a data source
- Provides dashboards for metrics visualization and alerting

---
