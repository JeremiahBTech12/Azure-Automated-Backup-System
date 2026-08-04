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

## Deploy
###
Deploy the infrastructure:

```
terraform init
terraform plan
terraform apply
```

## Logic App Configuration (Daily Confirmation)

## Upload A Test File & Versioning

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

## Teardown
###
To remove all Azure resources created by this project, run:
```
terraform destroy
```
