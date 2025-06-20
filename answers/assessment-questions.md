## ✅ Section 1: Architecture and Setup Understanding

### Q1: Container Orchestration Analysis

In this lab, the Node Exporter container had to access the host's `/proc`, `/sys`, and `/` directories. This is because it collects real-time system data such as CPU usage, memory info, and file system stats. If you don’t mount these directories, Node Exporter can only see data from *inside the container*, which is pretty much useless when you’re trying to monitor the actual server.

I tested this by removing the `/proc` mount once, and Prometheus started showing blank or zero values for most of the metrics. The dashboard panels didn’t make sense anymore.

This setup still respects the idea of containerization — the Node Exporter stays isolated as a container, but it reads from the host system in a read-only way. So we get real system data without running tools directly on the host.

---

### Q2: Network Security Implications

By using a custom network named `prometheus-grafana`, all the containers are placed in a controlled environment where only they can talk to each other. It’s definitely better than using Docker’s default network because it keeps things cleaner and reduces risks of random containers interacting.

That said, we still exposed ports `9090`, `9100`, and `3000` for Prometheus, Node Exporter, and Grafana. This means anyone on the same network can hit those endpoints. That’s risky.

For a real production setup, I would:

* Put them behind a reverse proxy with HTTPS (e.g. NGINX + Let’s Encrypt)
* Use firewall rules to restrict access
* Enable authentication on Grafana
* Never expose Prometheus or Node Exporter publicly

---

### Q3: Data Persistence Strategy

Both Prometheus and Grafana use named volumes in the Docker Compose file to keep data persistent. Prometheus stores its time series data (metrics) in `prometheus_data`, and Grafana keeps dashboards and settings in `grafana_data`.

I tried running `docker compose down -v` just to see what would happen, and I lost everything — the dashboards I made and all my past metrics were gone.

These volumes are critical because:

* Grafana needs them to keep dashboards and user settings between restarts
* Prometheus needs them to retain metrics for graphing and alerting

Since both tools handle different kinds of data, having separate volumes is a good idea.

---

## ✅ Section 2: Metrics and Query Understanding

### Q4: PromQL Logic Breakdown

The lab used:

```promql
node_time_seconds - node_boot_time_seconds
```

This gives us uptime. The first value is the system’s current time, and the second is when the system last booted. Subtract them and you get how long it’s been running.

But there's a catch. If the exporter restarts or gets scraped too early, you might get a `NaN` or wrong result. Also, slight time drift between containers can cause issues.

A more stable version would be:

```promql
time() - node_boot_time_seconds
```

This uses Prometheus’s internal clock instead, which is more reliable especially when monitoring multiple systems.

---

### Q5: Memory Metrics Deep Dive

Instead of using `node_memory_MemFree_bytes`, the lab suggests:

```promql
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```

This gives a better picture of how much memory is truly in use. In Linux, `MemFree` doesn’t tell the full story because the OS uses free memory for caching. So a low MemFree doesn’t mean the system is out of memory.

`MemAvailable` is smarter — it includes free memory *and* cached memory that the system can easily reclaim. So this method helps avoid false alarms and gives a more realistic usage stat.

---

### Q6: Filesystem Query Analysis

The query:

```promql
1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})
```

This calculates how full the root filesystem is. It divides available space by total space to get the free ratio, then subtracts from 1 to get used space.

The problem comes when you have many mount points. It can confuse the panel or mix up results. Also, some temporary filesystems like `tmpfs` or `/run` can distort the view.

To fix that, I filtered them out using:

```promql
1 - (
  node_filesystem_avail_bytes{fstype!="tmpfs", mountpoint!~"/(run|boot|dev)"} /
  node_filesystem_size_bytes{fstype!="tmpfs", mountpoint!~"/(run|boot|dev)"}
)
```

This cleaned things up and focused on actual disk usage.

---

## ✅ Section 3: Visualization and Dashboard Design

### Q7: Visualization Type Justification

* **Stat panel**: Great for things like uptime — one clear number.
* **Time Series**: Perfect for trends like CPU and memory usage.
* **Gauge**: Ideal for disk usage — shows how full things are in a visual way.

Each panel type was chosen based on what I needed to show. For example, a gauge for memory might be misleading because Linux caches a lot. So it depends on the data and what people need to see quickly.

---

### Q8: Threshold Configuration Strategy

The lab used 80% as the warning threshold for disk usage. That’s a common default, but depending on the server type, you may want different levels:

* **Database servers**: Warn at 70%
* **Web servers**: Maybe fine until 90%

A better system is using multiple thresholds:

* Yellow: 70%
* Orange: 85%
* Red: 95%

Grafana lets you set these in the panel settings. For alerts, I’d link it to Slack or email so we catch issues early.

---

### Q9: Dashboard Variable Implementation

The `$job` variable in Grafana makes the dashboard reusable. It pulls job names from Prometheus (e.g., node-exporter) and lets you switch between them in a dropdown.

If the query or label is wrong, the whole dashboard can break. I tried using an invalid variable and got no data.

To make it stable, I used:

```promql
label_values(node_cpu_seconds_total, job)
```

That lists all jobs, so the dropdown always works. I also tested with default values to make sure it doesn’t crash.

---

## ✅ Section 4: Production and Scalability Considerations

### Q10: Resource Planning and Scaling

If we monitored 100 servers, and each exports \~1000 metrics per scrape, then with a 15s interval:

* That’s 1000 × 100 × 4 × 60 × 24 = \~576 million datapoints/day

Based on that, Prometheus would need:

* CPU: 4–8 cores
* RAM: 8–16 GB
* Disk: 100–200 GB/month

First bottlenecks would be:

* Disk I/O
* Dashboard query load
* Retention policy size

To scale, I’d:

* Use remote storage like Thanos
* Reduce scrape intervals for low-priority nodes
* Split workloads across Prometheus shards

---

### Q11: High Availability Design

To make this setup highly available:

* **Prometheus**: Run two instances with different scrape targets
* **Grafana**: Use PostgreSQL as backend DB for shared state
* **Reverse proxy**: Put HAProxy or NGINX in front
* **Backups**: Set up volume backups or sync dashboards to GitHub

This increases complexity, but also makes sure metrics and dashboards stay up even if one piece fails.

---

### Q12: Security Hardening Analysis

Here are 5 key issues I noticed:

1. **Open ports** — Limit with firewall or reverse proxy
2. **No auth** — Enable login for Grafana, use middleware for Prometheus
3. **No HTTPS** — Add reverse proxy with SSL
4. **Unencrypted volumes** — Use encrypted storage or cloud volumes
5. **Secrets in plaintext** — Use `.env` files or Docker secrets

I would store tokens using Docker Secrets or tools like Vault. For HTTPS, I’d use Let’s Encrypt + NGINX.

---

## ✅ Section 5: Troubleshooting and Operations

### Q13: Debugging Methodology

If Prometheus shows a target as DOWN, here’s what I did:

1. Go to `http://localhost:9090/targets` and check the error message
2. Run `curl http://target:port/metrics` inside Prometheus container
3. Check `docker logs` of the target (e.g., node-exporter)
4. Look at `prometheus.yml` to confirm job/port
5. Restart the container if needed

Most common problems:

* Wrong port
* Crash
* Firewall blocking
* Wrong label in config

---

### Q14: Performance Optimization

Some panels were laggy. I noticed that queries using `rate()` over large time windows slowed things down.

So I:

* Switched to `irate()` when possible
* Scoped queries using `job` or `instance`
* Increased dashboard interval from 1s to 15s

To monitor Prometheus itself, I tracked:

* `prometheus_tsdb_head_series`
* `prometheus_engine_query_duration_seconds`

---

### Q15: Capacity Planning Scenario

Prometheus started eating a lot of disk. I realized it was due to scrape frequency + number of series.

To fix it:

* I set flags: `--storage.tsdb.retention.time=15d` and `--storage.tsdb.retention.size=20GB`
* I turned on data compression

For a long-term plan:

* Keep 30 days on local disk
* Archive old data to S3 using Thanos
* Purge expired data regularly

This keeps storage under control while still retaining what’s important.
