# Proxmox Kubernetes

Production-grade Kubernetes cluster on Proxmox, built from scratch to host AI and network automation workloads.

## What this is

A 3-node k3s cluster running on Proxmox VMs with Cilium (eBPF networking), MetalLB (bare metal load balancing), and NGINX Ingress. Designed to run real workloads on real infrastructure, not a toy lab.

This repo contains the runbooks, manifests, and configurations used throughout the build. It's the companion repo for the [From Proxmox to Production](https://youtube.com) video series on The Tech-E YouTube channel.

## Cluster stack

| Component | Choice |
|---|---|
| Hypervisor | Proxmox VE |
| Kubernetes | k3s (no Traefik, no servicelb, no Flannel) |
| CNI | Cilium with kube-proxy replacement (eBPF) |
| Load balancer | MetalLB (L2 mode) |
| Ingress | NGINX Ingress Controller |
| Storage | Longhorn (Phase 2) |
| Secrets | Sealed Secrets (Phase 3) |

## Cluster layout

| Node | Role | Resources |
|---|---|---|
| k3s-server | Control plane | 4 vCPU, 8GB RAM, 50GB disk |
| k3s-worker-01 | Worker | 8 vCPU, 32GB RAM, 100GB disk |
| k3s-worker-02 | Worker | 8 vCPU, 32GB RAM, 100GB disk |

## Build phases

**Phase 1 — Cluster build (current)**
VM template creation, k3s install, Cilium CNI, MetalLB, NGINX Ingress. Everything needed to get a working cluster that can accept workloads.

**Phase 2 — Storage**
Longhorn distributed storage across worker nodes. Persistent volumes for databases and application state.

**Phase 3 — Application deployment**
Deploy the full application stack: FastAPI backend, ChromaDB vector database, Redis, and supporting services. Sealed Secrets for API key management. Cilium network policies for least-privilege egress.

**Phase 4 — Observability**
Prometheus, Grafana, and Loki for monitoring, dashboards, and log aggregation across the cluster and workloads.

## Repo structure

```
docs/
  k3s-proxmox-runbook.pdf    # Phase 1 runbook — full cluster build
manifests/                    # Kubernetes manifests (coming in Phase 2+)
```

## Getting started

Download the Phase 1 runbook from [`docs/k3s-proxmox-runbook.pdf`](docs/k3s-proxmox-runbook.pdf). It walks through every command from VM creation to a validated cluster. All personal info has been replaced with placeholders — fill in the quick reference table on the last page with your environment details before starting.

### Prerequisites

- A Proxmox server (any hardware with enough RAM for 3 VMs)
- SSH key pair on your dev machine
- kubectl installed on your dev machine

## Links

- [The Tech-E](https://thetech-e.com)
- [YouTube — From Proxmox to Production series](https://youtube.com)
- [LinkedIn — Elliot Conner](https://www.linkedin.com/in/elliot-conner-49240111/)

## License

MIT
