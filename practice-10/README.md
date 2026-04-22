# Practice 10

Practical webpage: https://courses.cs.ut.ee/2026/cloud/spring/Main/Practice10

> Theme: Azure monitoring (App Service, Blob Storage, Cosmos DB) using Application Insights + Log Analytics + KQL.

> Reminder — Application Insights ingestion can take 5–30 minutes. **Archive every screenshot with a timestamp** (file mtime + filename pattern `10_x.png` / `Qx_y.png`). This counts as timing evidence for the deliverables.

---

## Exercise 10.1 — Deploy the Message Board App to Azure

### Step-by-step

1. Sign in to the Azure Portal with your UT student subscription.
2. Create a new Resource Group named `lab10` (region: `Sweden Central`).
3. Inside `lab10`, create:
   - an **App Service (Linux, Python 3.9)** — Free F1 tier.
   - a **Storage Account** → add a **Blob container** named `images`.
   - a **Cosmos DB account** (Core/SQL) → add database `lab5messagesdb` → container `lab5messages` (partition key `/id`, throughput 400).
4. Copy the following values and note them down:
   - `STORAGE_ACCOUNT` — storage account name
   - `CONN_KEY` — storage primary access key
   - `COSMOS_URL` — Cosmos DB endpoint URI
   - `MasterKey` — Cosmos DB primary key
5. In the Web App → **Configuration → Application settings**, add all four as environment variables. Prefix each name with `APPSETTING_` if using the Deployment Center / zip-deploy route.
6. Deploy the Practice 5 Message Board code (zip deploy or GitHub Actions). Verify the homepage renders at `https://<app-name>.azurewebsites.net`.

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

1. On the Web App blade → **Application Insights → Turn on Application Insights → Apply**. Create a new Log Analytics workspace in `lab10` (or reuse one).
2. On the Web App → **Monitoring → Diagnostic settings → Add diagnostic setting**. Enable **all** log categories + AllMetrics → destination: **Send to Log Analytics workspace** (the one just created).
3. Repeat step 2 on the Storage Account (pick the **blob** sub-resource) and on the Cosmos DB account. Enable every category and AllMetrics.

### Exercise Deliverables

- None — the deliverables materialise in 10.3–10.5 once data arrives.

### Checklist (fill in before proceeding)

- [ ] Application Insights is linked to the Web App.
- [ ] Diagnostic settings configured for App Service, Blob, Cosmos.
- [ ] Log Analytics workspace receives data (wait 5–30 min — you can move on and come back).

---

## Exercise 10.3 — Generate Workload & Capture Application Map

### Step-by-step

1. Open the Message Board in the browser. Post **~10 messages** with images (use small JPG/PNGs).
2. Refresh the homepage 10–15 times to generate GET traffic.
3. In the Azure Portal → Blob container `images`, confirm the uploaded files appear.
4. In Cosmos DB → **Data Explorer**, open `lab5messagesdb/lab5messages` and confirm the documents are present.
5. On the Web App → **Application Insights → Application Map**. Take a full-window screenshot that clearly shows your university username (`qun.yan.li@ut.ee`) in the top-right corner.

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

Run each query in **Log Analytics workspace → Logs**. Queries are single-line-friendly; pipes can stay on separate lines for readability in the query pane — that is the Azure UI, not a shell.

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

1. Azure Portal → Resource groups → `lab10` → **Delete resource group**. Confirm with the name.
2. Verify the resource group disappears from the list.

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
