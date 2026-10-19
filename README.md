# Project 5.3 – Advanced Monitoring, Alerting, and Distributed Tracing

Implement an observability solution using **Prometheus, recording rules, Alertmanager, Slack notifications, Jaeger, and the HotROD demo application**.

This guide uses the configuration files already included in this repository. **Do not recreate `prometheus.yml`, `alerts.yml`, or `recording_rules.yml`.** Review and reuse them, changing only environment-specific values when needed.

## 1. Project Overview

The objective is to collect metrics, pre-calculate useful PromQL expressions, detect infrastructure issues, deliver notifications, and investigate distributed traces.

**Technologies:** Prometheus, Node Exporter, cAdvisor, Alertmanager, Slack, Jaeger, HotROD, Docker, and Google Cloud Platform.

## 2. Problem Statement

Without centralized monitoring and observability, teams can struggle to identify unavailable services, detect sustained high CPU utilization, receive timely alerts, and trace slow requests across application components. This project combines metrics, recording rules, alerting, notifications, and distributed tracing to address these challenges.

## 3. Architecture

```text
                     Google Cloud Platform
                   ┌────────────────────────┐
                   │      monitoring-vm     │
                   │ Prometheus             │
                   │ Alertmanager           │
                   │ Jaeger                 │
                   └───────────┬────────────┘
                               │
                    Metrics    │    Traces (OTLP)
                               │
                   ┌───────────▼────────────┐
                   │       target-vm        │
                   │ Node Exporter           │
                   │ HotROD                  │
                   └────────────────────────┘

Node Exporter / cAdvisor → Prometheus → Alertmanager → Slack
HotROD → Jaeger → Trace visualization
```

Use private VM-to-VM communication wherever possible. Expose user interfaces only to trusted source IP addresses.

## 4. Prerequisites

- A Google Cloud project with permission to manage Compute Engine VMs and firewall rules.
- Two Ubuntu VMs or equivalent Linux hosts:
  - `monitoring-vm`: Prometheus, Alertmanager, and Jaeger.
  - `target-vm`: Node Exporter and HotROD.
- Docker installed and running on the relevant VMs.
- SSH access, a Slack workspace, and permission to create an incoming webhook.
- Basic Linux, Docker, YAML, PromQL, and GCP networking knowledge.

Example VM sizes used during this assignment are `e2-standard-2` for monitoring and `e2-medium` for the target. Adjust them to your needs.

## 5. Clone the Repository and Reuse Existing Configuration

On your workstation:

```bash
git clone https://github.com/SnehalShinde11/Advanced-Monitoring-pro-graph.git
cd Advanced-Monitoring-pro-graph
ls -la
```

The repository already contains these configuration files:

| File | Purpose |
|---|---|
| `prometheus.yml` | Scrape jobs, alertmanager connection, and rule-file references |
| `alerts.yml` | `InstanceDown` and `HighCPUUsage` alert rules |
| `recording_rules.yml` | CPU, memory, and network recording rules |
| `prometheus.yml.backup` | Backup copy of the Prometheus configuration |

Inspect the files before deployment:

```bash
cat prometheus.yml
cat alerts.yml
cat recording_rules.yml
```

### Copy the existing files to monitoring-vm

From your workstation, replace `MONITORING_VM_EXTERNAL_IP` with the current external IP address:

```bash
ssh USERNAME@MONITORING_VM_EXTERNAL_IP 'sudo mkdir -p /opt/prometheus-5.3'
scp prometheus.yml alerts.yml recording_rules.yml USERNAME@MONITORING_VM_EXTERNAL_IP:/tmp/
ssh USERNAME@MONITORING_VM_EXTERNAL_IP
sudo mv /tmp/prometheus.yml /tmp/alerts.yml /tmp/recording_rules.yml /opt/prometheus-5.3/
sudo chown root:root /opt/prometheus-5.3/*.yml
sudo chmod 644 /opt/prometheus-5.3/*.yml
ls -l /opt/prometheus-5.3
```

Replace `USERNAME` with your SSH username. If the VM cannot accept SCP to `/tmp`, clone the repository on the VM and copy the files locally instead.

**Do not create replacement versions of these three files.** Make edits to the existing files only when a setting needs to match your environment. In particular, confirm the Node Exporter address in `prometheus.yml` matches the target VM's private IP. The committed example uses `10.128.0.29:9100`.

The existing `prometheus.yml` already references both rule files and configures Alertmanager at `alertmanager:9093`. It also includes a Jaeger scrape job at `jaeger:8888`; verify that this endpoint is supported by the Jaeger version you deploy before expecting that target to be UP.

Keep credentials and webhook URLs out of Git.

## 6. Task 1 – Validate Prometheus Recording Rules

The existing `recording_rules.yml` defines four recording rules:
- `instance:cpu_usage_percent:10m`
- `instance:memory_usage_percent`
- `instance:network_receive_bytes_per_second`
- `instance:network_transmit_bytes_per_second`

On `monitoring-vm`, validate the existing file:

```bash
sudo docker run --rm \
  --entrypoint /bin/promtool \
  -v /opt/prometheus-5.3/recording_rules.yml:/etc/prometheus/recording_rules.yml:ro \
  prom/prometheus \
  check rules /etc/prometheus/recording_rules.yml
```

Expected result: `SUCCESS: 4 rules found`.

The CPU expression relies on Node Exporter metrics. If the query returns no data, check that the Node Exporter target is reachable and has been scraped long enough to provide a 10-minute rate window.

## 7. Task 2 – Validate Alerting Rules and Start Prometheus

The existing `alerts.yml` defines:

| Alert | Condition | Pending period |
|---|---|---|
| `InstanceDown` | `up == 0` | 1 minute |
| `HighCPUUsage` | `instance:cpu_usage_percent:10m > 80` | 5 minutes |

Validate the existing alert rules:

```bash
sudo docker run --rm \
  --entrypoint /bin/promtool \
  -v /opt/prometheus-5.3/alerts.yml:/etc/prometheus/alerts.yml:ro \
  prom/prometheus \
  check rules /etc/prometheus/alerts.yml
```

Validate the existing Prometheus configuration and both rule files:

```bash
sudo docker run --rm \
  --entrypoint /bin/promtool \
  -v /opt/prometheus-5.3/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
  -v /opt/prometheus-5.3/alerts.yml:/etc/prometheus/alerts.yml:ro \
  -v /opt/prometheus-5.3/recording_rules.yml:/etc/prometheus/recording_rules.yml:ro \
  prom/prometheus \
  check config /etc/prometheus/prometheus.yml
```

Resolve any validation errors before deployment. Confirm the existing Docker network named `monitoring` exists and that your Prometheus, Alertmanager, and cAdvisor container names/network configuration are consistent.

If Prometheus is not already running, start it using the repository files:

```bash
sudo docker network inspect monitoring >/dev/null 2>&1 || sudo docker network create monitoring

sudo docker run -d \
  --name prometheus \
  --restart unless-stopped \
  --network monitoring \
  -p 9090:9090 \
  -v /opt/prometheus-5.3/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
  -v /opt/prometheus-5.3/alerts.yml:/etc/prometheus/alerts.yml:ro \
  -v /opt/prometheus-5.3/recording_rules.yml:/etc/prometheus/recording_rules.yml:ro \
  prom/prometheus
```

If a container named `prometheus` already exists, inspect it first with `sudo docker ps -a`. Do not remove an existing container without confirming its configuration and data requirements. If you update the mounted host configuration, restart Prometheus to reload it.

Verify Prometheus and the rules:

```bash
curl http://localhost:9090/-/healthy
curl -s http://localhost:9090/api/v1/rules | python3 -m json.tool
curl -G -s http://localhost:9090/api/v1/query \
  --data-urlencode 'query=instance:cpu_usage_percent:10m' \
  | python3 -m json.tool
```

Open `http://MONITORING_VM_EXTERNAL_IP:9090` to review **Status → Targets** and **Alerts**. Restrict firewall access to trusted IPs.

## 8. Task 3 – Configure Alertmanager and Slack Notifications

The repository already configures Prometheus to send alerts to `alertmanager:9093`. You only need to create the runtime-specific Alertmanager configuration because it contains environment-specific routing and Slack integration details.

On `monitoring-vm`:

```bash
sudo mkdir -p /opt/alertmanager-5.3
sudo vi /opt/alertmanager-5.3/alertmanager.yml
```

Configure a default route and receiver. For Slack, use a receiver with `api_url_file` pointing to a mounted secret file; do not hardcode the webhook in the repository. The secret file must contain only the webhook URL.

Create a Slack incoming webhook:

1. Visit [Slack API app management](https://api.slack.com/apps).
2. Create or select an app for the intended workspace.
3. Enable **Incoming Webhooks**.
4. Add a webhook to the desired channel and copy its URL.
5. Treat the URL like a password. Never commit it or publish it in screenshots/logs.

Store the URL on the VM in a protected file:

```bash
sudo sh -c 'umask 077; printf "%s" "PASTE_YOUR_SLACK_WEBHOOK_URL_HERE" > /opt/alertmanager-5.3/slack_webhook_url'
sudo chmod 600 /opt/alertmanager-5.3/slack_webhook_url
```

Replace the placeholder locally with your actual webhook URL. Do not include quote characters or a trailing newline in the secret file. Configure the Slack receiver in `alertmanager.yml` to use:

```yaml
api_url_file: /etc/alertmanager/secrets/slack_webhook_url
```

Ensure the route references the Slack receiver and the configuration uses valid YAML. Validate it:

```bash
sudo docker run --rm \
  --entrypoint amtool \
  -v /opt/alertmanager-5.3/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro \
  -v /opt/alertmanager-5.3/slack_webhook_url:/etc/alertmanager/secrets/slack_webhook_url:ro \
  prom/alertmanager \
  check-config /etc/alertmanager/alertmanager.yml
```

Start Alertmanager on the same Docker network as Prometheus:

```bash
sudo docker run -d \
  --name alertmanager \
  --restart unless-stopped \
  --network monitoring \
  -p 9093:9093 \
  -v /opt/alertmanager-5.3/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro \
  -v /opt/alertmanager-5.3/slack_webhook_url:/etc/alertmanager/secrets/slack_webhook_url:ro \
  prom/alertmanager
```

If an Alertmanager container already exists, inspect it and recreate it only if necessary to mount the configuration and secret. Verify:

```bash
curl http://localhost:9093/-/healthy
curl -s http://localhost:9090/api/v1/alertmanagers | python3 -m json.tool
```

### Test the notification pipeline

On `target-vm`, temporarily stop Node Exporter (confirm the container name first):

```bash
sudo docker ps
sudo docker stop node-exporter
```

Wait for the scrape interval plus the alert's `for: 1m` duration (allow around 70–120 seconds). On `monitoring-vm`, inspect alerts:

```bash
curl -s http://localhost:9090/api/v1/alerts | python3 -m json.tool
curl -s http://localhost:9093/api/v2/alerts | python3 -m json.tool
sudo docker logs alertmanager --tail 100
```

Confirm `InstanceDown` fires and a Slack message reaches the configured channel. Restore Node Exporter immediately after the test:

```bash
sudo docker start node-exporter
```

If no notification arrives, inspect the alert state, Alertmanager route/receiver, container logs, webhook file permissions, and webhook URL validity. Do not print the webhook file contents to diagnose it.

## 9. Task 4 – Deploy Jaeger

On `monitoring-vm`, confirm the Docker network exists:

```bash
sudo docker network inspect monitoring
```

Run the Jaeger all-in-one image using a version compatible with the HotROD OTLP configuration:

```bash
sudo docker run -d \
  --name jaeger \
  --restart unless-stopped \
  --network monitoring \
  -p 16686:16686 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest
```

Check the container:

```bash
sudo docker ps | grep jaeger
sudo docker logs jaeger --tail 50
```

Port `16686` is the Jaeger UI and `4318` is the OTLP HTTP receiver. Do not expose OTLP ingestion to the public internet.

To access the UI, create a GCP ingress firewall rule for TCP `16686`, targeting the `jaeger-ui` network tag and allowing only your workstation's public IP with a `/32` range. Add the `jaeger-ui` tag to `monitoring-vm`. Open:

```text
http://MONITORING_VM_EXTERNAL_IP:16686
```

Use the current external IP and keep access restricted.

## 10. Task 5 – Run HotROD and Generate Traces

On `target-vm`, verify Docker is available and test access to Jaeger using the monitoring VM's private IP:

```bash
curl -sS -o /dev/null -w "HTTP status: %{http_code}\\n" \
  http://MONITORING_VM_INTERNAL_IP:4318/
```

An HTTP response, including `404`, indicates the endpoint is reachable. A timeout or connection refusal requires checking the private IP, container port binding, VPC firewall, and routing. Allow TCP `4318` from the target VM's private IP (`/32`) to the monitoring VM; do not open it publicly.

Run HotROD on `target-vm`, replacing the placeholder with the actual private IP of `monitoring-vm`:

```bash
sudo docker run -d \
  --name hotrod \
  --restart unless-stopped \
  -p 8080:8080 \
  -e OTEL_EXPORTER_OTLP_ENDPOINT=http://MONITORING_VM_INTERNAL_IP:4318 \
  jaegertracing/example-hotrod:latest \
  all
```

Check the container and local endpoint:

```bash
sudo docker ps | grep hotrod
curl -sS -o /dev/null -w "HTTP status: %{http_code}\\n" http://localhost:8080/
```

To access HotROD remotely, create a GCP ingress firewall rule for TCP `8080`, targeting the `hotrod-ui` tag and restricted to your workstation's public IP followed by `/32`. Ensure the target VM has the tag and that you use the correct VPC network. Open:

```text
http://TARGET_VM_EXTERNAL_IP:8080
```

Generate sample requests on `target-vm`:

```bash
for i in {1..5}; do
  curl -s -o /dev/null http://localhost:8080/
done
```

## 11. Task 6 – Analyze Distributed Traces

On `monitoring-vm`, check whether Jaeger has received traces:

```bash
curl -s http://localhost:16686/api/v3/services
```

In the Jaeger UI:

1. Select `frontend` from the **Service** dropdown.
2. Click **Find Traces** and open a recent trace.
3. Inspect the waterfall/timeline and compare span durations.
4. Identify slow operations and review span attributes and parent-child relationships.

If traces do not appear, generate more HotROD requests and check the application logs, OTLP endpoint, private network/firewall rules, and Jaeger logs.

## 12. Beyond the Basics – Scrape Jaeger Metrics with Prometheus

The existing `prometheus.yml` already includes a Jaeger scrape job targeting `jaeger:8888/metrics`. **Do not add a duplicate job.** First confirm that the Jaeger version you deployed exposes metrics at this address and that the Jaeger container is connected to the `monitoring` Docker network.

Inspect the existing job and back up the file before any edit:

```bash
sudo cat /opt/prometheus-5.3/prometheus.yml
sudo cp /opt/prometheus-5.3/prometheus.yml /opt/prometheus-5.3/prometheus.yml.backup
```

If your Jaeger version exposes metrics at a different port or path, edit the existing `jaeger` job in the repository's `prometheus.yml` and copy the updated file to the VM. Keep the other scrape jobs, rule file references, and Alertmanager configuration. Then validate and restart Prometheus:

```bash
sudo docker exec prometheus promtool check config /etc/prometheus/prometheus.yml
sudo docker restart prometheus
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool
```

Check **Prometheus → Status → Targets**. The Jaeger target should be `UP` only when the configured endpoint is actually served and reachable. If it is `DOWN`, verify the metrics endpoint for your specific Jaeger version and Docker network connectivity.

## 13. Validation and Troubleshooting

| Component | Validation | Expected result |
|---|---|---|
| Prometheus | `curl http://localhost:9090/-/healthy` | Healthy response |
| Recording rules | `promtool check rules` and Prometheus Rules UI | Four rules loaded and healthy |
| CPU recording metric | Query `instance:cpu_usage_percent:10m` | Data appears after sufficient samples are collected |
| Alerting rules | Prometheus Rules UI or `/api/v1/rules` | `InstanceDown` and `HighCPUUsage` loaded |
| Alertmanager | `curl http://localhost:9093/-/healthy` | Healthy response |
| Alert delivery | `/api/v1/alertmanagers` | Prometheus discovers Alertmanager |
| Slack | Temporarily stop Node Exporter | Alert message reaches the configured channel |
| Node Exporter | Query `up{job="node-exporter"}` | Value returns to `1` after recovery |
| Jaeger | Jaeger UI and HotROD requests | Services and traces visible |
| HotROD | `curl http://localhost:8080/` | Application responds |
| Jaeger metrics | Prometheus Targets page | Target is UP if the configured endpoint is supported |

Common issues:

- **Rules are missing:** Check `rule_files`, mounted file paths, YAML indentation, and Prometheus logs.
- **Target is down:** Verify Node Exporter, its private IP, scrape configuration, and firewall rules.
- **Alertmanager is not connected:** Confirm the containers share the `monitoring` network and Prometheus targets `alertmanager:9093`.
- **Slack notification fails:** Inspect Alertmanager logs, route/receiver configuration, secret file path, and permissions without exposing the webhook.
- **Jaeger UI does not load:** Verify the container, port `16686`, target network tag, firewall source range, and external IP.
- **No traces appear:** Check HotROD requests, OTLP port `4318`, private connectivity, and Jaeger logs.
- **Jaeger target is down:** Verify the metrics endpoint supported by your deployed Jaeger version; do not assume port `8888` exists in every release.

## 14. Assignment Details

**Assignment:** Project 5.3 – Advanced Monitoring, Alerting, and Distributed Tracing

**Submitted by:** Snehal Shinde
