# Practice 12

Practical webpage: https://courses.cs.ut.ee/2026/cloud/spring/Main/Practice12

> Theme: Infrastructure as Code — Azure resources (Resource Group, Cosmos DB, Storage Account, Linux App Service) deployed from a single Terraform project.

> Reminder: the only student-specific value you need to capture is your **Azure Student Subscription ID** (grab it from `az account list -o table` after `az login` in 12.1). Paste it into `variable.tf` when you reach Exercise 12.2.

> **NB!** (course wording) Replace the `prefix` default value with a unique prefix — the course suggests your last name plus `lab12` (e.g. `jakovitslab12`). This README uses `lilab12` (surname Li). Replace the `subscription_id` default with your **Student Subscription ID**.

> **Region note.** The course default is `northeurope`. This repository uses `swedencentral` to stay consistent with Practice 10 and 13 (and because Sweden Central is the regional default for the UT Azure tenant). Either is acceptable; just keep it consistent across all four resources or you will hit cross-region pairing errors with Cosmos DB + App Service.

> **Bonus credit:** Practice 12 has **no separate bonus tasks** — there is no "Bonus" section on the course page (and the course's "Possible solutions to potential issues" header is empty as of the snapshot used to write this README). What the course does flag are three "Individual task" pieces that students must author themselves:
> 1. **12.3** — Cosmos SQL database + container ("Update the Terraform manifest to also automatically create `sql_database` and `sql_container` for storing documents.")
> 2. **12.4** — Storage container ("Update the Terraform manifest to also automatically create an Azure storage account **Container** to store images in.")
> 3. **12.5** — Application zip preparation + App Service deployment (course relies on you to assemble the Lab 5 messageboard into a flat zip and wire it through `zip_deploy_file`).
>
> All three are pre-filled or fully scripted below — complete them as part of the normal flow and you have earned every point on offer.

> **Course submission status note (as of the snapshot used here):** the course page shows "Sellele ülesandele ei saa hetkel lahendusi esitada" — submissions may not be open yet. Keep all deliverables (`.out` files, screenshots, `main.tf`, `variable.tf`, `python-lab-app.zip`) ready in this folder so you can upload as soon as the form opens.

---

## Exercise 12.1 — Dev Machine + Azure CLI + Terraform

### Step-by-step

1. On OpenStack, launch a new VM for this practical:
   - **Project → Compute → Instances → Launch Instance**.
   - **Details:** Instance Name: `Lab12_Li` · Count: `1`.
   - **Source:** Boot Source: **Image** · Create New Volume: **Yes** · Volume Size: `20 GB` · Delete Volume on Instance Delete: **Yes** · Image: **Ubuntu 24.04 LTS** (up-arrow to select).
   - **Flavor:** `g4.r4c2` (up-arrow).
   - **Networks:** `shared` (or the default student network).
   - **Security Groups:** ensure the group allowing SSH (port 22) is selected.
   - **Key Pair:** select your existing key (matching `C:\Users\qunyan\Desktop\Qun Yan Li.pem`).
   - **Launch Instance**. Wait for **Status: Active** and copy the public IP — call it `<LAB12_VM_IP>`.

2. SSH in from Windows PowerShell:
   ```powershell
   ssh -i "C:\Users\qunyan\Desktop\Qun Yan Li.pem" ubuntu@<LAB12_VM_IP>
   ```

3. Install Azure CLI + Terraform via apt and verify both in one chained run:
   ```bash
   curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash && sudo apt-get update && sudo apt-get install -y gnupg software-properties-common && wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg && echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list && sudo apt-get update && sudo apt-get install -y terraform && terraform -help | head -n 5 && az --version | head -n 1
   ```

4. Authenticate to Azure and capture the subscription ID:
   ```bash
   az login --use-device-code && az account list -o table
   ```
   The command prints a device code — open the displayed URL in a browser on your Windows machine, paste the code, and sign in with `qun.yan.li@ut.ee`. Choose directory **Tartu Ülikool** if prompted.
   After `az account list`, copy the `SubscriptionId` column value for the UT student subscription — you will paste it into `variable.tf` in Exercise 12.2.

> **Why device-code login?** The course notes that an Azure **Service Principal** would be the cleaner, scriptable option, but UT student accounts do not have the directory permissions to create one. Personal `az login` is the only authentication path available to us, and that is what `provider "azurerm"` will use during `terraform apply`.

### Exercise Deliverables

- None — setup only.

### Checklist (fill in before proceeding)

- [ ] `terraform -help` and `az --version` both work.
- [ ] `az login` succeeded; subscription ID copied.

---

## Exercise 12.2 — First Terraform Manifest (Resource Group)

> The course suggests the folder name `terraform-azure`. We use `~/lab12` here to stay consistent with the other practicals in this repo — Terraform does not care about the folder name. If you prefer to follow the course wording verbatim, replace `~/lab12` with `~/terraform-azure` everywhere below.

### Step-by-step

1. Create a project folder and both files:
   ```bash
   mkdir -p ~/lab12 && cd ~/lab12
   ```
2. Create `main.tf`:
   ```terraform
   terraform {
     required_providers {
       azurerm = {
         source  = "hashicorp/azurerm"
         version = "~> 4.27.0"
       }
     }
     required_version = ">= 1.1.0"
   }

   provider "azurerm" {
     features {}
     subscription_id = var.subscription_id
   }

   resource "azurerm_resource_group" "lab-resource-group" {
     name     = "${var.prefix}-resource-group"
     location = var.location
   }
   ```
3. Create `variable.tf` (fill `<SUBSCRIPTION_ID>` with the value from 12.1):
   ```terraform
   variable "prefix" {
     description = "The prefix used for all resources"
     default     = "lilab12"
   }

   variable "location" {
     description = "The Azure Region in which all resources are created."
     default     = "swedencentral"
   }

   variable "subscription_id" {
     description = "Azure subscription ID"
     default     = "<SUBSCRIPTION_ID>"
   }
   ```
4. Initialise, validate, apply, and capture the output in one chain:
   ```bash
   terraform init && terraform validate && terraform apply -auto-approve | tee lab12_2.out && az group list -o table | grep lilab12 && az group exists -n lilab12-resource-group
   ```

### Exercise Deliverables

- `lab12_2.out` (or screenshot `lab12_2.png`) — terraform apply output.
- `main.tf`, `variable.tf`.

### Checklist (fill in before proceeding)

- [ ] `az group exists` returned `true`.
- [ ] Output captured.

---

## Exercise 12.3 — Cosmos DB Account + SQL Database + Container

> **Individual task (course wording):** the SQL database and container blocks below are the part the course asks you to author yourself by reading the Terraform Registry. Both blocks are pre-filled here; cross-check them against the registry pages if you want to understand each argument:
> - `azurerm_cosmosdb_sql_database` — https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cosmosdb_sql_database
> - `azurerm_cosmosdb_sql_container` — https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cosmosdb_sql_container

### Step-by-step

Append to `main.tf`:
```terraform
resource "azurerm_cosmosdb_account" "lab-cosmosdb" {
  name                = "${var.prefix}-cosmosdb"
  location            = azurerm_resource_group.lab-resource-group.location
  resource_group_name = azurerm_resource_group.lab-resource-group.name
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"

  consistency_policy {
    consistency_level = "Eventual"
  }

  geo_location {
    location          = azurerm_resource_group.lab-resource-group.location
    failover_priority = 0
  }
}

resource "azurerm_cosmosdb_sql_database" "lab-cosmosdb-db" {
  name                = "lab5messagesdb"
  resource_group_name = azurerm_resource_group.lab-resource-group.name
  account_name        = azurerm_cosmosdb_account.lab-cosmosdb.name
}

resource "azurerm_cosmosdb_sql_container" "lab-cosmosdb-container" {
  name                  = "lab5messages"
  resource_group_name   = azurerm_resource_group.lab-resource-group.name
  account_name          = azurerm_cosmosdb_account.lab-cosmosdb.name
  database_name         = azurerm_cosmosdb_sql_database.lab-cosmosdb-db.name
  partition_key_paths   = ["/id"]
  partition_key_version = 1
  throughput            = 400
}
```
Apply in two stages so you get both deliverables:
```bash
terraform validate && terraform apply -auto-approve -target=azurerm_cosmosdb_account.lab-cosmosdb | tee lab12_3a.out && az cosmosdb list -o table | grep lilab12
terraform apply -auto-approve | tee lab12_3b.out && az cosmosdb sql container list --account-name lilab12-cosmosdb --database-name lab5messagesdb -g lilab12-resource-group -o table
```

### Exercise Deliverables

- `lab12_3a.out` — Cosmos account only.
- `lab12_3b.out` — database + container.

### Checklist (fill in before proceeding)

- [ ] `lab5messagesdb` visible in Azure Portal.
- [ ] `lab5messages` container shows partition key `/id`.

---

## Exercise 12.4 — Storage Account + Blob Container

> **Individual task (course wording):** the `azurerm_storage_container` block is the bit the course asks you to add yourself. It is pre-filled below; the registry reference is https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/storage_container — `container_access_type = "blob"` is what makes uploaded images publicly readable, which the Lab 5 messageboard depends on.

### Step-by-step

Append to `main.tf`:
```terraform
resource "azurerm_storage_account" "lab-storageaccount" {
  name                     = "${var.prefix}storageaccount"
  resource_group_name      = azurerm_resource_group.lab-resource-group.name
  location                 = azurerm_resource_group.lab-resource-group.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_storage_container" "lab-storage-container" {
  name                  = "images"
  storage_account_id    = azurerm_storage_account.lab-storageaccount.id
  container_access_type = "blob"
}
```
Apply + verify:
```bash
terraform validate && terraform apply -auto-approve -target=azurerm_storage_account.lab-storageaccount | tee lab12_4a.out && az storage account list -o table | grep lilab12
terraform apply -auto-approve | tee lab12_4b.out && az storage container list --account-name lilab12storageaccount --auth-mode login -o table
```

### Exercise Deliverables

- `lab12_4a.out` — account only.
- `lab12_4b.out` — container added.

### Checklist (fill in before proceeding)

- [ ] `lilab12storageaccount` visible.
- [ ] Container `images` with **Blob** public access.

---

## Exercise 12.5 — App Service Plan + Linux Web App (Message Board)

> **Individual task (course wording):** the course expects you to assemble the Lab 5 application code yourself and wire it into `zip_deploy_file`. The Terraform blocks below are pre-filled; the work that is genuinely on you is preparing `python-lab-app.zip` with the **flat structure** the course mandates ("make sure zip container uses a flat structure. It should not contain another subfolder inside which are your main files.").

> **F1 quota — read this first.** Azure caps each subscription at **one Free (F1) App Service Plan per region** (course wording: "It is likely that you can only have a single App Service plan per Azure region."). Practice 10 already created an F1 plan in **Sweden Central**, so `terraform apply` here will fail with `OperationNotAllowed` until that earlier plan is gone. Either delete the whole Practice 10 resource group in the Azure Portal first, or run `terraform destroy` from your Practice 10 working folder if it was deployed via Terraform. Verify with `az appservice plan list -o table` — there must be **zero** F1 plans left in `swedencentral` before you continue.

> **Pre-flight check — Linux F1 in your chosen region.** Some Azure regions disable Linux F1 SKUs entirely (Practice 10 hit this on the Portal and had to bump Python). Run before you start:
> ```bash
> az appservice list-locations --sku F1 --linux-workers-enabled -o table
> ```
> If `Sweden Central` (or whichever region you chose in `variable.tf`) is **not** in the list, switch `variable.tf`'s `location` to one that is — `northeurope`, `westeurope`, or `polandcentral` are all known good. Keep all four resource types (RG, Cosmos, Storage, App Service Plan) in the same region or you'll hit cross-region pairing errors.

### Step-by-step

1. Prepare the application zip on the lab VM:
   ```bash
   mkdir -p ~/lab12/lab-app && cd ~/lab12/lab-app
   ```
   Copy the **Lab 5 messageboard** Python project into this folder. The Practice 5 source lives at `c:\MSc-Computer-Science\Semester-2\cloud\courses_2026_cloud_spring_Practice5\practice5_source\` on the Windows machine — note that Practice 5 is **not** under `2026-cloud-practicals\practice-5\` in this repo (only practice-10/11/12/13/14 ship locally). Options:
   - Option A (recommended) — `scp` the contents (the trailing `/*` is critical: it copies the *contents* into a flat layout, not the wrapping folder):
     ```powershell
     scp -i "C:\Users\qunyan\Desktop\Qun Yan Li.pem" -r "C:\MSc-Computer-Science\Semester-2\cloud\courses_2026_cloud_spring_Practice5\practice5_source\*" ubuntu@<LAB12_VM_IP>:~/lab12/lab-app/
     ```
   - Option B — clone the Practice 5 repo on the lab VM (`git clone <practice5-url> ~/lab12/practice5-tmp && mv ~/lab12/practice5-tmp/practice5_source/* ~/lab12/lab-app/`).

   Required flat layout at the zip root (no wrapping folder — verified against the actual `practice5_source/` tree):
   - `app.py` — Flask entrypoint. The course/template uses `app.py`, **not** `main.py`. App Service auto-detects `gunicorn app:app` from this filename, so no custom startup command is needed.
   - `requirements.txt` — actual contents from Practice 5 are `flask==3.0.2`, `azure-storage-blob==12.3.1`, `six`, `azure-cosmos`. Keep `six` — older `azure-storage-blob` pins still need it at import time.
   - `templates/` folder with `home.html` and `handle_message.html`.
   - `static/` folder (empty in the source — `app.py` creates `static/images/` at runtime via `os.makedirs(UPLOAD_FOLDER, exist_ok=True)` for temporary local uploads before pushing them to Blob Storage). Empty folders may be skipped by `zip -r`; that is fine, the runtime will recreate it.
   - `data.json` (sample data shipped with Practice 5; harmless to include, the running app does not read it).

   Zip with flat structure and sanity-check the contents:
   ```bash
   cd ~/lab12/lab-app && zip -r python-lab-app.zip . -x "python-lab-app.zip" && ls -la python-lab-app.zip && unzip -l python-lab-app.zip | head -n 20
   ```
   The `unzip -l` preview must show `app.py` and `requirements.txt` at the **top level** (no leading `practice5_source/` or other folder prefix). If they are nested, re-zip from *inside* the folder that contains them. Final on-disk path must be `~/lab12/lab-app/python-lab-app.zip` so the relative `zip_deploy_file = "./lab-app/python-lab-app.zip"` resolves from `~/lab12`.
2. Append to `main.tf`:
   ```terraform
   resource "azurerm_service_plan" "lab-app-serviceplan" {
     name                = "${var.prefix}-app-zip-python"
     location            = azurerm_resource_group.lab-resource-group.location
     resource_group_name = azurerm_resource_group.lab-resource-group.name
     os_type             = "Linux"
     sku_name            = "F1"
   }

   resource "azurerm_linux_web_app" "lab-app-service" {
     name                = "${var.prefix}-zipdeploy"
     location            = azurerm_resource_group.lab-resource-group.location
     resource_group_name = azurerm_resource_group.lab-resource-group.name
     service_plan_id     = azurerm_service_plan.lab-app-serviceplan.id

     app_settings = {
       SCM_DO_BUILD_DURING_DEPLOYMENT = "true"
       STORAGE_ACCOUNT                = azurerm_storage_account.lab-storageaccount.name
       COSMOS_URL                     = azurerm_cosmosdb_account.lab-cosmosdb.endpoint
       MasterKey                      = azurerm_cosmosdb_account.lab-cosmosdb.primary_key
       CONN_KEY                       = azurerm_storage_account.lab-storageaccount.primary_access_key
     }

     site_config {
       always_on = false
       application_stack {
         python_version = "3.9"
       }
     }

     zip_deploy_file = "./lab-app/python-lab-app.zip"
   }
   ```
3. Apply and open the site:
   ```bash
   terraform validate && terraform apply -auto-approve | tee lab12_5a.out && az webapp show -n lilab12-zipdeploy -g lilab12-resource-group --query defaultHostName -o tsv
   ```
4. Visit `https://lilab12-zipdeploy.azurewebsites.net/`. The first request after deploy can take **30 s – 2 min** (Free F1 cold start + `pip install -r requirements.txt` triggered by `SCM_DO_BUILD_DURING_DEPLOYMENT=true`). If you see an Azure default page or a 502, wait a minute and refresh — do not redeploy yet. Post a message with an image. Take the screenshot (URL bar + your message + the uploaded image must all be visible) — save as `lab12_5b.png`.

> If the deploy fails with `OperationNotAllowed` / "quota exceeded" on the F1 tier, you still have an F1 plan elsewhere in `swedencentral` (most likely from Practice 10). List with `az appservice plan list -o table`, delete the offender (`az appservice plan delete -g <rg> -n <plan>`), then re-run `terraform apply`.

> If `zip_deploy_file` errors with "ZIP file not found", you are probably running `terraform apply` from outside `~/lab12`. The path is **relative to the working directory of the Terraform CLI**, so always invoke from `~/lab12`.

### Exercise Deliverables

- `lab12_5a.out` — terraform apply output.
- `lab12_5b.png` — working messageboard screenshot.

### Checklist (fill in before proceeding)

- [ ] Message posted and image visible.
- [ ] URL, message text, and image captured in one screenshot.

---

## Further Reading (linked from the course page)

- HashiCorp's official Azure get-started tutorials — https://developer.hashicorp.com/terraform/tutorials/azure-get-started
- HashiCorp's CLI install guide (the apt-based path the course recommends) — https://developer.hashicorp.com/terraform/tutorials/azure-get-started/install-cli
- `terraform-provider-azurerm` examples on GitHub — https://github.com/hashicorp/terraform-provider-azurerm/tree/main/examples
- `azurerm` resource reference (use this whenever you extend the manifest) — https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/

---

## Possible solutions to potential issues

> The course page reserves a section with this exact heading but leaves it empty. Below is a consolidated list of the failure modes that actually bite during this practical and how to clear them — most of these are also flagged inline in the relevant exercise above.

**`terraform apply` errors with `azure CLI is not signed in` / token expired:**
- `az login` sessions silently expire (typically after a few hours). Re-run `az login --use-device-code` and `az account set --subscription <SUBSCRIPTION_ID>` to fix. The `subscription_id` in `variable.tf` and the active CLI subscription must match.

**`OperationNotAllowed` / "quota exceeded" on the F1 SKU (Exercise 12.5):**
- One Free F1 plan per subscription per region. List with `az appservice plan list -o table`, identify any leftover plan in your target region (most often the Practice 10 one), then `az appservice plan delete -g <rg> -n <plan> --yes`. Re-run `terraform apply`.

**`LinkedAuthorizationFailed` / `SubscriptionNotFound` on first apply:**
- The `subscription_id` value in `variable.tf` is wrong, or your account is not linked to it. Verify with `az account list -o table` and copy the **SubscriptionId** column verbatim. UT student accounts typically have access to exactly one paid subscription — pick that one.

**`zip_deploy_file` errors with "ZIP file not found":**
- Path is relative to the Terraform CLI's working directory, not the manifest. Always run `terraform apply` from `~/lab12`, not from a sub-folder. Confirm with `ls -la ~/lab12/lab-app/python-lab-app.zip` before applying.

**Web app returns 502 / Azure default page on first visit:**
- Cold start + `pip install` from `SCM_DO_BUILD_DURING_DEPLOYMENT=true` can take 30 s – 2 min. Wait and refresh. If still failing after 5 min, fetch the deploy log:
  ```bash
  az webapp log tail -n lilab12-zipdeploy -g lilab12-resource-group
  ```
  Common root causes: missing `six` in `requirements.txt`, nested folder inside the zip (so App Service runs `gunicorn app:app` against an empty root), or wrong `python_version`.

**Web app loads but uploads or message reads fail:**
- One of the four `app_settings` is missing or wrong. Hit `https://lilab12-zipdeploy.scm.azurewebsites.net/Env.cshtml` (Kudu env page) and confirm `STORAGE_ACCOUNT`, `CONN_KEY`, `COSMOS_URL`, `MasterKey` are all present and non-empty. Terraform's `app_settings` block sets them as plain names; the app reads either the plain or the `APPSETTING_`-prefixed variant, so both work.

**`python_version = "3.9"` is rejected by the API:**
- The course wording says Python 3.9, and `azurerm` still accepts it on Linux F1 as of provider `~> 4.27.0`. If your region/subscription has dropped 3.9 (Practice 10 hit this on the Portal), bump to `3.10` or `3.11` in the `application_stack` block — the Lab 5 messageboard runs cleanly on both. This is a defensible deviation; mention it in your submission notes.

**`storage_account_id` argument unknown (provider 3.x):**
- The Storage Container block in 12.4 uses `storage_account_id` (provider 4.x style). If `terraform init` pulled an older provider, either upgrade with `terraform init -upgrade` or fall back to the legacy `storage_account_name = azurerm_storage_account.lab-storageaccount.name` argument.

**Cosmos container creation hangs or fails with `BadRequest: PartitionKey`:**
- `partition_key_paths = ["/id"]` (note: list, not string) and `partition_key_version = 1` are required in provider 4.x. The string-form `partition_key_path` is deprecated; if you copied a 3.x example, swap it.

**Storage account name "is not available" / "name is invalid":**
- `azurerm_storage_account.name` must be **3–24 lowercase alphanumeric characters, globally unique**. With prefix `lilab12` plus suffix `storageaccount` we land on `lilab12storageaccount` (21 chars, ok). If yours collides, change the `prefix` default in `variable.tf` and re-run `terraform apply` (Terraform will rename the resource — destroy + recreate).

**`terraform destroy` leaves a "soft-deleted" Cosmos / KeyVault behind:**
- Azure soft-deletes some resources for 7 days. If you want to redeploy with the same name immediately, run `az cosmosdb purge -n lilab12-cosmosdb` (or use the Portal "Recover deleted accounts" blade) before the next `terraform apply`.

**Submissions form is closed (Estonian banner "Sellele ülesandele ei saa hetkel lahendusi esitada"):**
- The course occasionally locks submissions until a deadline opens. Keep the deliverables ready in this folder and check the course page daily; nothing on your side is broken.

---

## Tear-down

The course page does **not** require a destroy step, but Azure student credit is finite — running `terraform destroy` after submission frees the quota for later practicals. Run from the `~/lab12` folder:
```bash
terraform destroy -auto-approve && az group list -o table
```
After it finishes, `az group list` should no longer show `lilab12-resource-group`. Also confirm the F1 plan is gone with `az appservice plan list -o table` — Practice 13 will need that quota back.

### Final Practical Checklist

**Terraform outputs / screenshots archived:**

- [ ] `lab12_2.out` (or `lab12_2.png`) — resource group created (Ex 12.2)
- [ ] `lab12_3a.out` — Cosmos account only (Ex 12.3)
- [ ] `lab12_3b.out` — Cosmos database + container (Ex 12.3)
- [ ] `lab12_4a.out` — storage account only (Ex 12.4)
- [ ] `lab12_4b.out` — storage container added (Ex 12.4)
- [ ] `lab12_5a.out` — web app deployed (Ex 12.5)
- [ ] `lab12_5b.png` — working messageboard (Ex 12.5)

**Source committed:**

- [ ] `main.tf`, `variable.tf`.
- [ ] `lab-app/python-lab-app.zip` (flat layout, no `<SUBSCRIPTION_ID>` hardcoded anywhere outside `variable.tf`).

**Cleanup & submission:**

- [ ] Resource group `lilab12-resource-group` destroyed via `terraform destroy`.
- [ ] All deliverables uploaded to the course submission system.
