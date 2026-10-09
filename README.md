# Raptor - Facets Control Plane CLI

A kubectl-style CLI tool for managing Facets Control Plane resources, infrastructure, and deployments.

## Installation

### Install script (macOS and Linux)

```bash
curl -fsSL https://cross.facetsapp.cloud/cli/install.sh | sh -s -- raptor
```

The script downloads the release for your platform, checks its SHA-256, and puts
it in `~/.local/bin`. It does not use sudo, so a later `raptor upgrade` needs no
sudo either. If `~/.local/bin` is not on your PATH, the script adds one line to
your shell profile; open a new terminal after it. It also lists every other
raptor on your PATH.

Options (environment variables):
- `FACETS_INSTALL_DIR`: install into another folder.
- `FACETS_NO_MODIFY_PATH=1`: do not change your shell profile.
- `RAPTOR_CHANNEL=unstable`: install the newest dev build.

**An older install in `/usr/local/bin`:** on macOS, if you own the file, you
can keep it: `raptor upgrade` replaces it in place. On Linux, `raptor upgrade`
cannot replace it; it installs a new copy in `~/.local/bin` and tells you if your
PATH still runs the old one. To keep one copy, run the script, then remove the
old copy: `sudo rm /usr/local/bin/raptor`.

**Updates:** raptor upgrades itself. When a newer release exists, raptor
downloads it in the background after a command and puts it in place for the next
command. It does not do this in CI, in a container, as root, or when you cannot
write to raptor's folder; there it tells you to run `raptor upgrade`. Set
`RAPTOR_NO_AUTO_UPGRADE=1` to turn it off.

### Install script (Windows)

In PowerShell:

```powershell
$env:FACETS_CLI = 'raptor'; irm https://cross.facetsapp.cloud/cli/install.ps1 | iex
```

The script downloads `raptor.exe`, checks its SHA-256, and puts it in
`%USERPROFILE%\.local\bin` without administrator rights, so a later
`raptor upgrade` needs none either. It adds that folder to your user PATH and to
the current window. `praxis login` also installs raptor this way, so a praxis
user does not need this step.

### Download Pre-built Binaries

To install by hand, download the release for your platform from
[GitHub Releases](https://github.com/Facets-cloud/raptor-releases/releases) into a
folder that you own and that is on your PATH:

```bash
mkdir -p ~/.local/bin
OS=$(uname -s | tr '[:upper:]' '[:lower:]')                    # linux or darwin
ARCH=$(uname -m | sed 's/x86_64/amd64/; s/aarch64/arm64/')      # amd64 or arm64
curl -fL -o ~/.local/bin/raptor "https://github.com/Facets-cloud/raptor-releases/releases/latest/download/raptor-$OS-$ARCH"
chmod +x ~/.local/bin/raptor
```

If `~/.local/bin` is not on your PATH, add `export PATH="$HOME/.local/bin:$PATH"`
to your shell profile.

**Windows:** download `raptor-windows-amd64.exe` (or `-arm64.exe`) from the
releases page, save it as `%USERPROFILE%\.local\bin\raptor.exe`, and add that
folder to your user PATH.

### Build from Source

```bash
git clone https://github.com/Facets-cloud/raptor.git
cd raptor
go build -o raptor
mkdir -p ~/.local/bin
mv raptor ~/.local/bin/
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

### Read-Only AI Access

Control-plane administrators can restrict CLI/AI-driven access per user role via the role's **AI access level** (Settings → Roles in the UI, or the custom-role API). When a user's roles only grant `READ`:

- The control plane rejects all write operations (POST/PUT/DELETE) from raptor — `create`, `apply`, `set`, `update`, `delete`, `publish`, `plan`, and release commands — with a 403 and a self-explanatory message.
- Read commands (`get`, `describe`, `logs`, downloads) work normally.
- Enforcement is server-side per user role. If any of the user's roles grants `WRITE` (the default), writes are allowed.

This replaces the retired global `RAPTOR_READ_ONLY` setting (and its `RAPTOR_BYPASS_READ_ONLY` client-side escape hatch), which was advisory and enforced only in the CLI.

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

# Tag a release after it exists (see Labels below)
raptor set release-labels <RELEASE_ID> -p myproject -e dev --label stable
```

### Labels

Labels are reusable tags a release can carry. Manage the catalog, then put
labels on a release with `set release-labels`.

```bash
# List the release labels
raptor get labels
raptor get labels -o json

# Create one (colour optional; without --color the platform default is used)
raptor create label stable
raptor create label hotfix --color "#FF5733"

# Delete one — the control plane also removes it from every release carrying it
raptor delete label stale-tag

# Put labels on a release. This REPLACES its whole label set
raptor set release-labels 6a86f8c635cd953fd938b27c -p myproject -e dev -l stable
raptor set release-labels 6a86f8c6... -p myproject -e dev -l stable -l "TICKET-421"

# Remove every label from a release
raptor set release-labels 6a86f8c6... -p myproject -e dev --clear
```

To tag from a script, give the trigger a trace id you already know, then use it
to address the release afterwards — no output parsing:

```bash
TRACE=$(uuidgen)
raptor create release -p myproject -e dev --trace-id "$TRACE" -w
raptor set release-labels -p myproject -e dev --trace-id "$TRACE" -l "$TICKET"
```

`raptor plan` and `raptor apply plan` take `--trace-id` the same way.

Notes:
- A label name that does not exist is created automatically, with the platform's
  default colour.
- Names are matched **case-sensitively**, and may be at most 100 characters —
  both match the control plane.
- Labels cannot be attached while a release is being created; no release
  endpoint accepts one. Label the release once it exists.

### Environment Management

```bash
# List environments in a project
raptor get environments -p myproject

# Check whether a resource has changes pending release (same rule as the UI badge)
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
# (bound to the <user>-raptor service account; access = the role's AI permissions)
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

Bulk create accepts these same fields in YAML. `envValues` uses environment
names; `clusterIdToValueMap` accepts IDs (and legacy names). Raptor resolves
names once and sends one bulk POST. The entire batch must pass strict validation;
unknown fields/targets, duplicates, nulls, conflicting assignments and empty env
values fail before writes. Put per-item settings in the file, not single-variable
flags. Values must be strings; quote numeric/boolean-looking YAML values.

The server applies environment values separately and can return success after
partial failures. Verify persistence before retrying. See
[bulk-create behavior](docs/variables-and-secrets.md#bulk-create-with-environment-values).

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

### Notifications (Channels & Subscriptions)

Notification **channels** (delivery destinations) and **subscriptions** (which
notification types route to which channel) are control-plane-global. Manage them with
`create`/`set`/`delete`/`get`; a single entity is referenced by ID or unique name. Common cases use flags, full/complex
payloads use `-f` JSON/YAML. Subscriptions are scoped to a project via `-p` (which
becomes the required `STACK_NAME` filter, except for `AUDIT_LOG`).

```bash
# List / get channels and subscriptions
raptor get channels
raptor get channels -o wide
raptor get channels 64a1b2c3d4e5f6 -o yaml
raptor get subscriptions

# Create a channel (fields depend on --type)
raptor create channel oncall --type SLACK --address https://hooks.slack.com/services/XXX
raptor create channel alerts --type EMAIL --emails a@example.com,b@example.com
raptor create channel pd --type PAGER_DUTY --integration-key 0123456789abcdef
raptor create channel -f channel.yaml

# Update a channel (flags merge with current state; -f is a full replace)
raptor set channel 64a1b2c3d4e5f6 --address https://hooks.slack.com/services/NEW
raptor set channel 64a1b2c3d4e5f6 -f channel.yaml

# Delete one or more channels
raptor delete channel 64a1b2c3d4e5f6 74b2c3d4e5f6a7 --yes

# Discover notification types and the filter keys each type supports
raptor get notification-types
raptor get notification-tags --type APPLICATION_DEPLOYMENT_COMPLETE
raptor get notification-attributes --type APP_DEPLOYMENT   # webhook payload {{fields}} for --payload-json

# Create a subscription: -p sets the required STACK_NAME filter; channel by id or name
raptor create subscription deploys -p infra-dev --type APPLICATION_DEPLOYMENT_COMPLETE \
  --channel-name oncall --application my-app --environment production \
  --filter DEPLOYMENT_STATUS=SUCCEEDED
raptor create subscription -f subscription.yaml

# Update (read-modify-write: each flag changes only that filter) / delete
raptor set subscription 64a1b2c3d4e5f6 --environment production,staging
raptor delete subscription 64a1b2c3d4e5f6 --yes

# Send a test notification — by id/name (resolve a saved entity)…
raptor test channel oncall
raptor test subscription deploys
# …or ad-hoc, to test a config before creating it (mirrors the UI's create-dialog button)
raptor test channel --type MS_TEAMS_WORKFLOW --address https://...
raptor test subscription --channel-name oncall --type ALERT
```

**Channel types:** `SLACK`, `WEBHOOK`, `MS_TEAMS`, `MS_TEAMS_WORKFLOW` (use `--address`);
`EMAIL` (use `--emails`); `PAGER_DUTY`, `ZEN_DUTY` (use `--integration-key`).

**Subscription filters** are dynamic per notification type — list the valid keys with
`raptor get notification-tags --type <TYPE>` and pass any of them via `--filter KEY=value`
(e.g. `DEPLOYMENT_STATUS`, `RELEASE_TYPE`, `AUDIT_ACTION`, `ENVIRONMENT_TEARDOWN_NOTIFY_BEFORE`).
Repeat `--filter` or pass the whole set in one quoted flag, `;`-separated
(`--filter "CLUSTER_NAME=prod;SEVERITY=critical,warning"`); within a key, `,` sets multiple
values. The ubiquitous ones also have convenience flags: `--environment` (`CLUSTER_NAME`),
`--application` (`APPLICATION_NAME`), `--release-stream` (`CLUSTER_TYPE`). Filter values are
plain names (env/app/release-stream names), not IDs.

### Resource Output Expressions

```bash
# List all available resource output expressions in a project
raptor describe expressions -p myproject

# Filter to a kind (or a specific resource), or include a pending file
raptor describe expressions -p myproject postgres/main-db
raptor describe expressions -p myproject --kind postgres
raptor describe expressions -p myproject -f pending-resource.json

# Find candidate references for a concrete value in an environment
raptor describe expressions -p myproject -e stage postgres/main-db --value 'db.internal'

# Restrict to exact names within a type; repeat --value to share reads
raptor describe expressions -p myproject -e stage --kind postgres --names db,events \
  --value 'db.internal' --value 'events.internal' -o json

# Get output schema for a specific type
raptor get output-schema @facets/eks
```

Reverse value lookup requires `-p`, `-e`, and at least one resource filter.
Matches are exact public strings from stored runtime outputs; results identify
candidate expressions without echoing query values. Public output indexes are
cached for five minutes (unavailable outputs for 30 seconds). Use `--refresh`
to refresh selected records or `--no-cache` to bypass disk access. Results show
API calls, observation times and partial failures. See
[reverse value lookup](docs/2026-09-14-reverse-value-lookup.md).

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

# First publication for this type/flavor (two-step workflow)
raptor create iac-module -f ./modules/service-k8s    # Upload as PREVIEW
raptor publish iac-module service/k8s/0.3            # First publication when ready

# Upload and publish for the first time (one-step)
raptor create iac-module -f ./modules/service-k8s --publish

# Compatible update: keep the published contract version
raptor publish iac-module service/k8s/0.3 --backward-compatible yes

# Breaking update: upload and publish a version greater than all published versions
raptor create iac-module -f ./modules/service-k8s --version 0.4 --publish --versionupgrade

# Upload with version override
raptor create iac-module -f ./modules/service-k8s --version 0.4

# Skip validation for testing
raptor create iac-module -f ./modules/service-k8s --skip-validation

# Delete a module
raptor delete iac-module service/k8s/0.3
```

Publication is non-interactive and requires a nonempty root `README.md` in the
uploaded PREVIEW. Compatible updates require `--backward-compatible yes` on the
same published version; breaking updates require a greater version and
`--versionupgrade`. First publication uses neither flag, and the flags cannot
be combined. `--skip-validation` skips local upload checks only. If upload
succeeds but publication fails, the command returns nonzero and reports the
PREVIEW for recovery. See [publish guards](docs/2026-09-14-module-publish-guards.md).

### Audit Logs

Query Control Plane audit/activity logs (who changed what, when). Filtering is
server-side. `--project`, `--environment`, `--user`, and `--target` are
regex/substring matches; `--entity`, `--action`, and `--source` accept multiple
values (repeat the flag or comma-separate). Defaults to the last 7 days, newest
first, capped at 50 events.

`--limit` auto-paginates to collect up to that many events; `--page`/`--size`
switch to manual single-page mode and cannot be combined with `--limit`.
`--user` and `--performed-by` are aliases — use one or the other.

```bash
# Most recent 50 events (last 7 days)
raptor get audit-logs                    # aliases: raptor get activity / raptor get audit

# Everything a user did in the last 24h
raptor get audit-logs --since 24h --user alice@example.com

# Blueprint + module changes in a project, deeper history
raptor get audit-logs -p my-project --entity BLUEPRINT,MODULE --limit 200

# A specific action type
raptor get audit-logs --action SECRETS_VARIABLES_UPDATE

# Explicit window, JSON for scripting
raptor get audit-logs --start 2026-06-01 --end 2026-06-07 -o json

# Everything in the window (auto-paginates; hard safety cap of 1000 pages)
raptor get audit-logs --since 30d --limit 0

# Wide table with project, environment, target, source, id
raptor get audit-logs -o wide
```

Flags: `--since` (default `7d`), `--start`/`--end`, `-p/--project`,
`-e/--environment`, `--user` / `--performed-by` (mutually exclusive aliases),
`--target`, `--entity`, `--action`, `--source`, `--limit` (default 50; 0 = all
in window; auto-paginates, capped at 1000 pages), `--page`/`--size` (manual
single-page mode, mutually exclusive with `--limit`), `-o table|wide|json|yaml`.

### Planning & Release Control

Beyond `create release`, raptor exposes the planning and lifecycle operations a
deployment pipeline needs.

```bash
# Generate a plan without releasing (dry run); --target limits scope
raptor plan -p myproject -e dev
raptor plan -p myproject -e dev --target service/api --target service/worker

# See what depends on a resource (and what it depends on, with --reverse)
raptor impact service/api -p myproject
raptor impact service/api -p myproject --reverse

# Download the Terraform plan artifact for a release
raptor download plan <RELEASE_ID> -p myproject -e dev --save-to ./plan.txt

# Pause / resume the release queue for an environment
raptor pause releases -p myproject -e dev
raptor resume releases -p myproject -e dev

# Abort an in-flight release
raptor abort release <RELEASE_ID> -p myproject -e dev

# Run arbitrary commands inside the Terraform release pod (debugging)
raptor debug release <RELEASE_ID> terraform state list

# Submit a custom release made of explicit commands (repeat -c)
raptor create custom-release -p myproject -e dev -c "terraform plan" -c "terraform apply"
```

### Environment Lifecycle

```bash
# Launch (provision) an environment; -w tails the launch
raptor launch environment dev -p myproject -w

# Tear an environment down (requires --yes)
raptor destroy environment dev -p myproject --yes -w
```

### Copy Configuration Between Environments

`copy config` copies configuration from a source environment to a target
environment in the same project via
`PUT /cc-ui/v1/clusters/{targetClusterId}/copy-configurations-selective`. Selection
is a single INCLUDE **or** EXCLUDE — never both:

- `--include TYPES` — copy **only** the listed configuration types (INCLUDE mode)
- `--exclude TYPES` — copy **everything except** the listed types (EXCLUDE mode)
- neither flag — copy **all** configuration types

Valid types: `VARIABLES_SECRETS`, `ARTIFACTS`, `ENVIRONMENT_SETTINGS`, `SCHEDULES`,
`AVAILABILITY_SCHEDULES`, `OVERRIDES`, `TEMPLATE_INPUTS`.

Copying **overwrites** configuration on the target; you are prompted to confirm
unless `--yes`/`-y` is passed. Environment names are resolved to cluster IDs
automatically — you never pass raw cluster IDs.

```bash
# Copy every configuration type from dev to staging
raptor copy config -p myproject --from dev --to staging

# Copy only variables/secrets and artifacts
raptor copy config -p myproject --from dev --to staging --include VARIABLES_SECRETS,ARTIFACTS

# Copy everything except overrides, without a prompt
raptor copy config -p myproject --from dev --to staging --exclude OVERRIDES --yes
```

### Environment Overrides (apply override)

`apply override` is the ergonomic, field-level way to edit environment-specific
overrides (the read-modify-write counterpart to `set resource-overrides -f`).

```bash
# Merge individual fields into an existing override
raptor apply override service/api -p myproject -e dev \
  --set spec.env.LOG_LEVEL=debug --set spec.runtime.size.memory=2Gi

# Remove a single overridden field
raptor apply override service/api -p myproject -e dev --unset spec.env.LOG_LEVEL

# Removal works even when the override value already equals the blueprint value.
# The effective config does not change, but the key leaves the override document,
# so later blueprint edits reach this environment instead of staying pinned.
raptor apply override redis/cache -p myproject -e prod \
  --unset spec.sizing.snapshot_window --yes

# Replace the entire spec section (destructive)
raptor apply override service/api -p myproject -e dev --spec-file overrides.json

# Disable / re-enable a resource in this environment only
raptor apply override service/api -p myproject -e dev --disabled
raptor apply override service/api -p myproject -e dev --enabled

# Pin a different module flavor/version for this environment
raptor apply override service/api -p myproject -e dev --flavor k8s --version 0.3
```

`--overwrite` discards ALL existing overrides before applying, rebuilding the
document from the flags on that one command — keys you do not restate are lost,
and the command names them on stderr first. It cannot be combined with `--unset`:
the rebuild starts from an empty document, so there would be nothing for `--unset`
to remove and the result would just be an empty override. Use `--unset` on its own
to drop a key and keep the rest. `--yes`/`-y` skips the diff confirmation.
`--disabled`/`--enabled` are mutually exclusive.

The "nothing to apply" short-circuit compares the **override document** — the
artifact the command writes — not the merged blueprint ⊕ override config. Re-running
an identical apply is still skipped, but a change that only affects the override
document (dropping a key whose value coincides with the blueprint) is applied and
shown as a document diff. Containers that a removal empties are pruned, so
`--unset spec.sizing.snapshot_window` does not leave `spec.sizing: {}` behind.

If a `--unset` key is not present the command warns naming it on stderr. A call
that asked only for removals and removed nothing is reported as a no-op and exits
0 without writing — re-running a removal that has already landed is safe, and an
override document is never created for a resource that has none.

#### Bulk operations (many resources, one call)

Standing up or tearing down an environment is a bulk operation. Pass more than one
`KIND/NAME`, or use `--type` / `--resources-file`, and raptor writes the whole target
set in a **single** server call instead of one call per resource. `--type` enumerates
the resources of that kind in the **environment** (an environment holds names the
project blueprint does not: exploded template instances and env-level add-ons).

There are two bulk paths and they cannot be mixed in one invocation: **state**
(`--enabled/--disabled`, `--inherit/--no-inherit`) and **spec** (the `--set` family,
`--spec`/`--spec-file`, `--flavor`/`--version`). `--overwrite`, `--input` and
`--unset-input` stay single-resource only.

```bash
# Disable several resources at once
raptor apply override -p myproject -e eu-prod --disabled service/api service/web

# Enable every resource of a type in an environment
raptor apply override -p myproject -e eu-prod --enabled --type pubsub

# Toggle from a file of newline-delimited KIND/NAME lines
raptor apply override -p myproject -e eu-prod --disabled --resources-file topics.txt

# Preview the resolved set without applying
raptor apply override -p myproject -e eu-prod --enabled --type pubsub --dry-run
```

Child resources of the targeted resources may also be toggled by the server. The
bulk state call is atomic server-side (one git commit + one environment sync).

##### Bulk spec overrides

Spec flags in bulk mode write many override documents in one call
(`POST /cc-ui/v1/clusters/{cluster}/overrides`), with one git commit and one
environment sync.

```bash
# Change a field on every resource of a type
raptor apply override -p myproject -e eu-prod --type pubsub --set spec.retention=7

# Pin many resources to another module version
raptor apply override -p myproject -e eu-prod --type pubsub --version 0.4

# Preview the merged documents without writing
raptor apply override -p myproject -e eu-prod --type pubsub --set spec.retention=7 --dry-run
```

Three things differ from the state path:

- **The endpoint replaces each override document.** raptor reads the current documents
  first and merges, so keys you do not name (`disabled`, `inputs`, flavor/version pins,
  `advanced.inherit_from_base`) are carried over. A concurrent writer between the read
  and the write loses its change.
- **Child resources are not included.** There is no cascade.
- **The batch is not atomic.** A failure part-way can leave some resources written, and
  the response does not report which. Re-run with `--dry-run` to see what still
  differs — the set that still differs is the set that did not land — and re-running
  the command itself is safe, because targets that already match are skipped.

Targets whose merged document already matches are skipped and counted, so a completed
sweep can be re-run. The preview shows the document diff for the first 20 changing
targets plus a roll-up of which keys change.

### Resource Groups

Resource groups are a Control Plane/RBAC construct (independent of blueprint
branches/PRs). Membership edits are resource-centric and additive — each
resource's other group memberships are preserved.

```bash
# Manage groups
raptor create resource-group --name platform
raptor get resource-groups
raptor set resource-group <GROUP_ID> --name platform-core
raptor delete resource-group <GROUP_ID> --yes

# Attach / detach one or more resources (group by id or unique name)
raptor attach resource service/api service/worker --group platform -p myproject
raptor detach resource service/api --group platform -p myproject
```

### Accounts (Cloud & Version Control)

```bash
# List / inspect accounts
raptor get accounts                         # all; --type CLOUD|VERSION_CONTROL|CODER
raptor get account <ACCOUNT_ID>             # or: --name NAME
raptor get account-orgs <ACCOUNT_ID>
raptor get account-token-details --stack myproject

# Create a cloud account (interactive credential entry); -w waits for validation
raptor create account --provider aws --name my-aws -w

# Create a VCS account with a personal access token
raptor create github-account --name gh --username me --token <TOKEN> [--org ORG] [--enterprise-host HOST]
raptor create gitlab-account --name gl --username me --token <TOKEN>
raptor create bitbucket-account --name bb --username me --token <TOKEN> [--project-key KEY]

# Create a VCS account via the OAuth app flow (-w waits for authorization)
raptor create github-app-account --name gh-app -w
raptor create gitlab-app-account --name gl-app -w
raptor create bitbucket-app-account --name bb-app -w

# Update VCS account credentials
raptor set github-account <ACCOUNT_ID> --token <NEW_TOKEN>

# Delete an account
raptor delete account <ACCOUNT_ID> --yes
```

### Users & Groups

```bash
# List / inspect (use --expanded for full role detail)
raptor get users
raptor get user <USER_ID> --expanded
raptor get current-user
raptor get user-groups
raptor get user-group <GROUP_ID>

# Invite users (optionally straight into a group)
raptor create user --emails alice@example.com,bob@example.com --group <GROUP_ID>

# Create a group with a base role and optional scoping
raptor create user-group --name platform --base-role ADMIN \
  --projects proj-a,proj-b --accounts <ACCT_ID> --additional-roles <ROLE_ID>

# Update membership / roles
raptor set user <USER_ID> --roles <ROLE_ID> --groups <GROUP_ID>
raptor set user-group <GROUP_ID> --base-role VIEWER --changelog "scope down"

# Delete
raptor delete user <USER_ID> --yes
raptor delete user-group <GROUP_ID> --yes
```

### Projects & Project Types

```bash
# List project types
raptor get project-types

# Discover reference modules locally and choose exact identities from the result
git clone https://github.com/Facets-cloud/facets-modules-redesign.git ./reference-modules
raptor module discover --source ./reference-modules -o json
raptor create iac-module --source ./reference-modules \
  --module TYPE/FLAVOR/VERSION --module OTHER_TYPE/FLAVOR/VERSION -o json
# Test and publish each selected module; linked module repos retain their CI workflow.

# Optional: unrestricted project type, or select published pairs with --resource-type
raptor create project-type microservices --description "Microservices catalog"
raptor create project-type selected-services --description "Selected services" \
  --resource-type service/k8s --resource-type postgres/rds
# Old import project-type commands are disabled; manifests never select uploads.

# Delete a project type (by name). Projects still using it block the delete;
# raptor names them first so you know what to move.
raptor delete project-type microservices
raptor delete project-type old-a old-b --yes

# Then create a project of that type
raptor create project myproject --project-type microservices --description "..."

# GitOps-enabled project on one of your VCS integrations (name or ID from
# `raptor get accounts --type VERSION_CONTROL`); --org picks the org/group/workspace
raptor create project myproject --project-type microservices --vcs-account acme-github --org acme
# ...or attach an existing repository that already holds a blueprint, instead of creating one
raptor create project myproject --project-type microservices --vcs-account acme-github \
  --vcs-url https://github.com/acme/infra.git --branch main --relative-path projects/myproject

# Map which resource types are allowed for a project type.
# No mappings = ALL resource types allowed. Adding mappings preserves unrestricted
# types; create a new type with --resource-type to curate its initial catalog.
raptor create resource-type-mapping microservices --resource-type service/k8s --resource-type postgres/rds
raptor delete resource-type-mapping microservices --resource-type postgres/rds
```

### Template Inputs

Template inputs let a project type expose typed, per-environment values.

```bash
# Define a template input type (schema) for a project
raptor apply template-input-type -f input-type.json -p myproject [--name MY_TYPE]
raptor get template-input-types -p myproject

# Set / get values per environment
raptor create template-input <TYPE>/<UID> -p myproject -e dev --value myvalue
raptor set template-input <TYPE>/<UID> -p myproject -e dev -f value.json
raptor get template-inputs -p myproject -e dev
raptor get template-input <TYPE>/<UID> -p myproject -e dev
raptor delete template-input <TYPE>/<UID> -p myproject -e dev
```

### Artifacts & Registries

```bash
# Registries the control plane knows about
raptor get registries
raptor get registry-credentials

# Register an artifact, then point it at an image (per env / git-ref / release-stream)
raptor create artifact my-api -p myproject
raptor set artifact-uri my-api -p myproject -e dev --uri myrepo/my-api:abc123
raptor set artifact-uri my-api -p myproject --release-stream main --uri myrepo/my-api:latest

# Upload a zip bundle artifact
raptor set artifact-zip my-bundle -p myproject -e dev -f ./bundle.zip

# Inspect — `get builds` also carries the build ids and a PROMOTED column
raptor get artifacts -p myproject
raptor get builds my-api -p myproject

# Promote — registers the build at the NEXT stage of the artifact's promotion
# workflow (DEV -> QA -> ...). It does not flip a flag on the build you name.
raptor promote build -a my-api --tag v1.4.2
raptor promote build -a my-api --id 6a05ba178a06280c7b9c1974

# Delete a build registration, by id (does NOT remove the image from the registry)
raptor delete build 6a05ba178a06280c7b9c1974
raptor delete build 6a05ba178a06280c7b9c1974 62287c34fab757000147dea3 --yes

# Delete the artifact itself — refuses while builds are still registered under it
raptor delete artifact my-api -p myproject
```

An **artifact** is the named integration (the control plane calls it an Artifact
CI); a **build** is one registered image or zip beneath it. `create artifact` and
`delete artifact` act on the first, `promote build` and `delete build` on the
second.

**Promote** takes `-a ARTIFACT` plus exactly one of `--tag` or `--id` — the API is
addressed by both, and a build record carries no reference back to its artifact.
`--tag` reads the build's tag field, or the tag in its image URI when that field is
empty — which it is for anything registered by `set artifact-uri`. Prefer `--id`
anyway: a digest-only reference has no tag at all, and the same tag is often
registered against several streams, so an ambiguous `--tag` is rejected rather than
resolved arbitrarily. The control plane requires the artifact to have a promotion
workflow, the build to be classified, and a next stage to exist — otherwise it
reports why.

**Delete build** is addressed by build id alone, so that is all it takes. Ids come
from `raptor get builds ARTIFACT_NAME -p PROJECT -o json`.

### Web Components

Register custom micro-frontends that the Control Plane UI loads.

```bash
raptor get web-components
raptor create web-component cost-dashboard \
  --remote-url https://myorg.github.io/cost-dashboard/cost-dashboard.js \
  --enabled --icon-url https://example.com/icon.png --tooltip "Cost Dashboard"
raptor set web-component cost-dashboard --enabled=false
raptor delete web-component cost-dashboard
```

### Module Development

The `module` subtree scaffolds and edits an IaC module locally (its `facets.yaml`
and managed files) before you `create iac-module` / `publish iac-module`. See also
`raptor module design-guide` and `raptor module development-guide` for the full
authoring walkthrough printed by the CLI.

```bash
# Scaffold a new module
raptor module init \
  --intent s3 --flavor aws --version 0.1 --cloud aws \
  --description "S3 bucket with encryption" \
  --input account:@facets/cloud_account \
  --requires-provider "provider=aws;input=account" \
  --output-type @facets/s3

# Inspect the working module / validate it
raptor module show
raptor module validate            # -f points at a module dir (default: .)

# Preview the module's UI form locally — renders the REAL control-plane React
# form in headless Chrome (no control plane needed). Requires Chrome/Chromium.
raptor module preview .                              # PNG + JSON report to stdout
raptor module preview facets.yaml --json r.json --png form.png
raptor module preview . --mode override              # environment-override page view
# Exit codes: 0 = rendered clean; 1 = render errors (e.g. array with non-string
# items) or crash; 2 = tool failure (Chrome missing, bad yaml, timeout).
# Agent/CI loop: run with --json, assert .ok == true, inspect .errors/.warnings/
# .hidden and per-field .degraded flags; view the PNG for layout.
# Refresh the embedded UI bundle after form-engine changes in control-plane-ui-react:
#   scripts/vendor-preview.sh

# Edit inputs and outputs
raptor module add-input --name vpc --output-type @facets/vpc [--provider aws]
raptor module remove-input --name vpc
raptor module add-output ...
raptor module remove-output ...
raptor module set-output-provider ...

# Edit the spec schema (UI form fields)
raptor module set-spec --add bucket_name --type string --title "Bucket Name" \
  --description "S3 bucket name" --min-length 3 --max-length 63 --ui-order 1
raptor module set-spec --add environment --type enum --title "Environment" \
  --description "Target environment" --values "dev,staging,prod"
raptor module set-spec --remove bucket_name
raptor module set-spec --show

# Output type (interface) definitions
raptor module create-output-type @myorg/s3 -f schema.json   # alias: update-output-type
raptor module get-output-type @myorg/s3
raptor module delete-output-type @myorg/s3

# Discover available output types for a project type, download module sources
raptor module discover --project-type microservices
raptor module download service/k8s/0.2 --save-to ./modules/
```

### Utility & Maintenance

```bash
raptor whoami                # Show the authenticated identity / control plane
raptor upgrade               # Self-update raptor to the latest release
raptor report -m "..."       # Send a friction report to Facets (what went wrong, how you recovered)
RAPTOR_CHANNEL=unstable raptor upgrade  # Follow the unstable channel: a dev build of every merge to main
raptor install skill               # Install the Raptor skill into every agent host on this machine
raptor uninstall skill             # Remove it again (files you changed stay)
raptor blueprint-guide       # Print the blueprint authoring guide
raptor cache status          # Inspect the local schema/metadata cache
raptor cache clear           # Clear it
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
- `iac-module` = `iac-modules` = `module` = `modules`
- `variables` = `vars` = `var`
- `channels` = `channel`
- `subscriptions` = `subscription` = `subs` = `sub`
- `config` = `configs` = `configuration` = `configurations`

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
raptor describe expressions -p myproject

# 7. Apply the resource to blueprint
raptor apply -f my-service.json -p myproject

# 8. Deploy to dev environment
raptor create release -p myproject -e dev --plan -w  # Preview first
raptor create release -p myproject -e dev -w -m "Deploy my-service"

# 9. Monitor the deployment
raptor get releases -p myproject -e dev
raptor logs release -p myproject -e dev -f <RELEASE_ID>

# 10. Check whether the resource has changes pending release
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

# Pending-release status and overrides
raptor get resource-status -p <project> -e <env> [<type>/<name>] [--pending-only]
raptor get resource-overrides -p <project> -e <env> <type>/<name>
raptor set resource-overrides -p <project> -e <env> -f <file> <type>/<name>

# Get runtime outputs
raptor get resource-outputs -p <project> -e <env> <type>/<name>

# Get kubeconfig
raptor get kubeconfig -p <project> -e <env> [-o <file>]

# Get expressions
raptor describe expressions -p <project> [KIND[/NAME]]
raptor describe expressions -p <project> -e <env> KIND/NAME --value <value> [--refresh | --no-cache]
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
raptor create iac-module -f <directory> --publish [--backward-compatible yes | --versionupgrade]
raptor publish iac-module <type>/<flavor>/<version> [--backward-compatible yes | --versionupgrade]
raptor delete iac-module <type>/<flavor>/<version>
```

