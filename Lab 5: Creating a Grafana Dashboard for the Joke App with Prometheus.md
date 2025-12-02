# Lab 5: Creating a Grafana Dashboard for the Joke App with Prometheus

Welcome to Lab 4 of the SRE Workshop! This hands-on activity builds on previous labs by adding Grafana to visualize metrics from our funny Joke API monitored by Prometheus. You'll deploy Grafana alongside the existing setup, create a dashboard to display key metrics (like request rates, latency, and custom joke stats), and explore visualization options. This helps turn raw data into actionable insights—e.g., see when "bad jokes" (failures) spike!

This guide is designed for beginners. Each step includes explanations, why we're doing it, and troubleshooting tips. We'll assume you've completed Lab 3 (or at least Lab 1) and have the files ready. Reuse the Joke API with custom metrics.

**Time Estimate:** 45-60 minutes.  
**Goals:**  
- Integrate Grafana with Prometheus for visualization.  
- Build a dashboard with panels for SLIs/SLOs, metric types, and fun app stats.  
- Customize views (e.g., graphs, gauges) and export/share dashboards.  
- Connect to previous labs: Use the Joke API's metrics to create a "Humor Health" dashboard.

## Step 1: Set Up the Lab Directory
**Why?** We'll extend previous files to include Grafana—copy them for continuity.

1. In your terminal, create a new directory and copy files from Lab 3 (or earlier):  
   ```
   mkdir sre-lab5
   cd sre-lab5
   ```

2. We'll update files below. Use your text editor (e.g., VS Code).

## Step 2: Update the Setup to Include Grafana
**Why?** Grafana queries Prometheus data and builds interactive dashboards—great for SRE teams to monitor at a glance.

Update `app.py` (add logging; keep custom metrics from Lab 2):

```python
from flask import Flask, jsonify
import time
import random
from prometheus_flask_exporter import PrometheusMetrics
from prometheus_client import Counter, Gauge, Histogram, Summary
import logging

# Set up logging to stdout with INFO level
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

app = Flask(__name__)
metrics = PrometheusMetrics(app)

# Export default metrics
metrics.info('app_info', 'Application info', version='1.0.0')

# Custom metrics from Lab 2
joke_counter = Counter('joke_total', 'Total number of jokes told')
humor_gauge = Gauge('humor_level', 'Current humor level (fluctuates)')
joke_length_histogram = Histogram('joke_length_seconds', 'Distribution of joke "delivery" lengths', buckets=[10, 20, 30, 40, 50, 60])
latency_summary = Summary('request_latency_summary_seconds', 'Summary of request latencies')

# List of funny jokes/puns
jokes = [
    "Why did the SRE go to therapy? Too many incidents!",
    "Error budgets are like diets: easy to set, hard to stick to.",
    "Why don't programmers like nature? Too many bugs.",
    "SLOs: Because 'best effort' isn't measurable.",
    "Why was the computer cold? It left its Windows open!",
    "Failure is not an option—it's a feature in beta.",
    "Why did the database administrator leave his wife? She had one too many relationships.",
    "Alert: Your coffee is low— that's the real outage.",
    "Why do SREs make great musicians? They handle scales well.",
    "404: Joke not found. Wait, that's not funny."
]

@app.route('/success')
def success():
    start_time = time.time()
    delay = random.uniform(0.05, 0.15)
    time.sleep(delay)
    joke = random.choice(jokes)
    
    # Update custom metrics
    joke_counter.inc()
    humor_gauge.set(random.uniform(50, 100))
    joke_length_histogram.observe(len(joke))
    latency_summary.observe(time.time() - start_time)
    
    logging.info(f"Successful request! Joke delivered: {joke}")
    return jsonify(message=f"Success! Here's a joke: {joke}"), 200

@app.route('/failure')
def failure():
    start_time = time.time()
    delay = random.uniform(0.2, 0.5)
    time.sleep(delay)
    joke = random.choice(jokes)
    
    # Update custom metrics
    joke_counter.inc()
    humor_gauge.set(random.uniform(0, 50))
    joke_length_histogram.observe(len(joke))
    latency_summary.observe(time.time() - start_time)
    
    logging.error(f"Failure request! Bad joke attempted: {joke}")
    return jsonify(message=f"Oops, failure! But hey: {joke}"), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3000)
```

**Explanation:**  
- Added `import logging` and basic config—logs go to console (stdout).  
- In endpoints: Log info for success, error for failure, including the joke for fun.  
- This demonstrates logging: Track events without changing app behavior.


Update your prometheus.yml
```
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Load rules once and periodically evaluate them
rule_files:
  - "/etc/prometheus/alert-rules.yml"

# Scrape configuration
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'flask-app'
    static_configs:
      - targets: ['app:3000']
```


### File: `requirements.txt` (Dependencies)
Lists Python packages needed.
```
flask==3.0.3
prometheus-flask-exporter==0.23.1
```
**Explanation:** Docker will install these when building the container. No need to install them locally.
### File: `Dockerfile` (Build Instructions for the App Container)
Defines how to package the app into a Docker image.
```
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
EXPOSE 3000
CMD ["python", "app.py"]
```
**Explanation:**
- Starts from a lightweight Python image.
- Copies files and installs dependencies.
- Exposes port 3000 for access.

Update `docker-compose.yml` (add the grafana service):

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    container_name: sample-app

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alert-rules.yml:/etc/prometheus/alert-rules.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--web.enable-lifecycle'
    container_name: prometheus
    depends_on:
      - app

  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    container_name: alertmanager

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"  # Grafana on 3001 to avoid conflict with app
    volumes:
      - grafana_data:/var/lib/grafana  # Persists dashboards
    container_name: grafana
    depends_on:
      - prometheus

volumes:
  alertmanager_data:
  grafana_data:  # New for Grafana persistence
```

**Explanation:**  
- Grafana service uses official image, exposes on port 3001 (internal 3000).  
- Volumes persist data (e.g., saved dashboards).  
- No changes to `app.py`, `prometheus.yml`, etc.—Grafana pulls from Prometheus.

Create new file: `alert-rules.yml` (defines when to alert, e.g., high error rate breaching SLO):

```yaml
groups:
  - name: joke-api-alerts
    interval: 15s  # How often to evaluate these rules
    rules:
      # Alert when error rate exceeds 10%
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(flask_http_request_total{status="500"}[5m]))
            /
            sum(rate(flask_http_request_total[5m]))
          ) > 0.1
        for: 1m
        labels:
          severity: warning
          service: joke-api
        annotations:
          summary: "High error rate in Joke API"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 10%). Check application logs immediately."
          dashboard: "http://localhost:9090/graph?g0.expr=rate(flask_http_request_total%7Bstatus%3D%22500%22%7D%5B5m%5D)"
      
      # Alert when latency is too high
      - alert: HighLatency
        expr: |
          histogram_quantile(
            0.99,
            sum(rate(flask_http_request_duration_seconds_bucket[5m])) by (le)
          ) > 0.2
        for: 1m
        labels:
          severity: critical
          service: joke-api
        annotations:
          summary: "High latency detected in Joke API"
          description: "99th percentile latency is {{ $value | humanizeDuration }} (threshold: 200ms). Users experiencing slow responses."
          dashboard: "http://localhost:9090/graph"
      
      # Additional useful alert: Service is down
      - alert: ServiceDown
        expr: up{job="flask-app"} == 0
        for: 30s
        labels:
          severity: critical
          service: joke-api
        annotations:
          summary: "Joke API is down"
          description: "The Flask application has been unreachable for 30 seconds."
      
      # Additional useful alert: High request rate
      - alert: HighRequestRate
        expr: sum(rate(flask_http_request_total[5m])) > 100
        for: 2m
        labels:
          severity: info
          service: joke-api
        annotations:
          summary: "High request rate detected"
          description: "Request rate is {{ $value | humanize }} req/s (threshold: 100 req/s). Possible traffic spike."
```

**Explanation:**  
- `HighErrorRate`: Alerts if errors > 10% (breaching availability SLO).  
- `HighLatency`: Alerts if slow requests > 200ms.  
- Fun annotations tie back to jokes.

Create new file: `alertmanager.yml` (basic config; for demo, alerts show in UI—no email/slack):

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: blackhole

receivers:
  - name: blackhole
```

**Explanation:** Simple setup—alerts will be visible in Alertmanager UI.



## Step 3: Deploy the Updated Stack
**Why?** Starts Grafana connected to Prometheus.

1. Run:  
   ```
   docker-compose up -d --build
   ```

2. Verify:  
   - App: http://localhost:3000/success (joke response).  
   - Prometheus: http://localhost:9090 (metrics).  
   - Alertmanager: http://localhost:9093 (if from Lab 3).  
   - Grafana: http://localhost:3001 – Login with admin/admin (change password on first login).



   <img width="3358" height="1404" alt="image" src="https://github.com/user-attachments/assets/4421676d-c281-4903-b269-ad74be177ef9" />


**Troubleshooting:**  
- Grafana not starting? Check logs: `docker logs grafana`. Ensure no port conflicts (change "3001:3000" if needed).  
- First login prompt? Set new password when asked.

## Step 4: Configure Grafana Data Source
**Why?** Links Grafana to Prometheus for querying metrics.

1. In Grafana UI (http://localhost:3001): Left menu > Connections > Data sources > Add new.  
2. Search/select "Prometheus".

<img width="3336" height="1372" alt="image" src="https://github.com/user-attachments/assets/c7fe3280-437e-4901-b0fb-7d330fe8d954" />

4. HTTP URL: `http://prometheus:9090` (internal Docker network name).  
5. Save & Test—should say "Data source is working".
<img width="2674" height="488" alt="image" src="https://github.com/user-attachments/assets/6ce0407a-397f-4545-bbda-e1bfaf57e10c" />


**Explanation:** This lets Grafana pull all metrics (e.g., flask_http_request_total, custom joke_total).

**Troubleshooting:** Test fails? Use `http://host.docker.internal:9090` if on macOS/Windows (localhost bridging). Restart stack.

## Step 5: Create Your First Dashboard
**Why?** Dashboards visualize metrics—build one for the Joke API.

1. Grafana UI > Dashboards > New > New dashboard.  
2. Add panel: Click "Add panel" > New visualization.  
3. Data source: Prometheus.  
4. Build panels (examples below—add one by one):  
   - **Request Rate Graph (Counter Metric):** Query: `rate(flask_http_request_total[5m])`. Visualization: Time series. Title: "Joke Requests per Second".
  
   <img width="2100" height="1382" alt="image" src="https://github.com/user-attachments/assets/67a95d5c-21a4-444d-8305-5eb2b6147448" />

   - **Availability Gauge (SLI from Lab 1):** Query: `sum(rate(flask_http_request_total{status="200"}[5m])) / sum(rate(flask_http_request_total[5m])) * 100`. Visualization: Gauge (thresholds: green >99, yellow >95, red <95). Title: "Availability %".

   <img width="2726" height="1014" alt="image" src="https://github.com/user-attachments/assets/2dd717ad-d842-4563-b5a2-d70619b306cb" />

     
   - **Latency Histogram:** Query: `histogram_quantile(0.99, sum(rate(flask_http_request_duration_seconds_bucket[5m])) by (le))`. Visualization: Time series. Title: "99th Percentile Latency".  
   - **Humor Level Stat (Gauge Metric from Lab 2):** Query: `humor_level`. Visualization: Stat (shows current value, color-coded). Title: "Current Humor Level".  
   - **Joke Length Distribution (Histogram Metric):** Query: `sum(rate(joke_length_seconds_bucket[5m])) by (le)`. Visualization: Histogram. Title: "Joke Length Buckets".  
6. Arrange panels: Drag/resize on dashboard. Add rows if needed (via dashboard settings).  
7. Save dashboard: Top right > Save > Name: "Joke API Health Dashboard".

**Explanation:** Each panel uses PromQL queries from previous labs. Time series for trends, gauges for SLOs, histograms for distributions—fun way to monitor "humor health"!

**Activity:** Generate traffic with `simulate_traffic.sh`—watch dashboard update in real-time (refresh or set auto-refresh to 10s).

## Step 6: Customize and Export the Dashboard
**Why?** Make it shareable—add annotations, variables, export for teams.

1. Add variable: Dashboard settings (gear icon) > Variables > New > Query type, Data source: Prometheus, Query: `label_values(job)` (filters by job). Use in panels for dynamic views.  
2. Add annotation: Settings > Annotations > New (e.g., query alerts from Prometheus).  
3. Export: Settings > Save As > JSON > Download. Import in another Grafana for sharing.

**Explanation:** Variables make dashboards reusable (e.g., switch services). Export JSON for version control.

## Step 7: Cleanup and Reflect
1. Stop: `docker-compose down -v` (removes volumes to clean data).  
2. Think: How do dashboards help SREs? (Quick insights, correlate metrics like latency and errors.) What panels for real apps? (Add error budgets from Lab 1.)

**Troubleshooting Tips:**  
- No data in panels? Ensure queries match metrics (use Prometheus UI to test). Generate traffic.  
- Grafana errors? Check browser console or container logs.  
- Slow? Limit dashboard to 4-6 panels for lab.

You've completed Lab 4! Your Joke API now has a pro dashboard in Grafana. Save as `lab4-guide.md`. Experiment: Add a "Bad Jokes Alert" panel linking to alerts from Lab 3. Ready for more labs?
