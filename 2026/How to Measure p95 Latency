
1. Add a NestJS interceptor that logs {route, method, statusCode, durationMs} per request, then compute p95 by parsing/aggregating those logs (e.g., pipe into a script, or ship logs to something queryable). Cheapest to build, but you own the aggregation.

2. Reverse-proxy / infra-level metrics — if this runs behind Nginx, an ALB, or similar, enable access logs with $request_time/latency fields. p95 can then be computed without touching app code at all (e.g., awk/gnuplot on Nginx logs, or CloudWatch/ALB target response time percentiles if you're on AWS).

3. APM tool (recommended for "all APIs, ongoing") — wire in something like:
Prometheus + prom-client (nestjs-prometheus or manual middleware) exposing a histogram, scraped and visualized in Grafana with histogram_quantile(0.95, ...) per route. Self-hosted, free, industry standard for this exact question.
Datadog / New Relic / Elastic APM — drop-in agent, gives you p50/p95/p99 per endpoint out of the box, plus traces. Paid, but zero query-writing.
Sentry Performance — you likely already touch Sentry-adjacent tooling in Node backends; gives per-transaction p95 with less setup than Prometheus.
4. Load-test tools (k6, Artillery, autocannon) if you want p95 under synthetic load rather than real production traffic — these report p95 natively per run.


If you are opting 3. step follow these steps to integrate in your app : these Steps is for nestjs Application :- 
Step 1 — Instrument the app to expose metrics
Install prom-client (the standard Node.js Prometheus client) — no separate agent needed, it runs in-process.

npm install prom-client
Create a small metrics module that:
Registers a Histogram named something like http_request_duration_seconds, labeled by method, route, status_code.
Uses app.setGlobalPrefix('api')-aware route labels — for NestJS, pull the matched route template (req.route.path via the underlying Express/Fastify request) rather than the raw URL, so /api/v1/users/123 and /api/v1/users/456 collapse into one series /api/v1/users/:id instead of exploding into per-ID cardinality (important — unbounded label cardinality is the #1 way people break Prometheus).
Wire it in as a NestJS middleware or interceptor (applied globally in main.ts, alongside your existing helmet()/compression() middleware) that starts a timer on request-in and observes duration on response-finish.
Expose a GET /metrics endpoint (typically excluded from your global api versioned prefix, e.g. plain /metrics) that calls prom-client's register.metrics() and returns Prometheus's text exposition format. This endpoint should not require auth from Prometheus's scraper, but should be blocked from public internet access (see Step 4).
Step 2 — Run Prometheus alongside your app
Add a prometheus service to docker-compose.yml, on the same tejas_network:


prometheus:
  image: prom/prometheus:latest
  container_name: tejas_prometheus
  volumes:
    - ./prometheus.yml:/etc/prometheus/prometheus.yml
  ports:
    - "9090:9090"
  networks:
    - tejas_network
And a prometheus.yml scrape config pointing at your api container by its Docker network hostname:


scrape_configs:
  - job_name: 'tejas-api'
    scrape_interval: 15s
    static_configs:
      - targets: ['api:3000']
    metrics_path: /metrics
Prometheus will now poll /metrics every 15s and store the histogram buckets over time.

Step 3 — Run Grafana and build the p95 dashboard
Add a grafana service the same way:


grafana:
  image: grafana/grafana:latest
  container_name: tejas_grafana
  ports:
    - "3001:3000"
  networks:
    - tejas_network
Then:

Log into Grafana (localhost:3001), add Prometheus (http://prometheus:9090) as a data source.
Build a panel with the PromQL query:

histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route))
This gives you p95 per route, over a rolling 5-minute window.
For the single "maximum p95 across all APIs" number you originally asked about, drop the route grouping:

max(histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route)))
— i.e., compute p95 per route, then take the worst one.
Save as a dashboard; optionally set an alert rule if p95 crosses a threshold.
Step 4 — Secure it before deploying
Don't expose ports 9090/3001 publicly on apinew.esolveinfotech.com etc. — put Prometheus/Grafana behind your existing reverse proxy with auth, or keep them on an internal-only network/VPN.
Set a real Grafana admin password via env var (GF_SECURITY_ADMIN_PASSWORD), don't leave the default.
If /metrics is reachable from the internet, anyone can see your request-volume/latency shape — low risk but still worth restricting by IP or a shared-secret header check in front of it.
Step 5 — Validate
Hit a few endpoints, wait 15–30s for a scrape, confirm data shows up in Prometheus's own UI (localhost:9090/graph) before trusting the Grafana panel.
Load-test one endpoint (e.g. with autocannon) to confirm the p95 number moves as expected.
