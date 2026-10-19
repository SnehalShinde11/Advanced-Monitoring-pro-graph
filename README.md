# Project 5.3 – Advanced Monitoring, Alerting, and Distributed Tracing

Implement a monitoring and observability solution using **Prometheus, Prometheus Recording Rules, Alertmanager, Slack notifications, Jaeger, and the HotROD demo application**.

This guide is designed for learners using the existing repository. Clone the repository, follow the tasks in order, validate each component, and capture evidence of your implementation.

## 1. Project Overview

The objective of this project is to build an observability solution that collects metrics, evaluates alerts, delivers notifications, and visualizes distributed traces.

**Technologies used**

- **Prometheus:** Metrics collection, querying, recording rules, and alert evaluation.
- **Node Exporter:** Host-level metrics collection.
- **Alertmanager:** Alert grouping, routing, and notification management.
- **Slack:** Alert notification destination.
- **Jaeger:** Distributed tracing and trace visualization.
- **HotROD:** Sample application that generates distributed traces.
- **Docker:** Container deployment and networking.
- **Google Cloud Platform:** Virtual machines and firewall configuration.

## 2. Problem Statement

Without a centralized monitoring and observability solution, teams may struggle to identify unavailable services, investigate high resource utilization, receive timely alerts, and trace slow requests across application components.

This project addresses these challenges by implementing recording rules for frequently queried metrics, alerting rules for service availability and CPU utilization, Alertmanager notifications, and distributed tracing with Jaeger.

## 3. Architecture

```text
                    Google Cloud Platform
                   ┌─────────────────────┐
                   │    monitoring-vm    │
                   │                     │
                   │     Prometheus      │
                   │     Alertmanager    │
                   │       Jaeger        │
                   └──────────┬──────────┘
                              │
                  Metrics     │     Traces (OTLP)
                              │
                   ┌──────────▼──────────┐
                   │      target-vm      │
                   │                     │
                   │    Node Exporter    │
                   │       HotROD        │
                   └─────────────────────┘

Prometheus → Alertmanager → Slack
HotROD → Jaeger → Trace visualization
Node Exporter → Prometheus → Metrics and alerts
```

The architecture separates monitoring services from the target application and metrics exporter. Use private VM-to-VM communication wherever possible.

## 4. Prerequisites

Before starting, ensure you have:

- A Google Cloud project with permission to create Compute Engine VMs and firewall rules.
- Two Ubuntu virtual machines, or equivalent Linux hosts:
  - `monitoring-vm`: Prometheus, Alertmanager, and Jaeger.
  - `target-vm`: Node Exporter and HotROD.
- Docker installed and running on both machines.
- SSH access to both machines.
- A Slack workspace where you can configure an incoming webhook.
- Access to the existing GitHub repository.
- Basic knowledge of Linux, Docker, YAML, PromQL, and GCP networking.

Example VM specifications used in this assignment:

| VM | Example machine type | Operating system |
|---|---|---|
| monitoring-vm | e2-standard-2 | Ubuntu 24.04 |
| target-vm | e2-medium | Ubuntu 24.04 |

Actual machine types can be adjusted to your available resources.

## 5. Clone the Repository

Clone the existing project repository onto your workstation:

```bash
git clone https://github.com/SnehalShinde11/Advanced-Monitoring-pro-graph.git
cd Advanced-Monitoring-pro-graph
```

Review the existing files and instructions before making changes:

```bash
git status
git branch
ls -la
```

Follow the repository's existing structure. Do not create a separate repository for this assignment.

**Important:** The Prometheus and Alertmanager configuration directories used on the monitoring VM are `/opt/prometheus-5.3` and `/opt/alertmanager-5.3`. Keep credentials and other secrets out of Git.

## 6. Task 1 – Implement Prometheus Recording Rules

### Objective

Create recording rules to pre-calculate frequently used or computationally expensive PromQL expressions, such as CPU utilization over a 10-minute window.

### Step 1: Prepare the Prometheus configuration directory

On `monitoring-vm`:

```bash
sudo mkdir -p /opt/prometheus-5.3
sudo ls -la /opt/prometheus-5.3
```

Create or update the Prometheus configuration file:

```bash
sudo vi /opt/prometheus-5.3/prometheus.yml
```

Ensure that the configuration includes the appropriate scrape jobs, rule file paths, and target addresses for your environment.

### Step 2: Create recording rules

Create the recording rules file:

```bash
sudo vi /opt/prometheus-5.3/recording_rules.yml
```

Define four recording rules for frequently used metrics. The rules should include the CPU utilization expression used in the later verification step:

`instance:cpu_usage_percent:10m`

Ensure that the expressions match the metric names and labels exposed by your Node Exporter targets.

### Step 3: Validate recording rules

```bash
sudo docker run --rm \
  --entrypoint /bin/promtool \
  -v /opt/prometheus-5.3/recording_rules.yml:/etc/prometheus/recording_rules.yml:ro \
  prom/prometheus \
  check rules /etc/prometheus/recording_rules.yml
```

Expected result:

```text
SUCCESS: 4 rules found
```

### Step 4: Validate the Prometheus configuration

Create the alerting rules file in Task 2 before running this validation.

```bash
sudo docker run --rm \
  --entrypoint /bin/promtool \
  -v /opt/prometheus-5.3/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
  -v /opt/prometheus-5.3/alerts.yml:/etc/prometheus/alerts.yml:ro \
  -v /opt/prometheus-5.3/recording_rules.yml:/etc/prometheus/recording_rules.yml:ro \
  prom/prometheus \
  check config /etc/prometheus/prometheus.yml
```

Resolve any configuration errors before starting Prometheus.

## 7. Task 2 – Define Alerting Rules

### Objective

Configure alerts to detect unavailable monitoring targets and sustained high CPU utilization.

### Step 1: Create alerting rules

Create the alert rules file:

```bash
sudo vi /opt/prometheus-5.3/alerts.yml
```

Configure the following alerts:

| Alert | Purpose | Example condition |
|---|---|---|
| `InstanceDown` | Detects a monitored target that is unavailable | `up == 0`, pending for 1 minute |
| `HighCPUUsage` | Detects sustained high CPU utilization | CPU usage above 80%, pending for 5 minutes |

Set appropriate severity labels and annotations so that notifications contain useful context.

### Step 2: Validate the alerting rules

```bash
sudo docker run --rm \
  --entrypoint /bin/promtool \
  -v /opt/prometheus-5.3/alerts.yml:/etc/prometheus/alerts.yml:ro \
  prom/prometheus \
  check rules /etc/prometheus/alerts.yml
```

### Step 3: Start Prometheus with the rule files

If an older Prometheus container is running, inspect it before replacing it:

```bash
sudo docker ps -a | grep prometheus
```

Stop and remove the existing container only after confirming that its configuration and data requirements are understood:

```bash
sudo docker stop prometheus
sudo docker rm prometheus
```

Start Prometheus on the existing `monitoring` Docker network:

```bash
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

Check the container and its health endpoint:

```bash
sudo docker ps
curl http://localhost:9090/-/healthy
```

### Step 4: Verify loaded rules

```bash
curl -s http://localhost:9090/api/v1/rules \
  | python3 -m json.tool \
  | grep -E '"name"|"type"|"health"|"state"|"alert"' \
  | head -80
```

Verify that the CPU recording metric is returning data:

```bash
curl -G -s http://localhost:9090/api/v1/query \
  --data-urlencode 'query=instance:cpu_usage_percent:10m' \
  | python3 -m json.tool
```

Expected result: the four recording rules are loaded and healthy, and the CPU recording query returns a result for the monitored Node Exporter target, such as `10.128.0.29:9100` in the example environment.

**Note:** Rule health, rule evaluation state, and target availability are different things. A healthy rule does not mean every target is `UP`. Verify these independently.

## 8. Task 3 – Configure Alertmanager and Slack Notifications

### Objective

Connect Prometheus to Alertmanager, verify that alerts are delivered, and configure Slack as a notification receiver.

### Step 1: Prepare Alertmanager

On `monitoring-vm`:

```bash
sudo mkdir -p /opt/alertmanager-5.3
cd /opt/alertmanager-5.3
sudo vi alertmanager.yml
```

Configure Alertmanager with a valid receiver and routing configuration. Add the Slack receiver in the Slack configuration steps below.

### Step 2: Start Alertmanager

Confirm that the Docker network `monitoring` exists and is also used by Prometheus.

```bash
sudo docker run -d \
  --name alertmanager \
  --restart unless-stopped \
  --network monitoring \
  -p 9093:9093 \
  -v /opt/alertmanager-5.3/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro \
  prom/alertmanager
```

Validate the container:

```bash
sudo docker ps | grep alertmanager
curl http://localhost:9093/-/healthy
```

### Step 3: Connect Prometheus to Alertmanager

Update `/opt/prometheus-5.3/prometheus.yml` with an Alertmanager configuration pointing to the container name:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "alertmanager:9093"
```

Keep the existing `scrape_configs` and `rule_files` entries. Because both containers share the `monitoring` Docker network, Prometheus can resolve Alertmanager by its container name.

Validate the configuration again:

```bash
sudo docker run --rm \
  --entrypoint /bin/promtool \
  -v /opt/prometheus-5.3/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
  -v /opt/prometheus-5.3/alerts.yml:/etc/prometheus/alerts.yml:ro \
  -v /opt/prometheus-5.3/recording_rules.yml:/etc/prometheus/recording_rules.yml:ro \
  prom/prometheus \
  check config /etc/prometheus/prometheus.yml
```

Restart Prometheus to apply the updated configuration:

```bash
sudo docker restart prometheus
```

Verify the Alertmanager connection:

```bash
curl -s http://localhost:9090/api/v1/alertmanagers \
  | python3 -m json.tool
```

### Step 4: Test the InstanceDown alert

On `target-vm`, stop Node Exporter temporarily:

```bash
sudo docker ps
sudo docker stop node-exporter
```

Wait long enough for the scrape interval and alert's `for` duration to elapse. For an alert configured with `for: 1m`, allow at least 70–120 seconds.

On `monitoring-vm`, inspect the alert state:

```bash
curl -s http://localhost:9090/api/v1/alerts \
  | python3 -m json.tool
```

Look for:

- `alertname`: `InstanceDown`
- `state`: `firing`

Check that Alertmanager received the alert:

```bash
curl -s http://localhost:9093/api/v2/alerts \
  | python3 -m json.tool
```

Restore Node Exporter on `target-vm`:

```bash
sudo docker start node-exporter
sudo docker ps | grep node-exporter
```

After the next scrape, verify that the target is up again:

```bash
curl -G -s http://localhost:9090/api/v1/query \
  --data-urlencode 'query=up{job="node-exporter"}' \
  | python3 -m json.tool
```

The target's `up` value should return to `1`.

### Step 5: Create a Slack incoming webhook

1. Open the [Slack API app management page](https://api.slack.com/apps).
2. Create a new app using **From scratch**.
3. Name it, for example, `Prometheus Alertmanager`, and select the intended workspace.
4. Open **Incoming Webhooks** and enable them.
5. Add a webhook to the desired channel, such as `#all-devops-journey`.
6. Authorize Slack to create the webhook and copy the generated URL.

Treat the webhook URL like a password. Never commit it to Git, publish it in screenshots, or paste it into public logs.

### Step 6: Store the Slack webhook securely

On `monitoring-vm`, create a protected environment file:

```bash
sudo vi /opt/alertmanager-5.3/slack.env
```

Add the actual webhook URL:

```bash
SLACK_WEBHOOK_URL='PASTE_YOUR_SLACK_WEBHOOK_URL_HERE'
```

Restrict access:

```bash
sudo chmod 600 /opt/alertmanager-5.3/slack.env
sudo ls -l /opt/alertmanager-5.3/slack.env
```

Create the separate webhook file expected by the Alertmanager configuration. The following command extracts the value without the surrounding single quotes:

```bash
sudo sh -c 'sed -n "s/^SLACK_WEBHOOK_URL='\''\(.*\)'\''$/\1/p" /opt/alertmanager-5.3/slack.env > /opt/alertmanager-5.3/slack_webhook_url'
```

Ensure the file contains only the URL, with no quotes or extra whitespace. Do not print its contents in shared terminals, logs, or screenshots.

### Step 7: Configure Alertmanager's Slack receiver

Update `alertmanager.yml` to use the webhook file rather than embedding the secret directly. The receiver's Slack configuration should use `api_url_file` with this path:

```yaml
api_url_file: /etc/alertmanager/secrets/slack_webhook_url
```

Keep the appropriate Slack receiver and routing settings in the file. Ensure the YAML is correctly indented and that the receiver is referenced by the route.

Set permissions on the host-side webhook file so that the non-root Alertmanager process can read it:

```bash
sudo chown 65534:65534 /opt/alertmanager-5.3/slack_webhook_url
sudo chmod 600 /opt/alertmanager-5.3/slack_webhook_url
sudo ls -ln /opt/alertmanager-5.3/slack_webhook_url
```

The numeric owner shown above is appropriate for the common non-root container user with UID/GID `65534`; verify the actual user in your container if it differs.

### Step 8: Validate and recreate Alertmanager

Validate the configuration:

```bash
sudo docker run --rm \
  --entrypoint amtool \
  -v /opt/alertmanager-5.3/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro \
  -v /opt/alertmanager-5.3/slack_webhook_url:/etc/alertmanager/secrets/slack_webhook_url:ro \
  prom/alertmanager \
  check-config /etc/alertmanager/alertmanager.yml
```

If validation succeeds, recreate the running container so it can mount the webhook file:

```bash
sudo docker stop alertmanager
sudo docker rm alertmanager

sudo docker run -d \
  --name alertmanager \
  --restart unless-stopped \
  --network monitoring \
  -p 9093:9093 \
  -v /opt/alertmanager-5.3/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro \
  -v /opt/alertmanager-5.3/slack_webhook_url:/etc/alertmanager/secrets/slack_webhook_url:ro \
  prom/alertmanager
```

Check the container and verify file readability without printing the secret:

```bash
sudo docker ps | grep alertmanager
curl http://localhost:9093/-/healthy

sudo docker exec alertmanager \
  sh -c 'test -r /etc/alertmanager/secrets/slack_webhook_url && echo "Webhook file is readable"'
```

### Step 9: Test the Slack notification pipeline

Stop Node Exporter on `target-vm` again:

```bash
sudo docker stop node-exporter
```

Wait at least 70–120 seconds, depending on the scrape interval and alert configuration. On `monitoring-vm`, check:

```bash
curl -s http://localhost:9090/api/v1/alerts \
  | python3 -m json.tool
```

Confirm that `InstanceDown` is firing, then check the configured Slack channel for the notification.

If the message does not arrive, inspect Alertmanager's logs:

```bash
sudo docker logs alertmanager --tail 100
```

For a `permission denied` error, check the webhook file's owner, mode, mount path, and container user. If the file contains quote characters, correct the file contents and restart Alertmanager.

Finally, restore Node Exporter:

```bash
sudo docker start node-exporter
```

## 9. Task 4 – Deploy Jaeger

### Objective

Deploy Jaeger to collect, store, and visualize distributed traces generated by the sample application.

### Step 1: Deploy Jaeger

On `monitoring-vm`, ensure the `monitoring` Docker network exists:

```bash
sudo docker network inspect monitoring
```

Deploy the Jaeger all-in-one image using a version that supports the OTLP protocol required by HotROD. The following command is a starting point for compatible all-in-one releases; verify the selected image's supported ports and configuration if its defaults differ.

```bash
sudo docker run -d \
  --name jaeger \
  --restart unless-stopped \
  --network monitoring \
  -p 16686:16686 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest
```

Check that the container is running:

```bash
sudo docker ps | grep jaeger
sudo docker logs jaeger --tail 50
```

Port `16686` is the Jaeger UI. Port `4318` is the OTLP HTTP receiver used by the example application. Do not expose OTLP ingestion publicly.

### Step 2: Configure secure access to the Jaeger UI

To open the UI from your workstation, create a GCP firewall rule allowing TCP `16686` only from your public IP address.

Find your workstation's public IP on your Mac terminal:

```bash
curl -4 https://ifconfig.me
```

In Google Cloud Console, navigate to **VPC network → Firewall → Create firewall rule** and configure:

| Setting | Value |
|---|---|
| Name | `allow-jaeger-ui` |
| Direction | Ingress |
| Action | Allow |
| Target | Specified target tags |
| Target tag | `jaeger-ui` |
| Source IPv4 range | Your public IP followed by `/32` |
| Protocol and port | TCP `16686` |

Add the `jaeger-ui` network tag to `monitoring-vm`.

Open the UI at:

```text
http://MONITORING_VM_EXTERNAL_IP:16686
```

Replace the placeholder with the current external IP of your VM. Restrict access to your own IP, and do not expose the UI unnecessarily.

## 10. Task 5 – Instrument and Run a Traced Application

### Objective

Run the HotROD sample application and send its traces to Jaeger using OTLP HTTP.

### Step 1: Verify the target VM

Connect to `target-vm` and check the environment:

```bash
hostname
sudo docker ps
docker --version
```

### Step 2: Verify connectivity to Jaeger

From `target-vm`, test the OTLP endpoint using the monitoring VM's **internal IP address**:

```bash
curl -sS -o /dev/null -w "HTTP status: %{http_code}\n" \
  http://MONITORING_VM_INTERNAL_IP:4318/
```

An HTTP response, including a `404`, confirms that the endpoint is reachable. A timeout or connection refusal requires further investigation.

If connectivity fails, check the VM network, Jaeger container port bindings, and GCP firewall configuration. Allow TCP `4318` from the target VM's internal IP with a `/32` source range, targeting the monitoring VM. Do not allow public internet access to this port.

### Step 3: Run HotROD

On `target-vm`, replace `MONITORING_VM_INTERNAL_IP` with the actual private IP of `monitoring-vm`:

```bash
sudo docker run -d \
  --name hotrod \
  --restart unless-stopped \
  -p 8080:8080 \
  -e OTEL_EXPORTER_OTLP_ENDPOINT=http://MONITORING_VM_INTERNAL_IP:4318 \
  jaegertracing/example-hotrod:latest \
  all
```

Check the application:

```bash
sudo docker ps | grep hotrod
curl -sS -o /dev/null -w "HTTP status: %{http_code}\n" \
  http://localhost:8080/
```

### Step 4: Access the HotROD UI

If you need to access HotROD from your workstation, create a GCP firewall rule for TCP `8080` restricted to your public IP.

Example command from Google Cloud Shell, after replacing the placeholder:

```bash
gcloud compute firewall-rules create allow-hotrod-ui \
  --network=default \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:8080 \
  --source-ranges=YOUR_LAPTOP_PUBLIC_IP/32 \
  --target-tags=hotrod-ui
```

Ensure the target VM has the `hotrod-ui` network tag and that `default` is the correct VPC network for your environment. If the rule already exists, inspect and update it rather than creating a duplicate.

Open:

```text
http://TARGET_VM_EXTERNAL_IP:8080
```

## 11. Task 6 – Analyze Distributed Traces

### Objective

Generate application requests, locate traces in Jaeger, and inspect span timelines to identify latency bottlenecks and service dependencies.

### Step 1: Generate sample requests

On `target-vm`, execute:

```bash
for i in {1..5}; do
  curl -s -o /dev/null http://localhost:8080/
done
```

### Step 2: Verify Jaeger has discovered services

On `monitoring-vm`, run:

```bash
curl -s http://localhost:16686/api/v3/services
```

The response should list services when traces have been received and indexed.

### Step 3: Inspect traces in the Jaeger UI

1. Open the Jaeger UI.
2. Select `frontend` from the **Service** dropdown.
3. Click **Find Traces**.
4. Open the newest trace.
5. Inspect the waterfall view and individual span durations.
6. Identify the service calls contributing the most latency.
7. Review span attributes and parent-child relationships to understand the request flow.

If no traces appear, generate additional requests and verify the OTLP endpoint, private network connectivity, HotROD logs, and Jaeger logs.

## 12. Beyond the Basics – Integrate Jaeger Metrics into Prometheus

The goal is to monitor the tracing system itself.

First, inspect the running Jaeger configuration and identify the actual metrics endpoint for the deployed version. Metrics ports and paths can vary between Jaeger releases; do not assume that port `8888` is available without verification.

Inspect the Prometheus configuration and back it up before making changes:

```bash
sudo cat /opt/prometheus-5.3/prometheus.yml

sudo cp /opt/prometheus-5.3/prometheus.yml \
  /opt/prometheus-5.3/prometheus.yml.backup
```

If the deployed Jaeger version exposes Prometheus metrics at `jaeger:8888/metrics`, and the Jaeger container is reachable on the `monitoring` network, add this job under `scrape_configs`:

```yaml
  - job_name: "jaeger"
    metrics_path: /metrics
    static_configs:
      - targets:
          - "jaeger:8888"
```

If necessary, attach Jaeger to the shared Docker network:

```bash
sudo docker network connect monitoring jaeger
```

Validate the active Prometheus configuration:

```bash
sudo docker exec prometheus \
  promtool check config /etc/prometheus/prometheus.yml
```

If validation succeeds, restart Prometheus:

```bash
sudo docker restart prometheus
```

Check target discovery:

```bash
curl -s http://localhost:9090/api/v1/targets \
  | python3 -m json.tool
```

Confirm that the Jaeger target is `UP` in Prometheus **Status → Targets**. If it is not, verify the actual Jaeger metrics port and path, container network connectivity, and the Prometheus target configuration.

## 13. Validation and Troubleshooting

| Component | Validation | Expected result |
|---|---|---|
| Prometheus | `curl http://localhost:9090/-/healthy` | Healthy response |
| Recording rules | Prometheus Rules UI or `/api/v1/rules` | Four recording rules loaded and healthy |
| Recording metric | Query `instance:cpu_usage_percent:10m` | Metric data returned when sufficient input data exists |
| Alerting rules | Prometheus Rules UI | `InstanceDown` and `HighCPUUsage` loaded |
| Alertmanager | `curl http://localhost:9093/-/healthy` | Healthy response |
| Alert delivery | `/api/v1/alertmanagers` | Alertmanager discovered by Prometheus |
| Slack | Trigger `InstanceDown` | Message arrives in the configured channel |
| Node Exporter | Query `up{job="node-exporter"}` | Value returns to `1` after recovery |
| Jaeger | Jaeger UI and application requests | Services and traces visible |
| HotROD | `curl http://localhost:8080/` | Application responds |
| Jaeger metrics (optional) | Prometheus Targets | Jaeger target is `UP` if correctly configured |

### Common issues

- **Recording rules are missing:** Check `rule_files`, mounted file paths, YAML indentation, and Prometheus logs.
- **Target remains down:** Verify Node Exporter, the scrape target address, the scrape configuration, and private network access.
- **Alertmanager is not connected:** Check the shared Docker network and the `alertmanager:9093` target.
- **Slack notification fails:** Inspect Alertmanager logs, validate the receiver configuration, and check webhook file contents and permissions without exposing the secret.
- **Jaeger UI does not load:** Verify the container, port `16686`, VM network tag, firewall source range, and external IP.
- **No traces appear:** Verify HotROD requests, the OTLP endpoint, port `4318`, firewall rules, and Jaeger logs.
- **Jaeger target is down in Prometheus:** Verify the metrics endpoint supported by the installed Jaeger version and ensure the containers can communicate.

## 14. Assignment Details

**Assignment:** Project 5.3 – Advanced Monitoring, Alerting, and Distributed Tracing

**Submitted by:** Snehal Shinde
