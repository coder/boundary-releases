# Boundary

A simple process isolator for Linux that provides lightweight isolation focused
on AI and development environments.

## Overview

Boundary allows you to run processes in isolated environments using Linux namespaces
while providing:

- Filesystem isolation with OverlayFS or fallback file copying
- Docker/OCI container image support
- Network isolation with port forwarding
- User namespace isolation
- Mount host directories into containers
- Run commands as different users inside containers

## Installation

### From a preview release

```bash
wget https://github.com/coder/boundary-releases/releases/download/preview/boundary-$(uname -m) -O boundary
chmod +x boundary
sudo mv ./boundary /usr/local/bin/boundary
```

### From Source

```bash
make
make install
```

## Usage

### Basic Usage

Run a bash shell in an isolated environment:

```bash
boundary
```

### Using a Container Image

Run a command in a specific container image:

```bash
boundary --image alpine -- /bin/sh
```

### Using Claude Code

```bash
boundary \
  --image ghcr.io/thomask33/claude-code-devcontainer:latest \
  --run-as-user node -- claude
```

### Using Coder Claude Code integration

Make sure that the coder CLI is available on your local system.

```bash
boundary \
  --image ghcr.io/thomask33/claude-code-devcontainer:latest \
  --run-as-user node \
  --binaries coder -- coder agent claude
```

or via environment variables:

```bash
export BOUNDARY_IMAGE=ghcr.io/thomask33/claude-code-devcontainer:latest
export BOUNDARY_BINARIES=coder
export BOUNDARY_RUN_AS_USER=node

boundary -- coder agent claude
```

or via config file:

```bash
cat << EOF > boundary.yaml
container:
  image: ghcr.io/thomask33/claude-code-devcontainer:latest
  binaries:
    - coder
  user:
    runAs: node
EOF

boundary -c boundary.yaml -- coder agent claude

# or configured via environment variables

export BOUNDARY_CONFIG_PATH=$(realpath ./boundary.yaml)
boundary -- coder agent claude
```

### Limiting Claude Code internet access

The Claude Code devcontainer image contains a [firewall script](https://github.com/anthropics/claude-code/blob/555b6b5b8a5f06f1e8725a584e62fb6b7c8eece5/.devcontainer/init-firewall.sh)
to limit internet access using standard iptables rules.
This script comes embedded in the devcontainer image and will block any request
that is not targeted at IP addresses belonging to Github, `registry.npmjs.org`,
`api.anthropic.com`, `sentry.io`, `statsig.anthropic.com`, `statsig.com`.

<details>
<summary>init-firewall.sh</summary>

<!-- `$ curl https://raw.githubusercontent.com/anthropics/claude-code/refs/heads/main/.devcontainer/init-firewall.sh` as bash -->

```bash
#!/bin/bash
set -euo pipefail  # Exit on error, undefined vars, and pipeline failures
IFS=$'\n\t'       # Stricter word splitting

# Flush existing rules and delete existing ipsets
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X
ipset destroy allowed-domains 2>/dev/null || true

# First allow DNS and localhost before any restrictions
# Allow outbound DNS
iptables -A OUTPUT -p udp --dport 53 -j ACCEPT
# Allow inbound DNS responses
iptables -A INPUT -p udp --sport 53 -j ACCEPT
# Allow outbound SSH
iptables -A OUTPUT -p tcp --dport 22 -j ACCEPT
# Allow inbound SSH responses
iptables -A INPUT -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
# Allow localhost
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Create ipset with CIDR support
ipset create allowed-domains hash:net

# Fetch GitHub meta information and aggregate + add their IP ranges
echo "Fetching GitHub IP ranges..."
gh_ranges=$(curl -s https://api.github.com/meta)
if [ -z "$gh_ranges" ]; then
    echo "ERROR: Failed to fetch GitHub IP ranges"
    exit 1
fi

if ! echo "$gh_ranges" | jq -e '.web and .api and .git' >/dev/null; then
    echo "ERROR: GitHub API response missing required fields"
    exit 1
fi

echo "Processing GitHub IPs..."
while read -r cidr; do
    if [[ ! "$cidr" =~ ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}/[0-9]{1,2}$ ]]; then
        echo "ERROR: Invalid CIDR range from GitHub meta: $cidr"
        exit 1
    fi
    echo "Adding GitHub range $cidr"
    ipset add allowed-domains "$cidr"
done < <(echo "$gh_ranges" | jq -r '(.web + .api + .git)[]' | aggregate -q)

# Resolve and add other allowed domains
for domain in \
    "registry.npmjs.org" \
    "api.anthropic.com" \
    "sentry.io" \
    "statsig.anthropic.com" \
    "statsig.com"; do
    echo "Resolving $domain..."
    ips=$(dig +short A "$domain")
    if [ -z "$ips" ]; then
        echo "ERROR: Failed to resolve $domain"
        exit 1
    fi
    
    while read -r ip; do
        if [[ ! "$ip" =~ ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$ ]]; then
            echo "ERROR: Invalid IP from DNS for $domain: $ip"
            exit 1
        fi
        echo "Adding $ip for $domain"
        ipset add allowed-domains "$ip"
    done < <(echo "$ips")
done

# Get host IP from default route
HOST_IP=$(ip route | grep default | cut -d" " -f3)
if [ -z "$HOST_IP" ]; then
    echo "ERROR: Failed to detect host IP"
    exit 1
fi

HOST_NETWORK=$(echo "$HOST_IP" | sed "s/\.[0-9]*$/.0\/24/")
echo "Host network detected as: $HOST_NETWORK"

# Set up remaining iptables rules
iptables -A INPUT -s "$HOST_NETWORK" -j ACCEPT
iptables -A OUTPUT -d "$HOST_NETWORK" -j ACCEPT

# Set default policies to DROP first
# Set default policies to DROP first
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT DROP

# First allow established connections for already approved traffic
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Then allow only specific outbound traffic to allowed domains
iptables -A OUTPUT -m set --match-set allowed-domains dst -j ACCEPT

echo "Firewall configuration complete"
echo "Verifying firewall rules..."
if curl --connect-timeout 5 https://example.com >/dev/null 2>&1; then
    echo "ERROR: Firewall verification failed - was able to reach https://example.com"
    exit 1
else
    echo "Firewall verification passed - unable to reach https://example.com as expected"
fi

# Verify GitHub API access
if ! curl --connect-timeout 5 https://api.github.com/zen >/dev/null 2>&1; then
    echo "ERROR: Firewall verification failed - unable to reach https://api.github.com"
    exit 1
else
    echo "Firewall verification passed - able to reach https://api.github.com as expected"
fi
```

</details>

<br />

This script can be run in boundary by:

```bash
# Launch the boundary
boundary --image ghcr.io/thomask33/claude-code-devcontainer:latest

# Within the boundary we can now setup networking rules using
init-firewall.sh

# We can verify this by running curl and waiting for it to timeout in 5 seconds
curl --max-time 5 https://example.com

# Then we can launch Claude Code and work as normal
claude
```

### Configuration Options

Boundary offers several configuration options, one can list them using:

```$ as bash
boundary --help
```

```bash
USAGE:
  boundary

  Linux process isolation tool

SUBCOMMANDS:
    generate-config    Generate a boundary configuration file
    version            Display version information

OPTIONS:
      --boundary-dir string, $BOUNDARY_DIR (default: /home/thomask33.linux/.boundary)
          Base directory for boundary files and directories.

  -c, --config yaml-config-path, $BOUNDARY_CONFIG_PATH
          Specify a YAML file to load configuration from.

      --debug bool, $BOUNDARY_DEBUG
          Enable debug mode and logging.

      --fork bool, $BOUNDARY_FORK (default: false)
          Use fork+exec instead of syscall.Exec.

CONTAINER OPTIONS: 
      --binaries string-array, $BOUNDARY_BINARIES
          List of binaries to copy.

      --container-name string, $BOUNDARY_CONTAINER_NAME
          Assign a name to the container. (if empty a name will be generated).

      --image string, $BOUNDARY_IMAGE
          The image to use as rootfs.

CONTAINER / MOUNT OPTIONS: 
      --mount-certs bool, $BOUNDARY_MOUNT_CERTS (default: true)
          Mount SSL certificate files for HTTPS connections.

      --mount-dns bool, $BOUNDARY_MOUNT_DNS (default: false)
          Mount DNS configuration files for name resolution.

      --mount-pwd bool, $BOUNDARY_MOUNT_PWD (default: true)
          Mount the current pwd as read write into the namespace.

      --require-proc bool, $BOUNDARY_REQUIRE_PROC (default: true)
          Whether to require successful mounting of procfs (fail if mounting
          fails).

      --require-sysfs bool, $BOUNDARY_REQUIRE_SYSFS (default: true)
          Whether to require successful mounting of sysfs (fail if mounting
          fails).

CONTAINER / NETWORK OPTIONS: 
      --network bool, $BOUNDARY_NETWORK (default: true)
          Create a network namespace for the isolated process.

      --port-forward string, $BOUNDARY_PORT_FORWARD
          Port forwarding configuration in format
          'hostPort1:containerPort1,hostPort2:containerPort2'.

CONTAINER / OVERLAYFS OPTIONS: 
      --overlayfs bool, $BOUNDARY_OVERLAYFS (default: true)
          Use OverlayFS for container filesystem (improves performance and
          reduces disk usage).

      --overlayfs-persist bool, $BOUNDARY_OVERLAYFS_PERSIST (default: false)
          Persist container filesystem changes after exit.

CONTAINER / UTS OPTIONS: 
      --hostname string, $BOUNDARY_HOSTNAME (default: boundary)
          Set the hostname for the container.

CONTAINER / USER OPTIONS: 
      --run-as-user string, $BOUNDARY_RUN_AS_USER
          User to run the command as inside the container (defaults to root if
          empty).

      --user-namespace bool, $BOUNDARY_USER_NAMESPACE (default: true)
          Create a user namespace for the isolated process.

      --user-namespace-map-mode string, $BOUNDARY_USER_NAMESPACE_MAP_MODE (default: full)
          User namespace mapping mode: 'basic' or 'full'.

```

#### Generate Configuration File

Boundary can generate a complete configuration file with default values and documentation:

```$ as yaml
# Generate and display configuration (stdout)
boundary generate-config
```

```yaml
container:
    # List of binaries to copy.
    # (default: <unset>, type: string-array)
    binaries: []
    # Assign a name to the container. (if empty a name will be generated).
    # (default: <unset>, type: string)
    name: ""
    uts:
        # Set the hostname for the container.
        # (default: boundary, type: string)
        hostname: boundary
    # The image to use as rootfs.
    # (default: <unset>, type: string)
    image: ""
    mount:
        # Mount SSL certificate files for HTTPS connections.
        # (default: true, type: bool)
        certs: true
        # Mount DNS configuration files for name resolution.
        # (default: false, type: bool)
        dns: false
        # Mount the current pwd as read write into the namespace.
        # (default: true, type: bool)
        pwd: true
        # Whether to require successful mounting of procfs (fail if mounting fails).
        # (default: true, type: bool)
        require-proc: true
        # Whether to require successful mounting of sysfs (fail if mounting fails).
        # (default: true, type: bool)
        require-sysfs: true
    network:
        # Create a network namespace for the isolated process.
        # (default: true, type: bool)
        enable: true
        # Port forwarding configuration in format
        # 'hostPort1:containerPort1,hostPort2:containerPort2'.
        # (default: <unset>, type: string)
        portForward: ""
    overlayfs:
        # Use OverlayFS for container filesystem (improves performance and reduces disk
        # usage).
        # (default: true, type: bool)
        enable: true
        # Persist container filesystem changes after exit.
        # (default: false, type: bool)
        persist: false
    user:
        # User to run the command as inside the container (defaults to root if empty).
        # (default: <unset>, type: string)
        runAs: ""
        # Create a user namespace for the isolated process.
        # (default: true, type: bool)
        enable: true
        # User namespace mapping mode: 'basic' or 'full'.
        # (default: full, type: string)
        mode: full

```

The generated file includes:

- Default values pre-populated
- Detailed descriptions as comments
- Type information for each field
- Hierarchical organization of related settings

## Development

### Requirements

- Nix (only supported development environment)

### Build

```bash
# Enter development shell
nix develop

# Build the binary
nix build

# Clean build artifacts
make clean
```

### Testing

```bash
# Run all tests
go test ./...

# Run a specific test
go test github.com/coder/boundary/package -run TestName
```

### Code Quality

```bash
# Format code
nix fmt

# Lint code
GOOS=linux golangci-lint run
```

## Troubleshooting

If you encounter "operation not permitted" errors when mounting tmpfs in user
namespaces, it may be related to AppArmor restrictions. For development purposes,
you can temporarily disable AppArmor:

```bash
# Disable AppArmor completely (for development only)
sudo systemctl stop apparmor
sudo systemctl disable apparmor

# Set the profile for a specific binary to complain mode
sudo aa-complain /path/to/binary
```

## License

TBD: \[License information\]

