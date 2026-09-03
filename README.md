# End-to-End DevOps Infrastructure & GitOps Platform

A hands-on DevOps environment built on **Nutanix AHV-based private cloud infrastructure** to design and operate a complete software delivery, observability, and security monitoring workflow.

The environment integrates infrastructure automation, CI/CD, container image management, Kubernetes, GitOps-based deployment, metrics, alerting, centralized logging, and security monitoring.

The goal of this project is not to demonstrate individual tools in isolation, but to integrate them into a working end-to-end platform.

---

## Architecture

```text
                              GitHub
                                 │
                                 ▼
                              Jenkins
                                 │
                         Build Docker Image
                                 │
                                 ▼
                               Harbor
                         Private Registry
                                 │
                                 │
                    Update Kubernetes Manifest
                                 │
                                 ▼
                              GitHub
                                 │
                                 ▼
                              Argo CD
                                 │
                                 ▼
                         Kubernetes Cluster
                      ┌──────────┴──────────┐
                      │                     │
                 Control Plane          Workers
                                      demo-app
                                          │
                                          ▼
                                       Service


                       OBSERVABILITY

              ┌──────────── Metrics ─────────────┐
              │                                  │
        Node Exporter                   kube-state-metrics
              │                                  │
              └──────────► Prometheus ◄──────────┘
                               │
                               ▼
                            Grafana

                         Alert Rules
                               │
                               ▼
                         Alertmanager


                 CENTRALIZED LOGGING

        Linux Servers                Kubernetes Pods
             │                             │
             ▼                             ▼
        Grafana Alloy                 Grafana Alloy
             │                             │
             └──────────────┬──────────────┘
                            ▼
                           Loki
                            │
                            ▼
                         Grafana


                  SECURITY MONITORING

                      Linux Servers
                           │
                      Wazuh Agents
                           │
                           ▼
                      Wazuh Manager
                           │
                           ▼
                      Wazuh Indexer
                           │
                           ▼
                     Wazuh Dashboard
```

---

## Infrastructure

The environment runs on Ubuntu Server virtual machines hosted on Nutanix AHV.

| Host | IP Address | Role |
|---|---|---|
| `tarik-ci01` | `192.168.5.1` | Jenkins CI/CD |
| `tarik-git01` | `192.168.5.2` | Git services |
| `tarik-k8s-cp01` | `192.168.5.3` | Kubernetes control plane |
| `tarik-k8s-wrk01` | `192.168.5.4` | Kubernetes worker |
| `tarik-k8s-wrk02` | `192.168.5.5` | Kubernetes worker |
| `tarik-monitor01` | `192.168.5.6` | Prometheus, Grafana, Alertmanager |
| `tarik-ops01` | `192.168.5.7` | Ansible control node |
| `tarik-reg01` | `192.168.5.8` | Harbor container registry |
| `tarik-log01` | `192.168.5.9` | Loki centralized logging |
| `tarik-wazuh01` | `192.168.5.10` | Wazuh security monitoring |

---

## CI/CD and GitOps

Application delivery follows a GitOps workflow.

```text
Developer
   │
   ▼
GitHub
   │
   ▼
Jenkins
   │
   ├── Validate automation
   ├── Build Docker image
   ├── Authenticate to Harbor
   ├── Push versioned image
   └── Update Kubernetes image tag in Git
             │
             ▼
           GitHub
             │
             ▼
           Argo CD
             │
             ▼
         Kubernetes
```

Jenkins is responsible for the CI portion of the workflow.

When a code change is detected, Jenkins builds a new Docker image and pushes both versioned and latest tags to the private Harbor registry.

The pipeline then updates the image version in the Kubernetes deployment manifest and pushes the desired state back to Git.

Argo CD monitors the Git repository and synchronizes the Kubernetes cluster with the desired state stored in Git.

This keeps deployment responsibility outside Jenkins and makes Git the source of truth for the cluster state.

### Jenkins Pipeline

![Jenkins Pipeline](docs/images/jenkins-pipeline.png)

---

## Private Container Registry

Harbor is used as the internal container image registry.

Application images produced by Jenkins are stored as versioned artifacts before being pulled by Kubernetes.

```text
Jenkins
   │
   ▼
harbor.kubeops.local
   │
   ├── demo-app:<build-number>
   └── demo-app:latest
         │
         ▼
     Kubernetes
```

Jenkins authenticates to Harbor using a dedicated robot account rather than a personal user account.

### Harbor Registry

![Harbor Registry](docs/images/harbor-registry.png)

---

## Kubernetes

The Kubernetes environment consists of one control-plane node and two worker nodes.

The sample application runs with multiple replicas and is exposed through a Kubernetes Service.

The application deployment includes:

- Multiple pod replicas
- Readiness probes
- Liveness probes
- Versioned container images
- Private Harbor image pulls
- GitOps-based deployment through Argo CD

The container runtime is **containerd**.

---

## Infrastructure Automation

Ansible is used from the dedicated operations node to configure and maintain the Linux infrastructure.

Automation covers tasks such as:

- Base server configuration
- Package installation
- Common system configuration
- Node Exporter deployment
- Jenkins setup
- Harbor setup
- Kubernetes configuration
- Monitoring configuration
- Prometheus targets
- Alertmanager
- Loki
- Grafana Alloy
- Wazuh deployment
- Wazuh agent deployment

Current playbooks include:

```text
ansible/playbooks/
├── base-setup.yml
├── ci-setup.yml
├── registry-setup.yml
├── harbor-setup.yml
├── kubernetes-setup.yml
├── kubernetes-cluster.yml
├── kubernetes-harbor.yml
├── monitoring-setup.yml
├── node-exporter.yml
├── prometheus-targets.yml
├── alertmanager-setup.yml
├── prometheus-alerts.yml
├── loki-setup.yml
├── alloy-setup.yml
├── wazuh-setup.yml
└── wazuh-agent.yml
```

---

## Monitoring

Prometheus provides centralized metrics collection for the infrastructure and Kubernetes environment.

Metrics are collected from:

- Linux servers through Node Exporter
- Kubernetes through kube-state-metrics
- Infrastructure and workload targets configured in Prometheus

Grafana is used to visualize infrastructure and Kubernetes metrics.

### Infrastructure Overview

![Infrastructure Dashboard](docs/images/grafana-infrastructure-dashboard.png)

### Node Metrics

![Node Dashboard](docs/images/grafana-node-dashboard.png)

### Kubernetes Metrics

![Kubernetes Dashboard](docs/images/grafana-kubernetes-dashboard.png)

---

## Alerting

Prometheus alert rules and Alertmanager provide infrastructure and workload availability monitoring.

Implemented alerts include:

```text
NodeDown
KubernetesPodNotRunning
```

`NodeDown` was tested by stopping Node Exporter on one of the monitored servers and verifying the complete alert lifecycle from Prometheus to Alertmanager and recovery.

### NodeDown Test

![NodeDown Alert](docs/images/grafana-node-down-alert.png)

---

## Centralized Logging

Centralized logging is implemented using **Grafana Alloy, Loki, and Grafana**.

Two different log sources are collected.

### Linux System Logs

Grafana Alloy runs on the Linux servers and reads logs from the systemd journal.

```text
Linux Server
     │
     ▼
systemd journal
     │
     ▼
Grafana Alloy
     │
     ▼
Loki
     │
     ▼
Grafana
```

Logs can be filtered by host and other labels from Grafana Explore.

Example:

```logql
{hostname="tarik-git01"}
```

![Linux Logs](docs/images/grafana-linux-logs.png)

### Kubernetes Application Logs

A dedicated Grafana Alloy instance runs inside Kubernetes and discovers workloads through the Kubernetes API.

Pod logs are enriched with Kubernetes metadata before being forwarded to Loki.

Available labels include:

```text
cluster
namespace
pod
container
app
```

This makes it possible to isolate logs for a specific workload directly from Grafana.

Example:

```logql
{namespace="default", container="demo-app"}
```

![Kubernetes Application Logs](docs/images/grafana-kubernetes-logs.png)

The logging pipeline is:

```text
Kubernetes Pods
      │
      ▼
Grafana Alloy
      │
      ▼
Loki
      │
      ▼
Grafana Explore
```

Loki runs on the dedicated logging server and stores its local TSDB, WAL, chunks, and related data under:

```text
/var/lib/loki
```

---

## Security Monitoring

Security monitoring is implemented using **Wazuh**.

A dedicated Wazuh server provides centralized security event analysis, endpoint monitoring, File Integrity Monitoring, and Security Configuration Assessment for the Linux infrastructure.

```text
Linux Servers
      │
      │ Wazuh Agent
      ▼
tarik-wazuh01
      │
      ├── Wazuh Manager
      ├── Wazuh Indexer
      └── Wazuh Dashboard
```

Wazuh agents are deployed and managed across the infrastructure using Ansible.

### Endpoint Monitoring

Eight Linux servers are connected to the Wazuh manager and centrally monitored from the Wazuh Dashboard.

![Wazuh Endpoints](docs/images/wazuh-endpoints.png)

### SSH Authentication Detection

A controlled SSH authentication test was performed against `tarik-git01`.

Wazuh detected and correlated:

- Authentication attempts using a non-existent user
- PAM authentication failures
- Repeated password failures
- Security alerts generated from repeated authentication failures

![Wazuh SSH Detection](docs/images/wazuh-ssh-detection.png)

### File Integrity Monitoring

Wazuh File Integrity Monitoring was tested by modifying a monitored file under `/etc`.

The agent detected the change in real time and generated an integrity checksum alert.

```text
/etc/wazuh-fim-test.conf
        │
        ▼
   File Modified
        │
        ▼
   Wazuh Agent
        │
        ▼
 Integrity Alert
```

![Wazuh File Integrity Monitoring](docs/images/wazuh-file-integrity-monitoring.png)

### Security Configuration Assessment

Wazuh Security Configuration Assessment is used to evaluate Linux hosts against the **CIS Ubuntu Linux 24.04 LTS Benchmark**.

The assessment provides visibility into passed, failed, and non-applicable security controls and can be used to identify potential system-hardening improvements.

![Wazuh CIS Assessment](docs/images/wazuh-cis-assessment.png)

---

## Repository Structure

```text
.
├── ansible/
│   ├── inventory
│   ├── ansible.cfg
│   └── playbooks/
│       ├── base-setup.yml
│       ├── ci-setup.yml
│       ├── registry-setup.yml
│       ├── harbor-setup.yml
│       ├── kubernetes-setup.yml
│       ├── kubernetes-cluster.yml
│       ├── kubernetes-harbor.yml
│       ├── monitoring-setup.yml
│       ├── node-exporter.yml
│       ├── prometheus-targets.yml
│       ├── alertmanager-setup.yml
│       ├── prometheus-alerts.yml
│       ├── loki-setup.yml
│       ├── alloy-setup.yml
│       ├── wazuh-setup.yml
│       └── wazuh-agent.yml
│
├── app/
│
├── kubernetes/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── alloy-logs.yaml
│
├── docs/
│   └── images/
│       ├── jenkins-pipeline.png
│       ├── harbor-registry.png
│       ├── grafana-infrastructure-dashboard.png
│       ├── grafana-node-dashboard.png
│       ├── grafana-kubernetes-dashboard.png
│       ├── grafana-node-down-alert.png
│       ├── grafana-linux-logs.png
│       ├── grafana-kubernetes-logs.png
│       ├── wazuh-endpoints.png
│       ├── wazuh-ssh-detection.png
│       ├── wazuh-file-integrity-monitoring.png
│       └── wazuh-cis-assessment.png
│
├── Jenkinsfile
└── README.md
```

---

## Technology Stack

| Area | Technologies |
|---|---|
| Virtualization | Nutanix AHV |
| Operating System | Ubuntu Server |
| Configuration Management | Ansible |
| Source Control | Git, GitHub |
| CI | Jenkins |
| Containers | Docker |
| Container Registry | Harbor |
| Container Runtime | containerd |
| Orchestration | Kubernetes |
| GitOps / CD | Argo CD |
| Metrics | Prometheus, Node Exporter, kube-state-metrics |
| Visualization | Grafana |
| Alerting | Alertmanager |
| Logging | Loki, Grafana Alloy |
| Security Monitoring | Wazuh |

---

## What This Project Covers

This environment demonstrates practical implementation of:

- Linux infrastructure administration
- Infrastructure automation with Ansible
- CI pipeline design
- Docker image lifecycle management
- Private container registry integration
- Kubernetes cluster operations
- GitOps deployment practices
- Infrastructure and Kubernetes monitoring
- Prometheus alerting
- Centralized Linux logging
- Kubernetes application logging
- Centralized endpoint security monitoring
- SSH authentication event detection
- File Integrity Monitoring
- CIS-based Security Configuration Assessment
- Automated security agent deployment with Ansible
- End-to-end integration between DevOps, observability, and security tooling

---

## Current Delivery Flow

```text
GitHub
   ↓
Jenkins
   ↓
Docker Build
   ↓
Harbor
   ↓
Git Manifest Update
   ↓
GitHub
   ↓
Argo CD
   ↓
Kubernetes
```

## Current Observability Flow

```text
Metrics:
Linux / Kubernetes → Prometheus → Grafana

Logs:
Linux / Kubernetes → Grafana Alloy → Loki → Grafana

Alerts:
Prometheus → Alertmanager
```

## Current Security Monitoring Flow

```text
Linux Servers
      ↓
Wazuh Agents
      ↓
Wazuh Manager
      ↓
Wazuh Indexer
      ↓
Wazuh Dashboard
```
