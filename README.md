# Raptor - Facets Control Plane CLI

A kubectl-style CLI tool for managing Facets Control Plane resources, infrastructure, and deployments.

## Installation

### Download Pre-built Binaries

Download the latest release for your platform from [GitHub Releases](https://github.com/Facets-cloud/raptor-releases/releases):

**Linux:**
```bash
# AMD64
wget https://github.com/Facets-cloud/raptor-releases/releases/latest/download/raptor-linux-amd64
chmod +x raptor-linux-amd64
sudo mv raptor-linux-amd64 /usr/local/bin/raptor

# ARM64
wget https://github.com/Facets-cloud/raptor-releases/releases/latest/download/raptor-linux-arm64
chmod +x raptor-linux-arm64
sudo mv raptor-linux-arm64 /usr/local/bin/raptor
```

**macOS:**
```bash
# Intel (AMD64)
curl -LO https://github.com/Facets-cloud/raptor-releases/releases/latest/download/raptor-darwin-amd64
chmod +x raptor-darwin-amd64
sudo mv raptor-darwin-amd64 /usr/local/bin/raptor

# Apple Silicon (ARM64)
curl -LO https://github.com/Facets-cloud/raptor-releases/releases/latest/download/raptor-darwin-arm64
chmod +x raptor-darwin-arm64
sudo mv raptor-darwin-arm64 /usr/local/bin/raptor
```

**Windows:**
Download `raptor-windows-amd64.exe` or `raptor-windows-arm64.exe` from the releases page and add to your PATH.

### Build from Source

```bash
git clone https://github.com/Facets-cloud/raptor.git
cd raptor
go build -o raptor
sudo mv raptor /usr/local/bin/
```

## Quick Start

1. **View comprehensive help:**
   ```bash
   raptor --help
   ```

2. **List your projects:**
   ```bash
   raptor get projects
   ```

## Configuration

### Authentication

Raptor supports two authentication methods (in priority order):

#### 1. Environment Variables (Highest Priority)

Set these three environment variables for direct authentication:

```bash
export FACETS_USERNAME=user@example.com
export FACETS_TOKEN=your-api-token
export CONTROL_PLANE_URL=https://your-org.console.facets.cloud
```

This method is ideal for CI/CD pipelines and automated workflows.

#### 2. Profile-based Authentication

Use the interactive login command:

```bash
raptor login
```

This will:
1. Prompt for your Control Plane URL
2. Open your browser to generate a token
3. Prompt for your username and token
4. Save credentials to `~/.facets/credentials`

Alternatively, manually create the credentials file at `~/.facets/credentials`:

```ini
[default]
control_plane_url = https://your-org.console.facets.cloud
username = user@example.com
token = your-api-token

[production]
control_plane_url = https://prod.console.facets.cloud
username = user@example.com
token = prod-token
```

### Profile Selection

Set the profile using the `FACETS_PROFILE` environment variable (defaults to `default` if not set):

```bash
export FACETS_PROFILE=production
# Or use it inline with commands
FACETS_PROFILE=production raptor get projects
```

**Note:** Environment variables (`FACETS_USERNAME`, `FACETS_TOKEN`, `CONTROL_PLANE_URL`) always take precedence over profile-based authentication.

### Read-Only Mode

Control-plane administrators can put the Raptor CLI into read-only mode by enabling the `RAPTOR_READ_ONLY` setting (Settings → Control Plane in the UI, or via the settings API). While enabled:

- All write operations (POST/PUT/DELETE) are blocked with a clear error before any request is sent — this includes `create`, `apply`, `set`, `update`, `delete`, `publish`, `plan`, and release commands.
- Read commands (`get`, `describe`, `logs`, downloads) work normally and pay no extra latency — the setting is only checked when a write is attempted.
- The check fails open: if the setting can't be fetched (e.g. an older control plane that doesn't have it), writes proceed as usual.

For emergencies, setting `RAPTOR_BYPASS_READ_ONLY=true` in the environment skips the client-side check entirely. Note that this is an advisory client-side guard, not a server-side security control.

## Core Commands

### Blueprint Management (Resources)

```bash
# List available resource types for your project
raptor get resource-types -t <project-type>

# Get resource type schema
raptor get resource-type-schema service/k8s/0.2

# Get required inputs for a resource type
raptor get resource-type-inputs service/k8s/0.2

# List resources in a project
raptor get resources -p myproject

# Get a specific resource
raptor get resources -p myproject service/api -o json

# Apply a resource from file
raptor apply -f my-service.json -p myproject

# Set resource inputs (local file operation)
raptor set resource-inputs -f my-service.json \
  --input-name database \
  --resource postgres/main-db \
  --output-name default

# Delete resources
raptor delete resource -p myproject service/old-api
```

### Release Management (Deployments)

```bash
# List releases in an environment
raptor get releases -p myproject -e dev

# Create a release (deploy)
raptor create release -p myproject -e dev -w -m "Deploy new feature"

# Create a plan (dry-run)
raptor create release -p myproject -e dev --plan -w

# Selective release (hotfix)
raptor create release -p myproject -e dev --target service/api -w

# View release logs
raptor logs release -p myproject -e dev -f <RELEASE_ID>
```

### Environment Management

```bash
# List environments in a project
raptor get environments -p myproject

# Check resource deployment status
raptor get resource-status -p myproject -e dev service/api

# Get/set environment-specific overrides
raptor get resource-overrides -p myproject -e dev service/api
raptor set resource-overrides -p myproject -e dev -f overrides.json service/api

# List revision history for an override
raptor get overrides -p myproject -e dev service/api --history

# Inspect one revision
raptor get overrides -p myproject -e dev service/api --history --version 3

# Diff latest two revisions
raptor get overrides -p myproject -e dev service/api --history --diff

# Diff an explicit range
raptor get overrides -p myproject -e dev service/api --history --diff v2..v5

# Get runtime outputs from deployed resources
raptor get resource-outputs -p myproject -e dev service/api

# Get kubeconfig for Kubernetes environments
raptor get kubeconfig -p myproject -e dev
```

### Tekton Actions

Tekton Actions are pre-defined operations (e.g. `rollout-restart-deployment`, `scale-deployment`) that can be triggered against deployed resources. The Raptor CLI exposes the five `/cc-ui/v1/actions` endpoints.

#### Listing actions and their runs

```bash
# List the actions available for a resource in an environment
raptor get actions service/api -p myproject -e dev

# With full details (resource type, cloud-action flag, etc.)
raptor get actions service/api -p myproject -e dev -o wide

# List the run history for a resource (optionally filter by action name or displayName)
raptor get action-runs service/api -p myproject -e dev
raptor get action-runs service/api -p myproject -e dev -a rollout-restart-deployment
```

#### Triggering an action

```bash
# Simple — one action, no params
raptor trigger action service/api -p myproject -e dev -a rollout-restart-deployment

# String + array params, wait for completion (exits non-zero on FAILED/CANCELLED)
raptor trigger action service/api -p myproject -e dev \
  -a deploy --param branch=main --array-param targets=dev,staging -w

# Bulk: submit multiple runs in one call
raptor trigger action -p myproject -e dev -f runs.yaml
```

`-w / --wait` polls the run until it reaches a terminal status (`SUCCEEDED`, `FAILED`, or `CANCELLED`), printing status transitions to stderr. Exit code is `0` on SUCCEEDED and `1` otherwise. `--wait` is single-action only — for bulk submissions, poll runs individually with `raptor get action-runs`.

**Bulk file format** (YAML or JSON):

```yaml
runs:
  - resource: service/api
    action: deploy
    params:
      branch: main                  # scalar → string
      replicas: "3"                 # quoted scalar → string
      targets: [dev, staging]       # list → array
  - resource: service/worker
    action: restart
    params: {}
```

`--param` and `--array-param` are mutually exclusive with `--file`. Mix `--param` (strings) and `--array-param` (arrays) freely on the CLI; reach for `-f` when you need to submit multiple actions in one call.

> **Note** — `--param VALUE` no longer comma-splits. Commas in a string value are preserved verbatim: `--param csv=a,b,c` now sends the single string `"a,b,c"`. Use `--array-param` for arrays. Prior CLI versions silently split commas; scripts that relied on that need to migrate to `--array-param`.

If the resolved action declares a parameter schema, supplied params are validated against it before submission — unknown names and type mismatches (string vs. array) are rejected client-side with a clear error. Actions with no declared params pass through unchanged.

`--wait` has a default timeout of 30 minutes (`--timeout`); set `--timeout 0` to disable.

#### Tailing live logs

```bash
# Stream step-by-step logs for a specific action run
raptor get action-run-logs service/api <RUN_NAME> -p myproject -e dev -a deploy -f
```

After every successful `raptor trigger action`, the command emits a `→ Tail logs: ...` hint pointing at the exact `get action-run-logs` invocation you'd need for the runs it just submitted.

#### Downloading an artifact produced by a run

If an action run produces an output file, it's exposed via the upload endpoint
(server-side REST resource name) and downloaded by `get action-run-download`.

```bash
# Default: write ./<original-filename> using the server-provided filename
raptor get action-run-download service/api <RUN_NAME> -p myproject -e dev -a deploy

# Save into a directory
raptor get action-run-download service/api <RUN_NAME> --save-to ./out/ \
  -p myproject -e dev -a deploy

# Save with a specific name; --overwrite to clobber an existing file
raptor get action-run-download service/api <RUN_NAME> --save-to ./bundle.tgz --overwrite \
  -p myproject -e dev -a deploy
```

The command short-circuits with a clear error when the run has `hasUpload=false`, avoiding a confusing server 404.

`action-run-upload` is kept as an alias for backwards compatibility.

### Authorization & Access Control

```bash
# Show complete access overview (all permissions and resources)
raptor auth can-i

# Check if you have a specific global permission
raptor auth can-i RELEASE_FULL

# Check permission for a specific environment
raptor auth can-i RELEASE_FULL --environment prod --project myproject

# Show permission matrix across all environments in a project
raptor auth can-i --project myproject --matrix
```

### Secrets & Variables Management

```bash
# List all variables and secrets
raptor get variables -p myproject

# Filter by type (variables only or secrets only)
raptor get variables -p myproject --type variable
raptor get variables -p myproject --type secret

# List with environment-specific values
raptor get variables -p myproject -e production

# Show actual secret values (requires -e and VIEW_SECRETS permission)
raptor get variables -p myproject -e production --show-secrets

# Get specific variable across all environments
raptor get variable API_ENDPOINT -p myproject
raptor get variable DB_PASSWORD -p myproject --show-secrets

# Create a variable
raptor create variable API_URL --default https://api.example.com -p myproject

# Create a secret (secrets never have a stack default — values are set per
# environment only, even if the value should be the same in every environment)
raptor create variable DB_PASSWORD -p myproject --secret \
  --env-values prod=prodSecret123,staging=stagingSecret456

# Create with description and global flag
raptor create variable APP_NAME --default MyApp -p myproject \
  --description "Application name" --global

# Create with environment-specific values
raptor create variable API_URL --default https://api.example.com -p myproject \
  --env-values prod=https://api.prod.com,staging=https://api.staging.com

# Bulk create from JSON/YAML file
raptor create variable -p myproject -f variables.json

# Update a variable (--value updates the stack default; not allowed for secrets)
raptor set variable API_URL --value https://api.v2.com -p myproject

# Update description
raptor set variable API_URL --description "New API" -p myproject

# Update environment-specific values (the only way to set secret values)
raptor set variable DB_HOST --env-values prod=db.prod.com -p myproject
raptor set variable DB_PASSWORD --env-values prod=newSecret -p myproject

# Bulk update from file
raptor set variables -p myproject -f updates.json

# Delete a variable
raptor delete variable API_ENDPOINT -p myproject

# Delete multiple variables
raptor delete variable OLD_VAR UNUSED_CONFIG LEGACY_VAR -p myproject

# Delete without confirmation
raptor delete variable API_KEY -p myproject --yes
```

**File format for bulk create (JSON):**
```json
[
  {
    "variableName": "API_ENDPOINT",
    "stackDefault": "https://api.example.com",
    "description": "Main API endpoint",
    "global": false,
    "secret": false,
    "envValues": {
      "production": "https://api.prod.com",
      "staging": "https://api.staging.com"
    }
  },
  {
    "variableName": "DB_PASSWORD",
    "description": "Database password (secrets cannot have stackDefault)",
    "secret": true,
    "envValues": {
      "production": "prod-secret-password",
      "staging": "staging-secret-password"
    }
  }
]
```

**File format for bulk update (JSON):**
```json
[
  {
    "variableName": "API_ENDPOINT",
    "stackDefault": "https://api.v2.example.com",
    "description": "Updated API endpoint"
  },
  {
    "variableName": "DB_HOST",
    "envValues": {
      "production": "db.prod.example.com",
      "staging": "db.staging.example.com"
    }
  }
]
```

> **Note:** Secrets never have a stack-level default value — `stackDefault` is
> rejected for secrets in both create and update files. Secret values can only
> be set per environment via `envValues`/`clusterIdToValueMap`, even if the
> value should be the same in every environment.

**Variable Status Values:**
- `DEFAULT` - Using the stack default value
- `OVERRIDDEN` - Has environment-specific override
- `NOT_SET` - No value configured for this environment
- `NO_ACCESS` - No permission to view this environment

### Resource Output Expressions

```bash
# List all available resource output expressions
raptor get resource-output-expressions -p myproject

# Get output schema for a specific type
raptor get output-schema @facets/eks
```

### IaC Module Management

```bash
# List all modules
raptor get iac-module
raptor get iac-module -o wide              # Show full details (ID, creators, age)
raptor get iac-module -o json              # JSON output

# Filter modules
raptor get iac-module --source CUSTOM      # Only custom modules
raptor get iac-module --stage PUBLISHED    # Only published modules
raptor get iac-module --type service --flavor k8s

# Download module source code (by TYPE/FLAVOR/VERSION)
raptor get iac-module service/k8s/0.2
raptor get iac-module service/k8s/0.2 --save-to ./modules/

# Download module by ID (from wide output)
raptor get iac-module 68c26fb6a96f --save-to service-k8s.zip

# View module details and usages
raptor get iac-module service/k8s/0.2 --details
raptor get iac-module service/k8s/0.2 --usages

# List publish history
raptor get iac-module service/k8s/0.2 --history

# Inspect one revision
raptor get iac-module service/k8s/0.2 --history --version 40

# Diff latest two publishes
raptor get iac-module service/k8s/0.2 --history --diff

# Diff a range and include per-file changes
raptor get iac-module service/k8s/0.2 --history --diff v38..v40 --include-files

# Upload a new module (two-step workflow - recommended)
raptor create iac-module -f ./modules/service-k8s    # Upload as PREVIEW
raptor publish iac-module service/k8s/0.3            # Publish when ready

# Upload and publish immediately (one-step)
raptor create iac-module -f ./modules/service-k8s --publish

# Upload with version override
raptor create iac-module -f ./modules/service-k8s --version 0.4

# Skip validation for testing
raptor create iac-module -f ./modules/service-k8s --skip-validation

# Delete a module
raptor delete iac-module service/k8s/0.3
```

## Key Features

### Resource File Structure

Every resource file must include:

```json
{
  "kind": "service",
  "flavor": "k8s",
  "version": "0.2",
  "disabled": false,
  "metadata": {
    "name": "my-service"
  },
  "inputs": {
    "cloud_account": {
      "resource_type": "cloud_account",
      "resource_name": "my-aws",
      "output_name": "default"
    }
  },
  "spec": {
    "env": {
      "DB_HOST": "${postgres.main-db.out.attributes.host}"
    }
  }
}
```

**Key Points:**
- **Filename becomes resource name:** `my-service.json` → resource name `my-service`, full ID: `service/my-service`
- **Inputs:** Mandatory dependencies on other resources (set using `raptor set resource-inputs`)
- **Resource Output Expressions:** `${...}` syntax for dynamic values in spec fields
- **metadata.name:** Optional override for filename-based naming

**Important:** Always check the schema first:
```bash
raptor get resource-type-schema service/k8s/0.2
raptor get resource-type-inputs service/k8s/0.2
```

### Resource Output Expressions

Use `${...}` syntax to reference runtime values from other resources:

```bash
${RESOURCE_TYPE.RESOURCE_NAME.out.ATTRIBUTE_PATH}

Examples:
  ${postgres.main-db.out.attributes.host}
  ${service.api.out.endpoint}
  ${cloud_account.my-aws.out.account_id}
```

### Validation

The `apply` command validates resources against their schema:
- ✅ Structural validation (types, required fields)
- ✅ Pattern validation (regex patterns)
- ✅ Input validation (checks referenced resources exist)
- ✅ Resource output expression validation
- ✅ Auto-skips pattern validation for `${...}` expressions

## Output Formats

- `table` (default): kubectl-style table output
- `wide`: Table with additional columns
- `json`: JSON output
- `yaml`: YAML output

```bash
raptor get projects -o json
raptor get resources -p myproject -o yaml
raptor get releases -p myproject -e dev -o wide
```

## Command Aliases

The CLI supports kubectl-style aliases:

- `projects` = `project` = `stacks` = `stack`
- `environments` = `environment` = `env` = `envs` = `clusters` = `cluster`
- `resources` = `resource` = `res`
- `releases` = `release` = `deployments` = `deployment`
- `variables` = `vars` = `var`

## Examples

### Complete Workflow: Deploy a Service

```bash
# Set authentication (choose one method):
# Method 1: Environment variables
export FACETS_USERNAME=user@example.com
export FACETS_TOKEN=your-api-token
export CONTROL_PLANE_URL=https://your-org.console.facets.cloud

# Method 2: Profile (defaults to "default" if not set)
export FACETS_PROFILE=default

# 1. List your projects
raptor get projects

# 2. Find available resource types (get project type first)
raptor get project-types
raptor get resource-types -t microservices

# 3. Get the schema and required inputs
raptor get resource-type-schema service/k8s/0.2
raptor get resource-type-inputs service/k8s/0.2
raptor get resource-type-outputs service/k8s/0.2

# 4. Create your resource file (my-service.json)
# See "Resource File Structure" above

# 5. Set required inputs (repeat for each required input)
raptor set resource-inputs -f my-service.json \
  --input-name cloud_account \
  --resource cloud_account/my-aws \
  --output-name default

# 6. Get available output expressions for dynamic values
raptor get resource-output-expressions -p myproject

# 7. Apply the resource to blueprint
raptor apply -f my-service.json -p myproject

# 8. Deploy to dev environment
raptor create release -p myproject -e dev --plan -w  # Preview first
raptor create release -p myproject -e dev -w -m "Deploy my-service"

# 9. Monitor the deployment
raptor get releases -p myproject -e dev
raptor logs release -p myproject -e dev -f <RELEASE_ID>

# 10. Check deployment status
raptor get resource-status -p myproject -e dev service/my-service
```

### Working with Overrides

```bash
# Get current overrides
raptor get resource-overrides -p myproject -e dev service/api -o json > overrides.json

# Edit overrides.json
# {
#   "spec.env.LOG_LEVEL": "debug",
#   "spec.runtime.size.memory": "2Gi"
# }

# Apply overrides
raptor set resource-overrides -p myproject -e dev -f overrides.json service/api
```

## Common Commands

### Blueprint Management

```bash
# List resource types
raptor get project-types
raptor get resource-types -t <project-type>

# Get schema and metadata
raptor get resource-type-schema <type>/<flavor>/<version>
raptor get resource-type-inputs <type>/<flavor>/<version>
raptor get resource-type-outputs <type>/<flavor>/<version>
raptor get output-schema @<namespace>/<name>
raptor get iac-module <type>/<flavor>/<version> [-o <output-path>]

# Work with resources
raptor get resources -p <project>
raptor get resources -p <project> <type>/<name>
raptor apply -f <file> -p <project>
raptor delete resource -p <project> <type>/<name>

# Set inputs
raptor set resource-inputs -f <file> --input-name <name> --resource <type>/<name> --output-name <output>
```

### Environment Management

```bash
# List environments
raptor get environments -p <project>

# Resource status and overrides
raptor get resource-status -p <project> -e <env> <type>/<name>
raptor get resource-overrides -p <project> -e <env> <type>/<name>
raptor set resource-overrides -p <project> -e <env> -f <file> <type>/<name>

# Get runtime outputs
raptor get resource-outputs -p <project> -e <env> <type>/<name>

# Get kubeconfig
raptor get kubeconfig -p <project> -e <env> [-o <file>]

# Get expressions
raptor get resource-output-expressions -p <project>
```

### Release Management

```bash
# Create releases
raptor create release -p <project> -e <env> [--plan] [-w] [-m "message"]
raptor create release -p <project> -e <env> --target <type>/<name> -w

# View releases
raptor get releases -p <project> -e <env>
raptor get releases -p <project> -e <env> <release-id>

# View logs
raptor logs release -p <project> -e <env> [-f] <release-id>
```

### Variables & Secrets

```bash
# List variables and secrets
raptor get variables -p <project> [--type variable|secret] [-e <env>] [--show-secrets]

# Get specific variable across environments
raptor get variable <variable-name> -p <project> [--show-secrets]

# Create variable or secret
# (--default is variables-only; secrets are set per environment via --env-values)
raptor create variable <name> -p <project> [--default "..."] [--secret] [--global] [--description "..."] [--env-values ENV=VALUE,...]
raptor create variable -p <project> -f <file>  # Bulk create

# Update variable or secret
# (--value updates the stack default and is variables-only; secret values are set via --env-values)
raptor set variable <name> -p <project> [--value "..."] [--description "..."] [--global=true|false] [--env-values ENV=VALUE,...]
raptor set variables -p <project> -f <file>  # Bulk update

# Delete variable or secret
raptor delete variable <name> [<name>...] -p <project> [--yes]
```

### Authorization & Access Control

```bash
# Show complete access overview
raptor auth can-i

# Check specific permission
raptor auth can-i <PERMISSION>

# Check environment-specific permission
raptor auth can-i <PERMISSION> --environment <env> --project <project>

# Show permission matrix for project
raptor auth can-i --project <project> --matrix
```

### IaC Modules

```bash
# List and download modules
raptor get iac-module [-o wide|json|yaml]
raptor get iac-module --source CUSTOM --stage PUBLISHED
raptor get iac-module <type>/<flavor>/<version> [--save-to <path>]
raptor get iac-module <module-id> [--save-to <path>]
raptor get iac-module <type>/<flavor>/<version> --details
raptor get iac-module <type>/<flavor>/<version> --usages

# Upload and publish modules
raptor create iac-module -f <directory> [--type <type>] [--flavor <flavor>] [--version <version>]
raptor create iac-module -f <directory> --publish
raptor publish iac-module <type>/<flavor>/<version>
raptor delete iac-module <type>/<flavor>/<version>
```

