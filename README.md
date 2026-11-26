# Real User Monitoring with Grafana Faro

A browser observability demo using Grafana Faro, Grafana Alloy, Loki, and Grafana, deployed on AWS EC2 with Docker Compose.

The demo captures browser performance measurements, custom user events, and JavaScript errors from a shopping application.

## Architecture

**Website delivery:** Nginx → Browser

**Telemetry pipeline:** Browser / Faro SDK → Alloy → Loki

**Visualization:** Grafana queries Loki and displays the results.

Nginx, Alloy, Loki, and Grafana run as Docker containers. The Faro SDK runs in the user's browser.

## Components

| Component | Purpose |
| --- | --- |
| AWS EC2 | Ubuntu host for the demo |
| Docker Compose | Runs services and manages their network and volumes |
| Nginx | Serves the website and Faro SDK files |
| Grafana Faro SDK | Collects browser measurements, errors, and custom events |
| Grafana Alloy | Receives Faro telemetry and forwards log records to Loki |
| Loki | Stores browser telemetry as JSON log records |
| Grafana | Queries Loki and visualizes telemetry |

Browser performance values are stored as fields in Loki log records. This demo does not require a separate metrics database.

## Features

- Browser performance measurements: LCP, INP, CLS, and TTFB.
- Custom `add_to_cart` events.
- Intentionally generated JavaScript errors.
- A slow interaction scenario to demonstrate responsiveness issues.
- Application, browser, and session context.
- Grafana panels for counts, performance, trends, and detailed logs.

## Project Structure

```text
rum-demo/
├── compose.yaml
├── compose.override.yaml
├── config.alloy
├── loki-config.yaml
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── loki.yaml
├── web/
│   ├── index.html
│   └── faro.js
└── screenshots/
    ├── grafana-overview.png
    └── grafana-details.png
```

The `screenshots` directory contains the dashboard images used at the end of this README.

## Configuration Files

| File | Purpose |
| --- | --- |
| `compose.yaml` | Defines Loki and Grafana services |
| `compose.override.yaml` | Adds Alloy and Nginx services |
| `config.alloy` | Configures the Faro receiver, CORS, and forwarding to Loki |
| `loki-config.yaml` | Configures Loki storage, schema, and ingestion settings |
| `grafana/provisioning/datasources/loki.yaml` | Provisions the Grafana Loki datasource |
| `web/index.html` | Defines the application UI, Faro initialization, and button actions |
| `web/faro.js` | Contains the Faro browser SDK bundle |

Docker Compose automatically merges `compose.yaml` and `compose.override.yaml` when commands are run from the project directory.

The configuration can also be maintained in a single Compose file; using two files separates the initial observability services from the receiver and website.

## Prerequisites

- An Ubuntu server with Docker Engine and the Docker Compose plugin installed.
- SSH access to the server.
- A browser with JavaScript enabled.
- Network access to the required application ports.

Verify Docker:

```bash
sudo docker --version
sudo docker compose version
```

## Ports

| Port | Service | Access |
| --- | --- | --- |
| `22` | SSH | Administrator's IP |
| `8080` | Demo website | Demo user's IP |
| `12347` | Faro receiver | Demo user's IP |
| `3000` | Grafana | Demo user's IP |
| `3100` | Loki | Internal Docker network and host loopback |

For an AWS deployment, allow TCP ports `22`, `8080`, `12347`, and `3000` from your current public IP in the EC2 Security Group.

Loki does not need public access.

## Setup

### 1. Clone the Repository

Replace the repository URL below with your own:

```bash
git clone https://github.com/YOUR_USERNAME/YOUR_REPOSITORY.git
cd YOUR_REPOSITORY
```

Run the following commands from the directory containing the Compose files.

### 2. Configure the Website Origin

Update `config.alloy` with the address used to open the demo website:

```hcl
cors_allowed_origins = ["http://YOUR_PUBLIC_IP:8080"]
```

The origin must match the website's protocol, hostname or IP, and port.

The website runs on port `8080`, while the receiver runs on port `12347`. These are different browser origins, so the receiver needs the appropriate CORS configuration.

CORS is a browser access policy, not an authentication mechanism.

### 3. Check the Faro Collector URL

In `web/index.html`, the Faro collector URL must point to Alloy:

```javascript
url: 'http://' + window.location.hostname + ':12347/collect',
```

Using `window.location.hostname` allows the SDK to use the same hostname or IP as the website.

If the URL contains a hardcoded IP instead, update it to the current server IP.

The application identity used by the demo is:

```javascript
app: {
  name: 'rum-demo',
  version: '1.0.0',
  environment: 'demo'
},
```

### 4. Configure Grafana Credentials

If `compose.yaml` references `${GRAFANA_PASSWORD}`, create a `.env` file beside it:

```dotenv
GRAFANA_PASSWORD=REPLACE_WITH_A_STRONG_PASSWORD
```

Do not commit `.env` or private SSH keys to GitHub.

Add these entries to `.gitignore`:

```gitignore
.env
*.pem
```

### 5. Start the Services

```bash
sudo docker compose up -d
```

Check container status:

```bash
sudo docker compose ps
```

The setup contains four services:

- `web`
- `alloy`
- `loki`
- `grafana`

### 6. Open the Application and Grafana

Demo application:

```text
http://YOUR_PUBLIC_IP:8080/
```

Grafana:

```text
http://YOUR_PUBLIC_IP:3000/
```

Sign in using the credentials configured for Grafana.

### 7. Verify the Loki Datasource

The provisioning file configures the datasource with:

```yaml
apiVersion: 1

datasources:
  - name: Loki
    uid: loki
    type: loki
    access: proxy
    url: http://loki:3100
    isDefault: true
    editable: true
```

Grafana connects to Loki using the Docker service name `loki`.

Do not replace this internal URL with the EC2 public IP.

## Telemetry Configuration

### Grafana Alloy

The Faro receiver listens on port `12347` and accepts browser telemetry at `/collect`.

```hcl
logging {
  level = "info"
}

faro.receiver "rum" {
  server {
    listen_address = "0.0.0.0"
    listen_port = 12347
    cors_allowed_origins = ["http://YOUR_PUBLIC_IP:8080"]
  }

  log_format = "json"

  extra_log_labels = {
    job = "faro",
  }

  sourcemaps {
    download = false
  }

  output {
    logs = [loki.write.local.receiver]
  }
}

loki.write "local" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

- `logging` controls Alloy's own operational logs.
- `faro.receiver` receives browser telemetry.
- `log_format` formats emitted log records as JSON.
- `extra_log_labels` attaches `job="faro"` for Loki queries.
- `sourcemaps.download = false` disables automatic source-map downloads.
- `output.logs` connects the receiver to the Loki writer.
- `loki.write` sends records to Loki's ingestion API.

### Loki

The demo uses a single Loki instance with filesystem storage.

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: "2024-01-01"
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

storage_config:
  filesystem:
    directory: /loki/chunks

limits_config:
  ingestion_rate_mb: 4
  ingestion_burst_size_mb: 8

analytics:
  reporting_enabled: false
```

The Docker volume mounted at `/loki` preserves Loki data across container recreation, provided the volume is retained.

The `24h` index period is not a log retention period. This configuration does not define an explicit retention policy.

## Validate the Pipeline

### Send a Test Event

Run this on the EC2 instance:

```bash
curl -i -X POST http://localhost:12347/collect \
  -H 'Content-Type: application/json' \
  --data-binary @- <<EOF
{
  "meta": {
    "app": {
      "name": "rum-demo",
      "version": "1.0.0",
      "environment": "demo"
    },
    "session": {
      "id": "setup-test"
    }
  },
  "events": [
    {
      "name": "pipeline_test",
      "domain": "demo",
      "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
      "attributes": {
        "source": "terminal"
      }
    }
  ]
}
EOF
```

A `202 Accepted` response means the receiver accepted the request.

Confirm that the record reached Loki by running this query in Grafana Explore:

```logql
{job="faro"} |= "pipeline_test"
```

This terminal test verifies the ingestion path. It does not validate browser CORS behavior.

### Validate Browser Telemetry

1. Open the demo website.
2. Click **Add to Cart**.
3. Click **Generate Error**.
4. Click **Slow Interaction**.
5. Allow a few seconds for telemetry delivery.
6. Open Grafana Explore and select Loki.
7. Select a recent time range and run the queries below.

Custom events:

```logql
{job="faro"} | json | event_name="add_to_cart"
```

Demo checkout error:

```logql
{job="faro"} |= "Demo checkout failed"
```

Browser measurements:

```logql
{job="faro"} | json | kind="measurement"
```

## Demo Scenarios

| Action | Expected Observation |
| --- | --- |
| Open the website | Browser loading measurements |
| Add to Cart | A custom `add_to_cart` event |
| Generate Error | An intentionally generated JavaScript error |
| Slow Interaction | Increased interaction latency visible in reported INP |

The slow interaction intentionally blocks the browser's main thread for approximately 700 milliseconds.

During testing, reported INP increased from 8 ms to 720 ms. Actual results vary by browser, device, and interaction timing.

This is a frontend demonstration. It does not include a real checkout or payment backend.

## Browser Performance Measurements

| Measurement | Meaning | Unit |
| --- | --- | --- |
| LCP | Time until the largest visible content element renders | Milliseconds |
| INP | Interaction responsiveness, measured through the next paint | Milliseconds |
| CLS | Unexpected layout shift score | Unitless |
| TTFB | Navigation start to receipt of the first response byte | Milliseconds |

LCP, INP, and CLS are Core Web Vitals. TTFB is an additional loading diagnostic.

## Grafana Dashboard

The demo dashboard contains these panels:

| Section | Panels |
| --- | --- |
| User Activity & Errors | Demo Checkout Errors, Add to Cart Events |
| Browser Performance | TTFB — Max, LCP — Max, INP — Max, CLS — Max |
| Trends | INP Trend — 1m Max, User Events — Last 1m |
| Details | Recent User Events, Demo Checkout Error Details |

The performance Stat panels show maximum reported values within the selected time range.

The trend panels evaluate measurements or event counts over a rolling one-minute window.

The datasource is provisioned automatically. Dashboard panels must be created separately unless a dashboard export or provisioning configuration is added to the repository.

## Troubleshooting

### Public IP Changed

Update the website origin in `config.alloy`:

```hcl
cors_allowed_origins = ["http://NEW_PUBLIC_IP:8080"]
```

If the Faro collector URL in `web/index.html` uses a hardcoded IP, update it as well.

Restart Alloy:

```bash
sudo docker compose restart alloy
```

Open the website using the new IP and reload the page.

The internal Loki URL, `http://loki:3100`, does not need to change.

### Website or Grafana Is Not Reachable

- Confirm that the EC2 instance is running.
- Confirm the current EC2 public IP.
- Check container status with `sudo docker compose ps`.
- Verify the required inbound Security Group rules.
- If your Wi-Fi or public IP changed, update the allowed source IP.

### Browser Reports a CORS Error
Ensure `cors_allowed_origins` exactly matches the website origin.

Example:

```text
Website:       http://YOUR_PUBLIC_IP:8080
Allowed origin: http://YOUR_PUBLIC_IP:8080
```

Restart Alloy after updating the configuration, then reload the website.

### Grafana Shows No Data

- Generate fresh events on the website.
- Select **Last 15 minutes** and refresh Grafana.
- Check browser Developer Tools → Network for `/collect` requests.
- Verify that telemetry requests receive `202 Accepted`.
- Check Alloy and Loki logs.
- Run `{job="faro"}` in Explore to check for available records.

### Inspect Service Logs

```bash
sudo docker compose logs --tail=100 alloy
sudo docker compose logs --tail=100 loki
sudo docker compose logs --tail=100 web
sudo docker compose logs --tail=100 grafana
```

## Useful Commands

Start services:

```bash
sudo docker compose up -d
```

List project containers:

```bash
sudo docker compose ps
```

Follow Alloy logs:

```bash
sudo docker compose logs -f alloy
```

Stop services without removing containers:

```bash
sudo docker compose stop
```

Remove project containers and networks while retaining named volumes:

```bash
sudo docker compose down
```

Avoid `docker compose down -v` if you want to retain the stored logs and Grafana data.

## Demo Scope

This configuration is intended for learning and demonstration.

It uses HTTP endpoints, single-instance Loki, and local filesystem storage. A production deployment requires appropriate authentication, TLS, access controls, retention, capacity planning, and availability design.

## Grafana Dashboard Screenshots

### Overview: User Activity, Errors, and Browser Performance

![Grafana](./screenshots/grafana-overview.png)

### Details: User Events and Checkout Errors

![Grafana2](./screenshots/grafana-details.png)