## Basics
* It is an open source tool to scrape metrics from services. It is a time series database, a scraping engine, and a query language.
* It pulls metrics instead of waiting for others to push.
	* It is a good heuristic if a service is working - push can't differentiate a silent and a dead service.
	* central behavior for all services
	* not very well for short-lived services (like lambda - batch jobs): need using **Pushgateway**
* the main components Prometheus has are:
	* Server: Handles scraping, evaluating queries, evaluating rules, stores data
	* Exporter: Small HTTP servers that allows generating Prometheus readable metrics for things that don't do it natively
	* Service Discovery: Instead of hardcoding services, Prometheus can detect them automatically
	* Alertmanager: Recieves alerts from Prometheus and handles them (forwarding / silencing / grouping ) and sends them
	* PromQL: the query langauge it uses
## Data Model
Everything in Prometheus is a time series, and each has a metric name, and a set of key value pairs.

```
http_requests_total{method="GET", status="200"}
```

There are four types of data:
1. Counter: A raw value - increasing like total requests
2. Gauge: A value can be increased or decreased: like replica count
3. Historgram: Samples of observations and groups them into buckets: Very simple example would be that a teacher counts how many students finished their test in 10, 20, and 30 mins. (e.g. 5 $\le$ 10min, 15 $\le$ 20min, 20 $\le$ 30min). You loose a bit precision but you can have good metrics, like roughly what time did 90% of the students finish the test.
4. Summary: It is like histogram but in client-side you calcuate the question you asked in the previous part

## PromQL
* Instant Vector: It is just one value (e.g. `http_total_request` which could be 120)
* Range Vector: It is the values of the same instant vector in a window (e.g. `http_total_request[1m]` which can be 120, 125, 130, ...)
* `rate()` : It averages the range vector per second. `rate(http_total_request[1m])` $=\frac{\text{lastval} - \text{firstval}}{60}$
	* if the counter suddenly drops, `rate` understands it is a counter reset, so takes that into account.
* `irate()` : Takes only last two points in the window
* When to use `rate` and `irate`: Use `rate` when building a dashboard, alert and to see the real pattern and use the other to see sudden spikes.
* Aggregation: `sum(rate(http_requests_total[5m])) by (status)` - this sums request accross all instances and groups by the status code in that 5 min window
* `histogram_quantile`: This is used for computing quantiles. `histogram_quantile(0.99, sum(rate(http_requests_duration_seconds_bucket[5m])) by (le))` 

## Scraping & Service Discovery
* If we have a handful of servers that won't change, we just hardcode them in the config. Not realistic in k8s, since pods come and go, scale up/down.
* So we use **Service Discovery** (k8s SD, Consul, EC2, DNS, File SD...) — Prometheus asks the platform "what's running right now" and keeps the target list fresh automatically.
* Catch: k8s SD gives us **every** pod in the cluster, not just the ones we want. We need to filter.
* **`relabel_configs`**: runs **before scraping**, on target metadata (`__meta_*` labels — temporary "sticky notes" k8s SD attaches, like namespace, annotations, etc., which get discarded after this stage).
	* `action: keep` / `drop` → filter which targets get scraped (e.g. keep only pods with annotation `prometheus.io/scrape: true`)
	* `action: replace` → promote a `__meta_*` value into a permanent label (e.g. `__meta_kubernetes_namespace` → `namespace`), otherwise it's lost
* **`metric_relabel_configs`**: runs **after scraping**, on the actual returned metric data. Used to drop unwanted metrics or strip high-cardinality labels (e.g. `user_id`) *after* the fact.
* One-liner: `relabel_configs` = filter/shape **targets** before scrape. `metric_relabel_configs` = clean up **data** after scrape.
## Alerting

* **Alertmanager** is a separate process from Prometheus. Prometheus **evaluates** alert rules; Alertmanager decides whether/how to **notify**.
* Alert rules live in Prometheus config: an `expr` (PromQL condition), a `for` duration (must stay true this long before firing), `labels` (used for routing/priority), and `annotations` (human-readable text).
* **Alert lifecycle — 3 states:**
	* **Inactive** — condition false.
	* **Pending** — condition just became true, but hasn't held for the full `for` duration yet.
	* **Firing** — condition held true for the full `for` duration → sent to Alertmanager.
* Meeting the condition doesn't always mean a notification goes out — Alertmanager applies extra logic first:
	* **Grouping** — bundle related alerts (e.g. 50 pods failing at once) into one notification instead of spamming 50.
	* **Inhibition** — suppress smaller/related alerts if a broader, more severe one is already firing (e.g. skip individual `InstanceDown` alerts if `ClusterDown` is already firing).
	* **Silencing** — manual, temporary mute (e.g. during planned maintenance).
* **Alertmanager needs HA too** — run as a cluster (gossip protocol keeps replicas in sync) so a single Alertmanager crash doesn't mean nobody gets paged.
* **Dead man's switch** — a rule that always fires (`vector(1) > 0`), sent to an external heartbeat service. If Prometheus itself dies, it stops sending this signal, and the external service notices the silence and pages you. Solves "what alerts me if Prometheus itself is down."
