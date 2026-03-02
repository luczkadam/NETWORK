RHEL/CentOS Network Migration Scripts
A set of Bash scripts designed to simplify network recovery and reconfiguration for RHEL-based systems (like CentOS, RHEL 6, and RHEL 7 using network-scripts) after migrating Virtual Machines between different hypervisors (e.g., VMware to Proxmox).

During such migrations, MAC addresses and interface names often change (e.g., from eth0 to ens3), leaving the VM disconnected from the network. These scripts provide a two-step solution to regain access and properly set up the environment.

Script 1: MGMT Network Access (mgmt.sh)
Purpose
To quickly regain SSH access to a migrated Virtual Machine without manually configuring interfaces from the hypervisor's console.

How it works
When executed via the VM console, this script:

Creates a timestamped backup of the current network configuration in /etc/sysconfig/network-scripts/OLD_INTERFACE/.

Automatically detects all physical network interfaces (ignoring lo, docker, virbr, etc.).

Converts the provided CIDR prefix into a standard decimal netmask.

Generates a basic ifcfg-* configuration file for every detected interface, assigning them all the exact same Management IP address and binding their new MAC addresses (HWADDR).

Disables NetworkManager control (NM_CONTROLLED=no) for these interfaces.

Restarts the network service.

Note: Assigning the same IP to all interfaces acts as a "lifeline" ensuring that regardless of which virtual NIC is connected to the Management switch, the server will be reachable via SSH.

Usage
Bash

# Syntax: ./mgmt.sh <IP_ADDRESS> <CIDR_PREFIX>
./mgmt.sh 192.168.1.92 24
Script 2: Interface Configurator (interface.sh)
Purpose
To permanently and correctly configure all network interfaces (MGMT, PROD, NAS) once SSH access has been restored using the mgmt.sh script.

How it works
This script runs interactively, looping through every detected physical interface and prompting the administrator to assign a specific role to it. It displays the current MAC address and currently assigned IP to help identify the port.

Available roles:

[1] MGMT (Management): Skips configuration, preserving the lifeline IP assigned by the MGMT script.

[2] PROD (Production): * Prompts for IP, CIDR Prefix, and an optional Gateway.

Automatically calculates the Netmask.

Dynamic DNS Injection: Automatically assigns specific DNS servers based on the starting octets of the provided IP address (e.g., IPs starting with 192.1* receive 10.222.10.10, while IPs starting with 192.2* receive 20.222.10.10).

Appends predefined search domains.

[3] NAS (Storage): Prompts for IP and Prefix only. Configures the interface without Gateway or DNS.

[4] SKIP: Ignores the interface entirely.

After all interfaces are processed, the script asks for confirmation to restart the network service and apply the new topology.

Usage
Bash

./interface.sh
