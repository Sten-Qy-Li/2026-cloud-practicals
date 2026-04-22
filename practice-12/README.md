# Practice 12

Practical webpage: https://courses.cs.ut.ee/2026/cloud/spring/Main/Practice12

> Theme: Infrastructure as Code — Azure resources (Resource Group, Cosmos DB, Storage Account, Linux App Service) deployed from a single Terraform project.

> Reminder: the student info we will need — your **Azure Student Subscription ID** (grab it from `az account list -o table` after `az login`). Let me know the value when you have it so I can paste it into `variable.tf`.

---

## Exercise 12.1 — Dev Machine + Azure CLI + Terraform

### Step-by-step

1. On OpenStack, launch a new Ubuntu **24.04** VM (`g4.r4c2`). Name it `Lab12_Li`. Keep the existing key pair.
2. SSH in:
   ```powershell
   ssh -i "C:\Users\qunyan\Desktop\Qun Yan Li.pem" ubuntu@<LAB12_VM_IP>
   ```
3. Install Azure CLI + Terraform via apt, and verify both in one chained run:
   ```bash
   curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash && sudo apt-get update && sudo apt-get install -y gnupg software-properties-common && wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg && echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list && sudo apt-get update && sudo apt-get install -y terraform && terraform -help | head -n 5 && az --version | head -n 1
   ```
4. Authenticate to Azure and capture the subscription ID:
   ```bash
   az login --use-device-code && az account list -o table
   ```
   Record the `SubscriptionId` — you will paste it into `variable.tf`.

### Exercise Deliverables

- None — setup only.

### Checklist (fill in before proceeding)

- [ ] `terraform -help` and `az --version` both work.
- [ ] `az login` succeeded; subscription ID copied.

---

## Exercise 12.2 — First Terraform Manifest (Resource Group)

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
     default     = "northeurope"
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

### Step-by-step

1. Prepare the application zip. On your local Windows machine or the lab VM:
   ```bash
   mkdir -p ~/lab12/lab-app && cd ~/lab12/lab-app
   ```
   Drop the **Lab 5 messageboard** Python code here (flat layout — no nested folder at the zip root). Ensure:
   - `main.py` (or the Flask entrypoint)
   - `requirements.txt`
   - `templates/` folder with HTML
   - `images/` folder (empty, for temporary uploads)

   Zip with flat structure:
   ```bash
   cd ~/lab12/lab-app && zip -r ../lab-app/python-lab-app.zip . && ls -la ~/lab12/lab-app/python-lab-app.zip
   ```
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
4. Visit `https://lilab12-zipdeploy.azurewebsites.net/`. Post a message with an image. Take the screenshot (URL bar + your message + the uploaded image must all be visible) — save as `lab12_5b.png`.

> If the deploy fails with "quota exceeded" on the F1 tier, check that you have no other F1 plan in `northeurope`; delete it first.

### Exercise Deliverables

- `lab12_5a.out` — terraform apply output.
- `lab12_5b.png` — working messageboard screenshot.

### Checklist (fill in before proceeding)

- [ ] Message posted and image visible.
- [ ] URL, message text, and image captured in one screenshot.

---

## Tear-down

Destroy in a single command from the `~/lab12` folder:
```bash
terraform destroy -auto-approve && az group list -o table
```

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
