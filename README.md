# ASA - Projeto 01: DevOps com Vagrant e Ansible

## Introduction

This project automates the creation and configuration of a four-node virtual environment using Vagrant and Ansible, featuring role-specific servers and centralized management. It demonstrates Infrastructure as Code (IaC) principles by provisioning a complete network with DHCP, DNS, file storage, web, and database services from a single `Vagrantfile` and a set of Ansible playbooks.

- **Professor:** Leonidas Francisco de Lima Junior
- **Subject:** Administração de Sistemas Abertos
- **Students:** Bruno de Farias Andrade, Ícaro Machado da Silva

---

## Architecture Overview

The environment consists of four virtual machines (VMs) connected via a private network (`192.168.56.0/24`). Each VM has a specific role, simulating a multi-tier application infrastructure.

- **`arq` (192.168.56.128)**: The core infrastructure server providing DHCP, DNS, LVM storage, and NFS services.
- **`db` (DHCP)**: A dedicated MariaDB database server.
- **`app` (DHCP)**: An Apache web application server.
- **`cli` (DHCP)**: A client workstation with a graphical interface for testing.

All VMs are provisioned from a base `debian/bookworm64` image and configured by Ansible playbooks for consistency and automation.

---

## Technology Stack

| Component       | Technology Used                         | Purpose                                           |
| :-------------- | :-------------------------------------- | :------------------------------------------------ |
| **Virtualization** | VirtualBox, Vagrant                     | VM lifecycle management and environment isolation |
| **Provisioning**  | Ansible                                 | Configuration management and service setup        |
| **Core Services** | ISC DHCP Server, BIND9, LVM, NFS        | Network, storage, and file sharing foundation     |
| **Application**   | MariaDB, Apache2 (HTTPD)                | Database and web application hosting              |
| **Client Tools**  | Firefox-ESR, AutoFS, X11                | User access and graphical interface               |
| **Management**    | Chrony (NTP), SSH (hardened), Sudo      | Time sync, secure access, and user privileges     |

---

## Vagrantfile

The `Vagrantfile` serves as the main configuration blueprint for the entire infrastructure. It defines four virtual machines with distinct roles, network configurations, and provisioning steps using Ansible.

### Key Features:
- **Modular Design**: Each VM (`arq`, `db`, `app`, `cli`) is defined in separate blocks with role-specific configurations
- **Multi-Stage Provisioning**: Uses both a generic playbook (`generic.yml`) for common settings and role-specific playbooks for specialized configurations
- **Network Isolation**: Creates a private network (192.168.56.0/24) with mixed static and DHCP addressing
- **Resource Management**: Configures CPU, memory, and storage resources per VM requirements
- **Automated Provisioning**: Integrates Ansible playbooks directly into the Vagrant workflow

### Configuration Breakdown:

1. **Global Settings** (applies to all VMs):
   - Base OS: `debian/bookworm64`
   - Memory: 512MB (except `cli` with 1024MB)
   - SSH key management: Preserves existing keys
   - Synced folders: Disabled for performance
   - VirtualBox linked clones for faster provisioning

2. **Network Configuration**:
   - `arq`: Static IP (192.168.56.128) - acts as DHCP/DNS server
   - `db` & `app`: DHCP with fixed MAC addresses for consistent addressing
   - `cli`: Standard DHCP client
   - Automatic DHCP server management to prevent conflicts

3. **Storage Configuration**:
   - `arq` receives three additional 10GB virtual disks for LVM storage setup
   - Other VMs use default storage allocation

4. **Provisioning Flow**:
   - **Stage 1**: `generic.yml` applies common configurations to all VMs (users, SSH, packages)
   - **Stage 2**: Role-specific playbooks configure specialized services for each VM

---

## Virtual Machines and Playbooks

### 1. `arq` - Infrastructure Server (`arq.yml`)
The central server hosting foundational network and storage services.
*   **DHCP Server**: Configures `isc-dhcp-server` to manage the private network (`eth1`), assigning IPs to `db`, `app`, and `cli`.
*   **DNS Server**: Sets up `bind9` as an authoritative server for the `bruno.icaro.devops` domain, managing forward and reverse (`56.168.192.db`) lookups.
*   **Storage (LVM)**: Combines three 10GB virtual disks into a single LVM Volume Group (`dados`), creates a 15GB Logical Volume (`ifpb`), formats it as `ext4`, and mounts it at `/dados`.
*   **File Sharing (NFS)**: Exports the `/dados/nfs` directory via NFS to the private subnet. Uses `all_squash` to map all remote users to a local `nfs-ifpb` user for unified permissions.

### 2. `db` - Database Server (`db.yml`)
A dedicated server for database operations.
*   **Database**: Installs and starts the `mariadb-server` package.
*   **File Access**: Configures AutoFS to automatically mount the shared `/dados/nfs` directory from the `arq` server at `/var/nfs` on demand.

### 3. `app` - Web Server (`app.yml`)
A server for hosting web applications.
*   **Web Server**: Installs, starts, and enables the `apache2` HTTP server.
*   **Content**: Deploys a custom `index.html` file to the default web root (`/var/www/html/`).
*   **File Access**: Configures AutoFS to automatically mount the shared NFS directory from `arq` at `/var/nfs`.

### 4. `cli` - Client Workstation (`cli.yml`)
A graphical client machine for user access and testing.
*   **Graphical Environment**: Installs `firefox-esr`, `xauth`, and `x11-utils` to enable running graphical applications.
*   **X11 Forwarding**: Configures the SSH server (`X11Forwarding yes`, `X11UseLocalhost no`) to allow GUI applications from the VM to be displayed on the host machine.
*   **File Access**: Configures AutoFS to automatically mount the shared NFS directory from `arq` at `/var/nfs`.

### 5. Base Configuration (`generic.yml`)
Applies common security and management settings to **all VMs**.
*   **System**: Performs a full package update and installs NTP (`chrony`) and NFS client utilities.
*   **Users & Groups**: Creates the `ifpb` group (with `sudo` privileges) and users `bruno` and `icaro`. Generates SSH keys for them and deploys the host's public key for password-less Ansible access.
*   **SSH Hardening**: Configures the SSH daemon to use only key-based authentication, disable root login, restrict access to the `vagrant` and `ifpb` groups, and display a legal banner.

---

## Inventory

This section documents the key configuration files that enable the network services in the environment. These files are deployed to the `arq` server by the Ansible playbook and manage DHCP assignment and DNS resolution for all nodes.

### DHCP Configuration Files

**File: `files/dhcp/dhcpd.conf`**
- Main configuration file for the ISC DHCP server running on the `arq` server
- Defines the authoritative DHCP server for the 192.168.56.0/24 subnet
- Configures lease durations (default: 180 seconds, max: 3600 seconds)
- Sets domain name and DNS server options for all clients
- Defines a DHCP pool (192.168.56.50-100) for dynamic address assignment
- Maps fixed IP addresses to specific MAC addresses for `db` (192.168.56.110) and `app` (192.168.56.180) servers

**Key Parameters:**
- `authoritative`: Server is the official DHCP server for the network
- `range 192.168.56.50 192.168.56.100`: DHCP address pool for dynamic assignment
- `hardware ethernet`: MAC-to-IP mapping for static reservations

### DNS Configuration Files

**File: `files/dns/named.conf.options`**
- Global options configuration for BIND9 DNS server
- Creates an Access Control List (ACL) restricting access to the 192.168.56.0/24 subnet
- Enables listening on all interfaces (IPv4 and IPv6)
- Configures query permissions (localhost and internal-network only)
- Sets forwarders to public DNS servers (1.1.1.1, 8.8.8.8) for external resolution
- Enables DNS recursion for internal clients

**Security Features:**
- `allow-query`: Restricts DNS queries to authorized clients only
- `allow-transfer`: Limits zone transfers to localhost only
- `dnssec-validation`: Enables DNSSEC validation for enhanced security

**File: `files/dns/named.conf.internal-zones`**
- Zone declarations for the internal domain infrastructure
- Defines the forward lookup zone for `bruno.icaro.devops` domain
- Defines the reverse lookup zone for 192.168.56.0/24 network (56.168.192.in-addr.arpa)
- Configures both zones as master (authoritative) with no dynamic updates

**Zone Configuration:**
- `type master`: Server is authoritative for these zones
- `file`: Specifies the zone database file location
- `allow-update { none; }`: Disables dynamic DNS updates

**File: `files/dns/bruno.icaro.devops.db`**
- Forward zone database file containing hostname-to-IP mappings
- Defines Start of Authority (SOA) record with serial number and timing parameters
- Sets the Name Server (NS) record pointing to `arq.bruno.icaro.devops`
- Creates A records mapping hostnames to IP addresses:
  - `arq` → 192.168.56.128
  - `db` → 192.168.56.110
  - `app` → 192.168.56.180

**SOA Record Details:**
- `Serial`: 2025082501 (YYYYMMDDNN format for version tracking)
- `Refresh`: 3600 seconds (1 hour) - slave refresh interval
- `Retry`: 1800 seconds (30 minutes) - retry interval after failure
- `Expire`: 604800 seconds (1 week) - zone expiration time
- `Minimum TTL`: 86400 seconds (1 day) - minimum cache time

**File: `files/dns/56.168.192.db`**
- Reverse zone database file containing IP-to-hostname mappings
- Defines SOA and NS records identical to forward zone
- Creates PTR (Pointer) records for reverse DNS lookups:
  - 128 → `arq.bruno.icaro.devops`
  - 110 → `db.bruno.icaro.devops`
  - 180 → `app.bruno.icaro.devops`

**Note on PTR Records:** The records use the last octet of the IP address (e.g., 128 for 192.168.56.128) in the reverse zone file.

### Apache Configuration Files

**File: `files/html/index.html`**
- Replaces the default Apache page with the project description and team members' names and student IDs.

---

## Getting Started

### Prerequisites
Ensure you have the following installed on your host machine:
- **Vagrant** (latest version)
- **VirtualBox** (latest version)
- A pair of SSH keys generated on your machine. The public key must be named `id_ed25519.pub`

### Deployment
1. Clone the repository and navigate into the project directory:
```bash
git clone https://github.com/bruno-dfrs/asa-projeto1.git
cd asa-projeto1/
```

2. Create a python virtual environment and install pip dependencies into it. This will make sure that you don't need to install any pip library locally on your machine.
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

3. Start the environment by running the following command in the project root. This will create and provision all four VMs sequentially:
```bash
vagrant up
```

## Accessing the VMs
### SSH Access
Use SSH commands to access any VM:
```bash
ssh bruno|icaro@192.168.56.128      # arq 
ssh bruno|icaro@192.168.56.110      # db 
ssh bruno|icaro@192.168.56.180      # app 
ssh -X bruno|icaro@192.168.56.50    # cli - address may vary due to DHCP
```

### Web Server Access
1. SSH with X11 forwarding into the `cli` machine:
```bash
ssh -X bruno|icaro@192.168.56.50
```
2. Launch Firefox and navigate to:
- **URL**: `http://app.bruno.icaro.devops`

### NFS Share Verification
From any client VM (`db`, `app`, or `cli`), check the NFS mount:
```bash
touch /var/nfs/test
ls -l /var/nfs/          # Shows files shared from arq server
```

### DNS Resolution Test
Test internal DNS resolution by pinging other VMs by hostname:
```bash
ping arq.bruno.icaro.devops
ping db.bruno.icaro.devops
ping app.bruno.icaro.devops
```
