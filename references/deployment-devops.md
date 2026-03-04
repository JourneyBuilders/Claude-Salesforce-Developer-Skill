# Deployment & DevOps Guide

## Development Models

### Org Development (Traditional)
Source of truth: the org. Deploy via change sets, metadata API, or package.xml.
Best for small teams, simple orgs.

### Package Development (Recommended)
Source of truth: Git. Use scratch orgs for dev, unlocked packages for deployment.
Best for teams, CI/CD, ISVs, complex orgs.

### Hybrid (Most Common)
Git as source of truth. Develop in scratch orgs or sandboxes. Deploy via `sf project deploy start`.

## Environment Strategy

```
Developer Sandbox / Scratch Org → Dev/QA Sandbox → UAT/Staging → Production
```

### Scratch Orgs vs Sandboxes

| Aspect | Scratch Org | Sandbox |
|--------|-------------|---------|
| Lifespan | 1-30 days | Persistent |
| Data | Empty (seed with scripts) | Copy from prod |
| Creation | Minutes | Hours (full) |
| Tracking | Built-in source tracking | No auto tracking |
| Use case | Feature development | Integration, UAT |

### Sandbox Types
- **Developer**: Metadata only, 200MB, 1-day refresh — individual dev
- **Developer Pro**: Metadata only, 1GB, 1-day refresh — dev + data testing
- **Partial Copy**: Metadata + sample data, 5GB, 5-day refresh — QA
- **Full**: Full copy, same as prod, 29-day refresh — UAT, staging

## Git Workflow

```
main (production)
├── develop (integration)
│   ├── feature/JIRA-123-account-trigger
│   └── bugfix/JIRA-789-fix
└── release/v1.2
```

### Typical Workflow
```bash
# 1. Create feature branch
git checkout develop && git pull
git checkout -b feature/JIRA-123-description

# 2. Create scratch org
sf org create scratch -f config/project-scratch-def.json -a feature-123 -d 7
sf project deploy start -o feature-123

# 3. Develop and retrieve
sf project retrieve start -o feature-123

# 4. Commit
git add force-app/ && git commit -m "feat(JIRA-123): Add validation"
git push origin feature/JIRA-123-description

# 5. PR → CI validates → merge → deploy to QA
sf project deploy start --source-dir force-app --target-org qa \
  --test-level RunLocalTests --wait 30
```

### .gitignore
```
.sfdx/
.sf/
*.log
.DS_Store
.vscode/
node_modules/
coverage/
test-results/
```

## CI/CD Pipeline (GitHub Actions Example)

```yaml
name: Salesforce CI
on:
  pull_request:
    branches: [develop, main]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install SF CLI
        run: npm install -g @salesforce/cli
      - name: Auth
        run: |
          echo "${{ secrets.SFDX_AUTH_URL }}" > auth.txt
          sf org login sfdx-url --sfdx-url-file auth.txt --alias ci-org
      - name: Validate
        run: |
          sf project deploy start --source-dir force-app \
            --target-org ci-org --dry-run \
            --test-level RunLocalTests --wait 30
```

### Generating Auth URL for CI
```bash
sf org display --target-org myorg --verbose
# Copy "Sfdx Auth Url" → store as CI secret
```

## Package Development

### Unlocked Packages (Recommended)
```bash
# Create package (once)
sf package create --name "MyFeature" --package-type Unlocked \
  --path force-app --target-dev-hub devhub

# Create version
sf package version create --package "MyFeature" \
  --installation-key mykey --wait 20 --code-coverage

# Install
sf package install --package 04t... --target-org prod \
  --installation-key mykey --wait 10

# Promote (locks version)
sf package version promote --package 04t...
```

## Metadata Management

### package.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>AccountService</members>
        <members>AccountServiceTest</members>
        <name>ApexClass</name>
    </types>
    <types>
        <members>myComponent</members>
        <name>LightningComponentBundle</name>
    </types>
    <version>66.0</version>
</Package>
```

### Generate package.xml
```bash
sf project generate manifest --source-dir force-app --name package.xml --output-dir manifest
```

### Common Metadata Types
ApexClass, ApexTrigger, LightningComponentBundle, AuraDefinitionBundle, CustomObject,
CustomField, Layout, FlexiPage, Flow, PermissionSet, Profile, CustomMetadata,
CustomLabel, StaticResource, ExternalClientApp

### Destructive Changes
```bash
sf project deploy start --manifest manifest/package.xml \
  --post-destructive-changes manifest/destructiveChanges.xml --target-org myorg
```

### Source Decomposition
Use decomposed metadata format for cleaner Git diffs. Each field, validation rule
gets its own file. Spring '26 adds ExternalServiceRegistration decomposition:
```bash
sf project convert source-behavior --behavior decomposeExternalServiceRegistration
```
