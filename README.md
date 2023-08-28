# Validator-sysctl-settings
These are my current sysctl.conf tweaks.

**I offer ***NO*** guarantee that these will work for you. You assume all risks using these**

I highly ***recommend*** that before blinding applying these, as they could utterly fuck your server up, you need to ***research them*** and ensure you will be configuring them properly for your server.
Google is your friend, a plethora of information about everything here will come up on page 1. If you are in need of help still, do not hesitate to submit a question in the Discussions section.

- [Validator-sysctl-settings](#validator-sysctl-settings)
- [Instructions](#instructions)
  - [Linux Kernel](#linux-kernel)
  - [Grub](#grub)
  - [sysctl.conf](#sysctlconf)
  - [rc.local](#rclocal)
  - [service file template](#service-file-template)
    - [Optional](#optional)
  - [Blockchain config's](#blockchain-configs)

# Instructions

## Linux Kernel
I am using Ubuntu 22.04 LTS with a Pro subscription.  The Pro subscription is free.  You can have up to 5 servers added.
https://ubuntu.com/pro

There are some instructions on the page once you sign up to link your server to the subscription.
Once linked you now have access to the LTS realtime Kernel.  This is beneficial for us because it's supported in the LTS release.
Enable this by doing
```bash
pro enable realtime-kernel
```

## Grub
Once you have the realtime-kernel installed, you can update your grub config to take more advantage of the kernel.
We need to modify the parameter starting with `GRUB_CMDLINE_LINUX_DEFAULT=`
If you have settings in there already, put these at the end. Keeping a space between settings.
```bash
skew_tick=1 rcu_nocb_poll rcu_nocbs=1-35,37-71 nohz=on nohz_full=1-35,37-71 kthread_cpus=0,36 irqaffinity=0,36 isolcpus=managed_irq,domain,1-35,37-71 intel_pstate=disable nosoftlockup tsc=nowatchdog
```

I have purposely left ***MY*** settings here to demonstrate what you will need to do to use this configuration.

First, the goal here is to assign a cpu or two to handle certain tasks and restrict them so that they cannot be used for anything else. This happens with:
- kthread_cpus
- irqaffinity

My server is a dual processor.  Each physical processor has 36 cpus.  What you want to do is take the first cpu from each physical processor, in my case 0 & 36.  CPU counts start from 0, not 1. Because we used 1 CPU from each Processor, we wont "overload" one with all the work.

Now that you've determined which CPU's you will dedicate to the above two items, you can then fill out your remaining CPUs in:
- rcu_nocbs
- nohz_full
- isolcpus

1-35,37-71 translates to, cpu's 1 through 35 and 37 through 71.  Adjust this to fit your CPU count.

## sysctl.conf
After pasting the tweaks into your `/etc/sysctl.conf`, issue the command `sysctl -p` to apply them.
In your rc.local, add `sysctl -p` on an empty line and save. This ensures they are applied at boot.

To calculate the proper value for `vm.min_free_kbytes` run:
```bash
awk 'BEGIN {OFMT = "%.0f";} /MemTotal/ {print "vm.min_free_kbytes =", $2 * .03;}' /proc/meminfo
```
I took this result and multiplied it by 5 and use that for my setting.

There are quite a few items commented out. Those do not exist on my system but may on yours.  You can always check using the `sysctl` command.

## rc.local
A few other tweaks can be made and applied at boot using the `rc.local` file.  By default, Ubuntu LTS does not have an rc.local enabled.  My current `rc.local` which includes a link for instructions on setting up your `rc.local` as well as the contents of what you need to have in your `rc-local.service` file

You will need to modify a few things here:
- My eth device is labeled `enpls0fN` where N={1,2..N}.  You will need to change this to match your eth device names
- My drive names all start with `nvme`.  Yours may be different as well as the number of them. If you have more than four drives, you will want to change `{0..4}` and change `4` to the number of drives you have. Secondly, change `nvme${i}n1` to match your device names. For example, if your devices are /dev/{sda1,sdb1,sdc1} then you would modify the end of the command to `/dev/sd*${i}`.  Setting the read ahead on the top-level device propagates it down to the lowest sub-device(all your partitions/extend partitions/lvm/mdadm).
- First figure out how many CPU's you have. This should start at 0, not 1. Mine are `{0..71}`, so if you have 32 cpus you would need `{0..31}`.

## service file template
You will need to change a few things here. I've labeled items needing to be changed with `blockchain`.  As well as heavy comments.
- Description
- User/Group
- Working Directory
- ExecStart, location, daemon name
- Nice
- CPUAffinity (how many CPUs do you want this service to be restricted to use?)
- IOSchedulingPriority:  1-highest, 7-lowest

Double-check all your paths. I would have a second ssh session open that is purely running `journalctl` to view logs as your restart your nodes after making changes.

### Optional
I use sockets over TCP. I store the sockets into shared memory `/dev/shm`.  You can find out how to set your `/etc/fstab` to mount a shared memory drive.
My `ExecStart{Pre,Post}` executes a script to help ensure permissions are set appropriately and that the socket is properly taken care of.
`Sockets` is required if you use them, as it allows systemd to know the process that creates it and how it needs to handle it given certain situations.
Setting the group to `www-data` is more for an RPC node using sockets.  This is so Nginx can access the socket for a reverse proxy.


For the Pre and Post scripts, I recommend keeping them in the users `$HOME/.local/bin` and ensure that in your `.profile` that is an added `PATH`.
After downloading them, rename them to be more descriptive of what blockchain they're for. IE: `pre-checks-starsd.sh`
Then uncomment the lines starting with:
- ExecStart{Pre,Post}
- Sockets

## Blockchain config's
I've added comments in the file to show which config file I am speaking about.  I've removed sections that aren't relevant to the tweaking they should not be removed.  Just alter the single config line setting.


In short, the recommendation is to turn off everything a validator doesn't need.
- Reduce timeouts/threshholds/deltas
- Turn off indexing
- Increase allowed send/receive rates
- State Sync chunk quantity
- No index
- !!!!!!!Double sign check height!!!!!!!!

There are a few others.  Check out the discussions and share your experiences so we can improve upon them for everyone.
