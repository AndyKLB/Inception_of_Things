# Inception of Things (IoT)

42 School project exploring **Kubernetes**, **K3s/K3d**, **Vagrant**, **GitOps with Argo CD** and a **self-hosted GitLab**, starting from a minimal cluster up to a full CI/CD chain.

The repo is split into four independent parts, each building on the previous one.

## Table of contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Part 1 — K3s cluster with Vagrant](#part-1--k3s-cluster-with-vagrant-p1)
- [Part 2 — Three apps behind an Ingress](#part-2--three-apps-behind-an-ingress-p2)
- [Part 3 — K3d + Argo CD (GitOps)](#part-3--k3d--argo-cd-gitops-p3)
- [Bonus — Self-hosted GitLab + Argo CD](#bonus--self-hosted-gitlab--argo-cd-bonus)

## Overview

| Part    | Tools                             | Goal                                                                 |
|---------|------------------------------------|------------------------------------------------------------------------|
| `p1`    | Vagrant, Alpine, K3s                | K3s cluster with 2 VMs (1 server + 1 worker)                           |
| `p2`    | Vagrant, K3s                        | 1 VM, 3 apps deployed and routed through a Traefik Ingress             |
| `p3`    | K3d, Argo CD                        | Local cluster + GitOps deployment from this Git repo                  |
| `bonus` | K3d, Argo CD, GitLab (Helm)         | Same GitOps chain, but synced from a self-hosted GitLab                |

## Prerequisites

- Linux with `sudo` access
- [Vagrant](https://www.vagrantup.com/) + VirtualBox (parts `p1` and `p2`)
- Docker (parts `p3` and `bonus`, required by K3d)
- `kubectl`, `k3d`, `helm` (auto-installed by the `install.sh` scripts)
- Write access to `/etc/hosts`

## Part 1 — K3s cluster with Vagrant (`p1`)

Two Alpine VMs provisioned by Vagrant:

- **Server** (`192.168.56.110`): K3s control plane
- **Worker** (`192.168.56.111`): joins the cluster using the join token

The server installs K3s, waits for the API to respond, then copies its `node-token` into the shared `/vagrant` folder. The worker waits until it finds that token before installing itself in agent mode and joining the server.

```bash
cd p1
vagrant up
vagrant ssh ankammerS -c "kubectl get nodes -o wide"
```

## Part 2 — Three apps behind an Ingress (`p2`)

A single K3s VM, with automatic deployment of 3 apps (`app1`, `app2`, `app3`) and a Traefik Ingress routing by hostname:

- `app1.com` → `app1` (1 replica)
- `app2.com` → `app2` (3 replicas)
- everything else → `app3` (default path)

```bash
cd p2
vagrant up
```

## Part 3 — K3d + Argo CD (GitOps) (`p3`)

Switches to a local **K3d** cluster (K3s in Docker), with **Argo CD** continuously syncing the cluster state to the contents of `p3/confs` in this Git repo.

```bash
cd p3
./scripts/install.sh
```

The script installs Docker, `kubectl`, `k3d`, creates the cluster, deploys Argo CD, exposes it via Ingress, and applies the Argo CD `Application` pointing at `p3/confs`.

Access:

| Service   | URL                            | Credentials                              |
|-----------|----------------------------------|--------------------------------------------|
| App       | http://local.ankammer.com:8080   | —                                          |
| Argo CD   | http://local.argo.com:8080       | `admin` / password printed at end of script |

⚠️ Add to `/etc/hosts` beforehand:

```
127.0.0.1 local.ankammer.com local.argo.com
```

**GitOps demo**: change the image in `p3/confs/deployment.yaml` (`v1` → `v2`), commit + push, then wait for the automatic sync (~3 min, or force it from the Argo CD UI).

## Bonus — Self-hosted GitLab + Argo CD (`bonus`)

Builds on part 3 by replacing the Git source with a **self-hosted GitLab** running inside the cluster (official Helm chart), with PostgreSQL, Redis and MinIO deployed manually (these dependencies were dropped from the GitLab chart as of version 10.0).

```bash
cd bonus
./scripts/install.sh
```

The script installs Docker, `kubectl`, `k3d`, `helm`, creates the cluster (`myClusterBonus`), deploys Argo CD, the GitLab dependencies, then GitLab itself via Helm.

Access:

| Service   | URL                            | Credentials                                 |
|-----------|----------------------------------|-----------------------------------------------|
| App       | http://local.ankammer.com:8080   | —                                              |
| Argo CD   | http://local.argo.com:8080       | `admin` / password printed at end of script    |
| GitLab    | http://local.gitlab.com:8080     | `root` / password printed at end of script     |

⚠️ Add to `/etc/hosts` beforehand:

```
127.0.0.1 local.ankammer.com local.argo.com local.gitlab.com
```

Once GitLab is up: create a public project `ankammer-iot`, push this repo to it, then apply `bonus/confs/argocd/argocd-app.yaml` so Argo CD syncs from this GitLab instead of GitHub.
