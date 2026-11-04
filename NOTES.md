# Notes

## Pick up here

1. `k3d cluster start drills` (if stopped), then `kubectl config current-context` should say `k3d-drills`
2. Check: `curl.exe http://localhost:8082` returns the nginx page
3. Commit anything left over: the readiness probe in `web-deploy/web-deploy.yaml`, plus `drills/oom.yaml` (keep or delete)
4. **Warm-up questions (unanswered from last session):**
   - A pod is `Running` but `0/1` and isn't restarting. Readiness or liveness?
   - A Service has empty Endpoints. What's the first command you run, and what are you looking for in its output?
5. **Next drill (pick one):**
   - **Liveness probes**: the restart version of the readiness drill. Same YAML shape, very different consequences.
   - **Self-healing**: curl in a loop while deleting `web` pods. Watch the ReplicaSet replace them and the Service route around them.
   - Later: k3s on one real VM (systemd, `journalctl -u k3s`, join token): the machine layer k3d skips.

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
- **Triton is to models what nginx is to web pages.** You give it trained model files, clients send inputs (images), and it returns predictions. It serves models but doesn't train them.
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

### Stop / start persistence (2026-11-03)
- After `k3d cluster stop` / `start`, everything was still there without reapplying. Single-server k3s keeps state in **SQLite** at `/var/lib/rancher/k3s/server/db/state.db` inside server-0, and stopping a container keeps its disk.
- The API server only knows how to talk to etcd. k3s's **kine** layer translates the etcd protocol to SQL.
- Pod objects survived (AGE 25h). Their containers restarted (`Restarts 1`).
- `stop` keeps state; `delete` loses it. That's why the manifests are in git.
- `k3d-drills-tools` stays stopped after `start`. It's a setup helper (`noop`), not a node.

### Drill 2: OOMKilled (2026-11-03)
- Predictions: OOM ✓, restarts ✓, containerd kills it ✗ (**the kernel** kills it).
- How the limit is enforced: YAML limit → kubelet → containerd/runc writes cgroup `memory.max` → **Linux kernel OOM killer** sends SIGKILL → kubelet reports `OOMKilled`.
- **Exit 137 = 128 + 9 (SIGKILL)**, killed hard. 143 = 128 + 15 (SIGTERM), a normal shutdown.
- `CrashLoopBackOff` is a **waiting state** (10s, 20s, 40s, up to 5m), not a cause. The cause is in `describe` → Last State → Reason.
- The kubelet restarts the container in place: same pod, same node, same IP.
- `kubectl logs <pod> --previous` shows logs from the container that crashed.
- No image tag means `:latest`, which is re-pulled on every restart. Pin tags (this matters on slow edge networks).
- QoS: requests < limits → **Burstable**; requests = limits → **Guaranteed**; neither set → **BestEffort** (evicted first).
- Fix: raise the limit (more memory) or lower `--vm-bytes` (like quantizing). Change one thing at a time when debugging.
- Model serving: weights bigger than the limit → killed on load → 137 → CrashLoopBackOff, often with no useful logs.
- Units: `Mi` = 1,048,576 bytes (binary), `M` = 1,000,000 (decimal). Use `Mi`/`Gi`. **`memory: 100m` = 0.1 bytes** (`m` means milli, correct only for CPU).

### Drill 3: Bad readiness probe (2026-11-03)
- **Running** = the process is alive. **Ready** = the pod can serve traffic. Without a probe, Kubernetes treats them as the same.
- The kubelet runs the probe. HTTP 200–399 passes. Failing readiness → pod removed from the EndpointSlice → **no traffic, no restart**. (Failing liveness → restart.)
- For a pod to get traffic, it needs **both** a selector match **and** Ready.
- Added `readinessProbe: httpGet /healthz` → nginx returns 404 → pods stuck `0/1`, `Unhealthy (x14)`, `Restart Count: 0`.
- Predicted 5 pods. Actual: **7** (4 old `1/1` + 3 new `0/1`). Changing the template doesn't wipe the Deployment; it **rolls**: a new ReplicaSet scales up as the old one scales down, and each step waits for readiness.
  - maxSurge 25% of 5 → rounds **up** to 2 (max 7 pods). maxUnavailable 25% → rounds **down** to 1 (min 4 ready). So the rollout got stuck.
  - `rollout status` hangs. After 10 min (`progressDeadlineSeconds`) it's marked failed. **No automatic rollback.**
- curl still worked, because the 4 **old** pods served it. The new pods' nginx would have served `/` fine, but they weren't in the endpoint list. **A probe controls traffic based on what it checks, not on whether the app actually works.** A wrong probe can keep a healthy app out of service.
- Fix: `path: /` in the file, then apply → a **third** ReplicaSet (new template = new hash). The old ReplicaSets stay at 0 as rollout history (`kubectl rollout history deployment/web`, keeps 10).
- If the template exactly matched an old one, Kubernetes would scale that ReplicaSet back up. That's how `rollout undo` works.
- Triton exposes `/v2/health/ready`, which returns 200 only after models load. Point the readiness probe there.

### Diagnosing empty Endpoints
| `get pods` READY | Cause | Clue |
|---|---|---|
| `1/1` | selector doesn't match | compare `describe svc` Selector vs `get pods --show-labels` |
| `0/1` | readiness failing | `describe pod` Events: `Readiness probe failed` |

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
