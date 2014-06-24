---
id: "2014-06-24-linux-defaults"
title: "Improved default settings for Linux machines"
abstract: "A quick overview of Linux settings I change on almost every machine I manage."
tags: ["linux", "tuning"]
pubdate: 2014-06-24T18:30:00Z
---

I get asked about my default settings for Linux fairly frequently and was writing this in an email
and decided to post it for broader use. If you have better recommendations, by all means please
send me a pull request. The Edit button at the top of this page will get you there.

There are a couple groups of settings below. The first couple go in /etc/sysctl.conf or /etc/sysctl.d/filename.conf. I've
applied most of these to hundreds of machines and never had an issue. That said, test in non-production first! I run the
same settings across pretty much every Linux machine I touch, including laptops, Intel NUCs, Xeon workstations, and huge
NUMA servers. There's more to be done for each case to get the best performance, but I think this is where almost every
machine should start.

Edit: I've run across a few comments complaining about these large max values. The reason I set them
high is that the machines I work on are not multi-user in any way. These settings would be insane on
a shared university machine, but for a Linux workstation or server, there's only one
user/application that pretty much never wants or expects to hit the ridiculously low default limits.
As I mentioned, I've run these
settings on likely thousands of machines over the last few years and have never seen them cause
an issue; in fact quite the opposite happens: users are not surprised by silly limits like 1024
file handles or applications going away for non-deterministic amounts of time while the kernel
fetches application pages from swap that were only swapped out to make room for VFS.

```
# tell the kernel to only swap if it really needs it, rather than doing it all the time
# as of Linux >= 3.5 this disables swap entirely, which I think is virtuous
# set vm.swappiness=1 if you want swap available for overflow
vm.swappiness = 0
# increase the number of allowed mmapped files
vm.max_map_count = 1048576
# increase the number of file handles available globally
fs.file-max = 1048576
# increase the number of sysv ipc slots for each type
kernel.shmmax = 65536
kernel.msgmax = 65536
kernel.msgmnb = 65536
# allow up to 999999 processes with corresponding pids
# this looks nice and rarely rolls over - I've never had a problem with it.
kernel.pid_max = 999999 # unnecessary, but I like it
# panic if the kernel hangs for more than 300 seconds
kernel.panic = 300

# do not enable if your machines are not physically secured
# this can be used to force reboots, kill processses, cause kernel crashes, etc without logging in
# but it's very handy when a machine is hung and you need to get control
# that said, I always enable it
kernel.sysrq = 1
```

These are improved defaults for opening up the Linux network stack. I recommend Googling "Linux C10k" to learn more about what they mean.

```
net.ipv4.ip_local_port_range = 10000 65535
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 2500
net.core.somaxconn = 65000
```

These are some more advanced settings to control how much written data can be held in RAM before flushing to disk. These are generally safe to apply, but going crazy with numbers can (easily) adversely affect performance. I prefer a fairly low dirty_background setting to make sure IO
doesn't get backed up. Setting these numbers really high can be useful for large file transfers that are smaller than RAM, but eventually
you pay the cost of flushing to disk, so I don't recommend going crazy.

See Also:

<ul>
    <li><a href="https://github.com/torvalds/linux/blob/master/Documentation/sysctl/vm.txt">linux/Documentation/sysctl/vm.txt</a></li>
    <li><a href="https://lkml.org/lkml/2008/11/23/160">LKML post introducing dirty_bytes and dirty_background_bytes</a></li>
</ul>

```
# these will need local tuning, currently set to start flushing dirty pages at 256MB
# writes will start blocking at 2GB of dirty data, but this should only ever happen if
# your disks are far slower than your software is writing data
# If you have an older kernel, you will need to s/bytes/ratio/ and adjust accordingly.
vm.dirty_background_bytes = 268435456
vm.dirty_bytes = 1073741824
```

Finally, I think the whole pam limits business is useless on single-user systems (e.g. workstations, database servers), so I effectively disable it.
Put this /etc/security/limits.conf or /etc/security/limits.d/disable.conf (depending on your distro & preferences):

```
* - nofile     1048576
* - memlock    unlimited
* - fsize      unlimited
* - data       unlimited
* - rss        unlimited
* - stack      unlimited
* - cpu        unlimited
* - nproc      unlimited
* - as         unlimited
* - locks      unlimited
* - sigpending unlimited
* - msgqueue   unlimited
```

Once these settings are applied, a lot of issues go away. Please let me know if you run into any issues
either on Twitter as <a href="https://twitter.com/AlTobey">@AlTobey</a> or via email at
<a href="mailto:tobert@gmail.com">tobert@gmail.com</a>.
