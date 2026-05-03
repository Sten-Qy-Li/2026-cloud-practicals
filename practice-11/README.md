# Practice 11

Practical webpage: https://courses.cs.ut.ee/2026/cloud/spring/Main/Practice11

> Theme: Two-node K3s (Kubernetes) cluster on OpenStack — pods, replicasets, deployments, services, secrets, persistent volumes.

> VM topology (per course spec): the **Master** is your **existing Lab 1 instance** at `172.17.65.54`. The **Worker** is a **new VM** named `Lab11_Li_Worker`. Do **not** create a second Master — the practical reuses your Lab 1 VM as the K3s server.

> SSH into either VM with the same key: `ssh -i "C:\Users\qunyan\Desktop\Qun Yan Li.pem" ubuntu@<VM_IP>`.

> Reminder: If you need Docker Hub pulls and hit rate limits, swap the image prefix to `registry.hpc.ut.ee/mirror/<image>` (HPC mirror docs: https://docs.hpc.ut.ee/public/services/registry.hpc.ut.ee/). Avoid `docker login` — that lowers your daily quota.

> Nagios checks tracked across this practical: **Lab11-00, Lab11-01** (cluster up), **Lab11-02** (namespace), **Lab11-03** (pod), **Lab11-04** (replicaset), **Lab11-05** (first deployment), **Lab11-06** (secret), **Lab11-07, Lab11-08, Lab11-09** (services + scaling). All ten must turn green before submission.

---

## Exercise 11.1 — Build the Two-Node K3s Cluster

### Step-by-step

1. Prepare the OpenStack security group **before** modifying any VM:
   - OpenStack dashboard → **Project → Network → Security Groups** → check if `K8S cluster` already exists; otherwise **+ Create Security Group** → Name: `K8S cluster` → **Create**.
   - Open `K8S cluster` → **Manage Rules → + Add Rule** and add the three rules:
     - Rule: **SSH** (or Custom TCP Rule, port 22) · Remote: **CIDR** `0.0.0.0/0` → **Add**.
     - Rule: **Custom TCP Rule** · Port Range: `6443` · Remote: CIDR `0.0.0.0/0` → **Add** (K3s API server).
     - Rule: **Custom TCP Rule** · Port Range: `30000 – 32767` · Remote: CIDR `0.0.0.0/0` → **Add** (NodePort services).

2. Attach the `K8S cluster` security group to your **existing Lab 1 instance** (the Master):
   - **Project → Compute → Instances** → find your Lab 1 VM (the one at `172.17.65.54`) → **Actions ▾ → Edit Security Groups**.
   - Move `K8S cluster` from the left list to the right list (the **+** button) → keep the existing groups too → **Save**.

3. Launch **one** new VM for the Worker:
   - **Project → Compute → Instances → Launch Instance**.
   - **Details tab:** Instance Name: `Lab11_Li_Worker` · Count: `1`.
   - **Source tab:** Select Boot Source: **Image** · Create New Volume: **Yes** · Volume Size: `30 GB` · Delete Volume on Instance Delete: **Yes** · Image: **Ubuntu 24.04 LTS** (click the up-arrow next to it).
   - **Flavor tab:** select `g4.r4c2` (up-arrow).
   - **Networks tab:** `shared` or the default student network (up-arrow).
   - **Security Groups tab:** deselect `default` · select `K8S cluster` (up-arrow).
   - **Key Pair tab:** select your existing key (the one matching `C:\Users\qunyan\Desktop\Qun Yan Li.pem`).
   - **Launch Instance**.

4. Wait until the Worker shows **Status: Active** and **Power State: Running**, then copy its IP from the **IP Address** column. The Master IP stays `172.17.65.54`.

5. SSH into the **Master** (Lab 1 VM) from Windows PowerShell:
   ```powershell
   ssh -i "C:\Users\qunyan\Desktop\Qun Yan Li.pem" ubuntu@172.17.65.54
   ```

6. On Master, install the K3s server, relax the kubeconfig permissions, and read the node token in one chained command:
   ```bash
   curl -sfL https://get.k3s.io | sh - && sudo chmod 644 /etc/rancher/k3s/k3s.yaml && sudo cat /var/lib/rancher/k3s/server/node-token
   ```
   Copy the full token (long hex · `::` · `server:` · more hex) — you need it in step 8.

7. Open a **second** PowerShell window and SSH into the **Worker**:
   ```powershell
   ssh -i "C:\Users\qunyan\Desktop\Qun Yan Li.pem" ubuntu@<WORKER_IP>
   ```

8. On Worker, join the cluster (single line, substitute `172.17.65.54` for `<MASTER_IP>` and paste `<NODE_TOKEN>` from step 6):
   ```bash
   curl -sfL https://get.k3s.io | K3S_URL=https://172.17.65.54:6443 K3S_TOKEN=<NODE_TOKEN> sh -
   ```

9. Back on the **Master** SSH window, verify both nodes are `Ready`:
   ```bash
   sudo kubectl get nodes && sudo kubectl get nodes -o wide
   ```

10. (Recommended — avoid `sudo` on every `kubectl` call.) On Master:
    ```bash
    echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc && source ~/.bashrc
    ```

### Exercise Deliverables

- Screenshot of `kubectl get nodes -o wide` showing both nodes `Ready` — save as `11_1_nodes.png`.
- Nagios checks **Lab11-00** and **Lab11-01** are green.

### Checklist (fill in before proceeding)

- [ ] `K8S cluster` security group created with SSH (22), 6443, and 30000–32767 rules.
- [ ] Lab 1 instance has `K8S cluster` attached.
- [ ] `Lab11_Li_Worker` VM is `Active` with the same security group.
- [ ] Master shows Worker as `Ready` in `kubectl get nodes`.
- [ ] Lab11-00, Lab11-01 green.

---

## Exercise 11.2 — Pods, Namespaces, ReplicaSets

### 11.2.1 — Namespaces

On **Master**, list existing namespaces, create your own (the pseudonym `turbot` has no special characters so it is a valid namespace name — the Nagios checks look for resources inside this namespace, so it must match your registered course pseudonym exactly), then list again:
```bash
kubectl get namespaces && kubectl create namespace turbot && kubectl get namespaces
```

### 11.2.2 — Pod via CLI and YAML

Imperative busybox pod (default namespace), then read its log:
```bash
kubectl run busybox --image=busybox --restart=Never -- /bin/sh -c "echo Hello Kubernetes!" && kubectl get po && kubectl logs busybox
```

Create `first_pod.yml` on Master (container name and command match the course spec exactly):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: first
  namespace: turbot
  labels:
    app: myfirst
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 180']
```
Deploy, list (with `-o wide` to see node placement), inspect logs, describe, and exec — chained:
```bash
kubectl create -f first_pod.yml && kubectl get pods -n turbot -o wide && kubectl logs first -n turbot && kubectl describe pod first -n turbot
```
Shell into it and run the course-mandated `cat /etc/resolv.conf` to inspect how pods resolve names against the cluster DNS, the VM, and the university network (exit with `exit`):
```bash
kubectl exec -it first -n turbot -- /bin/sh
# inside the pod:
cat /etc/resolv.conf
exit
```

> The pod's command sleeps for 180 seconds and then exits — Kubernetes will restart it, and `kubectl exec` may "kick" you out mid-session. That is expected; just re-exec.

After Nagios **Lab11-03** is green, the course wants you to clean the pod up:
```bash
kubectl delete -f first_pod.yml
```

### 11.2.3 — ReplicaSet

Create `replicaset.yml`:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web
  namespace: turbot
  labels:
    env: dev
    role: web
spec:
  replicas: 4
  selector:
    matchLabels:
      role: web
  template:
    metadata:
      labels:
        role: web
    spec:
      containers:
      - name: nginx
        image: nginx
```
Apply and inspect — note the course uses `-n your_pseudonym` on the apply, which is equivalent to baking `namespace: turbot` into the YAML as we did:
```bash
kubectl apply -f replicaset.yml && kubectl get rs -n turbot && kubectl get pods -n turbot
```

**Course order — do the *isolation* edit first, then scale.** Pick any running pod from `kubectl get pods -n turbot` (e.g. `web-abcde`) and change its `role` label from `web` to `isolated`:
```bash
kubectl edit pod <web-pod-name> -n turbot
```
In the editor, find `role: web` under `metadata.labels` and replace it with `role: isolated`, then save and quit (`:wq` in `vi` — see https://www.atmos.albany.edu/daes/atmclasses/atm350/vi_cheat_sheet.pdf if you are stuck). The ReplicaSet immediately spawns a replacement pod (it now matches one fewer pod against its selector). Confirm:
```bash
kubectl get pods -n turbot -L role
```
The pod with `role=isolated` is now an orphan — still running, but no longer managed by `rs/web`.

Now run the scale experiments:
```bash
kubectl scale --replicas=10 rs/web -n turbot && kubectl get rs -n turbot && kubectl get pods -n turbot
kubectl scale --replicas=1  rs/web -n turbot && kubectl get rs -n turbot && kubectl get pods -n turbot
```

### Exercise Deliverables

- `first_pod.yml`, `replicaset.yml` committed to the repo.
- Screenshot of `kubectl get namespaces` showing `li` — save as `11_2_namespaces.png`.
- Screenshot of `kubectl describe pod first -n turbot` — save as `11_2_describe.png`.
- Nagios checks **Lab11-02**, **Lab11-03**, **Lab11-04** are green.

### Checklist (fill in before proceeding)

- [ ] Namespace `li` exists.
- [ ] Pod `first` runs and logs show the hello message.
- [ ] ReplicaSet scaled from 4 → 10 → 1 successfully.
- [ ] One pod re-labelled to `role: isolated`; ReplicaSet replaced it.
- [ ] Both YAML files saved.
- [ ] Lab11-02, Lab11-03, Lab11-04 green.

---

## Exercise 11.3 — Deployments, Multi-Service, Secrets

### 11.3.1 — First Flask Deployment + Rolling Update

Create `first_deployment.yml` (swap `<DOCKERHUB_ID>` for your own, e.g. `qunyanli`). The course uses the **data.json**-based Practice 3 image `lab3flask1.0` for this sub-task — the Postgres-backed image comes in 11.3.2:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-deployment
  namespace: turbot
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
Apply, list, describe, and grab the cluster-internal pod IP (`10.42.x.x`) from `kubectl get po -o wide`:
```bash
kubectl apply -f first_deployment.yml && kubectl get deployments -n turbot && kubectl describe deploy flask-deployment -n turbot && kubectl get po -n turbot -o wide
```
From the **Master** node, curl the pod IP directly (k3s pod CIDR is reachable from the host) — this is the course's check that the Flask app is alive inside the pod:
```bash
curl "http://<POD_IP>:5000?msg=hi"
```
You should get an HTML response. (If you prefer not to leave the cluster, you can also `kubectl exec` into another pod and `wget -qO-` the same URL.)

Trigger a rolling update with a different image, watch it complete, confirm via `describe`, then — once Nagios **Lab11-05** turns green — delete the deployment as the course requires:
```bash
kubectl set image deployments/flask-deployment flask=shivupoojar/message-board.v2 -n turbot && kubectl rollout status deployment/flask-deployment -n turbot && kubectl describe pods -n turbot
kubectl delete -f first_deployment.yml
```

### 11.3.2 — Drafting `message-board.yml` (Flask + Postgres)

Create `message-board.yml` with **two** deployments separated by `---`. The Flask image is now the **PostgreSQL-based** `flask1.0` image you built in **Practice 2 Task 2.4** (not `lab3flask1.0`, which uses `data.json`). Postgres uses the official `postgres:14.1-alpine`. The env blocks below already reference the secret you will create in 11.3.3 — that is the form the course wants by the end of 11.3.4:

Replace `<DOCKERHUB_ID>` (and `<YOUR_PASSWORD>` further down):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-deployment
  namespace: turbot
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
        image: <DOCKERHUB_ID>/flask1.0
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
  namespace: turbot
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

### 11.3.3 — Create the Postgres Secret (kubectl, not YAML)

The course is explicit that secrets must be created via `kubectl`, not embedded in `message-board.yml`. Pick any password you like (the value never leaves your cluster):
```bash
kubectl create secret generic postgres-secret-config -n turbot --from-literal=username=postgres --from-literal=password=<YOUR_PASSWORD>
kubectl get secret postgres-secret-config -n turbot
kubectl get secret postgres-secret-config -n turbot -o yaml
```
The `-o yaml` form should show `username` and `password` base64-encoded under `data:`. **Lab11-06** turns green at this point.

### 11.3.4 — Wiring the Deployments to the Secret

Already done in the YAML above — the `env:` blocks under both Flask and Postgres pull `POSTGRES_USER` / `POSTGRES_PASSWORD` from `secretKeyRef: postgres-secret-config`, and Flask additionally constructs `DATABASE_URL` using those values plus the (still-to-be-created) `postgresql` Service DNS name. No further file edits are needed for 11.3.4.

### 11.3.5 — Apply, Screenshot, then Delete

Apply the deployments and verify both pods come up. Flask will log `could not connect to postgresql` until services are added in 11.4 — that is **expected** because the Postgres Service does not yet exist:
```bash
kubectl apply -f message-board.yml && kubectl get deployments -n turbot && kubectl get pods -n turbot
kubectl logs $(kubectl get po -n turbot -l app=flask -o jsonpath='{.items[0].metadata.name}') -n turbot
kubectl logs $(kubectl get po -n turbot -l app=postgres -o jsonpath='{.items[0].metadata.name}') -n turbot
```

**Take the deliverable screenshot now** — `kubectl get deployments -n turbot` showing both `flask-deployment` and `postgres-deployment` — save as `11_3_deployments.png`.

The course then asks you to delete the deployments before continuing (so 11.4's apply re-creates them alongside the new Services as one atomic step):
```bash
kubectl delete -f message-board.yml
```

### Exercise Deliverables

- `first_deployment.yml`, `message-board.yml` committed.
- Screenshot of `kubectl get deployments -n turbot` — save as `11_3_deployments.png`.
- Nagios checks **Lab11-05**, **Lab11-06** are green.

### Checklist (fill in before proceeding)

- [ ] Secret `postgres-secret-config` created and visible in YAML (base64).
- [ ] Both deployments show desired replicas after apply.
- [ ] Transient `CrashLoopBackOff` / `could not connect to postgresql` on Flask noted (no Service yet — fixed in 11.4).
- [ ] `11_3_deployments.png` saved.
- [ ] `message-board.yml` deleted (will be re-applied with services in 11.4).
- [ ] Lab11-05, Lab11-06 green.

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
  namespace: turbot
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
  namespace: turbot
  labels:
    name: postgresql
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
```
Apply, list, grab the NodePort, then scale Flask:
```bash
kubectl apply -f message-board.yml && kubectl get pods -o wide -n turbot && kubectl get svc -n turbot
kubectl scale deployment flask-deployment --replicas=2 -n turbot && kubectl get pods -n turbot && kubectl get deployment -n turbot
```
The NodePort is the second number under `PORT(S)` for `service-flask` (e.g. `5000:31234/TCP` → NodePort is `31234`, always in `30000–32767`). From your laptop, browse:

```
http://172.17.65.54:<NODEPORT>
```

Post a couple of test messages through the form. If the page times out, double-check that the `K8S cluster` security group covers the NodePort range on **both** VMs.

> If Flask shows a `relation "messages" does not exist` error after a Postgres redeploy, scale the Flask deployment to `0` then back to `2` so it re-runs its table-creation logic: `kubectl scale deployment flask-deployment --replicas=0 -n turbot && kubectl scale deployment flask-deployment --replicas=2 -n turbot`.

### Exercise Deliverables

- Screenshot of the Message Board webpage reached via NodePort, showing the URL bar with the `172.17.65.54:<NODEPORT>` address visible — save as `11_4_webpage.png`.
- Screenshot of `kubectl get pods -n turbot` after scaling Flask to 2 replicas — save as `11_4_scale.png`.
- Nagios checks **Lab11-07**, **Lab11-08**, **Lab11-09** are green.

### Checklist (fill in before proceeding)

- [ ] Messages can be posted via the browser.
- [ ] Scaled Flask deployment shows 2 running replicas.
- [ ] Lab11-07, Lab11-08, Lab11-09 green.

---

## Exercise 11.5 — Persistent Volume + Claim

### Step-by-step

Create `data-volume.yml`:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv-volume
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
  namespace: turbot
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
  storageClassName: manual
```
Modify the **Postgres container** in `message-board.yml` — add the `PGDATA` env, the `volumeMounts`, and the pod-level `volumes` block (keep `POSTGRES_USER` and `POSTGRES_PASSWORD` from before):
```yaml
        env:
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
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
        volumeMounts:
        - name: data-storage-volume
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: data-storage-volume
        persistentVolumeClaim:
          claimName: postgres-db-claim
```
Apply both files, verify the PV/PVC bind, and check the Postgres pod's mounts:
```bash
kubectl apply -f data-volume.yml && kubectl get pv,pvc -n turbot
kubectl apply -f message-board.yml && kubectl get pods -n turbot
kubectl describe pod $(kubectl get po -n turbot -l app=postgres -o jsonpath='{.items[0].metadata.name}') -n turbot
```
Open the Message Board, post a few messages, then **delete and recreate** the Postgres deployment — the messages must survive:
```bash
kubectl delete deployment postgres-deployment -n turbot && kubectl apply -f message-board.yml && kubectl get pods -n turbot
```
If Flask shows the empty/no-table error after the redeploy, scale Flask to `0` then back up so it re-creates the table on the persistent data:
```bash
kubectl scale deployment flask-deployment --replicas=0 -n turbot && kubectl scale deployment flask-deployment --replicas=2 -n turbot
```

> Use the Postgres-backed Message Board image, not the `data.json` variant — the `data.json` version writes to a file inside the container and will not benefit from the PV.

### Exercise Deliverables

- `data-volume.yml`, updated `message-board.yml`.
- Screenshot of `kubectl describe pod <postgres-pod>` clearly showing the `Mounts:` and `Volumes:` sections — save as `11_5_mounts.png`.

### Checklist (fill in before proceeding)

- [ ] `kubectl get pv,pvc -n turbot` shows both `Bound`.
- [ ] Posted messages survive `kubectl delete deployment postgres-deployment -n turbot` + reapply.

---

## Bonus — Liveness & Readiness Probes

Reference walkthrough: https://github.com/sebinxavi/kubernetes-readiness

### Step-by-step

Add to the **Flask container** in `message-board.yml`:
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
Apply, then describe a Flask pod and look at the `Events:` section — you should see `Unhealthy` / `Liveness probe failed` lines while the container is still warming up:
```bash
kubectl apply -f message-board.yml && kubectl describe pod $(kubectl get po -n turbot -l app=flask -o jsonpath='{.items[0].metadata.name}') -n turbot
```

Now raise both `initialDelaySeconds` from `0` to `8`, re-apply, and describe again. The probe failures should disappear because the Flask process has time to bind port 5000 before Kubernetes starts probing.

### Exercise Deliverables

- Screenshot of the `Events` section from `kubectl describe pod <flask-pod>` — save as `11_bonus_events.png`.
- Short note `11_bonus_note.md` answering the course question: **Why did errors occur initially with lower delay values, and what changed when `initialDelaySeconds` was raised to 8?** Follow the written-response style in [`common/README.md`](../common/README.md) (B1-level English, short sentences, mostly third-person passive voice, cloud-computing terminology).

### Checklist (fill in before proceeding)

- [ ] Liveness + readiness probes applied; pod restarts/probe failures observed at `initialDelaySeconds: 0`.
- [ ] Probe failures gone at `initialDelaySeconds: 8`.
- [ ] `11_bonus_events.png` saved.
- [ ] `11_bonus_note.md` saved.

---

## Tear-down

1. On Master, optionally clear the namespace and PV: `kubectl delete namespace turbot && kubectl delete pv postgres-pv-volume`.
2. On the OpenStack dashboard, terminate **only** the `Lab11_Li_Worker` VM (and detach/delete its volume). **Do not delete the Lab 1 instance** — other practicals still need it.
3. (Optional) Uninstall K3s on the Master so the Lab 1 VM goes back to its original state: `/usr/local/bin/k3s-uninstall.sh`.

### Final Practical Checklist

**All Nagios checks green:**

- [ ] Lab11-00, Lab11-01 — cluster up (Ex 11.1)
- [ ] Lab11-02 — namespace `turbot` (Ex 11.2.1)
- [ ] Lab11-03 — pod `first` (Ex 11.2.2)
- [ ] Lab11-04 — replicaset `web` (Ex 11.2.3)
- [ ] Lab11-05 — flask deployment (Ex 11.3.1)
- [ ] Lab11-06 — secret `postgres-secret-config` (Ex 11.3.3)
- [ ] Lab11-07, Lab11-08, Lab11-09 — services + scaling (Ex 11.4)

**Mandatory screenshots (the four the course asks for):**

- [ ] `11_3_deployments.png` — `kubectl get deployments -n turbot` (Ex 11.3)
- [ ] `11_4_webpage.png` — Message Board via NodePort with the IP visible (Ex 11.4)
- [ ] `11_5_mounts.png` — Postgres pod `Mounts:` / `Volumes:` (Ex 11.5)
- [ ] `11_bonus_events.png` — probe events (Bonus)

**Extra screenshots archived for safety (recommended):**

- [ ] `11_1_nodes.png` — two `Ready` nodes (Ex 11.1)
- [ ] `11_2_namespaces.png` — `kubectl get namespaces` (Ex 11.2)
- [ ] `11_2_describe.png` — `kubectl describe pod first -n turbot` (Ex 11.2)
- [ ] `11_4_scale.png` — pods after scaling Flask to 2 (Ex 11.4)

**YAML & notes committed (the five files the course asks for):**

- [ ] `first_pod.yml`
- [ ] `replicaset.yml`
- [ ] `first_deployment.yml`
- [ ] `message-board.yml` (deployments + secret refs + services + PV mounts + probes)
- [ ] `data-volume.yml`
- [ ] `11_bonus_note.md` — probe behaviour write-up.

**Cleanup & submission:**

- [ ] `Lab11_Li_Worker` VM (and its volume) deleted in OpenStack.
- [ ] Lab 1 instance still alive at `172.17.65.54` (do not delete).
- [ ] All deliverables uploaded to the course submission system while logged in and registered for the course.
