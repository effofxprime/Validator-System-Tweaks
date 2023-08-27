# Validator-sysctl-settings
These are my current sysctl.conf tweaks.

**I offer ***NO*** guarantee that these will work for you. You assume all risks using these**

I highly ***recommend*** that before blinding applying these, as they could utterly fuck your server up, you need to ***research them*** and ensure you will be configuring them properly for your server.
Google is your friend, a plethora of information about everything here will come up on page 1. If you are in need of help still, do not hesitate to submit a question in the Discussions section.


## Instructions
After pasting the tweaks into your `/etc/sysctl.conf`, issue the command `sysctl -p` to apply them.
In your rc.local, add `sysctl -p` on an empty line and save. This ensures they are applied at boot.

To calculate the proper value for `vm.min_free_kbytes` run:
```bash
awk 'BEGIN {OFMT = "%.0f";} /MemTotal/ {print "vm.min_free_kbytes =", $2 * .03;}' /proc/meminfo
```

## rc.local

A few other tweaks can be made and applied at boot using the `rc.local` file.  By default, Ubuntu LTS does not have an rc.local enabled.  Below contains my current `rc.local` which includes a link for instructions on setting up your `rc.local` as well as the contents of what you need to have in your `rc-local.service` file

```bash
#!/bin/bash
#set -x
# https://www.linuxbabe.com/linux-server/how-to-enable-etcrc-local-with-systemd
############## Service file start ####################
#[Unit]
# Description=/etc/rc.local Compatibility
# ConditionPathExists=/etc/rc.local
#
#[Service]
# Type=forking
# ExecStart=/etc/rc.local start
# TimeoutSec=0
# StandardOutput=tty
# RemainAfterExit=yes
# SysVStartPriority=99
#
#[Install]
# WantedBy=multi-user.target
############## Service file end ####################

# Increase your transaction queue length
# and increase the eth device tx/rx buffers
# Does not work with linode para-virt, must use full-virt
ip link set dev enp1s0f0 txqueuelen 9000
ip link set dev enp1s0f1 txqueuelen 9000

ethtool -G enp1s0f0 tx 4096 rx 4096
ethtool -G enp1s0f1 tx 4096 rx 4096

#Set Read ahead
for i in {0..4}
do
        blockdev --setra 4096 /dev/nvme${i}n1

done

# Ensure sysctl settings are set
sysctl -p

# Set CPU performance
for i in {0..71}
do
    echo -n "performance" > /sys/devices/system/cpu/cpu$i/cpufreq/scaling_governor
    # Change all CPU's to 'performance'
    echo -n 2 > /sys/devices/system/cpu/cpu$i/power/energy_perf_bias
    # Change to 0-4, 0=Performance and 4=Balanced-Performance
done

exit 0
```

You will need to modify a few things here:
- My eth device is labeled `enpls0fN` where N={1,2..N}.  You will need to change this to match your eth device names
- My drive names all start with `nvme`.  Yours may be different as well as the number of them. If you have more than four drives, you will want to change `{0..4}` and change `4` to the number of drives you have. Secondly change `nvme${i}n1` to match your device names. For example, if your devices are /dev/{sda1,sdb1,sdc1} then you would modify the end of the command to `/dev/sd*${i}`.  Setting the read ahead on the top level device propigates it down to the lowest sub-device(all your partitions/extend partitions/lvm/mdadm).
- First figure out how many CPU's you have. This should start at 0, not 1. Mine are `{0..71}`, so if you have 32 cpus you would need `{0..31}`.

