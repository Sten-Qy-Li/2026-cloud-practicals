# Practice 11

Practical webpage: https://courses.cs.ut.ee/2026/cloud/spring/Main/Practice11

> Theme: Two-node K3s (Kubernetes) cluster on OpenStack — pods, replicasets, deployments, services, secrets, persistent volumes.

> Note: the existing VM at `172.17.65.54` is **not** used here — this practical requires **two fresh VMs**. SSH into each using the same key: `ssh -i "C:\Users\qunyan\Desktop\Qun Yan Li.pem" ubuntu@<VM_IP>`.

> Reminder: If you need Docker Hub pulls and hit rate limits, swap the image prefix to `registry.hpc.ut.ee/mirror/<image>`.

---

## Exercise 11.1 — Build the Two-Node K3s Cluster

### Step-by-step

1. Prepare the OpenStack security group first (so the VMs have it applied at launch):
   - OpenStack dashboard → **Project → Network → Security Groups** → check if `lab11-k3s` already exists; otherwise **+ Create Security Group** → Name: `lab11-k3s` → **Create**.
   - Open `lab11-k3s` → **Manage Rules → + Add Rule** and add the three rules:
     - Rule: **SSH** (or Custom TCP Rule, port 22) · Remote: **CIDR** `0.0.0.0/0` → **Add**.
     - Rule: **Custom TCP Rule** · Port Range: `6443` · Remote: CIDR `0.0.0.0/0` → **Add** (K3s API server).
     - Rule: **Custom TCP Rule** · Port Range: `30000 – 32767` · Remote: CIDR `0.0.0.0/0` → **Add** (NodePort services).

2. Launch two VMs. Do **each** twice (once named `Lab11_Li_Master`, once `Lab11_Li_Worker`):
   - **Project → Compute → Instances → Launch Instance**.
   - **Details tab:** Instance Name: `Lab11_Li_Master` (or `Lab11_Li_Worker`) · Count: `1`.
   - **Source tab:** Select Boot Source: **Image** · Create New Volume: **Yes** · Volume Size: `15 GB` · Delete Volume on Instance Delete: **Yes** · Image: **Ubuntu 22.04 LTS** (click the up-arrow next to it).
   - **Flavor tab:** select `g4.r4c2` (up-arrow).
   - **Networks tab:** `shared` or the default student network (up-arrow).
   - **Security Groups tab:** deselect `default` · select `lab11-k3s` (up-arrow).
   - **Key Pair tab:** select your existing key (the one matching `C:\Users\qunyan\Desktop\Qun Yan Li.pem`).
   - **Launch Instance**. Repeat for the second VM.

3. Once both show **Status: Active** and **Power State: Running**, copy both public IPs from the **IP Address** column (use the Floating IP if one is assigned; raw VMs on the `shared` net already have a routable IP).

4. SSH into **Master** from Windows PowerShell:
   ```powershell
   ssh -i "C:\Users\qunyan\Desktop\Qun Yan Li.pem" ubuntu@<MASTER_IP>
   ```
5. On Master, install K3s server and read the node token in one chained command:
   ```bash
   curl -sfL https://get.k3s.io | sh - && sudo cat /var/lib/rancher/k3s/server/node-token
   ```
   Copy the full token (starts with a long hex string, a `::`, then `server:` and more hex) — you need it in step 7.

6. Open a **second** Windows PowerShell window and SSH into **Worker**:
   ```powershell
   ssh -i "C:\Users\qunyan\Desktop\Qun Yan Li.pem" ubuntu@<WORKER_IP>
   ```

7. On Worker, join the cluster (single line, substitute `<MASTER_IP>` and paste `<NODE_TOKEN>` from step 5):
   ```bash
   curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN=<NODE_TOKEN> sh -
   ```

8. Back on **Master** (the first SSH window), verify both nodes are `Ready`:
   ```bash
   sudo kubectl get nodes -o wide
   ```

9. (Optional but recommended — avoid `sudo` on every `kubectl` call.) On Master:
   ```bash
   echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc && sudo chmod 644 /etc/rancher/k3s/k3s.yaml && source ~/.bashrc
   ```

### Exercise Deliverables

- Screenshot of `kubectl get nodes -o wide` output showing two `Ready` nodes — save as `11_1_nodes.png`.

### Checklist (fill in before proceeding)

- [ ] Both VMs up and reachable over SSH.
- [ ] Master shows Worker as `Ready`.
- [ ] Screenshot archived.

---

## Exercise 11.2 — Pods, Namespaces, ReplicaSets

### 11.2.1 — Namespaces

On **Master**, run the list/create pair:
```bash
kubectl get namespaces && kubectl create namespace li && kubectl get namespaces
```

### 11.2.2 — Pod via CLI and YAML

Imperative busybox pod:
```bash
kubectl run busybox --image=busybox --restart=Never -n li -- /bin/sh -c "echo Hello Kubernetes!" && kubectl logs busybox -n li
```

Create `first_pod.yml` on Master:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: first
  namespace: li
  labels:
    app: first
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["/bin/sh", "-c", "while true; do echo hello from first; sleep 5; done"]
```
Deploy, inspect, exec — chained:
```bash
kubectl create -f first_pod.yml && kubectl get pods -n li && kubectl logs first -n li
```
Shell into it when you want:
```bash
kubectl exec -it first -n li -- /bin/sh
```

### 11.2.3 — ReplicaSet

Create `replicaset.yml`:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web
  namespace: li
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
```
Apply + scale experiments:
```bash
kubectl apply -f replicaset.yml && kubectl get rs -n li && kubectl get pods -n li
kubectl scale --replicas=10 rs/web -n li && kubectl get pods -n li
kubectl scale --replicas=1 rs/web -n li && kubectl get pods -n li
```
Capture the describe output for the deliverable:
```bash
kubectl describe pod first -n li | less
```

### Exercise Deliverables

- `first_pod.yml`, `replicaset.yml` committed to the repo.
- Screenshot of `kubectl describe pod first -n li` — save as `11_2_describe.png`.
- Screenshot of `kubectl get namespaces` — save as `11_2_namespaces.png`.

### Checklist (fill in before proceeding)

- [ ] Namespace `li` exists.
- [ ] Pod `first` runs and logs show the loop output.
- [ ] ReplicaSet scaled up to 10 then back to 1 successfully.
- [ ] Both YAML files saved.

---

## Exercise 11.3 — Deployments, Multi-Service, Secrets

### 11.3.1 — Flask Deployment

Create `first_deployment.yml` (swap `<DOCKERHUB_ID>` for your own, e.g. `qunyanli/lab3flask1.0`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-deployment
  namespace: li
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask
  template:
    metadata:
      labels:
        app: flask
    spec:
      containers:
      - name: flask
        image: <DOCKERHUB_ID>/lab3flask1.0
        ports:
        - containerPort: 5000
```
Apply, rolling-update, clean up:
```bash
kubectl apply -f first_deployment.yml && kubectl get deployments -n li
kubectl set image deployments/flask-deployment flask=shivupoojar/message-board.v2 -n li && kubectl rollout status deployment/flask-deployment -n li
kubectl delete -f first_deployment.yml
```

### 11.3.2 + 11.3.3 + 11.3.4 — Combined Message-Board File

Create `message-board.yml` (two deployments separated by `---`). Replace `<DOCKERHUB_ID>` and `<YOUR_PASSWORD>`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-deployment
  namespace: li
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask
  template:
    metadata:
      labels:
        app: flask
    spec:
      containers:
      - name: flask
        image: <DOCKERHUB_ID>/lab3flask1.0
        ports:
        - containerPort: 5000
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret-config
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret-config
              key: password
        - name: DATABASE_URL
          value: "postgresql://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@postgresql:5432/postgres"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
  namespace: li
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14.1-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret-config
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret-config
              key: password
```
Create the secret and apply everything:
```bash
kubectl create secret generic postgres-secret-config -n li --from-literal=username=postgres --from-literal=password=<YOUR_PASSWORD> && kubectl get secret postgres-secret-config -n li -o yaml
kubectl apply -f message-board.yml && kubectl get deployments -n li && kubectl get pods -n li
```

### Exercise Deliverables

- `first_deployment.yml`, `message-board.yml` committed.
- Screenshot of `kubectl get deployments -n li` — save as `11_3_deployments.png`.

### Checklist (fill in before proceeding)

- [ ] Secret created and visible.
- [ ] Both deployments show desired replicas.
- [ ] Transient CrashLoopBackOff is acceptable at this stage — the services are not yet in place.

---

## Exercise 11.4 — Services + Scaling

### Step-by-step

Append these two service blocks to `message-board.yml` (at the bottom, each starting with `---`):
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: service-flask
  namespace: li
spec:
  type: NodePort
  selector:
    app: flask
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
    name: tcp-5000
---
apiVersion: v1
kind: Service
metadata:
  name: postgresql
  namespace: li
  labels:
    name: postgresql
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
```
Apply, inspect, grab the NodePort, test scaling:
```bash
kubectl apply -f message-board.yml && kubectl get pods -o wide -n li && kubectl get svc -n li
kubectl scale deployment flask-deployment --replicas=2 -n li && kubectl get pods -n li && kubectl get deployment -n li
```
From your laptop, browse `http://<MASTER_IP>:<NODEPORT>` (NodePort starts with `3...`). If it times out, open the NodePort range in the OpenStack security group.

### Exercise Deliverables

- Screenshot of the Message Board webpage reached via NodePort — save as `11_4_webpage.png`.
- Screenshot of `kubectl get pods` after scaling to 2 replicas — save as `11_4_scale.png`.

### Checklist (fill in before proceeding)

- [ ] Messages can be posted via the browser.
- [ ] Scaled Flask deployment shows 2 running replicas.

---

## Exercise 11.5 — Persistent Volume + Claim

### Step-by-step

Create `data-volume.yml`:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv-volume
  namespace: li
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-db-claim
  namespace: li
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
  storageClassName: manual
```
Modify the Postgres block of `message-board.yml` — add `PGDATA` env and the mount/volume:
```yaml
        env:
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        # keep POSTGRES_USER + POSTGRES_PASSWORD from before
        volumeMounts:
        - name: data-storage-volume
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: data-storage-volume
        persistentVolumeClaim:
          claimName: postgres-db-claim
```
Apply + verify:
```bash
kubectl apply -f data-volume.yml && kubectl get pv,pvc -n li
kubectl apply -f message-board.yml && kubectl get pods -n li
kubectl describe pod <POSTGRES_POD_NAME> -n li | less
```
Post a few messages through the web UI, then test persistence — delete and reapply Postgres, messages must stay:
```bash
kubectl delete deployment postgres-deployment -n li && kubectl apply -f message-board.yml
```

### Exercise Deliverables

- `data-volume.yml`, updated `message-board.yml`.
- Screenshot of `kubectl describe pod <postgres>` showing `Mounts:` and `Volumes:` sections — save as `11_5_mounts.png`.

### Checklist (fill in before proceeding)

- [ ] `kubectl get pv,pvc` shows `Bound`.
- [ ] Messages survive a Postgres redeploy.

---

## Bonus — Liveness & Readiness Probes

### Step-by-step

Add to the Flask container in `message-board.yml`:
```yaml
livenessProbe:
  httpGet:
    path: /
    port: 5000
  initialDelaySeconds: 0
readinessProbe:
  httpGet:
    path: /?msg=Hello
    port: 5000
  initialDelaySeconds: 0
```
Apply, watch, then raise `initialDelaySeconds` to 8 and compare:
```bash
kubectl apply -f message-board.yml && kubectl describe pod <flask_pod_name> -n li
```

### Exercise Deliverables

- Screenshot of the Events section from `kubectl describe pod` — save as `11_bonus_events.png`.
- Short note explaining the behaviour change after raising `initialDelaySeconds` — save as `11_bonus_note.md`. Follow the written-response style in [`common/README.md`](../common/README.md) (B1-level English, short sentences, mostly third person passive voice).

### Checklist (fill in before proceeding)

- [ ] Liveness + readiness probes applied and pod restarts observed.
- [ ] `11_bonus_events.png` saved.
- [ ] `11_bonus_note.md` saved.

---

## Tear-down

1. On the OpenStack dashboard, terminate both VMs. Detach and delete any leftover volumes.

### Final Practical Checklist

**Screenshots archived:**

- [ ] `11_1_nodes.png` — two `Ready` nodes (Ex 11.1)
- [ ] `11_2_namespaces.png` — `kubectl get namespaces` (Ex 11.2)
- [ ] `11_2_describe.png` — `kubectl describe pod first -n li` (Ex 11.2)
- [ ] `11_3_deployments.png` — `kubectl get deployments -n li` (Ex 11.3)
- [ ] `11_4_webpage.png` — Message Board via NodePort (Ex 11.4)
- [ ] `11_4_scale.png` — pods after scaling (Ex 11.4)
- [ ] `11_5_mounts.png` — Postgres pod with PV mount (Ex 11.5)
- [ ] `11_bonus_events.png` — probe events (Bonus)

> The course spec asks for 6 screenshots — pick the 6 most legible above. The bonus screenshot is additional.

**YAML & notes committed:**

- [ ] `first_pod.yml`, `replicaset.yml`, `first_deployment.yml`, `message-board.yml`, `data-volume.yml`.
- [ ] `11_bonus_note.md` — probe behaviour write-up.

**Cleanup & submission:**

- [ ] Both VMs deleted in OpenStack.
- [ ] All deliverables uploaded to the course submission system.
