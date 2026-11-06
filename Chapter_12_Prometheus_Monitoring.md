# Chapter 12 — Prometheus Monitoring

## Chapter objectives

This chapter documents the Prometheus monitoring work performed on the three-node Vagrant Kubernetes cluster. It follows Chapter 11, but covers a different metrics pipeline: Prometheus scrapes and stores monitoring time series, while Metrics Server exposes short-lived CPU and memory resource metrics for Kubernetes consumers such as the HPA.

The chapter records the Prometheus Helm installation, the initial storage-related Pending state, the Node Exporter and kube-state-metrics components, PromQL fundamentals, the metrics used by Grafana, and the CPU alert that was successfully delivered to Slack. It also adds clearly labelled exercises for deeper PromQL, scrape configuration, and Prometheus-native alert rules.

By the end, you should be able to:

- Explain Prometheus pull-based monitoring and its main components.
- Distinguish a scrape target, exporter, metric, label, sample, and time series.
- Explain the roles of Prometheus Server, Node Exporter, kube-state-metrics, Pushgateway, and Alertmanager.
- Install and inspect the Prometheus Helm release used in the lab.
- Verify Prometheus-related Kubernetes objects and understand their relationships.
- Use PromQL to inspect target health, CPU, memory, filesystem, and Kubernetes object metrics.
- Understand scrape configuration and Kubernetes service discovery.
- Distinguish Grafana-managed alert rules from native Prometheus alerting rules.
- Troubleshoot common scrape, target, query, and storage problems.

## 1. Lab environment and evidence boundaries

The practical context is the final three-node cluster:

| Node | Role | Host-only IP |
|---|---|---|
| k8master1 | Control plane | 192.168.56.10 |
| k8worker1 | Worker | 192.168.56.20 |
| k8worker2 | Worker | 192.168.56.21 |

The cluster uses Kubernetes v1.30.14, containerd, and Calico v3.28.0. The Prometheus release was installed in namespace prometheus using the prometheus-community/prometheus Helm chart.

### What the lab record confirms

- The Prometheus chart installation created Prometheus Server, Alertmanager, kube-state-metrics, Node Exporter, and Pushgateway resources.
- The Node Exporter DaemonSet reached 3 desired, 3 current, and 3 ready Pods.
- The initial Prometheus Server and Alertmanager Pods were Pending because their PVCs could not be bound while the cluster had no StorageClass.
- After the Local Path Provisioner and default local-path StorageClass were configured, Prometheus storage claims became Bound. The storage workflow is documented in Chapter 10.
- Later Grafana dashboards showed populated node and Kubernetes metrics from the Prometheus data source.
- A Grafana CPU alert evaluated Prometheus Node Exporter metrics and reached FIRING. The alert notification was successfully delivered to the Slack channel #k8s-alerts.

### What this chapter does not claim

The exported conversation includes teaching instructions and proposed commands as well as actual terminal output and screenshots. A proposed command is not proof that it was run. In particular:

- The transcript does not preserve a full Prometheus scrape configuration file or a complete Prometheus Targets-page export.
- The PromQL expressions below are documented as learning queries. The CPU expression is supported by the Grafana alert results; not every query is independently evidenced as having been typed into the Prometheus expression browser.
- The Grafana alert rules are confirmed as Grafana-managed rules. The transcript does not establish that a native PrometheusRule resource or Prometheus rule file was deployed.
- A Slack notification from Grafana is confirmed. A notification sent by Prometheus Alertmanager is not established.

These distinctions matter when writing an operational runbook or describing the project in an interview.

## 2. Prometheus and the monitoring model

Prometheus is an open-source monitoring and alerting system. It collects numeric measurements, stores them as time series, and provides PromQL for querying and analysis.

Typical monitoring questions include:

- Is a node reachable?
- How much CPU time is being spent in non-idle modes?
- How much memory is available?
- Is a filesystem approaching capacity?
- How many Pods are running, pending, or restarting?
- Is an application endpoint responding successfully?

Prometheus is commonly pull-based. It discovers or is configured with targets, then periodically sends HTTP requests to their metrics endpoints. The target exposes metrics in Prometheus exposition format, typically at /metrics.

### Pull model

~~~mermaid
flowchart LR
    P[Prometheus Server] -->|HTTP GET /metrics| E[Exporter or application]
    E -->|Metrics response| P
    P --> TSDB[(Time Series Database)]
    U[PromQL query] --> P
~~~

Prometheus does not normally require every monitored application to push every measurement to it. Instead, Prometheus initiates scrapes and records the samples returned by targets.

Pushgateway is a special-purpose component for short-lived or batch jobs that cannot be scraped reliably while running. It is not a general replacement for pull-based scraping and should not be used as a blanket proxy for every application.

## 3. Metrics Server versus Prometheus

These components solve different problems.

| Capability | Metrics Server | Prometheus |
|---|---|---|
| Main purpose | Current Kubernetes resource usage | Monitoring, time-series storage, querying, and alert evaluation |
| Typical data | CPU and memory usage | Exporter, application, infrastructure, and Kubernetes metrics |
| Main consumer | Kubernetes Metrics API, HPA, kubectl top | PromQL, dashboards, alerting integrations |
| Retention | Short-lived resource metrics | Configurable time-series retention |
| Query language | Kubernetes metrics API | PromQL |
| Typical command/UI | kubectl top | Prometheus expression browser and API |

Metrics Server is not a replacement for Prometheus. Prometheus is not a drop-in replacement for the Kubernetes Metrics API used by the standard HPA resource metrics pipeline.

For the kubelet → Metrics Server → Metrics API → HPA flow, refer to Chapter 11.

## 4. Prometheus architecture and components

The lab's monitoring stack included these components.

### 4.1 Prometheus Server

Prometheus Server performs the central monitoring work:

- Loads scrape and rule configuration.
- Discovers or reads configured scrape targets.
- Scrapes metrics endpoints on a schedule.
- Adds target labels and stores samples in its time-series database (TSDB).
- Evaluates PromQL queries and configured alerting/recording rules.
- Exposes an HTTP API and web interface.

The Prometheus Server is the component that answers PromQL queries.

### 4.2 Node Exporter

Node Exporter exposes operating-system and hardware-related metrics from a Linux host. Typical metric families include:

- CPU time counters.
- Memory totals and available memory.
- Filesystem capacity and availability.
- Network interface counters.
- System load and other host statistics.

The chart created a Node Exporter DaemonSet. The recorded output showed 3 desired, 3 current, and 3 ready Pods, corresponding to the three Linux cluster nodes.

Node Exporter is an exporter, not the time-series database. It exposes measurements; Prometheus scrapes and stores them.

### 4.3 kube-state-metrics

kube-state-metrics observes Kubernetes API objects and exposes their state as metrics. Examples include:

- Node conditions and labels.
- Pod phase and container status.
- Deployment desired and available replicas.
- StatefulSet replica state.
- Namespace and other Kubernetes object metadata.

It reports the state of Kubernetes objects; it is not the same as collecting host CPU or memory usage. Node Exporter and kube-state-metrics complement one another.

### 4.4 Alertmanager

Alertmanager receives alerts from Prometheus and handles alert grouping, deduplication, routing, silencing, and inhibition. It is separate from the Prometheus query engine.

The chart created an Alertmanager StatefulSet and Service. The initial Alertmanager Pod was Pending because its storage claim was not yet bound. Later installation output and monitoring stack summaries showed the component as part of the deployed stack.

Do not infer that a particular Alertmanager route or receiver was configured merely because the chart installed Alertmanager.

### 4.5 Pushgateway

Pushgateway allows short-lived batch jobs to publish metrics to a stable endpoint that Prometheus can scrape later. The chart created a Pushgateway Deployment and Service.

For continuously running services, prefer direct scraping of the service or its exporter. Pushgateway changes the collection pattern and introduces lifecycle and stale-metric considerations.

### 4.6 Component flow

~~~mermaid
flowchart TB
    subgraph K8S["Three-node Kubernetes cluster"]
      N1["Node Exporter on k8master1"]
      N2["Node Exporter on k8worker1"]
      N3["Node Exporter on k8worker2"]
      KSM["kube-state-metrics"]
      APP["Application / exporter metrics endpoints"]
    end
    P["Prometheus Server"]
    DB[("Prometheus TSDB")]
    Q["PromQL / HTTP API"]
    AM["Alertmanager"]
    G["Grafana"]
    P -->|"scrape /metrics"| N1
    P -->|"scrape /metrics"| N2
    P -->|"scrape /metrics"| N3
    P -->|"scrape metrics"| KSM
    P -->|"scrape metrics"| APP
    P --> DB
    DB --> Q
    Q --> G
    P -->|"firing alerts when rules are configured"| AM
~~~

This is a conceptual architecture diagram. It does not assert that every possible application endpoint or alert route was configured in the original lab.

## 5. Install and inspect the Prometheus Helm release

### 5.1 Installation recorded in the lab

The conversation records the following namespace and installation commands:

~~~bash
kubectl create namespace prometheus

helm install prometheus \
  prometheus-community/prometheus \
  -n prometheus
~~~

The exact chart version and Helm repository-add command were not preserved in the captured installation output. Do not assume that the chart's current version is the same as the version originally installed.

If setting up a fresh environment, add and refresh the community chart repository before installing:

~~~bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo update
helm search repo prometheus-community/prometheus
~~~

These repository commands are setup guidance for a new installation, not a claim about the exact commands recorded in the original session.

### 5.2 Inspect the release

~~~bash
helm list -n prometheus
helm status prometheus -n prometheus
helm get values prometheus -n prometheus
helm get manifest prometheus -n prometheus
~~~

Inspect Kubernetes resources:

~~~bash
kubectl get pods -n prometheus -o wide
kubectl get deployments,daemonsets,statefulsets -n prometheus
kubectl get services -n prometheus
kubectl get pvc -n prometheus
~~~

The initial captured resource listing included:

- prometheus-server Deployment and Service.
- prometheus-alertmanager StatefulSet and Services.
- prometheus-kube-state-metrics Deployment and Service.
- prometheus-prometheus-node-exporter DaemonSet and Service.
- prometheus-prometheus-pushgateway Deployment and Service.

The initial Node Exporter DaemonSet output showed 3/3 ready. The Prometheus Server and Alertmanager Pods were initially Pending; this was a storage provisioning problem, not evidence that Node Exporter itself was failing.

### 5.3 Prometheus UI access

The recorded Service was named prometheus-server and exposed Service port 80. The documented port-forward command is:

~~~bash
kubectl port-forward \
  -n prometheus \
  svc/prometheus-server \
  9090:80
~~~

Then open the following address on the machine running the port-forward command:

~~~text
http://localhost:9090
~~~

Keep the terminal running while using the browser. Stop the port-forward with Ctrl+C.

Port-forwarding is appropriate for a lab. For a shared or production environment, use a deliberately secured access path rather than exposing the Prometheus UI publicly without authentication and network controls.

## 6. Storage dependency and the Pending incident

The first Prometheus installation did not become fully ready immediately.

The captured output showed:

- prometheus-server with 0/2 containers ready and Pod status Pending.
- prometheus-alertmanager-0 with 0/1 containers ready and Pod status Pending.
- Node Exporter Pods running.
- kube-state-metrics and Pushgateway running.
- No StorageClass available at the time of the initial failure.

The diagnostic output identified the scheduling condition:

~~~text
0/3 nodes are available:
pod has unbound immediate PersistentVolumeClaims
~~~

The root cause was that the cluster did not yet have a StorageClass capable of dynamically provisioning the requested volumes. The Local Path Provisioner was installed and local-path was made the default StorageClass. The Prometheus and Alertmanager claims subsequently became Bound.

**Full storage procedure:** see [Chapter 10 — Storage Fundamentals and Dynamic Provisioning](Chapter_10_Storage_Fundamentals_and_Dynamic_Provisioning.md). This chapter intentionally does not repeat the full Local Path Provisioner installation and PVC troubleshooting runbook.

Operational lesson: a monitoring stack is itself a Kubernetes workload. Its Pods, PVCs, Services, resource requests, and scheduling constraints must be monitored and maintained. Monitoring does not become reliable merely because a Helm install command returned successfully.

## 7. Prometheus data model

Prometheus stores samples as time series.

A time series is identified by:

- A metric name.
- A set of labels.

Each sample has a timestamp and a numeric value.

Example:

~~~text
node_memory_MemAvailable_bytes{instance="192.168.56.20:9100",job="node-exporter"} 123456789
~~~

This is an illustrative sample, not a preserved historical value from the lab.

### 7.1 Metric names

Metric names describe what is measured. Examples include:

| Metric | Meaning |
|---|---|
| up | Whether a scrape target was successfully scraped |
| node_cpu_seconds_total | CPU time accumulated by CPU and mode |
| node_memory_MemAvailable_bytes | Available memory on a node |
| node_memory_MemTotal_bytes | Total memory on a node |
| node_filesystem_avail_bytes | Available bytes on a filesystem |
| kube_pod_info | Kubernetes Pod information exposed by kube-state-metrics |

Metric availability depends on the exporter, scrape configuration, exporter version, and metric relabeling.

### 7.2 Labels

Labels add dimensions to a metric. Common examples are:

- job — the scrape job name.
- instance — the target address.
- mode — CPU mode such as idle or user.
- cpu — CPU identifier.
- device — filesystem or block device.
- namespace — Kubernetes namespace.
- pod — Pod name.

A label set identifies one particular time series. Different label values create different series.

### 7.3 Cardinality

Cardinality is the number of unique time series. A metric with many possible values across labels can create a large number of series.

Avoid unbounded labels such as request IDs, session IDs, random tokens, or user-generated values. These can cause high memory usage and expensive queries.

Use labels for bounded dimensions such as environment, service, namespace, or status code where appropriate. Do not put sensitive information into metric labels.

## 8. Scraping, jobs, and targets

A scrape target is an endpoint Prometheus attempts to collect metrics from. A scrape job groups targets that share scrape configuration.

For example, a node-exporter job could contain three targets:

~~~yaml
scrape_configs:
  - job_name: node-exporter
    static_configs:
      - targets:
          - 192.168.56.10:9100
          - 192.168.56.20:9100
          - 192.168.56.21:9100
~~~

This is a simplified static example for learning. It is not asserted to be the exact scrape configuration used by the Helm chart. Helm chart defaults and Kubernetes service discovery may generate target configuration differently.

### 8.1 Scrape interval and timeout

- scrape_interval controls how often Prometheus scrapes a target.
- scrape_timeout controls how long Prometheus waits for a scrape response.
- A scrape timeout must not exceed the configured scrape interval.

A shorter interval gives more frequent samples but increases network, CPU, storage, and query load. Select intervals based on the operational need, not simply the smallest possible value.

### 8.2 Service discovery

In Kubernetes, target addresses change as Pods are rescheduled and Services or Endpoints change. Kubernetes service discovery allows Prometheus to discover targets from the Kubernetes API rather than relying on fixed Pod IPs.

Conceptually:

~~~mermaid
flowchart TD
    API["Kubernetes API"]
    SD["Prometheus Kubernetes service discovery"]
    RELABEL["Relabeling and target selection"]
    TARGETS["Discovered scrape targets"]
    SCRAPE["Scrape /metrics"]
    STORE[("Prometheus TSDB")]
    API --> SD --> RELABEL --> TARGETS --> SCRAPE --> STORE
~~~

Service discovery and relabeling are important because discovering every Pod does not mean every Pod should be scraped. The configuration must select the intended targets and metrics endpoints.

### 8.3 Inspect targets

In the Prometheus UI:

1. Open Status.
2. Select Targets.
3. Inspect each job and target.
4. Check the endpoint, last scrape, scrape duration, and last error.
5. Confirm that expected targets are UP.

The original learning session explicitly introduced node-exporter, prometheus, and kube-state-metrics as target groups to inspect. The later Grafana dashboards confirm that node and Kubernetes metrics were being ingested, but the exported transcript does not preserve a complete Targets-page table with every target's exact status and labels.

## 9. PromQL fundamentals

PromQL is Prometheus's query language. It can select time series, filter by labels, calculate rates, aggregate values, and evaluate expressions over time windows.

### 9.1 First query: up

~~~promql
up
~~~

The up metric is generated for scrape targets:

- 1 means the most recent scrape succeeded.
- 0 means the scrape failed.

Filter by a job:

~~~promql
up{job="node-exporter"}
~~~

The job name must match the actual target label shown in the Prometheus Targets page. Do not assume a job label from an example; inspect the live target labels first.

Find failed targets:

~~~promql
up == 0
~~~

If a target disappears entirely from service discovery, it may no longer have an up series. Therefore, up == 0 alone does not detect every possible missing-target scenario.

### 9.2 Selectors and label matching

~~~promql
node_memory_MemAvailable_bytes
~~~

Select a specific instance:

~~~promql
node_memory_MemAvailable_bytes{instance="192.168.56.20:9100"}
~~~

Select with multiple label conditions:

~~~promql
node_cpu_seconds_total{mode="idle",instance="192.168.56.20:9100"}
~~~

Common matchers:

- = exact match.
- != does not equal.
- =~ regular-expression match.
- !~ does not match a regular expression.

### 9.3 Instant vectors and range vectors

An instant vector returns the latest sample for each selected time series at the evaluation time.

A range vector selects samples over a time interval. For example:

~~~promql
node_cpu_seconds_total[5m]
~~~

Many functions such as rate() operate on range vectors.

### 9.4 Counters, gauges, and rate()

A counter generally increases over time and may reset when a process restarts. CPU seconds are represented by counters.

A gauge can move up or down. Available memory and current temperature are examples of gauge-like measurements.

Use rate() to estimate the per-second average increase of a counter over a time window:

~~~promql
rate(node_cpu_seconds_total[5m])
~~~

For counters, rate() is generally preferable to subtracting two samples manually because it handles counter resets and extrapolation.

### 9.5 CPU usage percentage per node

The lab's Grafana CPU alert used the following expression:

~~~promql
100 - (
  avg by(instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  ) * 100
)
~~~

Interpretation:

1. Select CPU time series whose mode is idle.
2. Calculate the per-second rate over five minutes.
3. Average idle fractions by instance across CPUs.
4. Convert the idle fraction to a percentage.
5. Subtract from 100 to estimate non-idle CPU percentage.

The expression is an estimate of CPU busy percentage, not a direct instantaneous reading. It depends on the Node Exporter CPU metrics and the labels returned by the scrape.

### 9.6 Memory usage percentage per node

~~~promql
(
  1 -
  (
    node_memory_MemAvailable_bytes
    /
    node_memory_MemTotal_bytes
  )
) * 100
~~~

This estimates memory in use as a percentage of total memory using MemAvailable. It is often more useful than treating all unused memory as free, because Linux uses memory for caches that can be reclaimed.

### 9.7 Filesystem available percentage

~~~promql
100 *
node_filesystem_avail_bytes{
  fstype!~"tmpfs|overlay|squashfs"
}
/
node_filesystem_size_bytes{
  fstype!~"tmpfs|overlay|squashfs"
}
~~~

This is a starting example. In an operational dashboard, filter out pseudo-filesystems and mount points that are not relevant to the alert. Validate labels and filesystem semantics for the environment.

### 9.8 Kubernetes object metrics

Examples from kube-state-metrics include:

~~~promql
kube_pod_info
~~~

~~~promql
kube_pod_status_phase{phase="Pending"}
~~~

~~~promql
kube_deployment_status_replicas_available
~~~

~~~promql
kube_deployment_spec_replicas
~~~

These metrics describe Kubernetes API object state. They are not the same as container CPU or memory usage. Metric names and labels can vary with kube-state-metrics version and configuration, so verify them in the live Prometheus expression browser.

### 9.9 Aggregation examples

Average CPU busy percentage across all instances:

~~~promql
avg(
  100 - (
    avg by(instance) (
      rate(node_cpu_seconds_total{mode="idle"}[5m])
    ) * 100
  )
)
~~~

Highest memory usage by node:

~~~promql
topk(
  3,
  (
    1 -
    node_memory_MemAvailable_bytes
    /
    node_memory_MemTotal_bytes
  ) * 100
)
~~~

Count targets currently reporting successful scrapes:

~~~promql
sum(up)
~~~

These are query exercises. Validate the result labels and metric availability before turning an expression into an alert.

## 10. What was verified in the lab

### 10.1 Node Exporter DaemonSet

The captured Kubernetes resource listing showed:

~~~text
daemonset.apps/prometheus-prometheus-node-exporter
DESIRED 3
CURRENT 3
READY 3
~~~

This confirms that the Node Exporter DaemonSet had one ready Pod per node at that point in the lab.

### 10.2 Grafana received Prometheus metrics

Later screenshots and the conversation record confirmed:

- The Prometheus data source was functioning in Grafana.
- The Kubernetes Views Global dashboard showed 3 nodes.
- The dashboard showed 31 running Pods at the time of the screenshot.
- CPU and RAM metrics were populated.
- Node Exporter Full showed node-level metrics.

These are time-specific observations from the lab, not permanent expected counts.

### 10.3 CPU alert query returned three node values

The Grafana CPU alert used the CPU expression from Section 9.5. The recorded results were:

| Node endpoint | CPU busy percentage |
|---|---:|
| 192.168.56.10:9100 | 65% |
| 192.168.56.20:9100 | 89% |
| 192.168.56.21:9100 | 35% |

The lab threshold was deliberately set to above 20% to make testing easy. All three instances exceeded that threshold, so Grafana showed three alert instances in FIRING state. The worker1 value was observed while the stress test was running.

This is evidence that Grafana successfully queried Prometheus and evaluated the alert expression. It is not evidence that Prometheus native alert rules were configured.

## 11. Grafana alerting versus Prometheus alerting

Prometheus and Grafana can both participate in alerting, but their alert rules and delivery paths are not interchangeable.

### Grafana-managed alerting

The lab created Grafana alert rules using the Prometheus data source. The CPU rule used:

- Query A: the CPU busy percentage expression.
- Reduce expression: Last.
- Threshold expression: Above 20.
- Evaluation interval: 1 minute.
- Pending period: 1 minute.
- Folder: k8s-alerts.

The alert moved to FIRING while all three node values were above the low test threshold.

The lab also documented Grafana rules for memory usage and node-down detection. The transcript's screenshots and confirmation specifically establish the CPU rule as FIRING and its Slack notification as delivered. Do not treat the memory and node-down exercises as equally verified unless you have current evidence for them.

### Grafana-to-Slack delivery path

The successful notification flow was:

~~~mermaid
flowchart TD
    N["Node Exporter metrics"] --> P["Prometheus scrape and TSDB"]
    P --> D["Grafana Prometheus data source"]
    D --> R["Grafana CPU alert rule"]
    R --> NP["Grafana notification policy"]
    NP --> CP["Grafana Slack contact point"]
    CP --> S["Slack channel: #k8s-alerts"]
~~~

The conversation confirms that a High CPU Usage alert reached #k8s-alerts. The captured notification title was [FIRING:2] High CPU Usage.

Treat the Slack incoming webhook URL as a secret. Never place it in Git, Kubernetes manifests committed to a public repository, screenshots, or this documentation.

### Prometheus-native alerting

Prometheus-native alert rules are defined in rule files and evaluated by Prometheus. Firing alerts are sent to Alertmanager, which handles routing and notification delivery.

A minimal illustrative rule file:

~~~yaml
groups:
  - name: node-health
    rules:
      - alert: NodeExporterDown
        expr: up{job="node-exporter"} == 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Node exporter scrape failed"
          description: "Prometheus cannot scrape {{ $labels.instance }}."
~~~

This is a recommended learning example, not a file confirmed as deployed in the original lab. Confirm the real job label before applying it.

An alert rule has:

- alert — alert name.
- expr — PromQL expression that determines whether the alert is active.
- for — duration the expression must remain active before the alert fires.
- labels — metadata such as severity or team.
- annotations — human-readable context for operators.

The high-level native flow is:

~~~mermaid
flowchart LR
    T["Scraped time series"] --> Q["Prometheus rule evaluation"]
    Q --> A["Pending alert"]
    A --> F["Firing alert"]
    F --> AM["Alertmanager"]
    AM --> R["Grouping and routing"]
    R --> N["Configured receiver"]
~~~

### Important distinction

The original lab's confirmed Slack integration used Grafana Contact Point and Notification Policy. It does not prove that Prometheus Alertmanager was configured with a Slack receiver. These are two possible alert delivery architectures; document and operate the one actually configured.

## 12. Recommended Prometheus-native rule exercise

This exercise extends the existing lab. It is not marked as completed.

### 12.1 Create a rule file

Create a file named node-alerts.yml:

~~~yaml
groups:
  - name: node-alerts
    rules:
      - alert: NodeExporterDown
        expr: up{job="node-exporter"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Node exporter is unavailable"
          description: "The target {{ $labels.instance }} has failed scrapes for 2 minutes."

      - alert: HighNodeCPU
        expr: |
          100 - (
            avg by(instance) (
              rate(node_cpu_seconds_total{mode="idle"}[5m])
            ) * 100
          ) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "Node CPU busy percentage has remained above 80% for 5 minutes."
~~~

The job selector and expression must be tested against the actual metric labels. The threshold and duration are examples, not universal production settings.

### 12.2 Validate before deployment

Before loading a rule file:

1. Verify the metric exists in the Prometheus expression browser.
2. Confirm the job and instance labels.
3. Check the expression returns the intended series.
4. Confirm the Prometheus version supports the rule syntax.
5. Review the Helm chart's values and configuration mechanism.
6. Use a ConfigMap or chart-supported rule mechanism rather than editing a running Pod's filesystem.

For a file-based configuration, promtool can validate rule syntax:

~~~bash
promtool check rules node-alerts.yml
~~~

The promtool binary must be available in the environment where this command is run. Do not assume it is installed on the Vagrant host.

### 12.3 Understand alert states

- Inactive: the expression is not currently producing an active alert.
- Pending: the expression is active, but the configured for duration has not elapsed.
- Firing: the expression has remained active for the required duration.

An alert can return to inactive when its expression no longer matches. A firing alert is not proof that a notification was delivered; verify Alertmanager routing and receiver delivery separately.

## 13. Troubleshooting Prometheus

### 13.1 Prometheus Pod is Pending

Check:

~~~bash
kubectl get pods -n prometheus
kubectl get pvc -n prometheus
kubectl get storageclass
kubectl describe pod <prometheus-pod> -n prometheus
kubectl describe pvc <claim-name> -n prometheus
~~~

Look for unbound PVCs, missing StorageClass, insufficient resources, taints, or scheduling constraints.

The original incident was caused by missing dynamic storage provisioning. Follow Chapter 10 for the complete Local Path Provisioner recovery procedure.

### 13.2 Target is DOWN

In the Prometheus UI, open Status → Targets and inspect Last Error.

Then investigate:

~~~bash
kubectl get pods -n prometheus -o wide
kubectl get svc -n prometheus
kubectl get endpoints -n prometheus
kubectl logs -n prometheus <prometheus-pod>
kubectl describe pod -n prometheus <exporter-pod>
~~~

Common causes:

- Exporter Pod is not Ready.
- Target address or port is incorrect.
- Service has no matching endpoints.
- Network policy or routing blocks traffic.
- Metrics path is incorrect.
- TLS or authentication configuration does not match the target.
- The application is not exposing a Prometheus metrics endpoint.

### 13.3 Target does not appear at all

A target absent from the Targets page may not have been discovered or selected. Check:

- scrape_configs and service discovery configuration.
- Kubernetes RBAC permissions used by Prometheus.
- Namespace and label selectors.
- Relabeling rules that may drop the target.
- Whether the expected Service, Pod, or Endpoint exists.

A target that is not discovered is different from a discovered target whose scrape is failing.

### 13.4 PromQL returns no data

Check:

1. Does the metric name exist?
2. Are label names and values correct?
3. Is the target currently discovered?
4. Is the target UP?
5. Is the selected time range appropriate?
6. Does the exporter expose the metric?
7. Has metric relabeling dropped the series?

Start with a broad metric query, then add label filters one at a time.

### 13.5 Query is slow or expensive

Potential causes include:

- Very broad selectors.
- Large range windows.
- High-cardinality metrics.
- Expensive regular expressions.
- Excessive aggregation over many series.
- Too many dashboard panels refreshing simultaneously.

Prefer selective labels and recording rules for repeatedly used expensive calculations where appropriate.

### 13.6 Prometheus data appears to disappear

Check:

~~~bash
kubectl get pvc -n prometheus
kubectl describe pod -n prometheus <prometheus-pod>
kubectl logs -n prometheus <prometheus-pod>
helm status prometheus -n prometheus
~~~

Prometheus retention, PVC capacity, Pod restarts, and storage lifecycle all affect data availability. The Local Path Provisioner used in this lab stores data on a node-local path; it is not replicated, highly available storage. Review Chapter 10 before treating the lab's storage design as production-ready.

### 13.7 Exporter is running but metrics are missing

A Ready Pod does not by itself prove Prometheus is scraping it. Confirm the metrics endpoint responds, the target is discovered, the scrape succeeds, and the expected metric has not been filtered.

For Node Exporter, verify its Service and target port rather than assuming a particular Pod IP or endpoint remains constant.

## 14. Security and production considerations

A learning deployment is not a production monitoring architecture.

- Restrict access to the Prometheus and Alertmanager web interfaces.
- Do not expose monitoring endpoints or administrative UIs publicly without appropriate authentication and network controls.
- Protect scrape credentials, TLS private keys, webhook URLs, and other secrets.
- Apply least-privilege Kubernetes RBAC to service discovery.
- Set storage capacity and retention deliberately.
- Monitor Prometheus itself, including target health, scrape failures, storage usage, restarts, and resource consumption.
- Use persistent storage with an appropriate availability and backup strategy.
- Review metric cardinality and avoid sensitive or unbounded labels.
- Keep chart versions and configuration under change control.
- Validate alert thresholds and durations against normal operating behavior to avoid alert fatigue.
- Test notification delivery and recovery, not just alert-rule creation.

The lab's local-path storage is suitable for learning dynamic provisioning, but it is node-local and should not be described as replicated or highly available.

## 15. Hands-on exercises

### Exercise 1 — Inspect the deployed components

Run:

~~~bash
helm list -n prometheus
kubectl get pods -n prometheus -o wide
kubectl get daemonsets,deployments,statefulsets -n prometheus
kubectl get svc -n prometheus
kubectl get pvc -n prometheus
~~~

Record the release status, the number of Node Exporter Pods, and the readiness of Prometheus Server and Alertmanager.

### Exercise 2 — Open the Prometheus UI

~~~bash
kubectl port-forward -n prometheus svc/prometheus-server 9090:80
~~~

Open http://localhost:9090 and inspect Status → Targets.

Record the actual job names, target addresses, and health states. Do not copy example job labels without checking them.

### Exercise 3 — Explore the up metric

Run:

~~~promql
up
~~~

Then:

~~~promql
up == 0
~~~

Explain the difference between a failed scrape and a target that is not discovered.

### Exercise 4 — Explore node CPU and memory

Run the CPU expression in Section 9.5 and the memory expression in Section 9.6. Compare the three instances and note how the values change over time.

### Exercise 5 — Explore Kubernetes object metrics

Try:

~~~promql
kube_pod_info
~~~

~~~promql
kube_deployment_status_replicas_available
~~~

Inspect the labels and explain how namespace, Pod, and deployment dimensions are represented.

### Exercise 6 — Compare Grafana and Prometheus alerting

In Grafana, inspect the existing High CPU Usage rule, its Prometheus query, evaluation settings, and Slack contact point. Separately review the example native Prometheus rule in Section 12.

Write down which component evaluates each rule and which component sends the notification.

### Exercise 7 — Controlled CPU alert test

Only perform this test when the worker VM has enough spare capacity.

On a worker node, a temporary stress process can generate CPU load:

~~~bash
sudo apt update
sudo apt install stress -y
stress --cpu 2 --timeout 300
~~~

Observe the Grafana CPU panel and alert state. Stop the stress process if it causes instability. Restore the production-like threshold after the test.

The historical conversation records a CPU stress test and the resulting Grafana alert values, but repeat the exercise to validate the current state of the cluster.

## 16. Command reference

| Task | Command |
|---|---|
| List release | helm list -n prometheus |
| Release status | helm status prometheus -n prometheus |
| List Pods | kubectl get pods -n prometheus -o wide |
| List Services | kubectl get svc -n prometheus |
| List DaemonSets | kubectl get ds -n prometheus |
| List PVCs | kubectl get pvc -n prometheus |
| Describe Pod | kubectl describe pod <pod> -n prometheus |
| Read Pod logs | kubectl logs <pod> -n prometheus |
| Port-forward UI | kubectl port-forward -n prometheus svc/prometheus-server 9090:80 |
| Check chart values | helm get values prometheus -n prometheus |
| Check rendered release | helm get manifest prometheus -n prometheus |

## 17. Knowledge-check questions

1. What is Prometheus, and what problem does it solve?
2. Explain the pull-based scrape model.
3. What is a scrape target?
4. What is the difference between a scrape job and a target?
5. What is an exporter?
6. What is the role of Node Exporter?
7. What information does kube-state-metrics expose?
8. How is Prometheus different from Metrics Server?
9. What is a time series?
10. What is the purpose of labels?
11. What does the up metric indicate?
12. What is the difference between a counter and a gauge?
13. Why is rate() commonly used with CPU counters?
14. How does Kubernetes service discovery help Prometheus?
15. What does a target DOWN status tell you?
16. What is the difference between a target that is DOWN and a target that is absent?
17. What is Alertmanager responsible for?
18. How does Grafana-managed alerting differ from Prometheus-native alerting?
19. Why can high-cardinality labels become a problem?
20. Why did the initial Prometheus Server and Alertmanager Pods remain Pending in this lab?
21. Why is node-local storage not equivalent to highly available storage?
22. Why should alert thresholds be tested and tuned before production use?

## 18. Chapter completion checklist

- [ ] Explain the Prometheus pull model and scrape lifecycle.
- [ ] Distinguish Prometheus from Metrics Server.
- [ ] Explain Prometheus Server, Node Exporter, kube-state-metrics, Pushgateway, and Alertmanager.
- [ ] Inspect the Prometheus Helm release and its Kubernetes resources.
- [ ] Verify the Node Exporter DaemonSet across all three nodes.
- [ ] Access the Prometheus UI through port-forwarding.
- [ ] Inspect live scrape targets and their labels.
- [ ] Query up and interpret its values.
- [ ] Query node CPU and memory metrics with PromQL.
- [ ] Explain counters, gauges, labels, and cardinality.
- [ ] Explain the difference between Grafana and Prometheus alert rules.
- [ ] Troubleshoot Pending Pods, failed scrapes, missing metrics, and query issues.
- [ ] Explain the storage dependency and refer to Chapter 10 for the full storage runbook.

## 19. References

- Prometheus documentation: https://prometheus.io/docs/introduction/overview/
- Prometheus getting started: https://prometheus.io/docs/prometheus/latest/getting_started/
- PromQL basics: https://prometheus.io/docs/prometheus/latest/querying/basics/
- PromQL functions: https://prometheus.io/docs/prometheus/latest/querying/functions/
- Prometheus configuration: https://prometheus.io/docs/prometheus/latest/configuration/configuration/
- Alerting rules: https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- Alertmanager: https://prometheus.io/docs/alerting/latest/alertmanager/
- Node Exporter: https://github.com/prometheus/node_exporter
- kube-state-metrics: https://github.com/kubernetes/kube-state-metrics
- Prometheus Community Helm charts: https://github.com/prometheus-community/helm-charts
- Grafana alerting documentation: https://grafana.com/docs/grafana/latest/alerting/

---

**End of Chapter 12 — Prometheus Monitoring**

**Next chapter:** Chapter 13 — Grafana Dashboards and Persistence.
