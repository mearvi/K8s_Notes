# Kubernetes & DevOps Homelab — Technical Notes

A practical, reproducible technical manual documenting the build-out of a local multi-node Kubernetes lab and the DevOps workflows developed on top of it.

**Repository:** [mearvi/K8s_Notes](https://github.com/mearvi/K8s_Notes)  
**Hypervisor:** Oracle VirtualBox  
**VM orchestration:** HashiCorp Vagrant  
**Guest OS:** Ubuntu 22.04 LTS (`ubuntu/jammy64`)  
**Kubernetes bootstrap:** kubeadm  
**Container runtime:** containerd  
**Cluster topology:** 1 control-plane node + 2 worker nodes

---

## Project overview

This repository records a hands-on learning journey from local infrastructure preparation through Kubernetes administration, application delivery, GitOps, observability, and persistent storage.

The foundation is a three-node virtual lab hosted on a Windows workstation. Vagrant defines the virtual machines and their static private-network addresses; VirtualBox supplies the virtualization layer. Each guest receives a NAT interface for outbound access and a host-only/private interface for stable node-to-node and host-to-node communication.

The lab is intentionally a **single-control-plane learning cluster**, not a highly available production control plane. It is large enough to practise Kubernetes scheduling, Services, Ingress, storage, Helm, GitOps, CI/CD, and monitoring without the overhead of the abandoned larger-node experiment.

### Lab topology

| Node | Kubernetes role | Host-only IP | vCPU | RAM |
|---|---|---:|---:|---:|
| `master1` | Control plane | `192.168.56.10` | 2 | 4 GB |
| `worker1` | Worker | `192.168.56.20` | 2 | 6 GB |
| `worker2` | Worker | `192.168.56.21` | 2 | 6 GB |

**Allocated VM resources:** 6 vCPU and 16 GB RAM in total. These are virtual allocations, not dedicated physical cores.

### High-level network layout

```text
Windows workstation
│
├── VirtualBox NAT networking
│   └── Outbound connectivity for guest VMs
│
└── VirtualBox Host-only network (192.168.56.0/24)
    ├── master1  192.168.56.10  (control plane)
    ├── worker1  192.168.56.20  (worker)
    └── worker2  192.168.56.21  (worker)
```

The host-only network is used for stable private addressing. In the guest, VirtualBox commonly presents the NAT interface as `enp0s3` and the host-only interface as `enp0s8`; interface names can vary and should be verified with `ip -br address`.

---

## Documentation principles

- The chapters are written as practical runbooks, with commands, configuration files, validation steps, and operational notes.
- The initial infrastructure chapter preserves the final three-node Vagrant configuration recovered from the lab notes.
- The manual excludes raw error dumps and abandoned six-node configuration details.
- Any later improvement or supplementary recommendation is labelled as such rather than being represented as a command that was necessarily executed during the original build.
- Credentials, private keys, access tokens, and other secrets must never be committed.

---

## Table of contents

The following 16 chapters form the master documentation framework. Chapters are published sequentially as their implementation and verification are documented.

1. **[Chapter 1 — Infrastructure Setup](Chapter_01_Infra_Setup.md)** — Windows host preparation, VirtualBox networking, Vagrant, three-node VM definitions, and guest OS prerequisites.
2. **[Chapter 2 — Kubernetes Cluster Bootstrap](Chapter_02_Kubernetes_Cluster_Bootstrap.md)** — kubeadm initialization, control-plane configuration, Calico CNI installation, worker joining, and cluster validation.
3. **[Chapter 3 — Cluster Networking with Calico](Chapter_03_Cluster_Networking_with_Calico.md)** — CNI installation, pod networking, node networking, and network validation.
4. **[Chapter 4 — Kubernetes Core Workloads](Chapter_04_Kubernetes_Core_Workloads.md)** — Pods, ReplicaSets, Deployments, DaemonSets, Jobs, CronJobs, labels, selectors, and namespaces.
5. **[Chapter 5 — Services, DNS, and Cluster Networking](Chapter_05_Services_DNS_and_Cluster_Networking.md)** — ClusterIP, NodePort, LoadBalancer, service discovery, CoreDNS, and traffic flow.
6. **[Chapter 6 — Ingress, Host/Path Routing, and TLS](Chapter_06_Ingress_Host_Path_Routing_and_TLS.md)** — NGINX Ingress Controller, MetalLB integration, host-based and path-based routing, URI rewriting, and HTTPS with a Kubernetes TLS Secret.
7. **[Chapter 7 — Kubernetes Configuration and Secrets](Chapter_07_Kubernetes_Configuration_and_Secrets.md)** — ConfigMaps, generic Secrets, environment and file injection, namespace scope, application configuration, Helm examples, and Secret security.
8. **[Chapter 8 — Helm Fundamentals](Chapter_08_Helm_Fundamentals.md)** — Helm installation and repositories, chart structure, values and templates, linting, rendering, release inspection, upgrades, rollbacks, and troubleshooting.
9. **Chapter 9 — Multi-Tier Application with Helm** — Frontend, backend, PostgreSQL, service discovery, persistent data, and application connectivity.
10. **Chapter 10 — Storage Fundamentals and Dynamic Provisioning** — PVs, PVCs, StorageClasses, access modes, and reclaim policies.
11. **Chapter 11 — Metrics Server and Autoscaling** — Resource requests/limits, metrics, and Horizontal Pod Autoscaler.
12. **Chapter 12 — Prometheus Monitoring** — Metrics collection, scrape configuration, targets, and alerting foundations.
13. **Chapter 13 — Grafana Dashboards and Persistence** — Dashboards, data sources, persistent storage, and operational checks.
14. **Chapter 14 — CI/CD Automation** — Source control workflows, build pipelines, image publishing, and deployment automation.
15. **Chapter 15 — Argo CD GitOps Delivery** — GitOps principles, repository structure, application synchronization, and drift management.
16. **Chapter 16 — Longhorn Storage** — Distributed block storage, replicas, volumes, backups, and Kubernetes integration.

---

## Repository layout

```text
K8s_Notes/
├── README.md
├── Chapter_01_Infra_Setup.md
├── Chapter_02_Kubernetes_Cluster_Bootstrap.md
├── ...
└── Chapter_16_Longhorn_Storage.md                   # planned
```

The later chapter filenames may be refined as each chapter is prepared, while the 16-part learning sequence remains the master framework.

---

## Getting started

Start with [Chapter 1 — Infrastructure Setup](Chapter_01_Infra_Setup.md). It documents the Windows host prerequisites, VirtualBox host-only networking, the complete Vagrantfile, and the common guest provisioning script.

After the virtual machines are provisioned, continue with the Kubernetes bootstrap chapter. Do not run cluster initialization until all three VMs have stable private IP connectivity and the guest operating-system prerequisites have been applied.

---

## Scope and limitations

This lab is intended for education, development, and controlled experimentation. It is not a production reference architecture. A single control-plane node is a single point of failure; production clusters require an architecture designed around availability, security, capacity, backup, and operational support.

Always review upstream project documentation before applying version-sensitive commands to a new environment.
