# BIOS Changes

If users ask us to perform BIOS changes for them, we generally grant any reasonable requests for them. Otherwise, it's worth a discussion because some BIOS changes can be fairly destructive.

Who, and how, bios changes should be made depends on the node type and Chameleon site in question. When unclear, defer to the site operator.

## The BIOS Changes Ledger

All BIOS changes must be logged in this spreadsheet, so that we keep track of anything that might need to be rolled back at a later date.

https://docs.google.com/spreadsheets/d/1etR4iVRCqszRbDYibUAJS8JCKXFi7AXWrbrlZt8KVnk/edit?usp=sharing

## Known Changes

The following are changes we've done relatively frequently, and could be candidates for automation.


## To Verify:

Unfortunately, evertthing below here needs verification, given that it references idracadm7.
We should have procedures for either updated idrac, or ideally redfish that can be ported to ironic eventually.


### Hyperthreading/SMT

How to check if enabled:

How to enable:

How to disable:

```
idracadm7 -r <host>-oob -u <username> -p <password> set BIOS.ProcSettings.LogicalProc Enabled
```

Where the host is `impi_address`, username is `ipmi_username`, and password is `ipmi_password` from ironic. You must also add a job to apply this action on reboot:

```
idracadm7 -r <host>-oob -u <username> -p <password> jobqueue create BIOS.Setup.1-1
idracadm7 -r <host>-oob -u <username> -p <password> serveraction hardreset

# The job may sit in the queue for up to 10-30 minutes after this, so be patient
# After reboot completes, verify
idracadm7 -r <host>-oob -u <username> -p <password> get BIOS.ProcSettings.LogicalProc
```

### CPU Frequency Scaling

How to check if enabled:

How to enable:

How to disable:

A user may report that the device files for each core (/sys/devices/system/cpu/cpu0/cpufreq) do not exist. Replace $1 with the node name

```
idracadm7 -r "$1-oob" -u <username> -p <password> set BIOS.SysProfileSettings.SysProfile Custom
idracadm7 -r "$1-oob" -u <username> -p <password> set BIOS.SysProfileSettings.ProcCStates Enabled
idracadm7 -r "$1-oob" -u <username> -p <password> set BIOS.SysProfileSettings.ProcC1E Enabled
idracadm7 -r "$1-oob" -u <username> -p <password> set BIOS.SysProfileSettings.ProcPwrPerf OsDbpm
idracadm7 -r $1-oob -u <username> -p <password> jobqueue create BIOS.Setup.1-1
idracadm7 -r $1-oob -u <username> -p <password> serveraction hardreset
```

### Enable CPU Virtualization

How to check if enabled:

How to enable:

How to disable:

This allows running level 1 hypervisors (KVM) on baremetal

```
racadm --nocertwarn -u <user> -p <pass> -r <idrac_ip> set BIOS.ProcSettings.ProcVirtualization Enabled
racadm --nocertwarn -u <user> -p <pass> -r <idrac_ip> jobqueue create BIOS.Setup.1-1
```

### IOMMU

How to check if enabled:

How to enable:

How to disable:


### SR-IOV

How to check if enabled:

How to enable:

How to disable:

```
# does the OS see that SR-IOV is enabled (run on node)
lspci | grep Mellanox | wc -l # 1 = no, >1 = yes, 0 = ??

. ~/scripts/idrac-functions.sh
# Check current configuration
idrac-cmd-raw c01-40-oob get BIOS.IntegratedDevices.SriovGlobalEnable
# Enable SR-IOV
idrac-cmd-raw c01-40-oob set BIOS.IntegratedDevices.SriovGlobalEnable Enabled
# Schedule a job to commit change & reboot the node
idrac-cmd-raw c01-40-oob jobqueue create BIOS.Setup.1-1 -r pwrcycle -s TIME_NOW
# Monitor the execution of the job
idrac-cmd-raw c01-40-oob jobqueue view

# If jobqueue gets stuck:
idrac-cmd-raw c01-40-oob jobqueue delete -i JID_CLEARALL_FORCE
sleep 120
idrac-cmd-raw c01-40-oob racreset
```

On the node, when booted, the instructions [here](https://community.mellanox.com/docs/DOC-2368#jive_content_id_Enable_SRIOV_on_the_MLNX_OFED_Driver) can be used to enable SR-IOV in the NIC itself.


### Enable/Disable PXE boot for Network Interfaces

How to check if enabled:

How to enable:

How to disable:

Some nodes will be unable to provision because more than one PXE device i-rs enabled in the BIOS. Check available interfaces:

```
racadm --nocertwarn -u <user> -p <pass> -r <idrac_ip> get nic.NICConfig.1.LegacyBootProto
racadm --nocertwarn -u <user> -p <pass> -r <idrac_ip> get nic.NICConfig.2.LegacyBootProto
racadm --nocertwarn -u <user> -p <pass> -r <idrac_ip> get nic.NICConfig.3.LegacyBootProto
racadm --nocertwarn -u <user> -p <pass> -r <idrac_ip> get nic.NICConfig.4.LegacyBootProto
```

Disable a particular interface:

```
racadm --nocertwarn -u <user> -p <pass> -r <idrac_ip> set nic.NICConfig.4.LegacyBootProto NONE
```

Apply the change:

```
racadm --nocertwarn -u <user> -p <pass> -r <idrac_ip> jobqueue create NIC.<nic>.<#-#-#>
```

By default, this change will be applied on the next reboot. To force a reboot, append `-r pwrcycle` to the end of the previous command
