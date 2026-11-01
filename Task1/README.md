# Kubernetes on KVM with minikube: Full Walkthrough

A beginner-friendly record of building a multi-node Kubernetes cluster on KVM virtual machines (watched in Virtual Machine Manager), deploying an nginx app with a sidecar, and every problem hit along the way, with root causes and fixes.

**Machine:** Intel i5-8350U (8 threads), 15 GB RAM, Ubuntu 24.04
**Tools:** KVM/libvirt, virt-manager, minikube v1.39.0 (kvm2 driver), kubectl, Kubernetes v1.37.0

---

## 1. Goal

1. Build a cluster with a **master (control-plane)** and **worker** node(s) on KVM VMs.
2. See the VMs in **Virtual Machine Manager**.
3. Deploy `web-deployment` (3 replicas) in namespace `webapp`: nginx plus a `time-updater` sidecar that rewrites nginx's home page every 5 seconds, with **no shared volumes**.

---

## 2. Setup from scratch

### 2.1 Check the machine
```bash
egrep -c '(vmx|svm)' /proc/cpuinfo   # must be > 0 (CPU supports virtualization)
df -h /                              # want 15+ GB free
free -h                              # want 8+ GB available
```

### 2.2 Install KVM, libvirt and the GUI
```bash
sudo apt update
sudo apt install -y cpu-checker qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager
kvm-ok
sudo usermod -aG libvirt,kvm $USER
sudo systemctl enable --now libvirtd
```
**Log out and back in**, then check that `groups` lists `libvirt` and `kvm`.

> Your teacher's `usermod -aG docker $USER` advice is for `--driver=docker`. With the KVM driver the groups that matter are `libvirt` and `kvm`. In both cases, run minikube as your normal user, never with `sudo`.

### 2.3 Install minikube and kubectl
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
sudo snap install kubectl --classic
```

### 2.4 Create the cluster (the command that worked)
```bash
# 2 nodes (1 master + 1 worker)
minikube start --driver=kvm2 --nodes 2 --cpus 2 --memory 3072 --disk-size 10g -p mycluster

# 3 nodes with Calico CNI (needs a LOWERCASE profile name)
minikube start --driver=kvm2 --nodes 3 --cni=calico --cpus 2 --memory 2560 --disk-size 10g -p mlopstaskcluster
```
If it errors at the end while joining the worker, run `minikube start -p <profile>` again.

```bash
kubectl get nodes -o wide
```
Open **virt-manager**: each node appears as a running VM.

---

## 3. Problems, root causes and solutions

### Problem 1: Disk almost full
- **Symptom:** `df -h /` showed 8 GB free (97% used).
- **Root cause:** minikube downloads about 750 MB (ISO plus Kubernetes preload) and creates a VM disk per node (default 20 GB each).
- **Fix:** cleaned up with `sudo apt clean && sudo apt autoremove -y`, `sudo journalctl --vacuum-size=200M`, and by deleting old files (17 GB free afterwards). Used `--disk-size 10g`.

### Problem 2: `domain didn't return IP after 1m30s`
- **Symptom:** minikube created the VM, waited 90 seconds, failed. It repeated on every retry.
- **Wrong guesses along the way:**
  - The libvirt log line `internal error: wrong nlmsg len` (a netlink error that only breaks IP reporting).
  - AppArmor denials for `virt-aa-helper` on `~/.minikube` (real, but not what stopped the VM).
- **How the real cause was found:** looking at the VM's own console.
  ```bash
  virsh -c qemu:///system list --all
  virsh -c qemu:///system console mycluster      # exit with Ctrl+]
  sudo tail -30 /var/log/libvirt/qemu/mycluster.log
  ```
  The console showed `Kernel panic - not syncing: System is deadlocked on memory`.
- **Root cause:** the minikube v1.39 boot image unpacks its root filesystem into RAM. With `--memory 2048` that did not fit, so the VM crashed about 5 seconds after boot. A crashed VM never requests an IP, so none ever appeared.
- **Fix:** more memory. `--memory 3072` worked, and later `2560` worked too.
- **Lesson:** when a VM has no IP, **look at its console first** before changing system settings.

### Problem 3: A fix undone by `--purge`
- ACLs were set on `~/.minikube`, then `minikube delete --purge` removed that folder, and the 750 MB was downloaded again.
- **Fix:** use `minikube delete -p <profile>` without `--purge` unless a full reset is wanted.

### Problem 4: `nodes "<name>-m02" not found` (worker join error), two different causes

| Case | Profile name | Cause | Fix |
|---|---|---|---|
| `mycluster` | lowercase | Timing: minikube labelled the node before the worker's kubelet registered | Run `minikube start -p mycluster` again |
| `mlopsTaskCluster` | has capitals | Kubernetes node names must be lowercase, so the node registers as `mlopstaskcluster-m02` but minikube looks for `mlopsTaskCluster-m02`. (Strong match to the error; not confirmed by a retest in this session.) | Recreate with a lowercase profile name |

```bash
minikube delete -p mlopsTaskCluster
minikube start --driver=kvm2 --nodes 3 --cni=calico --cpus 2 --memory 2560 --disk-size 10g -p mlopstaskcluster
```

### Problem 5: Mistakes in `deployment.yaml`
- `port:` must be **`ports:`** (plural), or the port is not declared.
- `shareProcessNamespace: true` was missing. Without it, `pgrep` in the sidecar cannot see nginx, `$PID` stays empty, and the page never updates (Pods still show Running, which hides the bug).

### Problem 6: Failed rollout while testing the init container
- **Symptom:** after editing and re-applying, the new Pods showed `1/2 CrashLoopBackOff`, and one old Pod vanished.
- **Reading it:** a failing init container gives `Init:Error` with `0/2`. `1/2` means init passed and a main container crashed. The exact crash was not diagnosed because the logs were not captured (`kubectl logs <pod> -c <container> --previous` would show it).
- **Why a Pod disappeared:** rolling updates replace old Pods as new ones count as available. If new Pods then crash, the rollout stalls with a mix of old and new Pods.
- **Fix:** `kubectl rollout undo deployment/web-deployment -n webapp`. Pods returned to the original ReplicaSet (`6754d96947`), 3 Pods at `2/2 Running`.
- **Gotchas:**
  - Always pass `-n webapp`, or kubectl says "not found".
  - `rollout undo` changes the cluster, not your local file. Keep a good copy: `cp deployment.yaml deployment-good.yaml`.
  - Check `kubectl config current-context` when several minikube profiles exist.

### Harmless warnings
- `Unable to resolve the current Docker CLI context`: Docker is installed but not used by kvm2. Fix with `docker context use default` if it bothers you.
- `kubectl is version 1.35, cluster 1.37`: within the supported version difference.
- Worker `ROLES` shows `<none>`: cosmetic. `kubectl label node <worker> node-role.kubernetes.io/worker=worker`.

---

## 4. The application

### 4.1 Namespace and skeleton
```bash
kubectl create namespace webapp
kubectl create deployment web-deployment --image=nginx:1.27 --replicas=3 -n webapp --dry-run=client -o yaml > deployment.yaml
sed -i 's/app: web-deployment/app: web/' deployment.yaml
```

### 4.2 Final `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
  name: web-deployment
  namespace: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      shareProcessNamespace: true
      initContainers:
        - name: init-done
          image: busybox:1.36
          command: ["sh", "-c", "echo done"]
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          command: ["/bin/sh", "-c"]
          args:
            - |
              echo "<h1>hello in MLOPS MENA COMMUNITY : $(date '+%Y-%m-%d %H:%M:%S')</h1>" > /usr/share/nginx/html/index.html
              nginx -g 'daemon off;'
        - name: time-updater
          image: busybox:1.36
          securityContext:
            capabilities:
              add: ["SYS_PTRACE"]
          command: ["sh", "-c"]
          args:
            - |
              while true; do
                PID=$(pgrep -o nginx)
                if [ -n "$PID" ]; then
                  echo "<h1>hello in MLOPS MENA COMMUNITY : $(date '+%Y-%m-%d %H:%M:%S')</h1>" \
                    > /proc/$PID/root/usr/share/nginx/html/index.html
                fi
                sleep 5
              done
```

### 4.3 Why each piece exists

| Piece | Purpose |
|---|---|
| `shareProcessNamespace: true` | Containers in the Pod see each other's processes (Pod-level setting) |
| Init container `init-done` | Runs once and must succeed before the main containers start |
| `command` + `args` on nginx | Writes the first page, then runs nginx in the foreground (`daemon off;`) so the container stays alive |
| `pgrep -o nginx` | Finds the oldest nginx process (the master) |
| `/proc/$PID/root/...` | nginx's filesystem seen from the sidecar, so no volume is needed |
| `SYS_PTRACE` capability | Permission required to write through `/proc/<pid>/root` |
| No `volumes` | The task forbids them; `/proc` replaces a shared volume |

### 4.4 Deploy and verify
```bash
kubectl apply --dry-run=server -f deployment.yaml     # validate first
kubectl apply -f deployment.yaml
kubectl get pods -n webapp -w                          # wait for 2/2 Running, Ctrl+C
kubectl get pods -n webapp -o wide                     # see which node each Pod is on
```
Pods show `2/2` because the two regular containers count and the finished init container does not.

Further checks used:
```bash
kubectl logs deploy/web-deployment -n webapp -c init-done          # prints: done
kubectl exec deploy/web-deployment -n webapp -c time-updater -- ps  # nginx processes visible
kubectl exec deploy/web-deployment -n webapp -c nginx -- cat /usr/share/nginx/html/index.html
kubectl get deploy web-deployment -n webapp -o jsonpath='{.spec.template.spec.volumes}'   # empty = no volumes
```
Running the `cat` twice a few seconds apart showed the timestamp changing (13:48:18, then 13:48:23), which proves the sidecar updates nginx's file.

### 4.5 Browser test
```bash
kubectl port-forward -n webapp deploy/web-deployment 8080:80     # keep running
curl -s localhost:8080; sleep 6; curl -s localhost:8080          # second terminal
```
Open http://localhost:8080 and refresh after 5 seconds: the time changes. `port-forward` is tied to one Pod, so restart it after any Pod restart.

---

## 5. Everyday operations

```bash
# Restart all Pods gradually
kubectl rollout restart deployment/web-deployment -n webapp
kubectl rollout status deployment/web-deployment -n webapp

# Bring everything down, then up
kubectl scale deployment/web-deployment -n webapp --replicas=0
kubectl scale deployment/web-deployment -n webapp --replicas=3

# History and rollback
kubectl rollout history deployment/web-deployment -n webapp
kubectl rollout undo deployment/web-deployment -n webapp

# Test a failing init container (keep a good copy first)
cp deployment.yaml deployment-good.yaml
# edit init command to: ["sh", "-c", "exit 1"], then:
kubectl apply -f deployment.yaml
kubectl get pods -n webapp -w          # expect Init:Error, READY 0/2
kubectl apply -f deployment-good.yaml  # restore

# Pause or remove the cluster
minikube stop -p <profile>
minikube delete -p <profile>
```

---

## 6. Troubleshooting cheat sheet

| Symptom | First thing to check |
|---|---|
| `didn't return IP` | `virsh -c qemu:///system console <vm>`: look for a kernel panic, raise `--memory` |
| `nodes "...-m02" not found` | Profile name has capitals? Otherwise just run `minikube start -p <profile>` again |
| Pod `CrashLoopBackOff` | `kubectl logs <pod> -n webapp -c <container> --previous` and `kubectl describe pod ...` |
| `Init:Error` / `0/2` | The init container failed: `kubectl logs <pod> -c init-done` |
| Page never updates | `shareProcessNamespace: true` missing, or `SYS_PTRACE` missing |
| `not found` on kubectl commands | Missing `-n webapp`, or wrong cluster (`kubectl config current-context`) |
| Cluster slow or host freezing | `free -h` and `df -h /`: reduce memory or free disk |

---

## 7. Lessons learned

1. Look at the VM console before changing host settings.
2. Fix one thing at a time, and retest to confirm the cause (some of today's early theories were wrong).
3. Use lowercase profile names in minikube.
4. Keep a known-good copy of every YAML before experimenting.
5. Always include `-n <namespace>` in kubectl commands.
6. Watch memory (about 2.5 to 3 GB per node needed) and disk space on a laptop.

## 8. Items for the report

- `kubectl get nodes -o wide`
- Virt-manager screenshot with all node VMs running
- `kubectl get pods -n webapp -o wide`
- Browser page before and after a refresh (time changed)
- `kubectl get deploy web-deployment -n webapp -o jsonpath='{.spec.template.spec.volumes}'` (empty output)
