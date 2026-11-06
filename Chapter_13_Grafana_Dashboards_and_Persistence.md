# Chapter 13 — Grafana Dashboards and Persistence

## Chapter objectives

This chapter documents the Grafana work from the three-node Vagrant Kubernetes lab. It builds on Chapter 12, where Prometheus was installed and its scrape targets and PromQL metrics were introduced.

The practice covered:

- Installing Grafana with Helm in namespace monitoring.
- Exposing Grafana through a Kubernetes LoadBalancer Service backed by MetalLB.
- Connecting Grafana to the Prometheus data source.
- Creating custom dashboards and PromQL panels.
- Importing three community dashboards.
- Creating Grafana-managed CPU, memory, and node-health alert rules.
- Creating a Slack app and Incoming Webhook.
- Configuring the Grafana Slack contact point and notification policy.
- Enabling persistent storage with a 5 GiB PVC using the local-path StorageClass.
- Troubleshooting Grafana database persistence, Pod startup, volume permissions, and Service exposure.
- Backing up the Grafana SQLite database.

By the end, you should be able to explain Grafana's role, configure data sources, build dashboards, import community dashboards, configure alert rules and notification routing, explain Grafana persistence, troubleshoot volume and startup problems, and plan a backup and recovery procedure.

## 1. Lab environment and evidence boundaries

The final Kubernetes lab contains one control-plane node and two worker nodes:

| Node | Role | Host-only IP |
|---|---|---|
| k8master1 | Control plane | 192.168.56.10 |
| k8worker1 | Worker | 192.168.56.20 |
| k8worker2 | Worker | 192.168.56.21 |

The cluster uses Kubernetes v1.30.14, containerd, Calico v3.28.0, and MetalLB for LoadBalancer Services in the host-only network.

The monitoring stack used two namespaces:

- prometheus — Prometheus Server and its chart components.
- monitoring — Grafana.

Historical MetalLB addresses recorded during the practice session:

| Service | Historical external IP |
|---|---|
| nginx-lb | 192.168.56.240 |
| ingress-nginx-controller | 192.168.56.241 |
| prometheus-server | 192.168.56.242 |
| grafana | 192.168.56.243 |

These are historical observations, not guaranteed assignments. MetalLB can allocate a different address after a rebuild or configuration change. Always inspect the live Service.

### Evidence classification

**Recorded and verified in the practice session:**

- Grafana was installed with Helm in namespace monitoring.
- Grafana was accessed through a LoadBalancer address.
- The Prometheus data source successfully queried the Prometheus API.
- Custom PromQL panels were created.
- Community dashboards 15757, 7249, and 1860 were imported and used.
- Grafana CPU alert evaluation reached FIRING.
- The Slack contact point and notification policy were configured.
- A CPU alert notification was delivered to the Slack channel #k8s-alerts.
- Grafana persistence was enabled with a 5 GiB local-path PVC, and the PVC reached Bound.
- A later Grafana Pod startup issue was resolved; the Pod returned to 1/1 Running and the persistent database was used.
- A manual database backup procedure was documented.

**Important historical caveat:** Grafana was initially installed without a PVC. Enabling persistence later created a new PVC and a new database. The previous configuration stored only in the old container filesystem was not automatically copied into the new PVC. The dashboards, data source, alert rules, and Slack integration had to be recreated after that transition. Later restart/recovery work demonstrated that the database on the PVC could be reused after the permission issue was resolved.

This distinction is essential: a PVC protects data written to that PVC; it does not retroactively migrate data from an earlier ephemeral container filesystem.

## 2. What is Grafana?

Grafana is an observability and visualization platform. It connects to data sources, queries data, and presents results through dashboards and panels. It can also evaluate alert rules and route notifications through configured contact points and notification policies.

In this lab:

- Prometheus collected and stored metrics.
- Grafana queried Prometheus through its data source integration.
- Dashboards presented node, cluster, namespace, Pod, and workload information.
- Grafana alert rules evaluated selected PromQL queries.
- Grafana's notification policy routed alerts to the Slack contact point.
- Slack delivered notifications to #k8s-alerts.

Grafana does not replace Prometheus. It visualizes and evaluates data from Prometheus and other supported data sources.

## 3. Architecture of the monitoring and alerting stack

The following diagram represents the alerting path configured in the lab. It is a Grafana-managed alerting route; it should not be confused with a Prometheus Alertmanager Slack receiver.

~~~mermaid
flowchart TD
    subgraph CLUSTER["Kubernetes monitoring sources"]
      NODE["Node Exporter<br/>CPU, memory, filesystem, network"]
      KSM["kube-state-metrics<br/>Kubernetes object state"]
      APP["Application metrics endpoints"]
    end
    PROM["Prometheus Server<br/>Scrape and TSDB"]
    DS["Grafana Prometheus data source"]
    DASH["Grafana dashboards<br/>Panels and PromQL"]
    RULE["Grafana alert rule<br/>Query → Reduce → Threshold"]
    POLICY["Grafana notification policy"]
    CP["Grafana Slack contact point<br/>slack-k8s-alerts"]
    SLACK["Slack channel<br/>#k8s-alerts"]
    OPS["Operations / engineer"]
    NODE --> PROM
    KSM --> PROM
    APP --> PROM
    PROM --> DS
    DS --> DASH
    DS --> RULE
    RULE --> POLICY
    POLICY --> CP
    CP --> SLACK
    SLACK --> OPS
~~~

### 3.1 Metric collection path

1. Node Exporter exposes Linux host metrics.
2. kube-state-metrics exposes Kubernetes object state.
3. Prometheus scrapes metrics endpoints and stores time series.
4. Grafana connects to Prometheus through a configured data source.
5. Dashboard panels execute PromQL and render the results.

### 3.2 Alert delivery path

1. A Grafana alert rule evaluates a query against the Prometheus data source.
2. The rule's expressions reduce and compare the query result to a threshold.
3. When the alert transitions to a firing state, Grafana evaluates its notification policy.
4. The policy selects the slack-k8s-alerts contact point.
5. The Slack integration posts the notification to #k8s-alerts.

**Important:** Prometheus Alertmanager was installed as part of the Prometheus Helm chart, but the Slack integration confirmed in this practice session was configured in Grafana. The recorded path is Grafana alert rule → Grafana notification policy → Grafana Slack contact point → Slack. Do not document this as an Alertmanager Slack receiver.

## 4. Install Grafana with Helm

### 4.1 Helm repository and installation

The recorded installation command was:

~~~bash
helm install grafana grafana/grafana \
  -n monitoring
~~~

For a fresh environment, add and refresh the Grafana chart repository first:

~~~bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm search repo grafana/grafana
~~~

The repository setup commands are repeatable setup guidance. The exact chart version used in the original installation was not preserved in the captured output.

### 4.2 Inspect the release

~~~bash
helm list -n monitoring
helm status grafana -n monitoring
helm get values grafana -n monitoring
helm history grafana -n monitoring
~~~

Inspect Kubernetes resources:

~~~bash
kubectl get pods -n monitoring -o wide
kubectl get deployments -n monitoring
kubectl get services -n monitoring
kubectl get pvc -n monitoring
~~~

### 4.3 Access Grafana

The Grafana Service was changed to type LoadBalancer. The recorded patch was:

~~~bash
kubectl patch svc grafana \
  -n monitoring \
  -p '{"spec":{"type":"LoadBalancer"}}'
~~~

Inspect the current external address:

~~~bash
kubectl get svc grafana -n monitoring
~~~

During the recorded session Grafana was accessed at:

~~~text
http://192.168.56.243
~~~

This address is historical. Use the current EXTERNAL-IP returned by kubectl get svc.

The Grafana chart stores an administrator password in a Kubernetes Secret. Retrieve it without placing the password in documentation or shell history:

~~~bash
kubectl get secret \
  -n monitoring grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode
~~~

The default username in the lab was admin. Treat the retrieved password as a secret. Do not paste it into chat, commit it to Git, or include it in screenshots.

### 4.4 Keep Service exposure in Helm values

A key issue appeared when Grafana persistence was enabled through a Helm upgrade: the Service returned to the chart's default ClusterIP type. Grafana still had storage, but the previous LoadBalancer address was no longer assigned.

The temporary recovery command was the Service patch shown above. For repeatable deployments, manage both persistence and Service type in a values file so future Helm upgrades do not silently revert Service exposure.

Example values file:

~~~yaml
persistence:
  enabled: true
  size: 5Gi
  storageClassName: local-path

service:
  type: LoadBalancer
~~~

Apply the values with:

~~~bash
helm upgrade grafana grafana/grafana \
  -n monitoring \
  -f grafana-values.yaml
~~~

This is a recommended configuration pattern based on the issue encountered. Before applying it to an existing release, inspect current values and chart version. Preserve any other required values instead of replacing them accidentally.

## 5. Configure the Prometheus data source

Grafana needs a data source before dashboards can query Prometheus.

### 5.1 Data source URL

The recorded Prometheus Service had external IP 192.168.56.242. The Prometheus URL configured in Grafana was:

~~~text
http://192.168.56.242
~~~

This was the historical external address in the lab. The address can change after a rebuild. Inspect the current Service:

~~~bash
kubectl get svc -n prometheus
~~~

For an in-cluster Grafana deployment, Kubernetes DNS is generally preferable to relying on a MetalLB external IP. The Prometheus Service DNS name used by this release can be entered as:

~~~text
http://prometheus-server.prometheus.svc.cluster.local
~~~

Verify the actual Service name, namespace, and port before using the DNS URL. The historical UI setup used the external Prometheus address above, and the connection test successfully queried the Prometheus API.

### 5.2 Add the data source in Grafana

In the Grafana UI:

1. Open Connections.
2. Open Data sources.
3. Select Add data source.
4. Choose Prometheus.
5. Enter the reachable Prometheus URL.
6. Save and test the data source.

The recorded validation succeeded: Grafana successfully queried the Prometheus API.

### 5.3 Useful first queries

Use Grafana Explore or a dashboard panel to run:

~~~promql
count(kube_node_info)
~~~

~~~promql
count(kube_pod_info)
~~~

~~~promql
count(up == 1)
~~~

These queries were used as introductory exercises. Returned values depend on the current cluster and scrape configuration.

## 6. Build a custom dashboard

Before importing community dashboards, the practice session created custom panels to understand PromQL and Grafana visualizations.

### 6.1 Cluster node count

Query:

~~~promql
count(kube_node_info)
~~~

Suggested panel title: Node count. Note that count(kube_node_info) counts node information series; it does not by itself guarantee that each node is Ready.

For a Ready-node count, use an appropriate kube-state-metrics node condition metric and validate its labels and semantics in the installed version.

### 6.2 Running Pod count

A simple Pod information count:

~~~promql
count(kube_pod_info)
~~~

This counts Pod information series and may include Pods in different phases. If the panel specifically means Running Pods, use a phase-filtered metric such as:

~~~promql
sum(kube_pod_status_phase{phase="Running"})
~~~

Verify the metric and labels in the live data source before relying on the panel.

### 6.3 CPU usage percentage

The CPU query used in the alert and dashboard work was:

~~~promql
100 - (
  avg by(instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  ) * 100
)
~~~

Suggested panel title: CPU Usage %.

The query estimates non-idle CPU time as a percentage per Node Exporter instance. It averages idle CPU time across CPU cores and subtracts that percentage from 100.

### 6.4 Memory usage percentage

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

Suggested panel title: Memory Usage %.

Using MemAvailable is generally more useful than treating all Linux cache memory as unavailable.

### 6.5 Disk usage percentage

The recorded root-filesystem panel used:

~~~promql
100 - (
  node_filesystem_avail_bytes{
    mountpoint="/",
    fstype!="tmpfs"
  }
  /
  node_filesystem_size_bytes{
    mountpoint="/",
    fstype!="tmpfs"
  }
) * 100
~~~

Suggested panel title: Disk Usage %.

For a production dashboard, filter pseudo-filesystems and devices deliberately. Validate mountpoint and fstype labels because host layouts differ.

### 6.6 Pod count by namespace

~~~promql
count(kube_pod_info) by (namespace)
~~~

Suggested visualization: Pie chart or bar chart.

This helps identify which namespaces contain the largest number of Pod objects. It is not a measure of CPU or memory consumption.

### 6.7 Dashboard design considerations

A useful cluster overview should answer operational questions quickly:

| Panel | Question answered |
|---|---|
| Node health | Are the cluster nodes available? |
| CPU usage | Which node or workload is CPU constrained? |
| Memory usage | Is memory pressure increasing? |
| Disk usage | Is a filesystem approaching capacity? |
| Running Pods | Are expected workloads present? |
| Namespace usage | Which namespace owns the resources? |
| Network traffic | Is network throughput changing unexpectedly? |
| Pod restarts | Are workloads repeatedly failing? |

Use consistent units, meaningful panel titles, useful legends, and sensible time ranges. Avoid presenting a count of objects as a readiness metric unless the query explicitly measures readiness.

## 7. Import the community dashboards

Three community dashboards were used in the practice session.

| Dashboard ID | Name | Main focus |
|---|---|---|
| 15757 | Kubernetes / Views / Global | Cluster, namespace, Pod, and workload overview |
| 7249 | Kubernetes Cluster Monitoring (via Prometheus) | Running Pods, cluster CPU, node load, and Prometheus metrics |
| 1860 | Node Exporter Full | Node CPU, memory, disk, filesystem, network, and load average |

The practice record confirms that all three dashboards were imported and used with the Prometheus data source.

### 7.1 Import procedure

In Grafana:

1. Open Dashboards.
2. Select New → Import.
3. Enter one dashboard ID.
4. Load the dashboard.
5. Select the Prometheus data source when prompted.
6. Import.
7. Repeat for the remaining dashboard IDs.

Dashboard catalogue entries can change over time. If an imported dashboard asks for a different data source or displays no data, inspect its variables, data source mapping, metric names, and required exporter versions.

### 7.2 What the dashboards showed

The recorded Kubernetes / Views / Global screenshot showed:

- 3 nodes.
- 31 running Pods at that time.
- CPU usage metrics populated.
- RAM usage metrics populated.
- Kubernetes resource counts populated.

Node Exporter Full showed node-level metrics from Node Exporter.

The Kubernetes Cluster Overview dashboard showed running Pods, cluster CPU usage, and node load. These counts and values are historical snapshots, not permanent expected values.

### 7.3 Dashboard variables

Community dashboards commonly use variables for:

- Prometheus data source.
- Cluster.
- Node or instance.
- Namespace.
- Pod.
- Device or filesystem.

If a dashboard is blank, inspect variable dropdowns before assuming Prometheus is broken. A dashboard may be filtering for a label or job name that does not exist in the current scrape configuration.

## 8. Grafana alerting fundamentals

Grafana alerting evaluates queries and expressions and changes alert state based on configured conditions.

The rule workflow in the practice session was:

~~~text
Query
  ↓
Reduce
  ↓
Threshold
  ↓
Alert state
  ↓
Notification policy
  ↓
Contact point
~~~

### 8.1 Alert states

- Normal — the condition is not currently met.
- Pending — the condition is met, but the configured pending period has not elapsed.
- Firing — the condition has remained active for the configured duration.
- Recovering/Normal — the expression returns below threshold and the alert resolves according to its evaluation behavior.

An alert can be firing without a notification being delivered. The notification policy and contact point must also be configured and healthy.

### 8.2 High CPU Usage alert

The recorded Grafana alert used:

~~~promql
100 - (
  avg by(instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  ) * 100
)
~~~

Rule settings from the practice session:

| Setting | Value |
|---|---|
| Rule name | High CPU Usage |
| Data source | Prometheus |
| Reduce expression | Last |
| Threshold | Above 20 |
| Evaluate every | 1 minute |
| Pending period | 1 minute |
| Folder | k8s-alerts |

The 20% threshold was deliberately low for a lab test. It is not a production recommendation. The practice discussion suggested considering a more realistic threshold such as 80% or 85%, after measuring the normal workload baseline.

The captured results were:

| Node Exporter endpoint | CPU busy percentage |
|---|---:|
| 192.168.56.10:9100 | 65% |
| 192.168.56.20:9100 | 89% |
| 192.168.56.21:9100 | 35% |

All three values exceeded the temporary 20% threshold, so Grafana showed three alert instances firing. The worker1 reading was captured while the CPU stress test was running.

### 8.3 High Memory Usage alert

The memory query used in the lab was:

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

A threshold above 80% was documented as an example for the memory alert.

The transcript lists High Memory Usage among the alert rules created, but the clearest captured firing confirmation and Slack delivery evidence relates to High CPU Usage. Do not assume the memory alert was independently load-tested unless you repeat the test.

### 8.4 Node Down alert

The practice notes used the up metric:

~~~promql
up == 0
~~~

A failed scrape returns 0 for a discovered target. A target that disappears entirely from service discovery may not produce an up == 0 series, so robust missing-target alerting may require additional expected-target or inventory logic.

The Node Down rule was included in the alert-rule list in the session. The exported evidence does not establish a completed node shutdown test and confirmed alert delivery for this rule.

## 9. Slack integration: contact point and notification policy

This section documents the exact Grafana-to-Slack integration from the practice session.

### 9.1 Slack channel and app

The session created:

- Slack channel: #k8s-alerts
- Slack app: grafana-alerts
- Slack Incoming Webhooks: enabled
- Grafana contact point: slack-k8s-alerts

A Slack Incoming Webhook URL is a credential. Treat it like a password. Never publish the real URL in notes, GitHub, screenshots, chat, or shell history.

### 9.2 Create the Slack Incoming Webhook

In the Slack API Apps dashboard:

1. Create a new app from scratch.
2. Name it grafana-alerts.
3. Select the intended Slack workspace.
4. Open Features → Incoming Webhooks.
5. Enable Incoming Webhooks.
6. Select Add New Webhook to Workspace.
7. Choose #k8s-alerts.
8. Authorize the integration.
9. Copy the generated webhook URL and store it securely.

For all examples in this chapter, the webhook is represented by a placeholder:

~~~text
https://hooks.slack.com/services/<SLACK_WORKSPACE_ID>/<SLACK_APP_ID>/<REDACTED_WEBHOOK_TOKEN>
~~~

This is deliberately not a working URL. Never replace it with a real credential in a committed file.

### 9.3 Create the Grafana Slack contact point

In Grafana:

1. Open Alerting → Contact points.
2. Select New contact point.
3. Set the name to slack-k8s-alerts.
4. Select Slack as the integration.
5. Enter the Slack Incoming Webhook URL directly into the protected Grafana configuration field.
6. Set the recipient/channel to #k8s-alerts where the UI requires it.
7. Save the contact point.
8. Use Test and confirm that the test notification arrives in Slack.

The recorded session confirmed that Grafana successfully posted a test/alert message through the Slack webhook.

**Credential safety:** Do not export a contact-point configuration containing the actual webhook URL into a public repository. If configuration-as-code is used, inject the webhook through a secret-management mechanism supported by the deployment rather than committing it as plaintext.

### 9.4 Configure the notification policy

In Grafana:

1. Open Alerting → Notification policies.
2. Edit the Default policy, as done in the practice session.
3. Set the contact point to slack-k8s-alerts.
4. Save the policy.

The recorded policy was intended to route all alerts through the Slack contact point.

Conceptually:

~~~text
All Alerts
    ↓
Default Notification Policy
    ↓
slack-k8s-alerts
    ↓
Slack #k8s-alerts
~~~

If a more complex policy tree is introduced later, use labels such as severity, team, or environment to route different alerts to different destinations. Ensure that child policies and matchers do not unintentionally bypass the intended default route.

### 9.5 Verify end-to-end notification

The practice session confirmed:

- Grafana alert rule High CPU Usage reached FIRING.
- Grafana used the configured Slack contact point.
- The notification was delivered to #k8s-alerts.
- The captured message title was [FIRING:2] High CPU Usage.

This is the strongest end-to-end verification in the alerting practice. The Slack delivery was performed by Grafana's notification system, not by a demonstrated Prometheus Alertmanager receiver.

### 9.6 Troubleshoot Slack notifications

If Grafana shows an alert as Firing but Slack receives nothing:

1. Open Alerting → Contact points.
2. Check that slack-k8s-alerts exists and the integration is Slack.
3. Use the Test action and inspect the response.
4. Confirm that the webhook has not been revoked or rotated.
5. Open Alerting → Notification policies.
6. Confirm the active policy routes the alert to slack-k8s-alerts.
7. Inspect alert labels and policy matchers.
8. Check Grafana server logs for notification errors.
9. Confirm the Slack app still has Incoming Webhooks enabled and is authorized for #k8s-alerts.

Useful commands:

~~~bash
kubectl get pods -n monitoring
kubectl logs -n monitoring deploy/grafana
kubectl describe pod -n monitoring <grafana-pod>
~~~

Never paste the real webhook URL into diagnostic output shared publicly. If it has been exposed, revoke or rotate it in Slack and update Grafana.

## 10. Grafana persistence: why it matters

Grafana stores important configuration and metadata in its database. In the lab's default SQLite setup, the database file was:

~~~text
/var/lib/grafana/grafana.db
~~~

This database can contain dashboards, dashboard folders, data source configuration, users and preferences, Grafana-managed alert rules, contact points, notification policies, and other Grafana configuration and metadata.

A Pod's writable container filesystem is not durable storage. If a Deployment replaces the Pod, data stored only in that container layer can be lost.

A PersistentVolumeClaim mounted at /var/lib/grafana allows the Grafana database to live outside the container filesystem.

### 10.1 Persistence architecture

~~~mermaid
flowchart TD
    G["Grafana Pod"]
    M["Volume mount<br/>/var/lib/grafana"]
    PVC["PVC: grafana<br/>5 GiB"]
    PV["PersistentVolume<br/>local-path"]
    DISK["Node-local VM disk"]
    DB[("grafana.db")]
    G --> M --> PVC --> PV --> DISK
    DB --- DISK
    G -->|"reads and writes"| DB
~~~

The diagram represents the lab's local-path persistence model. The physical backing directory is on the node selected by the Local Path Provisioner. It is not a replicated distributed filesystem.

## 11. The persistence incident and recovery lessons

This was one of the most useful troubleshooting sequences in the Grafana practice session.

### 11.1 Initial installation had persistence disabled

The first Grafana installation had no PVC in namespace monitoring:

~~~bash
kubectl get pvc -n monitoring
~~~

The output was:

~~~text
No resources found in monitoring namespace.
~~~

At that point, dashboards, data sources, alert rules, contact points, and Slack settings were stored in Grafana's container filesystem. Replacing the Pod could discard that configuration.

### 11.2 Enabling persistence created a new volume

Persistence was enabled with these values:

~~~yaml
persistence:
  enabled: true
  size: 5Gi
  storageClassName: local-path
~~~

The PVC named grafana was created and became Bound with 5 GiB capacity.

However, the new PVC was mounted at /var/lib/grafana, replacing the directory view previously backed by the container filesystem. Grafana initialized a new database on the empty volume. The earlier database was not automatically copied into it.

This explains why the Grafana UI initially appeared fresh and why the earlier dashboards and alerting configuration were not visible immediately after the persistence change.

**Key lesson:** Enabling persistence does not migrate existing Grafana data. Plan a backup and migration before switching an existing installation from ephemeral storage to a new PVC.

### 11.3 Recreate configuration on persistent storage

After the persistence transition, the Grafana configuration was rebuilt on the PVC, including:

- Prometheus data source.
- Community dashboards.
- Grafana alert rules.
- Slack contact point.
- Notification policy.

The session later verified the PVC was Bound at 5 GiB and mounted to /var/lib/grafana.

### 11.4 Service exposure changed during Helm upgrade

When the Helm release was upgraded to enable persistence, the Grafana Service returned to ClusterIP. This made the previous LoadBalancer address unavailable even though the Grafana Pod and PVC were healthy.

The Service was patched back to LoadBalancer. For repeatability, the permanent Helm values should include both persistence and service.type: LoadBalancer, as shown in Section 4.4.

### 11.5 Pod startup after VM restart

After the VMs were powered back on, Grafana did not immediately become healthy. The Pod had an init-container/volume-permission issue involving init-chown-data.

The troubleshooting sequence identified that the PVC and database still existed. After the permissions were corrected and the Grafana Pod was recreated, the Pod reached:

~~~text
grafana-7c8f6dd8d5-p4w54   1/1   Running
~~~

The session then checked that Grafana could use the database from the persistent volume and reviewed dashboards, the data source, alert rules, contact point, and notification policy.

The important distinction is that the data on the PVC survived the Pod issue. The earlier data that had existed only in the old ephemeral container filesystem was a separate, already-lost copy.

### 11.6 Verify persistence yourself

Check the PVC:

~~~bash
kubectl get pvc -n monitoring
~~~

Expected historical lab state:

~~~text
NAME      STATUS   CAPACITY
grafana   Bound    5Gi
~~~

Inspect the Pod's volume mount:

~~~bash
kubectl describe pod -n monitoring \
  $(kubectl get pod -n monitoring \
    -l app.kubernetes.io/name=grafana \
    -o jsonpath='{.items[0].metadata.name}')
~~~

Look for:

- Volume source: PersistentVolumeClaim.
- Claim name: grafana.
- Mount path: /var/lib/grafana.

Inspect the database file:

~~~bash
kubectl exec -n monitoring deploy/grafana -- \
  ls -lh /var/lib/grafana/grafana.db
~~~

A database file existing is not by itself proof that every dashboard and alert is intact. Verify the application-level objects in the Grafana UI as well.

## 12. Backup and restore considerations

### 12.1 Manual database backup used in the lab

The practice notes documented copying the SQLite database out of the Grafana Pod:

~~~bash
kubectl exec -n monitoring deploy/grafana -- \
  cp /var/lib/grafana/grafana.db /tmp/grafana.db

kubectl cp \
  monitoring/$(kubectl get pod -n monitoring \
    -l app.kubernetes.io/name=grafana \
    -o jsonpath='{.items[0].metadata.name}'):/tmp/grafana.db \
  ./grafana.db
~~~

This demonstrates the mechanics of extracting a file from a Pod. However, copying a live SQLite database with a plain filesystem copy is not a universally safe, application-consistent backup method while Grafana is writing to it.

For a reliable backup, use a supported Grafana backup/export strategy or take a consistent database snapshot while writes are quiesced. Also back up provisioning files, Helm values, Kubernetes manifests, and any required secrets through secure channels.

### 12.2 Node-local database backup observation

The practice record also showed a direct copy of the database from the Local Path Provisioner directory on the node hosting the volume. That path is implementation-specific and should not be hard-coded into automation. Inspect the PV and node affinity to identify the current backing node and path.

Do not modify files under the provisioner's backing directory while Grafana is running. Prefer Kubernetes-aware backup and restore procedures.

### 12.3 What should be backed up

| Item | Why it matters |
|---|---|
| Grafana database | Dashboards, data sources, users, alerting metadata, and configuration |
| Dashboard JSON exports | Portable dashboard definitions |
| Helm values file | Recreates chart configuration such as persistence and Service type |
| Kubernetes manifests | Recreates Services, PVCs, and supporting resources |
| Provisioning files | Recreates data sources, dashboards, and alert definitions where used |
| Secret material | Required credentials and webhook values; store securely, never in Git |

A PVC is not a backup. It protects against some Pod replacement scenarios, but it does not protect against deletion of the PVC/PV, destruction of the VM disk, node loss, or accidental data corruption.

## 13. Troubleshooting reference

### 13.1 Grafana Pod is Pending

~~~bash
kubectl get pods -n monitoring -o wide
kubectl describe pod -n monitoring <grafana-pod>
kubectl get pvc -n monitoring
kubectl get storageclass
~~~

Check for unbound PVCs, missing or incorrect StorageClass, insufficient CPU or memory, node affinity, scheduling constraints, taints, and tolerations.

### 13.2 Grafana Pod is CrashLoopBackOff or init container fails

~~~bash
kubectl describe pod -n monitoring <grafana-pod>
kubectl logs -n monitoring <grafana-pod> -c init-chown-data
kubectl logs -n monitoring <grafana-pod> -c grafana
~~~

Check volume permissions, ownership, mount paths, and security context. Do not apply broad recursive permission changes to an entire node filesystem. Restrict any ownership correction to the specific Grafana data directory and verify the expected UID/GID for the chart version.

### 13.3 Grafana UI is unavailable

~~~bash
kubectl get svc grafana -n monitoring
kubectl get endpoints grafana -n monitoring
kubectl get pods -n monitoring -o wide
kubectl describe svc grafana -n monitoring
~~~

Check that Service type is LoadBalancer if intended, EXTERNAL-IP is assigned, Service selector matches Grafana Pod labels, endpoints exist, MetalLB address pool is available, and the network path allows access.

A Helm upgrade may restore chart defaults. Inspect helm get values and persist the intended Service type in the values file.

### 13.4 Prometheus data source shows errors

- Confirm the Prometheus Service exists.
- Confirm the URL and port are correct.
- From inside the cluster, prefer the Kubernetes Service DNS name where possible.
- Confirm Prometheus Server is Ready.
- Test the Prometheus API endpoint.
- Review Grafana logs.

~~~bash
kubectl get svc -n prometheus
kubectl get pods -n prometheus
kubectl logs -n monitoring deploy/grafana
~~~

### 13.5 Dashboard panels show No data

- Confirm the Prometheus data source is selected.
- Open Explore and test the panel's PromQL directly.
- Inspect dashboard variables and selected namespace/node values.
- Check whether the required exporter and metric are present.
- Verify that the query's label names match the current Prometheus series.
- Check the dashboard time range.

A community dashboard may require metrics or labels that are not exposed by the current scrape configuration.

### 13.6 Slack alert not delivered

Check the contact point test, webhook validity, notification policy route, alert labels, and Grafana logs. See Section 9.6.

## 14. Security and production recommendations

The Vagrant lab is for learning, not a production-ready Grafana deployment.

- Protect Grafana with authentication and least-privilege roles.
- Avoid exposing Grafana publicly without TLS and appropriate access controls.
- Do not use the default administrator password in a shared environment.
- Store Slack webhook URLs in a secret manager or secure secret store.
- Never commit webhook URLs, passwords, or private keys to Git.
- Restrict who can create or edit contact points and notification policies.
- Keep Grafana, Helm charts, plugins, and dashboards updated through a controlled process.
- Pin chart versions for repeatable deployments.
- Use a deliberate backup and restore plan.
- Use storage with appropriate durability and availability guarantees.
- Monitor Grafana itself: availability, restarts, query latency, database size, and notification failures.
- Use dashboard provisioning and configuration-as-code for repeatability where appropriate.
- Avoid using the lab's node-local Local Path Provisioner as if it were replicated, highly available storage.

## 15. Hands-on exercises

### Exercise 1 — Verify Grafana resources

~~~bash
helm list -n monitoring
helm status grafana -n monitoring
kubectl get pods,svc,pvc -n monitoring -o wide
~~~

Record the Pod state, Service type, external IP, PVC name, capacity, and status.

### Exercise 2 — Verify the Prometheus data source

In Grafana, open Connections → Data sources → Prometheus. Run the connection test and record the URL used. Confirm it points to the live Prometheus Service, not an obsolete MetalLB IP.

### Exercise 3 — Import the three dashboards

Import:

- 15757 — Kubernetes / Views / Global.
- 7249 — Kubernetes Cluster Monitoring (via Prometheus).
- 1860 — Node Exporter Full.

Confirm that the dashboards use the Prometheus data source and that the panels return data.

### Exercise 4 — Create a custom node overview

Create panels for CPU usage percentage, memory usage percentage, root filesystem usage, and Pod count by namespace. Use the queries in Section 6. Give every panel a clear title, unit, legend, and useful time range.

### Exercise 5 — Verify the High CPU alert

Inspect the High CPU Usage alert. Record its query, Reduce expression, threshold, evaluation interval, and pending period. Confirm that the rule is associated with the Prometheus data source.

### Exercise 6 — Verify Slack contact point and policy

Open Alerting → Contact points → slack-k8s-alerts. Then open Alerting → Notification policies. Confirm that the policy routes alerts to the Slack contact point. Use the Test action and verify receipt in #k8s-alerts. Never expose the webhook URL.

### Exercise 7 — Verify Grafana persistence

Check the grafana PVC is Bound and mounted at /var/lib/grafana. Inspect the grafana.db file. Restart the Grafana Pod in a controlled lab window and verify that dashboards, data sources, alert rules, contact point, and notification policy remain available.

Do not delete the PVC as part of this exercise.

### Exercise 8 — Practice a backup

Export important dashboard JSON files and create a consistent backup of the Grafana database using a safe method. Store the backup outside the cluster and document how you would restore it.

## 16. Command reference

| Task | Command |
|---|---|
| List Grafana release | helm list -n monitoring |
| Release status | helm status grafana -n monitoring |
| Values | helm get values grafana -n monitoring |
| Release history | helm history grafana -n monitoring |
| Grafana Pods | kubectl get pods -n monitoring -o wide |
| Grafana Service | kubectl get svc grafana -n monitoring |
| Grafana PVC | kubectl get pvc -n monitoring |
| Describe Grafana Pod | kubectl describe pod -n monitoring <pod> |
| Grafana logs | kubectl logs -n monitoring deploy/grafana |
| Database file | kubectl exec -n monitoring deploy/grafana -- ls -lh /var/lib/grafana/grafana.db |
| Prometheus Services | kubectl get svc -n prometheus |

## 17. Knowledge-check questions

1. What is Grafana, and how does it differ from Prometheus?
2. What is a Grafana data source?
3. Why did the lab use Prometheus as its data source?
4. What does a dashboard panel represent?
5. What is the purpose of dashboard variables?
6. Which dashboard ID was used for Kubernetes / Views / Global?
7. Which dashboard ID was used for Node Exporter Full?
8. What does a Grafana alert rule evaluate?
9. What is the purpose of a Reduce expression?
10. What is the difference between Pending and Firing alert states?
11. What is a contact point?
12. What is a notification policy?
13. Explain the alert route from the Prometheus query to the Slack channel.
14. Why is the Slack Incoming Webhook URL a secret?
15. Did this lab use Grafana alerting or a demonstrated Prometheus Alertmanager Slack receiver?
16. What is stored in /var/lib/grafana/grafana.db?
17. Why did enabling persistence create a fresh Grafana database?
18. Why does a PVC not automatically migrate data from an old container filesystem?
19. What does a Bound PVC status mean?
20. Why can a local-path volume be unavailable if its backing node is lost?
21. Why did Grafana's LoadBalancer address disappear after a Helm upgrade?
22. Why is a PVC not a substitute for a backup?
23. What checks would you perform if Grafana is Running but dashboards show No data?
24. How would you troubleshoot a Firing alert that does not arrive in Slack?

## 18. Chapter completion checklist

- [ ] Explain Grafana's role in the Prometheus monitoring stack.
- [ ] Inspect the Grafana Helm release and Kubernetes resources.
- [ ] Access Grafana through the current LoadBalancer address.
- [ ] Configure and test the Prometheus data source.
- [ ] Create custom panels using PromQL.
- [ ] Import dashboards 15757, 7249, and 1860.
- [ ] Explain Grafana alert rule evaluation and alert states.
- [ ] Verify the High CPU Usage rule.
- [ ] Verify the slack-k8s-alerts contact point.
- [ ] Verify the notification policy routes to the Slack contact point.
- [ ] Confirm a test or alert notification reaches #k8s-alerts.
- [ ] Explain why webhook URLs must remain secret.
- [ ] Verify the 5 GiB grafana PVC is Bound and mounted at /var/lib/grafana.
- [ ] Explain the ephemeral-to-persistent database transition from the lab.
- [ ] Verify Grafana configuration survives a controlled Pod restart.
- [ ] Create and store a consistent backup outside the cluster.
- [ ] Explain the limitations of node-local storage.

## 19. References

- Grafana documentation: https://grafana.com/docs/grafana/latest/
- Grafana dashboards: https://grafana.com/docs/grafana/latest/dashboards/
- Grafana data sources: https://grafana.com/docs/grafana/latest/datasources/
- Grafana alerting: https://grafana.com/docs/grafana/latest/alerting/
- Grafana provisioning: https://grafana.com/docs/grafana/latest/administration/provisioning/
- Grafana Helm chart: https://github.com/grafana/helm-charts
- Prometheus documentation: https://prometheus.io/docs/introduction/overview/
- Node Exporter: https://github.com/prometheus/node_exporter
- kube-state-metrics: https://github.com/kubernetes/kube-state-metrics
- Slack Incoming Webhooks: https://api.slack.com/messaging/webhooks
- Kubernetes Persistent Volumes: https://kubernetes.io/docs/concepts/storage/persistent-volumes/

---

**End of Chapter 13 — Grafana Dashboards and Persistence**

**Next chapter:** Chapter 14 — CI/CD Automation.
