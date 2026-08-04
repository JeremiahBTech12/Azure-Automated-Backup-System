# Azure-Automated-Backup-System
In this project I build a fully automated backup system using Azure Blob Storage with versioning, lifecycle management policies that control cost by moving older data to cheaper storage tiers, and a Logic Apps workflow that sends a daily backup confirmation email.

## Video Walkthrough: 

## Project Overview

###
For many small businesses, a backup strategy consists of someone periodically copying files to an external drive. When that person is on vacation, busy, or simply forgets, backups stop. Eventually, hardware fails, files are accidentally deleted, or ransomware strikes—and without a reliable backup, critical business data can be lost permanently.

This project replaces that manual, unreliable process with a fully automated Azure-based backup solution that:

* Automatically replicates every uploaded file across multiple Azure data centers to improve durability and resilience.
* Preserves every previous version of each file, allowing accidentally deleted or overwritten data to be restored quickly.
* Reduces long-term storage costs by automatically moving files to lower-cost storage tiers after 30 days and archiving them after 90 days.
* Sends a daily confirmation email so the business owner can verify the backup system is operating successfully without manually checking logs or storage accounts.

The result is a backup system that is automated, resilient, and cost-efficient. Instead of relying on manual processes and hoping backups were completed, the business gains confidence that its data is continuously protected, recoverable, and managed with minimal operational effort.

## Architecture Flow
### <img width="1920" height="1080" alt="Resource Group (3)" src="https://github.com/user-attachments/assets/5d73ca77-121d-4816-abf4-7ad34e7fa665" />

The system operates using an event-driven architecture designed to automate file protection and eliminate the need for manual backups. Whenever a new file is uploaded to the primary Azure Storage container, the storage account emits an event that triggers a Logic App. The Logic App then processes the event and creates a backup copy in a separate backup container, ensuring that newly added files are protected automatically without requiring user intervention. From the moment a file is uploaded to the time the backup is successfully created, the entire workflow is fully automated.

Supporting this workflow are several resilience and monitoring components. Blob versioning preserves previous versions of files whenever they are modified or overwritten, providing protection against accidental changes and enabling point-in-time recovery. Azure Monitor, diagnostic settings, and Log Analytics collect operational telemetry, while alert rules continuously monitor backup activity and notify administrators if expected backup operations fail to occur. Together, these services provide comprehensive visibility into backup health, storage activity, and the overall reliability of the automated backup system.

Designing the architecture before implementing the solution ensured that each Azure service had a clearly defined responsibility, resulting in a scalable, resilient, and maintainable backup solution.


## What Gets Built

###
```
rg-backup-[yourname]
├── Storage Account (stbackup[yourname]) — GRS 
│   ├── Container: documents
│   ├── Container: database-exports
│   ├── Container: application-files
│   ├── Blob Versioning — enabled · 30-day soft delete
│   └── Lifecycle Policy — Hot → Cool (30d) → Archive (90d) → Delete (365d)
├── Log Analytics Workspace — 30-day retention
├── Storage Diagnostic Settings — StorageRead · StorageWrite · StorageDelete
├── Logic App Workflow — daily confirmation email
├── Monitor Alert Rule — fires if zero writes in 24 hours
└── Monitor Action Group — routes notifications to configured email
```

## Tools & Services Used
###
```

Infrastructure as Code |Terraform (azurerm provider ~> 3.0)
Cloud Platform |Microsoft Azure
Storage	|Azure Blob Storage (GRS, versioning, lifecycle management)
Monitoring |Azure Monitor + Log Analytics Workspace
Automation / Notifications |Azure Logic Apps
CLI / Deployment |Azure CLI, PowerShell
```

## Prerequisites 
### Before deploying, install and configure:
  - Terraform
  - Azure CLI
  - An active Azure subscription
  

## Terraform Configuration
###
Folder Setup


Windows (PowerShell):
```
New-Item -ItemType Directory -Path "$HOME\backup-system-001"
cd "$HOME\backup-system-001"
New-Item -ItemType File main.tf, variables.tf, outputs.tf, terraform.tfvars
```

## Step 1 — Write variables.tf

###
```terraform
variable "yourname" {
  description = "Your name, lowercase, no spaces. Used to make resource names unique."
  type        = string
}
 
variable "location" {
  type    = string
  default = "East US"
}
 
variable "alert_email" {
  description = "Email address to receive daily backup confirmation."
  type        = string
}
 
variable "tags" {
  type = map(string)
  default = {
    project     = "backup-system"
    environment = "dev"
    managed_by  = "terraform"
  }
}
```

## Step 2 — Write terraform.tfvars

###
```terraform
yourname    = "jeremiah"
location    = "East US"
alert_email = "your.email@example.com"
```

## Step 3 — Write main.tf


Provider

```terraform
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}
 
provider "azurerm" {
  features {}
}
 
data "azurerm_client_config" "current" {}
```

## Resource group

```terraform
resource "azurerm_resource_group" "main" {
  name     = "rg-backup-${var.yourname}"
  location = var.location
  tags     = var.tags
}
```

## Storage account

###
This is the core of the backup system. Every file uploaded here is automatically replicated across multiple physical Azure data centers — not just different rooms in the same building, but geographically separate facilities.

account_replication_type = "GRS" stands for Geo-Redundant Storage. Azure keeps your data in your primary region (East US) and asynchronously copies it to a secondary region (West US) automatically. If an entire region goes offline, your data still exists in the secondary. For a backup system, GRS is the correct choice — LRS (used in the data lab) only replicates within a single region.

min_tls_version = "TLS1_2" enforces that all connections to the storage account use TLS 1.2 or higher. Older TLS versions have known vulnerabilities and should not be used for backup data.

blob_properties with versioning_enabled = true is what makes this a real backup system rather than just a file store. Every time a file is overwritten or deleted, Azure keeps the previous version. You can restore any file to any point in time. A user who accidentally deletes a folder or saves a corrupted file over a good one does not lose data permanently.

delete_retention_policy with days = 30 means that even after a blob is deleted, Azure retains it in a soft-deleted state for 30 days before permanently removing it. This is a safety net on top of versioning.

```terraform
resource "azurerm_storage_account" "backup" {
  name                     = "stbackup${var.yourname}"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
  min_tls_version          = "TLS1_2"
 
  blob_properties {
    versioning_enabled = true
 
    delete_retention_policy {
      days = 30
    }
 
    container_delete_retention_policy {
      days = 30
    }
  }
 
  tags = var.tags
}
```


## Storage containers

###
Containers are the top-level organizational unit inside a storage account — similar to folders, but at the root level. Separating data by type (documents, database exports, application files) matters enormously when you need to restore something under pressure. You do not want to be searching through a mixed pile of files during an incident.

container_access_type = "private" means no public internet access. Files can only be accessed by authenticated Azure identities or connection strings. Backup data should never be publicly readable.

```terraform
resource "azurerm_storage_container" "documents" {
  name                  = "documents"
  storage_account_name  = azurerm_storage_account.backup.name
  container_access_type = "private"
}
 
resource "azurerm_storage_container" "database_exports" {
  name                  = "database-exports"
  storage_account_name  = azurerm_storage_account.backup.name
  container_access_type = "private"
}
 
resource "azurerm_storage_container" "application_files" {
  name                  = "application-files"
  storage_account_name  = azurerm_storage_account.backup.name
  container_access_type = "private"
}
```

## Lifecycle management policy

###
This is what keeps backup costs from growing unbounded over time. The policy defines rules that automatically move files between storage tiers based on how old they are.

Azure has four storage tiers — Hot, Cool, Cold, and Archive — each progressively cheaper to store but more expensive to read. Hot is for data accessed frequently. Cool is for data accessed occasionally. Archive is for data that almost never needs to be read but must be retained for compliance or recovery.

The base_blob rule applies to the current (live) version of each file. After 30 days of not being modified, a file moves from Hot to Cool storage automatically. After 90 days, it moves to Archive. After 365 days, the file is deleted.

The version rule applies to older versions that have been superseded. These are kept for 30 days (long enough to catch any data corruption or accidental deletion) then permanently removed. Keeping old versions indefinitely would grow storage costs without limit.

filters with prefix_match = ["documents/", "database-exports/", "application-files/"] means this policy applies across all three containers.

```terraform
resource "azurerm_storage_management_policy" "lifecycle" {
  storage_account_id = azurerm_storage_account.backup.id
 
  rule {
    name    = "backup-lifecycle"
    enabled = true
 
    filters {
      blob_types   = ["blockBlob"]
      prefix_match = ["documents/", "database-exports/", "application-files/"]
    }
 
    actions {
      base_blob {
        tier_to_cool_after_days_since_modification_greater_than    = 30
        tier_to_archive_after_days_since_modification_greater_than = 90
        delete_after_days_since_modification_greater_than          = 365
      }
 
      version {
        delete_after_days_since_creation = 30
      }
    }
  }
}
```

## Log Analytics Workspace

```terraform
resource "azurerm_log_analytics_workspace" "main" {
  name                = "law-backup-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
  tags                = var.tags
}
```


## Storage diagnostic settings

###
This routes storage account logs and metrics into Log Analytics. The StorageWrite log category records every file write — which is the signal that backups are landing. StorageRead and StorageDelete record reads and deletions. Together these give you a complete audit trail of everything that has happened to your backup data.

metric with category Transaction sends metrics about storage operations (request count, latency, errors) to Log Analytics, making it possible to alert on unusual patterns like a sudden drop in write activity.

```terraform
resource "azurerm_monitor_diagnostic_setting" "storage_logs" {
  name                       = "diag-storage-to-law"
  target_resource_id         = "${azurerm_storage_account.backup.id}/blobServices/default"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
 
  enabled_log { category = "StorageRead"   }
  enabled_log { category = "StorageWrite"  }
  enabled_log { category = "StorageDelete" }
 
  metric {
    category = "Transaction"
    enabled  = true
  }
}
```

## Action Group and Logic App for daily confirmation

###
The Action Group defines where notifications go. The Logic App is what sends the confirmation email each morning. The Logic App is provisioned here and configured in the portal in Step 6.

```terraform
resource "azurerm_monitor_action_group" "backup_alerts" {
  name                = "ag-backup-${var.yourname}"
  resource_group_name = azurerm_resource_group.main.name
  short_name          = "backupalert"
 
  email_receiver {
    name                    = "owner-email"
    email_address           = var.alert_email
    use_common_alert_schema = true
  }
 
  tags = var.tags
}
 
resource "azurerm_logic_app_workflow" "backup_confirmation" {
  name                = "la-backup-confirm-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  tags                = var.tags
}
```

## Monitor alert — detect if backups stop

###
This alert fires if the storage account receives zero write transactions in a 24-hour window. In other words: if nothing is being written to the backup storage, something has gone wrong upstream, and the business owner should know about it.

metric_name = "Transactions" is the metric being watched. operator = "LessThan" with threshold = 1 means the alert fires when the transaction count drops below 1 — i.e., zero transactions.

frequency = "PT1H" means Azure checks the condition every hour. window_size = "P1D" means each check looks at the past 24 hours of data.

dimension with name = "ApiName" and values = ["PutBlob", "PutBlock"] filters the transaction count to only write operations. Without this filter the alert would fire on any quiet period, including normal times when nobody is reading files.

```terraform
resource "azurerm_monitor_metric_alert" "no_writes" {
  name                = "alert-no-backup-writes"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_storage_account.backup.id]
  description         = "Fires if no files have been written to backup storage in 24 hours."
  severity            = 2
  frequency           = "PT1H"
  window_size         = "P1D"
 
  criteria {
    metric_namespace = "Microsoft.Storage/storageAccounts"
    metric_name      = "Transactions"
    aggregation      = "Total"
    operator         = "LessThan"
    threshold        = 1
 
    dimension {
      name     = "ApiName"
      operator = "Include"
      values   = ["PutBlob", "PutBlock"]
    }
  }
 
  action {
    action_group_id = azurerm_monitor_action_group.backup_alerts.id
  }
 
  tags = var.tags
}
```

## Step 4 — Write outputs.tf

```terraform
output "storage_account_name" {
  value = azurerm_storage_account.backup.name
}
 
output "storage_account_connection_string" {
  value     = azurerm_storage_account.backup.primary_connection_string
  sensitive = true
}
 
output "log_analytics_workspace_id" {
  value = azurerm_log_analytics_workspace.main.id
}
 
output "logic_app_endpoint" {
  value = azurerm_logic_app_workflow.backup_confirmation.access_endpoint
}
```

###
sensitive = true on the connection string means Terraform will not print it in plain text in your terminal. To view it: terraform output -raw storage_account_connection_string


## Deploy

###
Deploy the infrastructure:

```
terraform init
terraform plan
terraform apply
```

## Logic App Configuration (Daily Confirmation)

###
- In the portal, navigate to la-backup-confirm-[yourname]
- Click Logic app designer
- Click Add a trigger → search for Recurrence → select Recurrence
-  Set Frequency to Day and Interval to 1. Set At these hours to 8 (8:00 AM)
- Click + New step → search for Azure Blob Storage → select List blobs
- Create a connection. Switch Authentication Type to Access Key, and split the Terraform connection string into its two components:
Storage Account Name: stbackup<name>
Access Key: only the value between AccountKey= and ;EndpointSuffix — not the full connection string. 

- Set the folder to documents
- Click + New step → Office 365 Outlook → Send an email (V2)
- Fill in the email — To: your alert email; Subject: Daily Backup Confirmation — @{formatDateTime(utcNow(), 'yyyy-MM-dd')}; Body: Backup system status: Active. Files in documents container: @{length(body('List_blobs')?['value'])}. All backup containers are protected and healthy.
- Click Save

## Upload A Test File & Versioning

###
Windows (PowerShell):
```
"Backup test file created $(Get-Date)" | Out-File -FilePath "$env:TEMP\backup_test.txt" -Encoding utf8
 
az storage blob upload `
  --account-name stbackupjeremiah `
  --container-name documents `
  --name test/backup_test.txt `
  --file "$env:TEMP\backup_test.txt" `
  --auth-mode login
```

###
Now overwrite the file to create a second version:

Windows (PowerShell):
```
"Updated content — second version $(Get-Date)" | Out-File -FilePath "$env:TEMP\backup_test.txt" -Encoding utf8
 
az storage blob upload `
  --account-name stbackupjeremiah `
  --container-name documents `
  --name test/backup_test.txt `
  --file "$env:TEMP\backup_test.txt" `
  --auth-mode login `
  --overwrite
```

List the versions to confirm both exist:

Windows (PowerShell):
```
az storage blob list `
  --account-name stbackupjeremiah `
  --container-name documents `
  --include v `
  --auth-mode login `
  --output table
```
You should see two rows for test/backup_test.txt — the current version and one previous version. This confirms versioning is working.


## Verification Checklist
###
- Storage account stbackup[yourname] exists in the portal
- Storage account → Data management → Versioning shows Enabled
- Storage account → Data management → Lifecycle management shows the backup-lifecycle rule
- Three containers exist: documents, database-exports, application-files
- Logic App la-backup-confirm-[yourname] runs on a recurrence trigger
- Alert rule alert-no-backup-writes exists in Monitor → Alerts
- Test file upload produced two versions in blob list output


## Troubleshooting & Lessons Learned

###
- Azure CLI blob operations returned permission errors.
Running az storage blob commands with --auth-mode login depends on Azure AD data-plane permissions, such as the Storage Blob Data Contributor role. Creating the storage account with Terraform does not automatically assign these roles. The issue was resolved by authenticating with the storage account key using --auth-mode key --account-key "<key>", with the key obtained from the Terraform connection string output.

- Logic App validation failed because of an invalid action reference.
The error indicating that the List_blobs action could not be found occurred because a dynamic content expression was pointing to an action that had been renamed, deleted, or was no longer producing output. Recreating the dynamic content reference through the Logic Apps designer—or verifying the action name in Code View—ensured the expression referenced the correct internal action.

- Office 365 Outlook connector authentication was unsuccessful.
The Office 365 Outlook connector is designed for Microsoft 365 work or school accounts and does not support personal Microsoft email addresses such as @outlook.com, @hotmail.com, or @msn.com. Replacing it with the Outlook.com connector allowed the Logic App to authenticate successfully and send email notifications.

## Teardown
###
To remove all Azure resources created by this project, run:
```
terraform destroy
```
