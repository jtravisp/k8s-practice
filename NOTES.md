# Notes

## Pick up here

**Next: OOMKilled drill.** Not started. `drills/oom.yaml` doesn't exist yet.

1. `k3d cluster start drills` (if stopped), then `kubectl config current-context` should say `k3d-drills`
2. Check: `curl.exe http://localhost:8082` returns the nginx page
3. Create `drills/oom.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom
spec:
  containers:
  - name: stress
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```

4. **Predict first:** What STATUS will it show, first and then after a minute or two? Will RESTARTS climb? Which component kills the process?
5. `kubectl apply -f drills/oom.yaml`, then `kubectl get pod oom -w` (Ctrl+C after about a minute), then `kubectl describe pod oom`. Find Last State, Reason, and Exit Code.

Why this drill matters: memory limits are the most common way model serving fails in production.
Requests are what the scheduler reserves. Limits are a hard cap the kernel enforces through cgroups.

Later drills: kill pods while curling in a loop (self-healing); k3s on one real VM (systemd, `journalctl -u k3s`, join token).

Interview prep to-do: check the Striveworks forks before mentioning them.
`gh repo list striveworks --fork --limit 100`. Look for Triton, KServe, MLServer.

---

## Concepts

### k3s vs k3d
- **k3s** is a Kubernetes distribution: one binary, runs as a systemd service on real machines. This is what Striveworks installs.
- **k3d** is a tool that runs k3s inside Docker containers. It isn't Kubernetes itself. Installing k3d alone runs nothing.
- In k3d, each node is a Docker container running k3s, and k3s's containerd runs the pods inside it. So pods are containers inside a container, all sharing one kernel.
- **What k3d replaces:** the machine layer (no systemd, no SSH, no real disks). The node containers stand in for the machines, and `serverlb` stands in for the network path to them.

### Cluster vs workload files
- `k3d/k3d-drills.yaml` describes **the cluster** and only k3d understands it.
- `web-deploy/*.yaml` describe **what runs inside** a cluster. They're standard Kubernetes files and work on k3d, kind, or EKS (the Ingress needs an ingress controller).
- That's the same split as Striveworks' setup: getting a cluster to exist (Terraform provider / Stampede / EKS AMI) is a separate problem from running workloads on it.

### Nodes: server vs agent
- Every server is a node, but not every node is a server. "Node" is the general term.
- **Server**: control plane (API server, scheduler, controller manager, datastore). One server uses SQLite; three or more use embedded etcd.
- **Agent**: kubelet plus containerd. It joins with the server address and a **join token**.
- **k3s servers also run workloads.** Traefik and one svclb pod ran on `server-0`. EKS hides its control plane; k3s doesn't, because at the edge one machine may be the whole cluster.
- Join tokens connect all three Striveworks repos: Terraform passes them over SSH, Stampede's elected leader hands them out, and k3d does it automatically.

### Contexts
- kubectl's target cluster is the current context in `~/.kube/config`. It isn't tied to the folder you're in.
- `k3d cluster create` switched the context to `k3d-drills` automatically.
- `kubectl config get-contexts` lists contexts, and `use-context` switches. Check before any apply or delete. A stale context pointing at prod is a classic mistake.

### Service types
Pods are disposable and their IPs change, so a Service gives them a stable name and IP and forwards to pods matching its **selector**.

| type | reachable from | adds |
|---|---|---|
| ClusterIP (default) | inside the cluster only | a stable internal IP |
| NodePort | anyone who can reach a node | a port in the 30000-32767 range on every node |
| LoadBalancer | an external address | **asks** for an outside address; something else has to provide it |

Kubernetes never creates load balancers itself. Who fulfills `type: LoadBalancer`:
EKS: AWS controller creates an NLB. kind: nobody, so it stays `<pending>`. k3s: **ServiceLB** (svclb pods). Bare metal in production: usually MetalLB.
Bare metal has no cloud API for load balancers. EKS hides this problem, but edge deployments have to solve it.

### Three things that aren't the same
| | serverlb | svclb-traefik pods | traefik |
|---|---|---|---|
| is | Docker container (nginx proxy) | pods, one per node | pod (ingress controller) |
| made by | k3d | k3s ServiceLB | k3s, via a Helm chart |
| k8s can see it | no | yes | yes |
| job | laptop -> nodes | node port 80 -> traefik Service | which app is this request for? |

- In `traefik   LoadBalancer   ...`, "LoadBalancer" is the Service **type**. Traefik is the app; the svclb pods fulfill its Service's LoadBalancer request.
- serverlb exists only because the nodes are containers. On real hardware it wouldn't be there.
- serverlb also proxies kubectl traffic: laptop `6693` -> API server `6443`.
- There are two different nginx: serverlb uses it as a **proxy** (first stop), and my `web` pods use it as a **web server** (final destination).
- `docker ps` shows serverlb but not my nginx pods, because the pods live inside the node containers' containerd.

### Full request path
```
browser -> localhost:8082
  -> serverlb          k3d, outside k8s       get it to a node
  -> svclb pod         node port 80           get it to traefik
  -> traefik           ingress controller     which app?
  -> web Service       ClusterIP + selector   which pod is alive?
  -> EndpointSlice     list of pod IPs
  -> web pod           nginx web server       returns the page
```

### The controller pattern
**The API stores what you want; controllers make it real.**
- LoadBalancer Service -> ServiceLB creates svclb pods
- Ingress -> traefik starts routing
- Service selector -> endpoint controller writes EndpointSlices
- Deployment -> ReplicaSet -> pods (`pod-template-hash` label keeps versions separate)
- KServe `InferenceService` -> KServe controller creates the serving stack (CRD + controller)

### Model serving (for the interview)
- **Triton** (NVIDIA): the inference server process inside the pod. Serves ONNX/PyTorch/TensorRT/TF over HTTP/gRPC, batches requests, shares GPUs.
- **KServe**: Kubernetes layer around it. You declare an `InferenceService`; its controller builds the Deployment, Service, routing, and autoscaling. Often runs Triton underneath.
- **MLServer** (Seldon): a Python server doing Triton's job. All three use the Open Inference Protocol.
- The claim that Striveworks forks these comes from my research notes. Verify it (see to-do above).

---

## Drill log

### Drill 0: Ingress (2026-11-02)
- Before the Ingress: `curl.exe http://localhost:8082` returned `404 page not found` from traefik. The traffic reached it but there was no rule.
- Added `web-deploy/web-ingress.yaml`. Now returns "Welcome to nginx!" CLASS shows `traefik`, the default ingress class in k3s.

### Drill 1a: Delete the Service (2026-11-02)
- Predicted 404 from traefik. **Correct.**
- Same 404 as having no Ingress at all, but a different cause: traefik drops a route whose Service doesn't exist.
- Diagnose with `kubectl describe ingress web` (Backends shows the error).
- The pods stayed healthy the whole time. Only the route was broken.
- Fix: `kubectl apply -f web-deploy/web-svc.yaml`

### Drill 1b: Selector typo (2026-11-02)
- `kubectl set selector svc web app=webb` (changed the live cluster, not the file, which causes drift)
- Predicted 404. **Wrong:** got `no available server` (HTTP 503).
- My mistake: I thought the selector pointed at a Service. **A selector picks pods by label.** The Service `web` still existed and the Ingress still found it; its endpoint list was just empty.
- Diagnose: `describe svc web` shows an empty `Endpoints:`, `get endpointslices -l kubernetes.io/service-name=web` shows `<unset>`, and `get pods --show-labels` shows the pods still have `app=web`.
- Fix: reapply the file (`service/web configured`). The file in git is the source of truth, and reapplying it fixes drift. GitOps tools (Argo CD, Flux) automate this.
- If a **pod's label** were changed instead, the ReplicaSet would stop counting that pod and create a replacement, leaving the old pod running unmanaged.

### What the response tells you (traefik)
| Symptom | Meaning | Look at |
|---|---|---|
| `404 page not found` | no working route | Ingress, and whether its Service exists |
| `no available server` / 503 | route exists, no healthy pods behind it | selector, endpoints, pod readiness |

Other controllers differ (ingress-nginx returns 503 for a missing backend). An error code tells you roughly where the request stopped, not why; check each layer to find out.

### Gotchas
- PowerShell: use `curl.exe`, because `curl` is an alias for `Invoke-WebRequest`.
- `kubectl` 1.34 (Docker Desktop) is within the supported ±1 skew of k3s 1.35, but 3 versions from the kind 1.37 cluster.
- Images built with local `docker build` aren't visible to the cluster. Use `k3d image import <img> -c drills`.
- `status: loadBalancer: {}` in `web-svc.yaml` is left over from `--dry-run -o yaml`. It's harmless and can be deleted.
- Rename folders with `git mv` so git records a rename instead of a delete plus an add.
