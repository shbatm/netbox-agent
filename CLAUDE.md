# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

netbox-agent is a Python tool that automatically creates and updates hardware infrastructure in Netbox (a datacenter infrastructure management platform). It collects hardware information from servers using system tools (dmidecode, lldpd, ethtool, etc.) and synchronizes it with Netbox.

**Key Capabilities:**
- Auto-discovers physical servers, virtual machines, chassis, and blade servers
- Creates/updates network interfaces (physical, bonding, VLAN) with IP addresses
- Performs automatic cabling using LLDP data
- Manages local inventory (CPU, GPU, RAM, RAID cards, disks, PSUs)
- Supports hypervisor-to-VM relationships

## Development Commands

### Running the Agent
```bash
# Run directly from source (for development)
python3 -m netbox_agent.cli --register

# With specific config file
python3 -m netbox_agent.cli -c /etc/netbox_agent.yaml --register

# Update specific components
python3 -m netbox_agent.cli --update-network
python3 -m netbox_agent.cli --update-inventory
python3 -m netbox_agent.cli --update-location
python3 -m netbox_agent.cli --update-psu
```

### Testing
```bash
# Run all tests (requires Docker for Netbox instance)
./tests.sh

# Run pytest directly (with existing Netbox)
pytest

# Run specific test file
pytest tests/server.py
```

The `tests.sh` script automatically spins up a Netbox Docker container, runs tests, and tears it down.

### Linting and Formatting
```bash
# Check code style
ruff check

# Auto-format code
ruff format
```

### Installation for Development
```bash
pip3 install -e .
pip3 install -e ".[dev]"    # With dev dependencies
pip3 install -e ".[style]"  # With style/linting tools
```

## Architecture

### Entry Point Flow
1. [netbox_agent/cli.py](netbox_agent/cli.py) - Main entry point (`main()` function)
2. Parses DMI data using [dmidecode.py](netbox_agent/dmidecode.py)
3. Determines if system is VM or physical hardware
4. Instantiates appropriate vendor-specific host class
5. Calls `netbox_create_or_update()` to sync with Netbox

### Vendor Support System
The codebase uses a vendor abstraction pattern:
- [netbox_agent/server.py](netbox_agent/server.py) - `ServerBase` class provides core functionality
- [netbox_agent/vendors/](netbox_agent/vendors/) - Vendor-specific implementations inherit from `ServerBase`:
  - [dell.py](netbox_agent/vendors/dell.py) - `DellHost`
  - [hp.py](netbox_agent/vendors/hp.py) - `HPHost`
  - [supermicro.py](netbox_agent/vendors/supermicro.py) - `SupermicroHost`
  - [qct.py](netbox_agent/vendors/qct.py) - `QCTHost`
  - [generic.py](netbox_agent/vendors/generic.py) - `GenericHost` (fallback)

Each vendor class implements blade detection (`is_blade()`) and slot identification (`get_blade_slot()`) specific to their hardware.

### Core Modules
- [config.py](netbox_agent/config.py) - Configuration management using jsonargparse; supports config files, env vars, and CLI args
- [network.py](netbox_agent/network.py) - `Network` class scans interfaces via `/sys/class/net/`, manages IPs, VLANs, and LLDP-based cabling
- [inventory.py](netbox_agent/inventory.py) - `Inventory` class discovers and syncs hardware components (CPU, GPU, RAM, disks, RAID cards)
- [location.py](netbox_agent/location.py) - Driver-based system for determining datacenter/rack/tenant using regex patterns

### Driver System
The location module uses a pluggable driver architecture:
- [drivers/cmd.py](netbox_agent/drivers/cmd.py) - Executes command and extracts value via regex
- [drivers/file.py](netbox_agent/drivers/file.py) - Reads file and extracts value via regex
- Custom drivers can be loaded via `driver_file` config parameter

Configuration example:
```yaml
datacenter_location:
  driver: "cmd:cat /etc/qualification | tr [A-Z] [a-z]"
  regex: "datacenter: (?P<datacenter>[A-Za-z0-9]+)"
```

### RAID Support
[netbox_agent/raid/](netbox_agent/raid/) contains vendor-specific RAID controller parsers:
- [hp.py](netbox_agent/raid/hp.py) - HP RAID (hpassacli)
- [omreport.py](netbox_agent/raid/omreport.py) - Dell RAID (omreport)
- [storcli.py](netbox_agent/raid/storcli.py) - LSI/Broadcom RAID (storcli)

All implement a common interface defined in [base.py](netbox_agent/raid/base.py).

### Blade Server Handling
Blades are registered using Netbox's parent/child device relationship:
1. Chassis is created/found first
2. Blade server is created as child with specific device bay slot
3. Blade slot names must match device type bay definitions in Netbox (e.g., "Bay 1", "Slot 01")
4. Expansion blades (GPU, storage) can be registered as separate devices with `--expansion-as-device`

### Virtual Machine Support
- [virtualmachine.py](netbox_agent/virtualmachine.py) - Handles VM detection and registration
- [hypervisor.py](netbox_agent/hypervisor.py) - Associates VMs with hypervisor cluster
- Supports Hyper-V, VMWare, VirtualBox, AWS, GCP

## Configuration

Config precedence: CLI args > environment variables (prefixed with `NETBOX_AGENT_`) > config file > defaults

Default config file locations checked in order:
1. `/etc/netbox_agent.yaml`
2. `~/.config/netbox_agent.yaml`
3. `~/.netbox_agent.yaml`

See [netbox_agent.yaml.example](netbox_agent.yaml.example) for full configuration options.

## Important Considerations

### System Information Overrides
The agent detects system information (manufacturer, model, serial number) from dmidecode, but you can override these values via configuration. This is useful for:
- Testing with specific device types
- Correcting misdetected hardware
- Standardizing device naming

Override methods (in precedence order):
1. **CLI arguments**: `--device.manufacturer HP --device.model "ProLiant DL380 Gen10" --device.serial MYSERIAL123`
2. **Environment variables**: `NETBOX_AGENT__DEVICE__MANUFACTURER`, `NETBOX_AGENT__DEVICE__MODEL`, `NETBOX_AGENT__DEVICE__SERIAL`
3. **Config file**:
   ```yaml
   device:
     manufacturer: HP
     model: ProLiant DL380 Gen10
     serial: MYSERIAL123
   ```

Implementation details:
- Manufacturer override affects vendor class selection in [cli.py:39-42](netbox_agent/cli.py#L39-L42)
- Model override is used in [server.py:140-146](netbox_agent/server.py#L140-L146) `get_product_name()`
- Serial override is used in [server.py:148-154](netbox_agent/server.py#L148-L154) `get_service_tag()`

### MAC Address Conflicts
The agent scopes MAC address lookups to `interface.id` to avoid primary_mac_address conflicts in Netbox (see commit 5d762f6).

### Anycast IP Handling
IPs must be set to "Anycast" mode in Netbox to prevent duplicate IP conflicts when the same IP exists on multiple servers.

### Custom Fields for Disks
When using `--process-virtual-drives`, these custom fields must be pre-created in Netbox as `DCIM > inventory item` text fields:
- `mount_point` - Device mount point(s)
- `pd_identifier` - Physical disk identifier in RAID controller
- `vd_array` - Virtual drive array the disk is member of
- `vd_consistency` - Virtual disk array consistency
- `vd_device` - Virtual drive system device
- `vd_raid_type` - Virtual drive array RAID type
- `vd_size` - Virtual drive array size

### Netbox Version Compatibility
- Requires Netbox >= 3.7
- Version check occurs in [cli.py:44](netbox_agent/cli.py#L44)

## Testing Environment

The test suite uses netbox-community/netbox-docker. Tests are configured to use:
- URL: `http://localhost:8000` (mapped from container)
- Token: `0123456789abcdef0123456789abcdef01234567`
- Superuser: test/test (test@test.com)

See [tests.sh](tests.sh) for the full test environment setup.
