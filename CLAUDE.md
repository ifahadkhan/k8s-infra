# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

Pure Kubernetes manifests (YAML) for the **Synapse** warehouse/logistics platform deployed on a Rancher K3s cluster. No application code lives here — only infrastructure configuration.

## Architecture Overview

### Microservices

14 services communicating over internal cluster DNS, all pulling from `docfahad/synapse:<service>-api` on Docker Hub with `imagePullPolicy: Always`:

| Service | Role | Replicas |
|---|---|---|
| API Gateway | External entry point | 1 |
| Gatekeeper API | Auth/authorization | 2 |
| Query Engine API | Data retrieval | 1 |
| Hangfire API | Background job scheduling | 1 |
| Pick Job Assigner API | Assigns warehouse pick jobs | 1 |
| Pick Message Consumer | Consumes RabbitMQ pick messages | 1 |
| Cherry Picker API | Cherry-picking operations | 3 |
| Cherry Packer API | Packing/boxing operations | 4 |
| Stock Issues API | Stock discrepancy management | 1 |
| Stock Transfer API | Inter-location stock transfers | 1 |
| Print Engine API | Print job management (NFS) | 1 |
| Replen API | Inventory replenishment | 1 |
| Allocation Changer API | Inventory allocation changes | 1 |
| RabbitMQ | Message broker | 1 |

### Infrastructure Components

- **MetalLB** — Bare-metal load balancer; IP pools `192.168.1.131` and `192.168.1.7`
- **RabbitMQ** — AMQP broker; requires node label `rabbit-storage: ssd` for PV affinity; local PV at `/run/desktop/mnt/host/c/storage/rabbit-ques` (5Gi)
- **NFS** — Network share `192.168.1.80:/public/CMD_Files/` (5Gi) mounted into Print Engine at `/app/cmdfiles`
- **Timezone** — All pods mount `/usr/share/zoneinfo/Europe/London` via hostPath

### Service Pattern

Every microservice follows this standard layout in a single `*-deploy.yaml` file:

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <service>-deploy
spec:
  replicas: <n>
  selector:
    matchLabels:
      app: <service>-api
  template:
    spec:
      containers:
        - image: docfahad/synapse:<service>-api
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
---
# ClusterIP Service
apiVersion: v1
kind: Service
metadata:
  name: <service>-clusterip-service
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
```

Some services also have a separate NodePort or LoadBalancer service file.

## Common Commands

```bash
# Apply entire cluster state
kubectl apply -f .

# Apply a single manifest
kubectl apply -f <service-name>-deploy.yaml

# Check status
kubectl get deployments,pods,services

# Stream logs
kubectl logs -f deployment/<deployment-name>

# Scale a service
kubectl scale deployment/<deployment-name> --replicas=<n>

# Check persistent storage
kubectl get pv,pvc

# Port-forward RabbitMQ Management UI (guest/guest)
kubectl port-forward svc/rabbitmq-loadbalancer 15672:15672
```

## File Naming Conventions

| Pattern | Contents |
|---|---|
| `*-deploy.yaml` | Deployment + ClusterIP Service (primary manifest per service) |
| `*-service.yaml` | Additional NodePort or LoadBalancer service |
| `*-pv.yaml` | PersistentVolume |
| `*-pvc.yaml` | PersistentVolumeClaim |
| `*-configmap.yaml` | ConfigMap (e.g., RabbitMQ config) |
| `jenkins-clusterrole*.yaml` | RBAC for CI/CD |
| `metallb-*.yaml` | MetalLB IPAddressPool / L2Advertisement |
| `nfs-*.yaml` | NFS StorageClass / PV / PVC |

Service name conventions:
- Deployment: `<service>-deploy`
- ClusterIP: `<service>-clusterip-service` or `<service>-api-clusterip-service`
- NodePort: `<service>-np-service`
- LoadBalancer: `<service>-loadbalancer`

## Git Branches

| Branch | Purpose |
|---|---|
| `master` | Main/stable branch |
| `rancher-infra` | Rancher K3s cluster (current) |
| `kind-infra` | KinD cluster (local dev) |

## Deployment Dependencies

Deploy in this order when standing up a fresh cluster:

1. MetalLB (`metallb-*.yaml`) — network first
2. PersistentVolumes (`*-pv.yaml`, `nfs-*.yaml`) — storage before workloads
3. PersistentVolumeClaims (`*-pvc.yaml`)
4. RabbitMQ (`rabbitmq-deploy.yaml`) — message broker before consumers
5. All remaining services (`kubectl apply -f .`)

Verify RabbitMQ PVC is `Bound` and the pod is `Running` before deploying Pick Message Consumer or other queue-dependent services.
