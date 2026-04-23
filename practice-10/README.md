# Practice 10

Practical webpage: https://courses.cs.ut.ee/2026/cloud/spring/Main/Practice10

> Theme: Azure monitoring (App Service, Blob Storage, Cosmos DB) using Application Insights + Log Analytics + KQL.

> Reminder — Application Insights ingestion can take 5–30 minutes. **Archive every screenshot with a timestamp** (file mtime + filename pattern `10_x.png` / `Qx_y.png`). This counts as timing evidence for the deliverables.

---

## Exercise 10.1 — Deploy the Message Board App to Azure

> Portal naming note: the "App Service" resource is created from the **Web App** tile (Portal → Create a resource → Web App, or App Services blade → **+ Create → Web App**). "Cosmos DB (Core/SQL)" is now labelled **Azure Cosmos DB for NoSQL** in the Portal.

### Step-by-step

1. Open [https://portal.azure.com](https://portal.azure.com) and sign in with the UT student account `qun.yan.li@ut.ee`. Top-right avatar → confirm the directory is **Tartu Ülikool (TartuUlikool.onmicrosoft.com)** and the subscription is the UT student subscription.

2. Create the resource group:
   - Top search bar → type `Resource groups` → **Resource groups** → **+ Create**.
   - Subscription: UT student subscription · Resource group: `lab10` · Region: `(Europe) Sweden Central`.
   - **Review + create → Create**. Wait for "Your deployment is complete".

3. Create the **Web App** (this is the "App Service" resource):
   - Top search bar → `App Services` → **+ Create → Web App**.
   - **Basics tab:**
     - Subscription: UT student subscription · Resource Group: `lab10`.
     - Name: `lilab10-<random>` (must be globally unique; the full URL will be `https://lilab10-<random>.azurewebsites.net`).
     - Publish: **Code** · Runtime stack: **Python 3.9** · Operating System: **Linux** · Region: `Sweden Central`.
     - Linux Plan: **Create new** → name it `lilab10-plan` · Pricing plan: **Free F1 (Shared infrastructure)**.
   - Leave Database / Deployment / Networking / Monitoring / Tags at defaults. **Review + create → Create**.

4. Create the **Storage Account**:
   - Top search bar → `Storage accounts` → **+ Create**.
   - Subscription: UT student · Resource group: `lab10` · Storage account name: `lilab10storage<random>` (3–24 lowercase letters/digits, globally unique) · Region: `Sweden Central` · Primary service: **Azure Blob Storage or Azure Data Lake Storage Gen 2** · Performance: **Standard** · Redundancy: **Locally-redundant storage (LRS)**.
   - **Review + create → Create**.
   - Once deployed → open the storage account → left blade **Data storage → Containers → + Container** → Name: `images` → Anonymous access level: **Private (no anonymous access)** → **Create**.

5. Create the **Cosmos DB account** (Core/SQL API):
   - Top search bar → `Azure Cosmos DB` → **+ Create** → pick the **Azure Cosmos DB for NoSQL** tile → **Create**.
   - Subscription: UT student · Resource group: `lab10` · Account name: `lilab10-cosmos-<random>` · Location: `Sweden Central` · Capacity mode: **Provisioned throughput** · Apply Free Tier Discount: **Apply** (if offered).
   - **Review + create → Create**. Deployment takes 5–10 minutes.
   - Once deployed → open the account → left blade **Data Explorer → New Container**:
     - Database id: **Create new** → `lab5messagesdb` · Share throughput across containers: **unchecked**.
     - Container id: `lab5messages` · Partition key: `/id` · Container throughput (autoscale/manual): **Manual** · RU/s: `400`.
     - **OK**.

6. Collect the four values (write them into a scratch note — they go into Web App configuration in step 7):
   - `STORAGE_ACCOUNT` — Storage account blade → **Overview** → name at the top (e.g. `lilab10storage<random>`).
   - `CONN_KEY` — Storage account blade → **Security + networking → Access keys → key1 → Show → copy the Key** (the long base64 value, not the connection string).
   - `COSMOS_URL` — Cosmos DB account blade → **Overview → URI** (form: `https://lilab10-cosmos-<random>.documents.azure.com:443/`).
   - `MasterKey` — Cosmos DB account blade → **Settings → Keys → Read-write Keys → PRIMARY KEY → copy**.

7. Add the four env vars to the Web App:
   - Web App blade (`lilab10-<random>`) → left blade **Settings → Environment variables → App settings tab → + Add**.
   - Add each name / value pair (plain names, **no** `APPSETTING_` prefix needed when setting them here — the `APPSETTING_` prefix only matters when baking them into the deployment payload):
     - `STORAGE_ACCOUNT` = `<value from step 6>`
     - `CONN_KEY` = `<value from step 6>`
     - `COSMOS_URL` = `<value from step 6>`
     - `MasterKey` = `<value from step 6>`
   - Click **Apply → Confirm** (the app will restart).

8. Deploy the Practice 5 Message Board code:
   - On your local machine, zip the Practice 5 project at the project root (flat layout — `main.py`, `requirements.txt`, `templates/`, `images/` must be at the **root** of the zip, not inside a top folder).
   - Web App blade → left blade **Development Tools → Advanced Tools → Go** → this opens the Kudu site in a new tab.
   - In Kudu → top menu **Tools → Zip Push Deploy** → drag the zip onto the page. Wait for the `Deployment successful` response.
   - Alternative (CLI): `az webapp deploy --resource-group lab10 --name lilab10-<random> --src-path ./messageboard.zip --type zip`.
   - Open `https://lilab10-<random>.azurewebsites.net` in the browser — the Message Board homepage must render.

### Exercise Deliverables

- None for 10.1 (setup only) — but keep the resource names consistent; later screenshots must show them.

### Checklist (fill in before proceeding)

- [ ] Resource group `lab10` exists.
- [ ] Web App reachable via public URL.
- [ ] Blob container `images` created.
- [ ] Cosmos DB database + container created.
- [ ] All four app settings present on the Web App.

---

## Exercise 10.2 — Enable Application Insights + Diagnostic Settings

### Step-by-step

1. Turn on Application Insights for the Web App:
   - Web App blade → left blade **Settings → Application Insights → Turn on Application Insights**.
   - Change your resource name: accept the default (e.g. `lilab10-<random>`).
   - Log Analytics workspace: **Create new** → name `lilab10-logs` · Resource group: `lab10`.
   - **Apply → Yes** (confirm restart). Wait for the "Change applied" banner.

2. Route Web App logs to the Log Analytics workspace:
   - Web App blade → left blade **Monitoring → Diagnostic settings → + Add diagnostic setting**.
   - Diagnostic setting name: `lilab10-webapp-to-logs`.
   - **Logs** section: tick **allLogs** (or tick every individual category).
   - **Metrics** section: tick **AllMetrics**.
   - **Destination details** → tick **Send to Log Analytics workspace** → Subscription: UT student · Workspace: `lilab10-logs`.
   - **Save**.

3. Repeat for the Storage Account — diagnostic settings live under the **blob** sub-resource, not the account root:
   - Storage account blade → left blade **Monitoring → Diagnostic settings** → you will see a list with four rows (blob, file, queue, table). Click **+ Add diagnostic setting** on the **blob** row.
   - Name: `lilab10-blob-to-logs` · tick **allLogs** + **AllMetrics** · destination: Log Analytics workspace `lilab10-logs` → **Save**.

4. Repeat for the Cosmos DB account:
   - Cosmos DB account blade → left blade **Monitoring → Diagnostic settings → + Add diagnostic setting**.
   - Name: `lilab10-cosmos-to-logs` · tick **allLogs** + **AllMetrics** · destination: Log Analytics workspace `lilab10-logs` → **Save**.

### Exercise Deliverables

- None — the deliverables materialise in 10.3–10.5 once data arrives.

### Checklist (fill in before proceeding)

- [ ] Application Insights is linked to the Web App.
- [ ] Diagnostic settings configured for App Service, Blob, Cosmos.
- [ ] Log Analytics workspace receives data (wait 5–30 min — you can move on and come back).

---

## Exercise 10.3 — Generate Workload & Capture Application Map

### Step-by-step

1. Open `https://lilab10-<random>.azurewebsites.net` in the browser. Post **~10 messages** with images (small JPG/PNGs, < 200 KB each).
2. Refresh the homepage 10–15 times (hold F5 / Ctrl+R) to generate GET traffic on the `/` route and `/handle_message` POSTs.
3. Verify uploads arrived: Portal → Storage account → **Data storage → Containers → images** — the uploaded files must be listed.
4. Verify writes arrived: Portal → Cosmos DB account → **Data Explorer** → expand `lab5messagesdb → lab5messages → Items` — the documents must be listed.
5. On the Web App blade → left blade **Settings → Application Insights → View Application Insights data** (this opens the AI resource) → left blade **Investigate → Application map**. Wait until at least one node appears (5–30 min ingestion lag is normal). Take a full-browser-window screenshot with the top-right user chip showing `qun.yan.li@ut.ee` (or the UT avatar) clearly visible. Save as `10_3.png`.

### Exercise Deliverables

- `10_3.png` (or `.jpg`) — Application Map screenshot with username visible.

### Checklist (fill in before proceeding)

- [ ] ~10 messages + images posted.
- [ ] `10_3.png` saved with filename matching spec.
- [ ] Username visible in screenshot.

---

## Exercise 10.4 — KQL Analysis

> **Timing reminder:** every query result screenshot is a timed snapshot; archive with local timestamp.

### 10.4.1 — App Service (`AppServiceHTTPLogs`)

Portal navigation for every query below:
1. Portal top search → `Log Analytics workspaces` → click `lilab10-logs`.
2. Left blade **General → Logs** → close the Queries dialog if it pops up.
3. Paste the KQL block into the query editor → **Run**.
4. Screenshot the **full browser window** (URL bar + query + results) with the UT user avatar visible in the top-right. Save under the filename listed next to the query.

Queries are single-line-friendly; pipes can stay on separate lines for readability in the query pane — that is the Azure UI, not a shell.

**Q1.1.3** — root endpoint invocations in the last 24 hours:
```kql
AppServiceHTTPLogs | where TimeGenerated > ago(24h) | where CsUriStem == "/" | count
```
Save as `Q1_1_3.png`.

**Q1.2** — `/handle_message` invocations in last 7 days:
```kql
AppServiceHTTPLogs | where TimeGenerated > ago(7d) | where CsUriStem == "/handle_message" | count
```
Save as `Q1_2.png`.

**Q1.4** — top 5 slowest `/handle_message` requests:
```kql
AppServiceHTTPLogs | where TimeGenerated > ago(7d) | where CsUriStem == "/handle_message" | order by TimeTaken desc | take 5
```
Save as `Q1_4.png`.

**Q1.7** — IP-address request counts:
```kql
AppServiceHTTPLogs | where TimeGenerated > ago(7d) | summarize RequestCount = count() by CIp | order by RequestCount desc
```
Save as `Q1_7.png`.

**Q1.8** — latency statistics (avg / min / max / p90 / p95 / p99):
```kql
AppServiceHTTPLogs | where CsUriStem == "/" or CsUriStem == "/handle_message" | summarize AverageResponseTime = avg(TimeTaken), MinimumResponseTime = min(TimeTaken), MaximumResponseTime = max(TimeTaken), Percentile90ResponseTime = percentile(TimeTaken, 90), Percentile95ResponseTime = percentile(TimeTaken, 95), Percentile99ResponseTime = percentile(TimeTaken, 99)
```
Save as `Q1_8.png`.

### 10.4.2 — Blob Storage (`StorageBlobLogs`)

**Q2.2** — successful PutBlob uploads:
```kql
StorageBlobLogs | where OperationName has_all ("PutBlob") | where StatusCode has_all ("201") | count
```
Save as `Q2_2.png`.

**Q2.3** — average upload duration:
```kql
StorageBlobLogs | where OperationName == "PutBlob" | summarize AvgUploadMs = avg(DurationMs)
```
Save as `Q2_3.png`.

**Q2.4** — average blob read duration:
```kql
StorageBlobLogs | where OperationName == "GetBlob" | summarize AvgReadMs = avg(DurationMs)
```
Save as `Q2_4.png`.

**Q2.5** — pie chart of operations in last 3 days:
```kql
StorageBlobLogs | where TimeGenerated > ago(3d) | summarize count() by OperationName | sort by count_ desc | render piechart
```
Save as `Q2_5.png`.

### 10.4.3 — Cosmos DB (`CDBDataPlaneRequests`)

**Q3.4** — average read latency for the Flask app (Windows user-agent):
```kql
CDBDataPlaneRequests | where OperationName == "Read" | where UserAgent contains "Windows" | summarize AverageReadTimeWindows = avg(DurationMs)
```
Save as `Q3_4.png`.

**Q3.5** — slowest read in the last hour:
```kql
CDBDataPlaneRequests | where TimeGenerated > ago(1h) | where OperationName == "Read" | order by DurationMs desc | take 1
```
Save as `Q3_5.png`.

**Q3.7** — pie chart of requests by client IP over 3 days:
```kql
CDBDataPlaneRequests | where TimeGenerated > ago(3d) | summarize count() by ClientIpAddress | render piechart
```
Save as `Q3_7.png`.

### Exercise Deliverables

- `Q1_1_3.png`, `Q1_2.png`, `Q1_4.png`, `Q1_7.png`, `Q1_8.png`
- `Q2_2.png`, `Q2_3.png`, `Q2_4.png`, `Q2_5.png`
- `Q3_4.png`, `Q3_5.png`, `Q3_7.png`
- **12 screenshots** — each must show your username in the portal corner.

### Checklist (fill in before proceeding)

- [ ] All 12 KQL screenshots saved with correct names.
- [ ] Each screenshot shows non-zero data (widen the time range if empty).
- [ ] Username visible in every screenshot.

---

## Exercise 10.5 — Deployment Logs (`AppServicePlatformLogs`)

### Step-by-step

**Q4.2** — container restart events:
```kql
AppServicePlatformLogs | where tolower(Message) contains "restart" | where tolower(Message) contains "container"
```
Save as `Q4_2.png`.

**Q4.3** — error-level deployment logs:
```kql
AppServicePlatformLogs | where Level == "Error"
```
Save as `Q4_3.png`.

**Q4.4** — warnings mentioning "unhealthy":
```kql
AppServicePlatformLogs | where Level == "Warning" | where tolower(Message) contains "unhealthy"
```
Save as `Q4_4.png`.

### Exercise Deliverables

- `Q4_2.png`, `Q4_3.png`, `Q4_4.png`.

### Checklist (fill in before proceeding)

- [ ] All 3 platform-log screenshots saved.
- [ ] Total of **16 screenshots** collected across 10.3–10.5.

---

## Tear-down (do last)

1. Portal top search → `Resource groups` → click `lab10` → top command bar **Delete resource group**.
2. In the confirmation blade, type `lab10` into the text field → tick **Apply force delete for selected Virtual machines and Virtual machine scale sets** (if present) → **Delete**.
3. Wait for the green "Deleted resource group" notification, then refresh the Resource groups list — `lab10` must be gone.

### Final Practical Checklist

**All deliverables archived (16 screenshots):**

- [ ] `10_3.png` — Application Map (Ex 10.3)
- [ ] `Q1_1_3.png`, `Q1_2.png`, `Q1_4.png`, `Q1_7.png`, `Q1_8.png` — App Service KQL (Ex 10.4.1)
- [ ] `Q2_2.png`, `Q2_3.png`, `Q2_4.png`, `Q2_5.png` — Blob Storage KQL (Ex 10.4.2)
- [ ] `Q3_4.png`, `Q3_5.png`, `Q3_7.png` — Cosmos DB KQL (Ex 10.4.3)
- [ ] `Q4_2.png`, `Q4_3.png`, `Q4_4.png` — Platform logs (Ex 10.5)

**Cleanup & submission:**

- [ ] Username visible in every screenshot.
- [ ] Resource group `lab10` deleted.
- [ ] All deliverables uploaded to the course submission system.
