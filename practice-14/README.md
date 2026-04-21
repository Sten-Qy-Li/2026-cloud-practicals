# Practice 14

Practical webpage: https://courses.cs.ut.ee/2026/cloud/spring/Main/Practice14

> Theme: Migrate the messageboard away from Azure — replace Cosmos DB with **MongoDB** and Azure Blob Storage with **MinIO**, deployed as Docker containers on an OpenStack VM.

> Reminder: this practical asks you to **keep the VM and services running after completion** — do not delete until the course grader has reviewed.

> Student info needed: pick a **MongoDB root password** and **MinIO access key / secret** now. Let me know if you want me to suggest values.

---

## Exercise 14.1 — Fresh OpenStack VM with Docker

### Step-by-step

1. On OpenStack, launch a new VM:
   - Image: Ubuntu **24.04** · Flavor: `g4.r4c2` · Volume: **20 GB** with "Delete Volume on Instance Delete" enabled.
   - Security group: create / reuse `lab14` opening 22, 5000 (Flask), 9000 (MinIO), 9001 (MinIO Console), 27017 (MongoDB, optional external access).
2. Note the new public IP — call it `<LAB14_IP>` below.
3. SSH in:
   ```powershell
   ssh -i "C:\Users\qunyan\Desktop\Qun Yan Li.pem" ubuntu@<LAB14_IP>
   ```
4. Fix Docker networking (per Practice 2) and install Docker in one chained command:
   ```bash
   sudo apt-get update && sudo apt-get install -y ca-certificates curl gnupg && sudo install -m 0755 -d /etc/apt/keyrings && curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg && sudo chmod a+r /etc/apt/keyrings/docker.gpg && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list && sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin && sudo usermod -aG docker $USER && sudo docker run --rm hello-world
   ```
5. Log out and back in so group changes take effect (`exit`, then the SSH command again).

### Exercise Deliverables

- None — setup only.

### Checklist (fill in before proceeding)

- [ ] `docker run hello-world` succeeds without `sudo`.
- [ ] Security group allows all four ports.

---

## Exercise 14.2 — MongoDB Integration

### Step-by-step

1. Launch MongoDB in Docker. Single line for easy re-run:
   ```bash
   docker run -d --name mongodb -v ${HOME}/mongo-data:/data/db -p 27017:27017 -e MONGO_INITDB_ROOT_USERNAME=sample-db-user -e MONGO_INITDB_ROOT_PASSWORD=sample-password mongo:latest && docker ps | grep mongodb
   ```
2. Pull the Lab 5 messageboard source into `~/lab14/app`, then edit the Python file:
   - Remove every Cosmos DB import / client / helper.
   - Add PyMongo to `requirements.txt`: `pymongo`.
   - Replace the client init block:
     ```python
     from pymongo import MongoClient
     import os

     mongo_client = MongoClient(
         os.getenv('MONGO_HOST', '<LAB14_IP>'),
         username=os.getenv('MONGO_INITDB_ROOT_USERNAME'),
         password=os.getenv('MONGO_INITDB_ROOT_PASSWORD'),
         port=27017)
     db = mongo_client.flask_db
     messages = db.messages
     ```
   - Rename `insert_cosmos(...)` → `insert_mongo(...)`. Inside, use:
     ```python
     messages.insert_one(new_message)
     ```
   - Rewrite `read_cosmos()` → `read_mongo()` returning the latest 10:
     ```python
     def read_mongo():
         return list(messages.find().sort('_id', -1).limit(10))
     ```
   - Update every call site accordingly.
3. Install deps and run the Flask app on the VM:
   ```bash
   cd ~/lab14/app && python3 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt && export MONGO_INITDB_ROOT_USERNAME=sample-db-user MONGO_INITDB_ROOT_PASSWORD=sample-password && python main.py
   ```
4. Post a message via the browser at `http://<LAB14_IP>:5000/`.
5. Verify in the Mongo shell — exec into the container, query:
   ```bash
   docker exec -it mongodb bash -c "mongosh -u sample-db-user -p sample-password --authenticationDatabase admin --eval 'use flask_db; db.messages.find().pretty(); db.messages.find({username:\"turbot\"}).pretty()'"
   ```

### Exercise Deliverables

- `14_2_mongo_shell.png` — terminal showing the queries above and their results.
- Modified messageboard Python file.

### Checklist (fill in before proceeding)

- [ ] Message posted via browser appears in the mongosh output.
- [ ] Screenshot captured.

---

## Exercise 14.3 — MinIO Blob Storage

### Step-by-step

1. Launch MinIO:
   ```bash
   docker run -d --name minio -p 9000:9000 -p 9001:9001 -e MINIO_ROOT_USER=adminuser -e MINIO_ROOT_PASSWORD=StrongPasswd123 -v ${HOME}/minio/data:/data quay.io/minio/minio server /data --console-address ":9001" && docker ps | grep minio
   ```
2. Open the console at `http://<LAB14_IP>:9001`. Log in with `adminuser` / `StrongPasswd123`.
3. Create bucket `images` (Administrator → Buckets → Create Bucket).
4. Create an access key: **Administrator → Access Keys → Create access key**. Copy both halves — call them `MINIO_ACCESS_KEY` and `MINIO_SECRET_KEY`.
5. Set anonymous download policy from the VM:
   ```bash
   docker exec -it minio mc alias set minio http://<LAB14_IP>:9000 <MINIO_ACCESS_KEY> <MINIO_SECRET_KEY> && docker exec -it minio mc anonymous set download minio/images
   ```
6. Update the messageboard:
   - `requirements.txt` — add `minio`.
   - Remove any Azure Storage imports. Add:
     ```python
     from minio import Minio
     import os

     minio_client = Minio(
         f"{os.getenv('MINIO_HOST', '<LAB14_IP>')}:9000",
         access_key=os.getenv('MINIO_ACCESS_KEY'),
         secret_key=os.getenv('MINIO_SECRET_KEY'),
         secure=False)
     ```
   - Rewrite `insert_blob(img_path)`:
     ```python
     def insert_blob(img_path):
         filename = os.path.basename(img_path)
         minio_client.fput_object('images', filename, img_path)
         return f"http://{os.getenv('MINIO_HOST', '<LAB14_IP>')}:9000/images/{filename}"
     ```
   - Make sure the returned URL is stored on the message document passed to `insert_mongo()`.
7. Restart the app with all env vars:
   ```bash
   export MINIO_ACCESS_KEY=<MINIO_ACCESS_KEY> MINIO_SECRET_KEY=<MINIO_SECRET_KEY> MINIO_HOST=<LAB14_IP> && python main.py
   ```
8. Post a message with an image. Open the MinIO console → `images` bucket and screenshot the file listing; the address bar must include `<LAB14_IP>`.

### Exercise Deliverables

- `14_3_minio_bucket.png` — MinIO web UI with `images` bucket contents + VM IP visible.
- Updated Python file + `requirements.txt`.

### Checklist (fill in before proceeding)

- [ ] Image appears in the `images` bucket.
- [ ] Public URL from `insert_blob` loads the image in an incognito browser.

---

## Exercise 14.4 — Dockerise the App + Restart Policy

### Step-by-step

1. Make MongoDB and MinIO restart on reboot:
   ```bash
   docker update --restart=always mongodb && docker update --restart=always minio
   ```
2. In `~/lab14/app`, create a `Dockerfile`:
   ```dockerfile
   FROM python:3.11-slim
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY . .
   RUN mkdir -p ./images
   EXPOSE 5000
   CMD ["python", "main.py"]
   ```
3. Build + run with `--restart=always`. Substitute the same env vars:
   ```bash
   docker build -t lab14flaskapp . && docker run -d --name lab14flask --restart=always -p 5000:5000 -e MONGO_INITDB_ROOT_USERNAME=sample-db-user -e MONGO_INITDB_ROOT_PASSWORD=sample-password -e MONGO_HOST=<LAB14_IP> -e MINIO_ACCESS_KEY=<MINIO_ACCESS_KEY> -e MINIO_SECRET_KEY=<MINIO_SECRET_KEY> -e MINIO_HOST=<LAB14_IP> lab14flaskapp && docker ps
   ```
4. Verify `http://<LAB14_IP>:5000/` renders, and screenshot the page — the URL bar (with the VM IP) must be in frame. Save as `14_4_webpage.png`.
5. Post a message with an image. Right-click → Inspect → copy the `<img src=...>` URL into the address bar; screenshot with that image + URL visible. Save as `14_4_image_url.png`.
6. Reboot the VM from OpenStack, wait ~1 min, reload the site, confirm it comes back up without manual intervention.

### Exercise Deliverables

- `14_4_webpage.png`, `14_4_image_url.png`.
- `Dockerfile`.

### Checklist (fill in before proceeding)

- [ ] All three containers auto-restart (`docker ps` shows them after reboot).
- [ ] Page loads after reboot with no manual steps.

---

## Bonus — Docker Compose Deployment

### Step-by-step

1. In `~/lab14`, create `.env` (kept out of the repo via `.gitignore`):
   ```
   MONGO_INITDB_ROOT_USERNAME=sample-db-user
   MONGO_INITDB_ROOT_PASSWORD=sample-password
   MINIO_ROOT_USER=adminuser
   MINIO_ROOT_PASSWORD=StrongPasswd123
   MINIO_ACCESS_KEY=<MINIO_ACCESS_KEY>
   MINIO_SECRET_KEY=<MINIO_SECRET_KEY>
   ```
2. `docker-compose.yaml`:
   ```yaml
   services:
     mongodb:
       image: mongo:latest
       restart: always
       environment:
         MONGO_INITDB_ROOT_USERNAME: ${MONGO_INITDB_ROOT_USERNAME}
         MONGO_INITDB_ROOT_PASSWORD: ${MONGO_INITDB_ROOT_PASSWORD}
       volumes:
         - ./mongo-data:/data/db

     minio:
       image: quay.io/minio/minio
       restart: always
       command: server /data --console-address ":9001"
       environment:
         MINIO_ROOT_USER: ${MINIO_ROOT_USER}
         MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
       volumes:
         - ./minio-data:/data
       ports:
         - "9000:9000"
         - "9001:9001"

     flaskapp:
       build: ./app
       image: lab14flaskapp
       restart: always
       depends_on:
         - mongodb
         - minio
       environment:
         MONGO_INITDB_ROOT_USERNAME: ${MONGO_INITDB_ROOT_USERNAME}
         MONGO_INITDB_ROOT_PASSWORD: ${MONGO_INITDB_ROOT_PASSWORD}
         MONGO_HOST: mongodb
         MINIO_ACCESS_KEY: ${MINIO_ACCESS_KEY}
         MINIO_SECRET_KEY: ${MINIO_SECRET_KEY}
         MINIO_HOST: minio
       ports:
         - "5000:5000"
   ```
3. Update the Python code so the bucket is created + made public on startup (if not already):
   ```python
   import json
   bucket_name = 'images'
   if not minio_client.bucket_exists(bucket_name):
       minio_client.make_bucket(bucket_name)
       policy = {
         "Version": "2012-10-17",
         "Statement": [
           {"Effect": "Allow", "Principal": {"AWS": ["*"]},
            "Action": ["s3:ListBucketMultipartUploads","s3:GetBucketLocation","s3:ListBucket"],
            "Resource": ["arn:aws:s3:::images"]},
           {"Effect": "Allow", "Principal": {"AWS": ["*"]},
            "Action": ["s3:PutObject","s3:AbortMultipartUpload","s3:DeleteObject","s3:GetObject","s3:ListMultipartUploadParts"],
            "Resource": ["arn:aws:s3:::images/*"]}
         ]}
       minio_client.set_bucket_policy(bucket_name, json.dumps(policy))
   ```
4. Because services talk over the Compose network, change `insert_blob`'s returned URL to use the public IP for browsers, not `minio`:
   ```python
   return f"http://<LAB14_IP>:9000/images/{filename}"
   ```
5. Bring the stack up from an empty state to prove reproducibility:
   ```bash
   docker compose down -v && rm -rf mongo-data minio-data && docker compose up -d --build && docker compose ps
   ```

### Bonus Deliverables

- `docker-compose.yaml`, `.env.example` (scrub secrets), proof screenshots.

---

## Tear-down

Do **not** terminate the VM — the practical requires the app to remain reachable for grading. When grading is done:
```bash
docker compose down -v
```
then delete the VM from OpenStack.

### Final Practical Checklist

- [ ] 4 screenshots: `14_2_mongo_shell.png`, `14_3_minio_bucket.png`, `14_4_webpage.png`, `14_4_image_url.png`.
- [ ] Submit app URL `http://<LAB14_IP>:5000/`.
- [ ] Source code: Python files, `Dockerfile`, `requirements.txt`, optional `docker-compose.yaml` — no `.venv`, no `.env`.
- [ ] VM + services still running for grader access.
- [ ] Deliverables uploaded.
