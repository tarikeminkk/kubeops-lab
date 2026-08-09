# KubeOps Lab

KubeOps Lab is a DevOps laboratory environment built on Nutanix AHV.

The environment combines infrastructure automation, CI/CD, containerization, a private container registry, and Kubernetes into a complete deployment workflow.

## Architecture

```text
                         Developer
                             |
                         git push
                             |
                             v
                         +--------+
                         | GitHub |
                         +---+----+
                             |
                             v
                       +-------------+
                       |   Jenkins   |
                       | tarik-ci01  |
                       +------+------+
                              |
                    Docker Build
                              |
                              v
                       +-------------+
                       |   Harbor    |
                       | tarik-reg01 |
                       +------+------+
                              |
                         Image Pull
                              |
                              v
                    +-------------------+
                    | Kubernetes Cluster|
                    +---------+---------+
                              |
                 +------------+------------+
                 |                         |
                 v                         v
         +---------------+         +---------------+
         |tarik-k8s-wrk01|         |tarik-k8s-wrk02|
         |    Worker     |         |    Worker     |
         +---------------+         +---------------+
                 |                         |
                 +------------+------------+
                              |
                              v
                           Demo App
                         NodePort 30080
```

## Infrastructure

| Server | Role |
|---|---|
| `tarik-ops01` | Ansible Control Node |
| `tarik-git01` | Git Server |
| `tarik-ci01` | Jenkins CI Server |
| `tarik-reg01` | Harbor Container Registry |
| `tarik-k8s-cp01` | Kubernetes Control Plane |
| `tarik-k8s-wrk01` | Kubernetes Worker |
| `tarik-k8s-wrk02` | Kubernetes Worker |
| `tarik-monitor01` | Monitoring Server |

All systems run as virtual machines in a Nutanix AHV lab environment.

## Technology Stack

- Nutanix AHV
- Ubuntu Server
- Git / GitHub
- Ansible
- Jenkins
- Docker
- Harbor
- Kubernetes
- NGINX

## Ansible Automation

Ansible is used for configuration management and infrastructure automation.

The control node manages the lab servers over SSH using a centralized inventory.

```text
ansible/
├── inventory
├── ansible.cfg
└── playbooks/
    ├── base-setup.yml
    ├── ci-setup.yml
    ├── registry-setup.yml
    ├── harbor-setup.yml
    ├── kubernetes-setup.yml
    ├── kubernetes-cluster.yml
    └── kubernetes-harbor.yml
```

Automation covers Linux base configuration, package management, Docker installation, Jenkins configuration, Harbor deployment, Kubernetes node preparation, cluster configuration, and container registry integration.

## CI/CD Pipeline

Application deployments are handled by Jenkins through the repository `Jenkinsfile`.

```text
GitHub
   |
   | push to main
   v
Jenkins
   |
   +-- Checkout
   |
   +-- Ansible Syntax Check
   |
   +-- Docker Build
   |
   +-- Harbor Authentication
   |
   +-- Docker Push
   |
   +-- Kubernetes Deployment Update
   |
   +-- Rollout Verification
   |
   v
Application
```

Each Jenkins build generates a versioned Docker image.

```text
harbor.kubeops.local/kubeops/demo-app:11
harbor.kubeops.local/kubeops/demo-app:latest
```

The image is pushed to Harbor and the Kubernetes deployment is updated automatically.

## Harbor Registry

Harbor provides the private container registry used by the CI/CD pipeline.

```text
harbor.kubeops.local
└── kubeops
    └── demo-app
        ├── latest
        ├── 11
        ├── 10
        └── 9
```

Jenkins authenticates to Harbor using a dedicated robot account.

Kubernetes nodes are configured to pull images from the internal Harbor registry.

## Kubernetes

The cluster consists of one control plane and two worker nodes.

```text
             tarik-k8s-cp01
              Control Plane
                    |
          +---------+---------+
          |                   |
          v                   v
 tarik-k8s-wrk01       tarik-k8s-wrk02
      Worker                Worker
```

The demo application runs with two replicas distributed across the worker nodes.

```text
NAME                        READY   STATUS    NODE
demo-app-xxxxxxxxxx         1/1     Running   tarik-k8s-wrk01
demo-app-yyyyyyyyyy         1/1     Running   tarik-k8s-wrk02
```

The application is exposed through a NodePort service on port `30080`.

## Kubernetes RBAC

Jenkins uses a dedicated Kubernetes ServiceAccount:

```text
jenkins-deployer
```

A namespace-scoped Role and RoleBinding provide the permissions required for deployments.

The Jenkins pipeline can update the `demo-app` deployment and monitor its rollout without using Kubernetes administrator credentials.

## Deployment Flow

When a change is pushed to the main branch:

```text
Code Change
    |
    v
GitHub
    |
    v
Jenkins
    |
    v
Docker Build
    |
    v
Harbor Registry
    |
    v
Kubernetes Deployment
    |
    v
Rolling Update
```

Jenkins updates the running image with:

```bash
kubectl set image deployment/demo-app \
  demo-app=harbor.kubeops.local/kubeops/demo-app:${IMAGE_TAG}
```

The deployment is then verified with:

```bash
kubectl rollout status deployment/demo-app --timeout=120s
```

## Demo Application

The demo application is a lightweight NGINX container used to validate the end-to-end pipeline.

```text
KubeOps Lab

KubeOps CI/CD v2 Works!

GitHub -> Jenkins -> Harbor -> Kubernetes
```

It is accessible through the Kubernetes NodePort service:

```text
http://<worker-node>:30080
```

## Repository Structure

```text
kubeops-lab/
├── ansible/
│   ├── inventory
│   ├── ansible.cfg
│   └── playbooks/
├── app/
│   ├── Dockerfile
│   └── index.html
├── kubernetes/
│   ├── deployment.yaml
│   └── service.yaml
├── Jenkinsfile
└── README.md
```
