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

## Usage

```bash
raptor --help
```
