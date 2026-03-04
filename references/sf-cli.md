# SF CLI Reference Guide

## Table of Contents
1. [Installation & Setup](#installation--setup)
2. [Authentication](#authentication)
3. [Scratch Org Management](#scratch-org-management)
4. [Sandbox Management](#sandbox-management)
5. [Source Deployment & Retrieval](#source-deployment--retrieval)
6. [Metadata Operations](#metadata-operations)
7. [Testing](#testing)
8. [Data Operations](#data-operations)
9. [Package Development](#package-development)
10. [Configuration & Aliases](#configuration--aliases)
11. [Troubleshooting](#troubleshooting)

---

## Installation & Setup

```bash
# Install via npm (recommended)
npm install @salesforce/cli --global

# Verify installation
sf --version

# Update CLI
sf update

# Check for available plugins
sf plugins --core

# Install additional plugins
sf plugins install @salesforce/plugin-packaging
```

### Project Initialization
```bash
# Create new project
sf project generate --name my-project --template standard

# Generate manifest (package.xml) from source
sf project generate manifest --source-dir force-app --name package.xml \
  --output-dir manifest

# Convert source format
sf project convert source --root-dir force-app --output-dir mdapi-output
sf project convert mdapi --root-dir mdapi-output --output-dir force-app
```

---

## Authentication

### Interactive (Browser-Based)
```bash
# Production or Developer Edition
sf org login web --alias prod --instance-url https://login.salesforce.com

# Sandbox
sf org login web --alias sandbox --instance-url https://test.salesforce.com

# Custom domain (My Domain)
sf org login web --alias myorg --instance-url https://mycompany.my.salesforce.com
```

### JWT (CI/CD Headless)
```bash
# Create connected app first, then:
sf org login jwt \
  --client-id <CONSUMER_KEY> \
  --jwt-key-file server.key \
  --username user@example.com \
  --alias prod \
  --instance-url https://login.salesforce.com \
  --set-default
```

### Auth Management
```bash
sf org list                           # List all authenticated orgs
sf org list --all                     # Include expired scratch orgs
sf org display --target-org myorg     # Show org details
sf org logout --target-org myorg      # Logout from specific org
sf org logout --all                   # Logout from all (prompts for each)
```

---

## Scratch Org Management

### Creating Scratch Orgs
```bash
# Basic creation
sf org create scratch \
  --definition-file config/project-scratch-def.json \
  --alias scratch1 \
  --duration-days 7 \
  --target-dev-hub devhub \
  --set-default

# With specific edition
sf org create scratch \
  --definition-file config/project-scratch-def.json \
  --alias scratch1 \
  --edition developer \
  --duration-days 30

# From scratch definition inline (no file)
sf org create scratch --edition developer --alias quick-scratch --duration-days 1
```

### Scratch Org Definition Examples

**Standard Developer Edition:**
```json
{
  "orgName": "My Dev Scratch",
  "edition": "Developer",
  "features": [],
  "settings": {
    "lightningExperienceSettings": {
      "enableS1DesktopEnabled": true
    }
  }
}
```

**With Communities & Multi-Currency:**
```json
{
  "orgName": "Full Featured Scratch",
  "edition": "Enterprise",
  "features": [
    "Communities",
    "ServiceCloud",
    "MultiCurrency",
    "PersonAccounts"
  ],
  "settings": {
    "communitiesSettings": { "enableNetworksEnabled": true },
    "lightningExperienceSettings": { "enableS1DesktopEnabled": true },
    "experienceBundleSettings": { "enableExperienceBundleMetadata": true }
  }
}
```

**Healthcare-specific:**
```json
{
  "orgName": "Health Cloud Scratch",
  "edition": "Enterprise",
  "features": ["HealthCloud", "Communities"],
  "settings": {
    "lightningExperienceSettings": { "enableS1DesktopEnabled": true }
  }
}
```

### Managing Scratch Orgs
```bash
sf org open --target-org scratch1          # Open in browser
sf org open --path lightning/setup          # Open directly to Setup
sf org display --target-org scratch1        # Show details & expiry
sf org delete scratch --target-org scratch1 --no-prompt  # Delete
sf org resume scratch --job-id 2SR...       # Resume creation if timed out
```

### Push/Pull (Scratch Org Source Tracking)
```bash
# Push local changes to scratch org
sf project deploy start --target-org scratch1

# Pull changes from scratch org to local
sf project retrieve start --target-org scratch1

# Check tracking status
sf project deploy preview --target-org scratch1
sf project retrieve preview --target-org scratch1

# Force overwrite conflicts
sf project deploy start --target-org scratch1 --ignore-conflicts
sf project retrieve start --target-org scratch1 --ignore-conflicts
```

---

## Sandbox Management

### Create/Refresh Sandboxes
```bash
# Create sandbox (must have sandbox licenses)
sf org create sandbox \
  --definition-file config/sandbox-def.json \
  --alias devsandbox \
  --target-org production \
  --wait 60

# Refresh existing sandbox
sf org refresh sandbox \
  --name DevSandbox \
  --target-org production \
  --wait 60

# Resume sandbox creation/refresh
sf org resume sandbox --name DevSandbox --target-org production
```

### Sandbox Definition
```json
{
  "sandboxName": "DevSandbox",
  "licenseType": "Developer",
  "autoActivate": true
}
```

License types: `Developer`, `Developer_Pro`, `Partial`, `Full`

### Deploying to Sandboxes
```bash
# Deploy specific source directory
sf project deploy start \
  --source-dir force-app \
  --target-org sandbox \
  --wait 33

# Deploy with manifest
sf project deploy start \
  --manifest manifest/package.xml \
  --target-org sandbox

# Deploy specific metadata
sf project deploy start \
  --metadata ApexClass:MyClass \
  --metadata ApexTrigger:MyTrigger \
  --target-org sandbox

# Validate only (dry run) with test execution
sf project deploy start \
  --source-dir force-app \
  --target-org sandbox \
  --dry-run \
  --test-level RunLocalTests

# Deploy with specific tests
sf project deploy start \
  --source-dir force-app \
  --target-org production \
  --test-level RunSpecifiedTests \
  --tests MyTestClass AnotherTestClass

# Quick deploy after successful validation
sf project deploy quick --job-id 0Af... --target-org production
```

---

## Source Deployment & Retrieval

### Deploy Options
```bash
# By source directory
sf project deploy start --source-dir force-app/main/default/classes

# By metadata type
sf project deploy start --metadata ApexClass

# By specific component
sf project deploy start --metadata ApexClass:AccountService

# Multiple components
sf project deploy start \
  --metadata ApexClass:AccountService \
  --metadata ApexClass:AccountServiceTest \
  --metadata ApexTrigger:AccountTrigger

# By manifest (package.xml)
sf project deploy start --manifest manifest/package.xml

# Resume a deploy
sf project deploy resume --job-id 0Af...

# Cancel a deploy
sf project deploy cancel --job-id 0Af...

# Report on deploy status
sf project deploy report --job-id 0Af...
```

### Retrieve Options
```bash
# By manifest
sf project retrieve start --manifest manifest/package.xml --target-org myorg

# By metadata type
sf project retrieve start --metadata ApexClass --target-org myorg

# By specific component
sf project retrieve start --metadata ApexClass:AccountService --target-org myorg

# To specific output directory
sf project retrieve start \
  --metadata ApexClass \
  --target-org myorg \
  --output-dir retrieved-source
```

### Test Levels for Deployment
| Level | Description | When to use |
|-------|-------------|-------------|
| `NoTestRun` | Skip tests | Sandbox deployments |
| `RunSpecifiedTests` | Run named tests | When you know which tests cover your code |
| `RunLocalTests` | All non-managed tests | Production deployments (most common) |
| `RunAllTestsInOrg` | All tests including managed | Rarely needed, very slow |
| `RunRelevantTests` | Auto-detected relevant tests | Spring '26 beta — auto-detects tests |

---

## Testing

```bash
# Run all local tests
sf apex run test --test-level RunLocalTests --target-org myorg --wait 10

# Run specific test class
sf apex run test --class-names MyTestClass --target-org myorg

# Run specific test method
sf apex run test --tests MyTestClass.testMethod1 --target-org myorg

# Run multiple classes
sf apex run test \
  --class-names MyTestClass,AnotherTestClass \
  --target-org myorg --wait 10

# Run tests with code coverage results
sf apex run test --test-level RunLocalTests \
  --code-coverage --result-format human \
  --target-org myorg --wait 10

# Get test results as JSON
sf apex run test --test-level RunLocalTests \
  --result-format json --target-org myorg --wait 10 \
  --output-dir test-results

# Execute anonymous Apex
sf apex run --file scripts/apex/init.apex --target-org myorg
echo "System.debug('Hello');" | sf apex run --target-org myorg
```

---

## Data Operations

```bash
# Query records
sf data query --query "SELECT Id, Name FROM Account LIMIT 10" \
  --target-org myorg

# Query with result format
sf data query --query "SELECT Id, Name FROM Account" \
  --target-org myorg --result-format csv

# Bulk query (Bulk API 2.0)
sf data query --query "SELECT Id, Name FROM Account" \
  --target-org myorg --bulk --wait 10

# Create a record
sf data create record --sobject Account \
  --values "Name='Acme Corp' Industry='Technology'" \
  --target-org myorg

# Update a record
sf data update record --sobject Account --record-id 001XX... \
  --values "Name='Acme Corporation'" --target-org myorg

# Delete a record
sf data delete record --sobject Account --record-id 001XX... \
  --target-org myorg

# Import data tree (preserves relationships)
sf data import tree --files data/Account.json --target-org myorg
sf data import tree --plan data/data-plan.json --target-org myorg

# Export data tree
sf data export tree \
  --query "SELECT Id, Name, (SELECT Id, LastName FROM Contacts) FROM Account" \
  --target-org myorg \
  --output-dir data

# Bulk upsert
sf data upsert bulk --sobject Account \
  --file data/accounts.csv \
  --external-id External_Id__c \
  --target-org myorg --wait 10

# Bulk delete
sf data delete bulk --sobject Account \
  --file data/account-ids.csv \
  --target-org myorg --wait 10
```

---

## Package Development

### Unlocked Packages (recommended for most projects)
```bash
# Create package
sf package create \
  --name "MyPackage" \
  --description "My unlocked package" \
  --package-type Unlocked \
  --path force-app \
  --target-dev-hub devhub

# Create package version
sf package version create \
  --package "MyPackage" \
  --installation-key test1234 \
  --wait 10 \
  --target-dev-hub devhub \
  --code-coverage

# List versions
sf package version list --target-dev-hub devhub

# Install package
sf package install \
  --package 04t... \
  --target-org myorg \
  --installation-key test1234 \
  --wait 10

# Promote to released
sf package version promote --package 04t... --target-dev-hub devhub
```

---

## Configuration & Aliases

```bash
# Set defaults
sf config set target-org=myorg
sf config set target-dev-hub=devhub
sf config set org-api-version=66.0

# View config
sf config list

# Set aliases
sf alias set myorg=user@example.com
sf alias set sandbox=user@example.com.sandbox
sf alias list

# Unset
sf config unset target-org
sf alias unset myorg
```

---

## Troubleshooting

### Common Issues & Fixes

**"The org cannot be found"**
```bash
sf org list --all              # Check if auth expired
sf org login web --alias myorg # Re-authenticate
```

**Deploy conflicts**
```bash
sf project deploy preview --target-org myorg  # See what would deploy
sf project deploy start --ignore-conflicts    # Force deploy
```

**Source tracking reset**
```bash
sf project delete tracking --target-org scratch1  # Reset tracking
```

**Debug logs**
```bash
sf apex log list --target-org myorg
sf apex log get --log-id 07L... --target-org myorg
sf apex log tail --target-org myorg  # Stream logs in real-time
```

**Check org limits**
```bash
sf org display --target-org myorg    # See API version, status
sf limits api display --target-org myorg  # See API call limits
```

### Environment Variables
```bash
SF_LOG_LEVEL=debug          # Verbose CLI logging
SF_ORG_API_VERSION=66.0     # Force API version
SFDX_USE_PROGRESS_BAR=false # Disable progress bar (CI)
SF_SINGLE_USE_ORG_OPEN_URL  # Removed in Spring '26
```
