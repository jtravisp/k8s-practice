# k8s-practice

Kubernetes review for a Striveworks interview. Practice runs on a local **k3d** cluster
(k3s v1.35.5 in Docker), because Striveworks ships k3s to edge and bare-metal customers.
Earlier work used kind (v1.37); those clusters still exist as separate kubectl contexts.

Ground rule: I run every command myself. Claude explains, I type.

## Tracks

- Deploy, expose, update
- Break-fix drills
- Config and security
- Own image + CI/CD
- Concepts review

## Layout

| Path | What it is | Read by |
|---|---|---|
| `k3d/k3d-drills.yaml` | Cluster definition: 1 server, 2 agents, k3s pinned to v1.35.5, laptop `8082` -> loadbalancer `80` | k3d |
| `web-deploy/` | nginx Deployment (5 replicas), ClusterIP Service, Ingress (`/` -> `web:80`) | kubectl |
| `NOTES.md` | What I've learned, drill results, where to pick up | me |

## Start / stop / rebuild

```powershell
# Is the cluster there?
k3d cluster list

# Stop and start it without losing anything (frees CPU and RAM overnight)
k3d cluster stop drills
k3d cluster start drills

# Rebuild from scratch
k3d cluster delete drills
k3d cluster create --config k3d/k3d-drills.yaml
kubectl apply -f web-deploy/
```

Before any `apply` or `delete`, run `kubectl config current-context`. It should say `k3d-drills`.

Smoke test: `curl.exe http://localhost:8082` returns "Welcome to nginx!"
