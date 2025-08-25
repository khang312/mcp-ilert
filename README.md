[![Releases](https://img.shields.io/badge/Releases-Download-blue?logo=github)](https://github.com/khang312/mcp-ilert/releases) https://github.com/khang312/mcp-ilert/releases

# MCP iLert Server — Lightweight Incident Bridge for iLert 🚨☁️

A compact MCP server that bridges incident events to iLert. It accepts incoming MCP events, enriches and normalizes payloads, and forwards them to iLert via webhook or API. Built for low CPU use, predictable latency, and easy deployment in containers or bare-metal.

![architecture](https://raw.githubusercontent.com/github/explore/main/topics/microservices/microservices.png)

Quick facts
- Protocols: MCP (custom), HTTP webhooks, REST API
- Targets: iLert incidents, alerts, schedules
- Runtime: Go binary with optional Docker image
- Observability: Prometheus metrics, structured logs

Table of contents
- Features
- Why use MCP iLert Server
- Architecture diagram
- Supported platforms
- Installation
- Quick start
- Configuration
- Runtime modes
- API and webhook mapping
- Metrics and logging
- Troubleshooting
- Contributing
- License

Features 🔧
- Receive MCP events over HTTP(S) or TCP.
- Normalize event payloads to iLert format.
- Authenticate to iLert using API token or webhook key.
- Map MCP priorities to iLert severity levels.
- Rate limit outgoing requests to avoid API throttling.
- Graceful shutdown and retry with exponential backoff.
- Systemd unit and Docker image for production use.
- Prometheus metrics endpoint for alerting and dashboards.

Why use this bridge
- Keep your existing MCP workflow.
- Send consistent alerts to iLert.
- Avoid building custom middleware.
- Run on edge devices or in the cloud.

Architecture diagram
- Inbound: MCP clients -> MCP iLert Server
- Processing: validation -> enrichment -> mapping
- Outbound: iLert API / iLert webhook

![flow](https://raw.githubusercontent.com/khang312/mcp-ilert/main/docs/images/flow.png)

Supported platforms
- Linux x86_64, ARM64
- macOS (for testing)
- Docker (amd64/arm64 multi-arch)
- Kubernetes (Helm chart support)

Installation 🚀

Download a release and run the included executable. Download the release file from the Releases page and execute it:

1) Visit the Releases page and pick an artifact for your platform:
https://github.com/khang312/mcp-ilert/releases

2) Example Linux install (replace version and file as needed):
```bash
# Example file name: mcp-ilert-linux-amd64.tar.gz
wget https://github.com/khang312/mcp-ilert/releases/download/v1.2.0/mcp-ilert-linux-amd64.tar.gz
tar xzf mcp-ilert-linux-amd64.tar.gz
chmod +x mcp-ilert
sudo mv mcp-ilert /usr/local/bin/
```

3) Example run:
```bash
mcp-ilert --config /etc/mcp-ilert/config.yaml
```

If you prefer Docker:
```bash
docker run -d \
  --name mcp-ilert \
  -p 8080:8080 \
  -v /etc/mcp-ilert:/etc/mcp-ilert \
  ghcr.io/khang312/mcp-ilert:latest
```

Releases link
- The main release page lists builds and assets. Download the file for your platform from:
https://github.com/khang312/mcp-ilert/releases
- Choose the binary or container image asset that matches your environment, then run or deploy it.

Quick start — minimal config
1) Create a config file /etc/mcp-ilert/config.yaml
```yaml
listen:
  address: 0.0.0.0
  port: 8080

ilert:
  api_key: "ILERT_API_KEY_GOES_HERE"
  api_url: "https://api.ilert.com"

mapping:
  priority_map:
    1: critical
    2: high
    3: medium
    4: low
```

2) Start the server:
```bash
mcp-ilert --config /etc/mcp-ilert/config.yaml
```

3) Send a sample MCP event:
```bash
curl -X POST http://localhost:8080/events \
  -H "Content-Type: application/json" \
  -d '{
    "mcp_id":"abc-123",
    "source":"monitoring-ng",
    "priority":2,
    "summary":"CPU usage above threshold",
    "details":"host: web-01, cpu: 92%"
  }'
```

4) The server will map priority 2 to iLert severity "high" and create/update an incident in iLert.

Configuration — fields and examples
- listen.address: interface to bind.
- listen.port: HTTP port.
- tls.cert_file: path to TLS cert (optional).
- tls.key_file: path to TLS key (optional).
- ilert.api_key: API key to authenticate to iLert.
- ilert.api_url: iLert endpoint.
- mapping.priority_map: map MCP numeric priority to iLert severity.
- rate_limit.per_minute: max outgoing requests per minute.
- retries.max_attempts: retry count for failed deliveries.
- logging.level: info, debug, error.
- metrics.enabled: expose Prometheus metrics endpoint.

Example full config (comments removed in YAML):
```yaml
listen:
  address: 0.0.0.0
  port: 8080

tls:
  cert_file: /etc/ssl/certs/mcp.pem
  key_file: /etc/ssl/private/mcp.key

ilert:
  api_key: "XXXXXXXX-XXXXXXXX-XXXXXXXX"
  api_url: "https://api.ilert.com"
  webhook_key: "" # optional fallback

mapping:
  priority_map:
    1: critical
    2: high
    3: medium
    4: low

rate_limit:
  per_minute: 120

retries:
  max_attempts: 5
  base_interval_seconds: 2

logging:
  level: info

metrics:
  enabled: true
  listen_address: 0.0.0.0
  listen_port: 9090
```

Runtime modes
- Standalone: Run the binary on a host or VM.
- Container: Use the Docker image with mounted config.
- Kubernetes: Use Helm or plain manifests. Expose service and create a ClusterIP for internal traffic.
- Systemd: Use the included unit file to run as a service.

Example systemd unit:
```ini
[Unit]
Description=MCP iLert Server
After=network.target

[Service]
ExecStart=/usr/local/bin/mcp-ilert --config /etc/mcp-ilert/config.yaml
Restart=on-failure
User=mcp
Group=mcp

[Install]
WantedBy=multi-user.target
```

API and webhook mapping
- Endpoint: POST /events
- Headers: Content-Type: application/json
- Sample MCP payload:
```json
{
  "mcp_id": "evt-0001",
  "source": "probe-1",
  "priority": 1,
  "summary": "Disk full",
  "details": "root disk at 98%",
  "tags": ["disk", "storage"]
}
```
- The server validates the payload, maps priority, and posts to iLert.
- Outbound call uses iLert REST API or webhook URL. Retry on 5xx. Drop on repeated 4xx.

Metrics and logging
- Metrics endpoint (Prometheus): /metrics (default port 9090)
- Key metrics:
  - mcp_in_events_total
  - mcp_out_success_total
  - mcp_out_failed_total
  - mcp_out_retries_total
  - mcp_processing_latency_seconds
- Logs: JSON structured logs by default. Use logging.level to adjust verbosity.

Troubleshooting
- App fails to start
  - Check config path and YAML syntax.
  - Verify file permissions for TLS certs and keys.
- No outgoing requests to iLert
  - Verify ilert.api_key and ilert.api_url values.
  - Check rate_limit settings.
- Events dropped
  - Inspect mcp_out_failed_total metric.
  - Increase retries.max_attempts or check iLert API limits.
- Metrics not visible
  - Confirm metrics.enabled is true.
  - Ensure firewall rules allow access to metrics port.

Security
- Use TLS for inbound MCP traffic.
- Rotate ilert.api_key on a schedule.
- Run the service under an unprivileged account.
- Restrict access to metrics and admin endpoints.

Operational tips
- Use systemd or container orchestrator for restart and updates.
- Monitor mcp_processing_latency_seconds to detect slowdowns.
- Use Prometheus alerting rules to notify on high failure rates.
- Keep rate_limit below your iLert plan quota.

Contributing 🤝
- Fork the repo and open a pull request.
- Keep changes small and focused.
- Include tests for core logic: mapping, retry, and auth.
- Follow the repository code style and run `go fmt` and `go vet`.
- Add or update docs in /docs for new features.

Release process
- Tag a semantic version: vMAJOR.MINOR.PATCH
- Build artifacts: linux, mac, windows, docker images
- Push release assets to the Releases page:
https://github.com/khang312/mcp-ilert/releases

Links and resources
- Releases: https://github.com/khang312/mcp-ilert/releases
- iLert API docs: https://docs.ilert.com (use real link in your environment)
- Prometheus docs: https://prometheus.io

Contact
- Open issues in the repository for bugs and feature requests.
- Use PR comments for implementation discussion.

License
- MIT License. Check LICENSE file in the repository for full terms.