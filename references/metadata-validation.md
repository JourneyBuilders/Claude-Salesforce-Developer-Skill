# Metadata Validation, Error Detection & Troubleshooting

## Table of Contents
1. [Pre-Deployment Validation](#pre-deployment-validation)
2. [Common XML Syntax Errors](#common-xml-syntax-errors)
3. [Metadata Deployment Errors & Solutions](#metadata-deployment-errors)
4. [VS Code XML Validation Issues](#vscode-xml-issues)
5. [Git & Source Control Metadata Issues](#git-metadata-issues)
6. [Automated Validation in CI/CD](#automated-validation)
7. [Reporting Bugs](#reporting-bugs)

---

## Pre-Deployment Validation

**Always validate before deploying.** Use `--dry-run` to catch errors without modifying the org:

```bash
# Validate deployment without actually deploying (checks compile, dependencies, etc.)
sf project deploy start --source-dir force-app --dry-run --target-org myorg

# Validate with test execution (required for production)
sf project deploy start --source-dir force-app --dry-run --test-level RunLocalTests --target-org myorg

# Validate specific metadata
sf project deploy start --metadata ApexClass:MyClass --dry-run --target-org myorg

# Validate from manifest
sf project deploy start --manifest manifest/package.xml --dry-run --target-org myorg
```

### Quick XML Syntax Check (before deploy)
Run a local XML well-formedness check on all metadata files:

```bash
# Using xmllint (install: apt-get install libxml2-utils / brew install libxml2)
find force-app -name "*.xml" -exec xmllint --noout {} \; 2>&1 | grep -v "validates"

# Quick Python check for well-formed XML
python3 -c "
import xml.etree.ElementTree as ET
import glob, sys
errors = []
for f in glob.glob('force-app/**/*.xml', recursive=True):
    try:
        ET.parse(f)
    except ET.ParseError as e:
        errors.append(f'{f}: {e}')
for e in errors:
    print(e)
if not errors:
    print('All XML files are well-formed')
sys.exit(1 if errors else 0)
"

# Using sf CLI scanner (PMD-based, catches code + some metadata issues)
sf scanner run --target force-app --format table
```

---

## Common XML Syntax Errors

### 1. Wrong or Missing XML Namespace

**Error**: `Cannot find the declaration of element 'CustomObject'` or deploy silently
ignores components.

**Cause**: Missing or incorrect `xmlns` on the root element.

**Fix**: Every Salesforce metadata XML file MUST have the correct namespace:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomObject xmlns="http://soap.sforce.com/2006/04/metadata">
    ...
</CustomObject>
```

Common root elements and their expected format:
```xml
<!-- Custom Object -->
<CustomObject xmlns="http://soap.sforce.com/2006/04/metadata">
<!-- Custom Field (standalone) -->
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
<!-- Apex Class -->
<ApexClass xmlns="http://soap.sforce.com/2006/04/metadata">
<!-- LWC meta -->
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
<!-- Permission Set -->
<PermissionSet xmlns="http://soap.sforce.com/2006/04/metadata">
<!-- Flow -->
<Flow xmlns="http://soap.sforce.com/2006/04/metadata">
<!-- Layout -->
<Layout xmlns="http://soap.sforce.com/2006/04/metadata">
<!-- Profile -->
<Profile xmlns="http://soap.sforce.com/2006/04/metadata">
<!-- package.xml -->
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
```

**IMPORTANT**: The namespace is always `http://soap.sforce.com/2006/04/metadata` (HTTP,
not HTTPS). Changing to HTTPS will be reverted on next retrieve. Do NOT change this.


### 2. Wrong API Version in Metadata

**Error**: `The field apiVersion can't be "XX.0"` or unexpected behavior.

**Cause**: API version in `-meta.xml` doesn't match what the org supports, or is too old/new.

**Fix**: Ensure `<apiVersion>` matches your target (currently 66.0 for Spring '26):
```xml
<!-- In .cls-meta.xml, .trigger-meta.xml, LWC .js-meta.xml -->
<apiVersion>66.0</apiVersion>
```

**Bulk fix** for all meta files:
```bash
# Find and update all apiVersion references
grep -rl "<apiVersion>" force-app --include="*.xml" | \
  xargs sed -i 's/<apiVersion>[0-9.]*<\/apiVersion>/<apiVersion>66.0<\/apiVersion>/g'
```


### 3. BOM (Byte Order Mark) in XML Files

**Error**: `Content is not allowed in prolog` or `XML document structures must start and
end within the same entity`.

**Cause**: Some editors (especially Windows Notepad) add an invisible BOM character
(`\xEF\xBB\xBF`) at the start of UTF-8 files. Salesforce Metadata API chokes on this.

**Detection**:
```bash
# Find files with BOM
grep -rl $'\xEF\xBB\xBF' force-app --include="*.xml"

# Or with hexdump
find force-app -name "*.xml" -exec sh -c 'head -c 3 "$1" | xxd | grep -q "efbb bf" && echo "$1"' _ {} \;
```

**Fix**:
```bash
# Remove BOM from all XML files
find force-app -name "*.xml" -exec sed -i '1s/^\xEF\xBB\xBF//' {} \;

# Prevent in VS Code: add to settings.json
# "files.encoding": "utf8"
# (NOT "utf8bom")
```


### 4. Invalid Characters / Encoding Issues

**Error**: `An invalid XML character (Unicode: 0xNN) was found` or mojibake in labels.

**Cause**: Non-UTF-8 characters, smart quotes, em-dashes, or invisible control characters
copied from Word, email, or web pages into labels, descriptions, or help text.

**Detection**:
```bash
# Find non-ASCII characters in XML files
grep -rPn '[^\x00-\x7F]' force-app --include="*.xml" | head -20

# Find specific problem characters (smart quotes, etc.)
grep -rPn '[\x80-\x9F]' force-app --include="*.xml"
```

**Fix**: Replace smart quotes with straight quotes, em-dashes with hyphens, and ensure
all files are saved as UTF-8 without BOM. In VS Code, check bottom-right encoding indicator.


### 5. Incorrect XML Element Ordering

**Error**: `Invalid content was found starting with element 'X'. One of '{Y}' is expected.`

**Cause**: Salesforce metadata XML requires elements in a **specific order** defined by
the metadata XSD schema. Manual edits often put elements in the wrong order.

**Common ordering issues**:
```xml
<!-- CustomField: correct order -->
<fields>
    <fullName>MyField__c</fullName>
    <defaultValue>false</defaultValue>
    <description>Description here</description>
    <externalId>false</externalId>
    <label>My Field</label>
    <trackTrending>false</trackTrending>
    <type>Checkbox</type>
</fields>

<!-- CustomObject: correct top-level order (abbreviated) -->
<CustomObject>
    <actionOverrides>...</actionOverrides>
    <fields>...</fields>
    <listViews>...</listViews>
    <recordTypes>...</recordTypes>
    <validationRules>...</validationRules>
    <webLinks>...</webLinks>
</CustomObject>
```

**Fix**: Retrieve the component fresh from the org to see correct ordering:
```bash
sf project retrieve start --metadata CustomObject:MyObject__c --target-org myorg
```
Then compare with your local version and fix the ordering.

**Prevention**: Use `sfdx-git-delta` or `@jayree/sfdx-plugin-prettier` to auto-format:
```bash
# Install prettier plugin for Salesforce XML
sf plugins install @jayree/sfdx-plugin-prettier
```


### 6. Unclosed or Malformed XML Tags

**Error**: `XML document structures must start and end within the same entity` or
`The element type "X" must be terminated by the matching end-tag "</X>"`.

**Cause**: Manual editing left unclosed tags, typos in tag names, or self-closing tags
where content is expected.

**Detection**:
```bash
# xmllint will catch these
xmllint --noout path/to/file.xml
```

**Common mistakes**:
```xml
<!-- WRONG: unclosed tag -->
<label>My Label

<!-- WRONG: mismatched case -->
<Label>My Label</label>

<!-- WRONG: self-closing where content expected -->
<label/>  <!-- should be <label>Something</label> -->

<!-- CORRECT -->
<label>My Label</label>
```


### 7. Duplicate Members / Components

**Error**: `Duplicate value found: <X> duplicates value on record with id: <Y>` or
`A component with the same name already exists`.

**Cause**: Same component defined twice in package.xml, or two files map to the same
metadata component.

**Detection**:
```bash
# Check for duplicate members in package.xml
grep '<members>' manifest/package.xml | sort | uniq -d

# Check for duplicate file names (case-insensitive, matters on Mac/Windows)
find force-app -name "*-meta.xml" | xargs -I{} basename {} | sort -f | uniq -di
```


### 8. Missing Required Fields in Metadata

**Error**: `Required field is missing: [fieldName]` or `Field is required`.

**Common missing fields by metadata type**:

| Metadata Type | Required Fields |
|--------------|-----------------|
| CustomField | fullName, label, type |
| CustomObject | fullName, label, pluralLabel, nameField, sharingModel, deploymentStatus |
| PermissionSet | label |
| Profile | (depends on what's included) |
| Flow | label, processType, status |
| ValidationRule | active, errorConditionFormula, errorMessage |
| ApexClass (-meta.xml) | apiVersion, status |
| LWC (.js-meta.xml) | apiVersion, isExposed |


### 9. Invalid package.xml Structure

**Error**: Various errors during retrieve/deploy, or components silently skipped.

**Correct package.xml structure**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>MyClass</members>
        <members>AnotherClass</members>
        <name>ApexClass</name>
    </types>
    <types>
        <members>MyObject__c</members>
        <name>CustomObject</name>
    </types>
    <version>66.0</version>
</Package>
```

**Common mistakes**:
```xml
<!-- WRONG: <name> before <members> — technically works but bad practice -->
<!-- WRONG: missing <version> — defaults to old API version -->
<!-- WRONG: using <n> instead of <name> — seen in some docs/examples (typo) -->
<!-- WRONG: wildcard * for types that don't support it -->
<!-- WRONG: namespace prefix missing for managed package components -->
```

**Validate your package.xml**:
```bash
# Generate a fresh package.xml from org (compare with yours)
sf project generate manifest --from-org myorg --output-dir manifest/

# Generate from local source
sf project generate manifest --source-dir force-app --output-dir manifest/
```

---

## Metadata Deployment Errors

### Dependency Errors

**Error**: `Cannot find a user that matches any of the following usernames` or
`Entity of type X named Y cannot be found`.

**Cause**: Metadata references components, users, or objects not present in the target org.

**Fix**:
1. Deploy dependencies first (custom objects before fields, fields before validation rules)
2. Use `sf project deploy start --source-dir force-app` (deploys everything together)
3. For user references: ensure the user exists in target org or remove hardcoded usernames
4. For managed package references: install the package in target org first

### Permission Errors

**Error**: `Cannot modify managed object` or `Insufficient access`.

**Cause**: Trying to modify managed package metadata or deploying as user without
sufficient permissions.

**Fix**:
1. Ensure deploying user has: Modify All Data, Author Apex, Modify Metadata
2. Don't include managed package components in your deployment
3. Use `.forceignore` to exclude managed package metadata from retrieves:
```
# .forceignore
**/managed_package_namespace__*
```

### Test Coverage Errors

**Error**: `Code Coverage Failure` or `Average test coverage across all Apex Classes and
Triggers is X%, at least 75% test coverage is required`.

**Fix**: This is not a metadata syntax error — you need more/better test coverage.
See `references/apex-patterns.md` for testing best practices.

### Profile / Permission Set Deployment Issues

**Error**: `You can't edit tab settings for X, as it's not a valid tab` or
`Field X is not available for this permission set`.

**Cause**: Profile/PermissionSet XML references tabs, objects, or fields that don't exist
in the target org (different editions, missing packages, etc.).

**Fix**:
1. Retrieve profiles fresh from target org, merge your changes
2. Use permission sets instead of profiles (more portable)
3. Remove references to components that don't exist in target
4. Use `.forceignore` for problematic profile sections

### Layout Errors

**Error**: `Fields in highlights panel must be a subset of fields in detail section` or
`Invalid field: X in related list: Y`.

**Cause**: Layout references fields that don't exist, or highlight panel contains fields
not in the detail section.

**Fix**: Edit the layout XML to remove invalid field references, or deploy the missing
fields first.

### Flow Deployment Errors

**Error**: `The version of the flow you're updating is active. To update it, you need
to deactivate it first` or `Invalid condition logic`.

**Fix**:
1. For active flow updates: SF allows deploying a new version (the old version stays active)
2. Invalid condition logic: check `conditionLogic` element matches the number of `conditions`
3. Missing references: ensure all referenced objects/fields/actions exist in target

---

## VS Code XML Validation Issues

### False Positive Errors in Problems Panel

**Error**: `cvc-elt.1.a: Cannot find the declaration of element 'CustomApplication'`
(and similar for all metadata types).

**Cause**: The `vscode-xml` extension (RedHat XML / Lemminx) tries to validate against
XSD schemas but can't find the Salesforce metadata schema.

**Fix options** (add to VS Code `settings.json`):

```json
{
  // Option 1: Disable validation for all -meta.xml files (RECOMMENDED)
  "xml.validation.filters": [
    {
      "pattern": "**/*-meta.xml",
      "enabled": false
    },
    {
      "pattern": "**/package.xml",
      "enabled": false
    }
  ],

  // Option 2: Only validate when explicit schema is present
  "xml.validation.schema.enabled": "onValidSchema",

  // Also useful: prevent BOM issues
  "files.encoding": "utf8"
}
```

### Salesforce Extension Pack Issues

If the Salesforce Extension Pack itself shows errors:
```bash
# Clear the Salesforce extension cache
# On Mac/Linux:
rm -rf ~/.sfdx/tools/
# On Windows:
rmdir /s /q %USERPROFILE%\.sfdx\tools

# Restart VS Code, then re-authorize:
sf org login web --alias myorg
```

---

## Git & Source Control Metadata Issues

### Whitespace / Line Ending Problems

**Cause**: Different developers on different OS (Windows CRLF vs Unix LF) or editors
adding trailing whitespace, causing noisy diffs and merge conflicts.

**Fix**: Add `.gitattributes` to your project root:
```
# .gitattributes - normalize line endings
* text=auto eol=lf
*.xml text eol=lf
*.cls text eol=lf
*.trigger text eol=lf
*.js text eol=lf
*.html text eol=lf
*.css text eol=lf
*.json text eol=lf

# Binary files
*.png binary
*.jpg binary
*.gif binary
*.zip binary
*.resource-meta.xml binary
```

### Namespace Pollution in Retrieved XML

**Cause**: After manual edits, `sf project retrieve` may add `xmlns` attributes to
child nodes that didn't have them before. This is cosmetic noise but pollutes diffs.

**Fix**: Use metadata normalizers in your CI pipeline:
```bash
# Using prettier with XML plugin
npx prettier --write "force-app/**/*.xml" --xml-whitespace-sensitivity ignore

# Or @jayree/sfdx-plugin-prettier
sf jayree manifest cleanup
```

### Merge Conflicts in Metadata XML

**Prevention**: Decompose large metadata types:
- Profiles: use `@rdietrick/sfdx-profile-decompose` or switch to permission sets
- Custom Objects: SF CLI source format already decomposes fields, but layouts and
  profiles can still conflict
- Permission Sets: decompose into smaller, purpose-specific sets

### .forceignore Best Practices

Create a `.forceignore` to exclude noisy/undeployable metadata:
```
# .forceignore

# Managed package components
**/lwc/managed__*
**/aura/managed__*

# Standard value sets that cause issues
**/standardValueSets/

# Profiles (deploy via permission sets instead)
**/profiles/

# Settings that differ per org
**/settings/

# Scratch org specific
**/connectedApps/

# VS Code / IDE files
.vscode/
.sfdx/

# OS files
.DS_Store
Thumbs.db
```

---

## Automated Validation in CI/CD

### Pre-Deploy Validation Script

Add this to your CI/CD pipeline (GitHub Actions, GitLab CI, etc.):

```bash
#!/bin/bash
# validate-metadata.sh — Run before every deployment

set -e

echo "=== Step 1: XML Well-Formedness Check ==="
ERRORS=0
while IFS= read -r -d '' file; do
    if ! xmllint --noout "$file" 2>/dev/null; then
        echo "MALFORMED XML: $file"
        xmllint --noout "$file" 2>&1 | head -5
        ERRORS=$((ERRORS + 1))
    fi
done < <(find force-app -name "*.xml" -print0)

if [ $ERRORS -gt 0 ]; then
    echo "Found $ERRORS malformed XML files. Fix before deploying."
    exit 1
fi
echo "All XML files well-formed."

echo ""
echo "=== Step 2: Check for BOM Characters ==="
BOM_FILES=$(grep -rl $'\xEF\xBB\xBF' force-app --include="*.xml" 2>/dev/null || true)
if [ -n "$BOM_FILES" ]; then
    echo "WARNING: BOM characters found in:"
    echo "$BOM_FILES"
    echo "Run: find force-app -name '*.xml' -exec sed -i '1s/^\xEF\xBB\xBF//' {} \;"
    exit 1
fi
echo "No BOM characters found."

echo ""
echo "=== Step 3: Check API Version Consistency ==="
VERSIONS=$(grep -rh "<apiVersion>" force-app --include="*.xml" | sort -u)
VERSION_COUNT=$(echo "$VERSIONS" | wc -l)
if [ "$VERSION_COUNT" -gt 1 ]; then
    echo "WARNING: Multiple API versions found:"
    echo "$VERSIONS"
    echo "Consider standardizing to a single version."
fi

echo ""
echo "=== Step 4: Validate Package XML ==="
if [ -f manifest/package.xml ]; then
    xmllint --noout manifest/package.xml
    # Check for duplicate members
    DUPES=$(grep '<members>' manifest/package.xml | sort | uniq -d)
    if [ -n "$DUPES" ]; then
        echo "WARNING: Duplicate members in package.xml:"
        echo "$DUPES"
    fi
fi

echo ""
echo "=== Step 5: Dry-Run Deploy ==="
sf project deploy start --source-dir force-app --dry-run --target-org "$TARGET_ORG" --wait 30

echo ""
echo "=== All Validations Passed ==="
```

### GitHub Actions Example

```yaml
# .github/workflows/validate.yml
name: Validate Metadata
on: [pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install SF CLI
        run: npm install --global @salesforce/cli

      - name: Install xmllint
        run: sudo apt-get install -y libxml2-utils

      - name: XML Well-Formedness Check
        run: |
          find force-app -name "*.xml" -exec xmllint --noout {} \;

      - name: BOM Check
        run: |
          if grep -rl $'\xEF\xBB\xBF' force-app --include="*.xml" 2>/dev/null; then
            echo "::error::BOM characters found in XML files"
            exit 1
          fi

      - name: Auth to Target Org
        run: |
          echo "${{ secrets.SF_AUTH_URL }}" > auth.txt
          sf org login sfdx-url --sfdx-url-file auth.txt --alias target-org

      - name: Validate Deployment
        run: |
          sf project deploy start --source-dir force-app \
            --dry-run --target-org target-org --wait 30
```

---

## Reporting Bugs

> **SECURITY WARNING — NEVER include any of the following in bug reports, issues, or PRs:**
>
> - API tokens, access tokens, refresh tokens, session IDs
> - Passwords, security questions, secret answers
> - OAuth client secrets, consumer secrets, JWT private keys
> - Salesforce org IDs, usernames with real domains, auth URLs (`force://...`)
> - Named Credential secrets, External Credential secrets
> - Any value from `sf org display --verbose` (contains access tokens!)
> - Real customer/employee data (PII, email addresses, phone numbers)
>
> **Always sanitize** error output, logs, and metadata before posting. Replace real
> values with placeholders like `<REDACTED>`, `myuser@example.com`, or `xxxx-xxxx`.
> GitHub issues are **public** — anything you post is visible to the entire internet.
> If you accidentally post a secret, **rotate it immediately** (revoke the token,
> change the password, regenerate the key) — editing/deleting the post is not enough
> as Git history and caches preserve the original content.

### Skill Issues vs Salesforce Platform Issues

**If the issue is with this skill** (wrong guidance, missing metadata type, outdated info,
suggested fix doesn't work, missing error pattern):

| Report To | URL |
|-----------|-----|
| **This skill's repo** | **github.com/JourneyBuilders/Claude-Salesforce-Developer-Skill** |

Open an issue with: what you were trying to do, what the skill suggested, what actually
happened, and (if possible) the correct solution. PRs with fixes are welcome!

**If the issue is with Salesforce itself** (CLI bug, Metadata API bug, VS Code extension):

### Where to Report Salesforce Bugs

| Issue Type | Repository | URL |
|-----------|-----------|-----|
| **SF CLI bugs** (deploy, retrieve, org commands) | forcedotcom/cli | github.com/forcedotcom/cli/issues |
| **VS Code Extension bugs** | forcedotcom/salesforcedx-vscode | github.com/forcedotcom/salesforcedx-vscode/issues |
| **Source Deploy/Retrieve library** (metadata registry, type resolution) | forcedotcom/source-deploy-retrieve | github.com/forcedotcom/source-deploy-retrieve/issues |
| **Metadata API bugs** (server-side Salesforce behavior) | Salesforce Known Issues | issues.salesforce.com |
| **LWC framework bugs** | nicholasglesmann/sfdc-lwc-lightning-datatable (community) or salesforce/lwc | github.com/nicholasglesmann/sfdc-lwc-lightning-datatable/issues |
| **Apex language bugs** | Salesforce Known Issues | issues.salesforce.com |
| **Metadata registry** (missing type, wrong mapping) | forcedotcom/source-deploy-retrieve | Check: github.com/forcedotcom/source-deploy-retrieve/blob/main/src/registry/metadataRegistry.json |

### How to Write a Good Bug Report

Follow this template for `forcedotcom/cli` issues:

```markdown
### Summary
Brief description of the bug.

### Steps To Reproduce
1. Create project with `sf project generate --name repro`
2. Add the following metadata file: [attach or show content]
3. Run: `sf project deploy start --source-dir force-app --target-org myorg`
4. Observe error: [paste full error output]

### Expected Result
What should have happened.

### Actual Result
What actually happened. Include the full error message.

### System Information
[Paste output of: `sf version --verbose --json`]
⚠️ DO NOT paste output of `sf org display` — it contains access tokens!

### Additional Context
- Org edition: Developer / Enterprise / etc.
- API version in sfdx-project.json: 66.0
- Does it reproduce in a fresh scratch org? Yes/No
- Did it work in a previous CLI version? If so, which one?

### Workaround
If you found one, share it to help others.
```

### Before Reporting

1. **Update CLI**: `npm update --global @salesforce/cli` — your bug may already be fixed
2. **Run doctor**: `sf doctor` — catches common config issues
3. **Search existing issues**: Check both open AND closed issues on the relevant repo
4. **Check Known Issues**: issues.salesforce.com for server-side Metadata API bugs
5. **Check Metadata Coverage Report**: developer.salesforce.com/docs/metadata-coverage —
   confirms if the metadata type is even supported
6. **Reproduce minimally**: Create the smallest possible project that reproduces the issue
7. **Include `sf version --verbose --json`**: Always include your full version info
8. **Scrub sensitive data**: Remove ALL tokens, passwords, secrets, org IDs, real usernames,
   and customer data from error output, logs, and metadata before posting. Use
   `sf org display` output for your own debugging only — NEVER post it publicly

### Other Reporting Channels

- **Salesforce Trailblazer Community**: trailhead.salesforce.com/trailblazer-community
  (for general questions, Salesforce staff monitors these)
- **Salesforce Stack Exchange**: salesforce.stackexchange.com (community Q&A)
- **Salesforce Known Issues**: issues.salesforce.com (for platform/Metadata API bugs,
  requires Salesforce login to report — this is the official channel for non-CLI issues)
- **Partner Community**: For ISV/Partner-specific issues
- **Salesforce Support**: For critical production issues (requires support contract)

### Contributing Fixes

**To this skill** (wrong guidance, missing errors, outdated patterns):
```bash
# Fork and clone the skill repo
git clone https://github.com/JourneyBuilders/Claude-Salesforce-Developer-Skill.git
# Edit reference files, submit PR
```

**To Salesforce CLI and tools** (open source, accepts PRs):

```bash
# Fork and clone the CLI repo
git clone https://github.com/forcedotcom/cli.git

# For metadata registry issues (missing types, wrong mappings):
git clone https://github.com/forcedotcom/source-deploy-retrieve.git
# Edit: src/registry/metadataRegistry.json
# Submit PR with: description of the issue, the fix, test evidence
```

---

## Quick Troubleshooting Checklist

When a deployment fails, work through this checklist:

1. **Is the XML well-formed?** → `xmllint --noout <file>`
2. **Correct namespace?** → Must be `http://soap.sforce.com/2006/04/metadata`
3. **BOM characters?** → `grep -rl $'\xEF\xBB\xBF' force-app --include="*.xml"`
4. **API version correct?** → Check `<apiVersion>` in all `-meta.xml` files
5. **Element order correct?** → Retrieve fresh copy, compare ordering
6. **Dependencies present?** → All referenced objects/fields/classes in the deployment?
7. **Target org compatible?** → Right edition, packages installed, features enabled?
8. **User permissions sufficient?** → Modify All Data, Author Apex, Modify Metadata
9. **Known Salesforce issue?** → Check issues.salesforce.com
10. **CLI up to date?** → `sf update` or `npm update --global @salesforce/cli`
