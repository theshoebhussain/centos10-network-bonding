# Enterprise Linux 10 (RHEL/CentOS Stream) Network Bonding Guide

This repository contains infrastructure documentation and generic deployment code for configuring native Linux kernel network bonding using NetworkManager (`nmcli`). 

## Architecture Overview
Starting with Enterprise Linux 9 and 10, the legacy team daemon (`teamd`) has been deprecated and completely removed. High-availability link aggregation must now be handled via native kernel bonding drivers managed by NetworkManager.

This runbook focuses on the **Active-Backup (Mode 1)** configuration pattern. It provides automatic fault tolerance by ensuring a secondary standby interface immediately intercepts traffic if the primary path loses physical carrier link status.

---

## Generic Deployment Runbook

### 1. Variables Definition
Before executing, identify the target physical network interfaces on the host system:
* **Master Interface Name:** `bond0`
* **Primary Interface Interface (Active):** Replace `INTERFACE_PRIMARY` (e.g., `eth0`, `enp0s8`)
* **Secondary Interface Interface (Backup):** Replace `INTERFACE_BACKUP` (e.g., `eth1`, `enp0s9`)

---

### 2. Implementation Commands

Execute these steps as a privileged user (`root` or via authorized `sudo` profiles):

#### Step A: Initialize Infrastructure Environments
Clean up any conflicting network profiles on the target interfaces to avoid state synchronization deadlocks:
```bash
# Delete existing master profiles if tearing down a broken implementation
nmcli connection delete bond0 2>/dev/null || true
```

#### Step B: Instantiate the Bond Master Controller
Create the logical bonding interface specifying the fault-tolerance configuration parameters:
```bash
nmcli connection add type bond \
  con-name bond0 \
  ifname bond0 \
  bond.options "mode=active-backup,miimon=100"
```
*Note: `miimon=100` configures the kernel driver to inspect physical link state changes every 100 milliseconds.*

#### Step C: Provision the Subordinate Slave Interfaces
Bind your physical network infrastructure cards under the master bond interface controller. 

*(Replace `INTERFACE_PRIMARY` and `INTERFACE_BACKUP` with your environment's explicit device outputs from `nmcli device status`)*:

```bash
# Attach Primary Interface Link
nmcli connection add type bond-slave \
  con-name bond0-port1 \
  ifname INTERFACE_PRIMARY \
  master bond0

# Attach Secondary Interface Link
nmcli connection add type bond-slave \
  con-name bond0-port2 \
  ifname INTERFACE_BACKUP \
  master bond0
```

#### Step D: Establish Primary Fallback Preferences
Configure the kernel cluster policy to ensure traffic explicitly reverts back to your high-performance primary adapter whenever its physical carrier link is healthy:
```bash
nmcli connection modify bond0 +bond.options "primary=INTERFACE_PRIMARY"
```

#### Step E: Interface Activation
Commit changes and force NetworkManager to bind the hardware components and acquire network addressing:
```bash
nmcli connection up bond0
```

---

## Post-Deployment Validation

### Runtime Kernel Telemetry Inspection
Read the real-time operational state metrics managed directly within the kernel space:
```bash
cat /proc/net/bonding/bond0
```

### Expected Production Metrics:
* **Bonding Mode:** `fault-tolerance (active-backup)`
* **Primary Slave:** Your explicit primary interface name
* **Currently Active Slave:** Your explicit primary interface name
* **MII Status:** `up` (Must be validated across the master controller and all subordinate slave blocks)
