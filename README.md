# Raptor Releases

Public releases for the Raptor CLI tool.

## Installation

Download the latest release for your platform:

### Linux
```bash
# AMD64
curl -L -o raptor https://github.com/Facets-cloud/raptor-releases/releases/latest/download/raptor-linux-amd64
chmod +x raptor
sudo mv raptor /usr/local/bin/

# ARM64
curl -L -o raptor https://github.com/Facets-cloud/raptor-releases/releases/latest/download/raptor-linux-arm64
chmod +x raptor
sudo mv raptor /usr/local/bin/
```

### macOS
```bash
# Intel
curl -L -o raptor https://github.com/Facets-cloud/raptor-releases/releases/latest/download/raptor-darwin-amd64
chmod +x raptor
sudo mv raptor /usr/local/bin/

# Apple Silicon
curl -L -o raptor https://github.com/Facets-cloud/raptor-releases/releases/latest/download/raptor-darwin-arm64
chmod +x raptor
sudo mv raptor /usr/local/bin/
```

## Getting Started

### Setup Authentication

After installation, configure your Facets Control Plane credentials:

```bash
raptor login
```

This will:
- Prompt for your Control Plane URL
- Open your browser to generate a token
- Save credentials to `~/.facets/credentials`

### Usage

```bash
# View all available commands
raptor --help

# List projects
raptor get projects

# Get resources in a project
raptor get resources -p myproject
```

### Multiple Profiles

To manage multiple environments, use the `FACETS_PROFILE` environment variable:

```bash
# Login with a custom profile name
raptor login  # Enter "production" when prompted for profile name

# Use the profile
export FACETS_PROFILE=production
raptor get projects
```
