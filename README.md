# CentOS Stream 10 - Network Bonding Setup Guide

This guide outlines the steps required to configure native kernel network bonding (`active-backup` mode) with a primary slave preference on CentOS Stream 10 using NetworkManager (`nmcli`).

---

## Step 1: VirtualBox Hardware Configuration
1. Power down the virtual machine completely: `poweroff`
2. Open VM Settings -> Network -> Adapter 2.
3. Check "Enable Network Adapter" and set "Attached to: Bridged Adapter".
4. Ensure the name matches Adapter 1, click OK, and boot the VM.

## Step 2: Verify Network Interfaces
```bash
nmcli device status
```

## Step 3: Configure the Network Bond
```bash
nmcli connection delete bond0
nmcli connection add type bond con-name bond0 ifname bond0 bond.options "mode=active-backup"
nmcli connection add type bond-slave con-name bond0-port1 ifname enp0s8 master bond0
nmcli connection add type bond-slave con-name bond0-port2 ifname enp0s9 master bond0
nmcli connection up bond0
```

## Step 4: Configure Primary Slave Preference
```bash
nmcli connection modify bond0 +bond.options "primary=enp0s8"
nmcli connection up bond0
```

## Step 5: Verification
```bash
cat /proc/net/bonding/bond0
```
