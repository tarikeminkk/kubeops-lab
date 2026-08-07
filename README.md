# KubeOps Lab

Production-inspired DevOps laboratory environment built on Nutanix AHV.

The project focuses on infrastructure automation, configuration management, CI/CD, containerization, Kubernetes and monitoring.

# KubeOps Lab

Production-inspired DevOps laboratory environment built on Nutanix AHV.

The project focuses on infrastructure automation, configuration management, CI/CD, containerization, Kubernetes and monitoring.

## Architecture

```text
                              GitHub
                                 |
                                 v
                         +---------------+
                         |  tarik-ops01  |
                         |    Ansible    |
                         |      Git      |
                         +-------+-------+
                                 |
               +-----------------+-----------------+
               |                 |                 |
               v                 v                 v
        +-------------+   +-------------+   +-------------+
        | tarik-git01 |   | tarik-ci01  |   | tarik-reg01 |
        | Git Server  |   | CI Server   |   |  Registry   |
        +-------------+   +-------------+   +-------------+
                                 |
                                 v
                       +---------------------+
                       |  Kubernetes Cluster |
                       +----------+----------+
                                  |
                    +-------------+-------------+
                    |                           |
                    v                           v
             +-------------+             +-------------+
             | k8s-cp01    |             | k8s-wrk01   |
             | Control     |             | Worker      |
             | Plane       |             +-------------+
             +-------------+                    |
                                                v
                                         +-------------+
                                         | k8s-wrk02   |
                                         | Worker      |
                                         +-------------+

                                  |
                                  v
                           +-------------+
                           | monitor01   |
                           | Monitoring  |
                           +-------------+

Infrastructure
Server	Role
tarik-ops01	Automation / Management
tarik-git01	Git Server
tarik-ci01	CI Server
tarik-reg01	Container Registry
tarik-k8s-cp01	Kubernetes Control Plane
tarik-k8s-wrk01	Kubernetes Worker
tarik-k8s-wrk02	Kubernetes Worker
tarik-monitor01	Monitoring

Infrastructure is hosted on a Nutanix AHV test environment.

Automation

Ansible is used for infrastructure configuration and server provisioning.

Current automation includes:

Package management
Common server configuration
Time synchronization
Service management
Centralized inventory management
Technology Stack
Nutanix AHV
Linux
Git
Ansible
CI/CD
Docker
Kubernetes
Container Registry
Monitoring
