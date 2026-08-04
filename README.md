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


## Tools & Services Used

## What Gets Built

## Prerequisites 

## Terraform Configuration

## Deploy

## Logic App Configuration (Daily Confirmation)

## Upload A Test File & Versioning

## Verification Checklist

## Troubleshooting

## Teardown
