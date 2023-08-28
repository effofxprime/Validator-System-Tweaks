# Validator-sysctl-settings
These are my current sysctl.conf tweaks.

**I offer ***NO*** guarantee that these will work for you. You assume all risks using these**

I highly ***recommend*** that before blinding applying these, as they could utterly fuck your server up, you need to ***research them*** and ensure you will be configuring them properly for your server.
Google is your friend, a plethora of information about everything here will come up on page 1. If you are in need of help still, do not hesitate to submit a question in the Discussions section.


# Instructions

## sysctl.conf
After pasting the tweaks into your `/etc/sysctl.conf`, issue the command `sysctl -p` to apply them.
In your rc.local, add `sysctl -p` on an empty line and save. This ensures they are applied at boot.

To calculate the proper value for `vm.min_free_kbytes` run:
```bash
awk 'BEGIN {OFMT = "%.0f";} /MemTotal/ {print "vm.min_free_kbytes =", $2 * .03;}' /proc/meminfo
```
I took this result and multiplied it by 5 and use that for my setting.


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
