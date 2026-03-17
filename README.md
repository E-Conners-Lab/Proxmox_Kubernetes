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
| Storage | Longhorn |
| Secrets | HashiCorp Vault (K8s auth + Agent Injector) |
| Vector DB | ChromaDB |
| Network Policy | Cilium (least-privilege egress) |

## Cluster layout

| Node | Role | Resources |
|---|---|---|
| k3s-server | Control plane | 4 vCPU, 8GB RAM, 50GB disk |
| k3s-worker-01 | Worker | 8 vCPU, 32GB RAM, 100GB disk |
| k3s-worker-02 | Worker | 8 vCPU, 32GB RAM, 100GB disk |

## Build phases

**Phase 1 — Cluster build** ✅
VM template creation, k3s install, Cilium CNI, MetalLB, NGINX Ingress. Everything needed to get a working cluster that can accept workloads.

**Phase 2 — Storage, secrets, and first workload** ✅
Longhorn distributed storage, HashiCorp Vault in production mode with Kubernetes auth and Agent Injector, ChromaDB vector database, and Cilium network policies enforcing least-privilege egress.

**Phase 3 — Agent deployment (current)**
Deploy the FastAPI agent backend, wire up Vault secret injection, connect to ContainerLab network devices, and run the first autonomous troubleshooting session.

**Phase 4 — Observability**
Prometheus, Grafana, and Loki for monitoring, dashboards, and log aggregation across the cluster and workloads.

## Repo structure

```
docs/
  k3s-proxmox-runbook.pdf                  # Phase 1 runbook — full cluster build
phase2-storage-secrets-runbook.html        # Phase 2 runbook — source (HTML)
phase2-storage-secrets-runbook.pdf         # Phase 2 runbook — printable (PDF)
```

## Getting started

Start with the Phase 1 runbook ([`docs/k3s-proxmox-runbook.pdf`](docs/k3s-proxmox-runbook.pdf)) for the full cluster build, then follow the Phase 2 runbook ([`phase2-storage-secrets-runbook.pdf`](phase2-storage-secrets-runbook.pdf)) for storage, secrets, and the first workload. All personal info has been replaced with placeholders — fill in the quick reference table on the last page of each runbook with your environment details before starting.

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
