## Update: 2019-05-13

TL;DR: ACPI error workaround as of May 13th, add `pci=nommconf`.

Small update on the endless ACPI errors. The issue appears to be related to
trying to increase the io size allocated to a specific pci hotplug device on
the thunderbolt 3 bridge. What I'm seeing is consistent with the results of the
git bisect I did earlier that added a call to reallocate unassigned bridge
resources; a change introduced in that commit tries to optimize the layout of
memory and io across a bridge, and allocates more resources to hotplug buses
that would otherwise go to waste.

When the above is performed in combination with MMCONFIG/MCFG (memory-mapped
config space/table provided by motherboard (firmware, bios?), some kind of
corruption occurs that immediately begins spamming the ACPI interrupt handler
with junk events; this may be due to memory corruption/offset errors, but I'm
not sure yet. They don't appear to be valid events; they seem to be randomized
and each event has flags set on them that should make it impossible to reach
the code paths that are generating the errors due to guards on the dispatch
side. ACPI errors have their event's dispatch type set to none, which should
never happen...it's not the GPE XX that's the problem, they're just bogus.

I'm still trying to narrow the issue further to see if it's a kernel bug or on
Intel/LGs side. Unfortunately, I won't have more time for several days. Most
likely the issue is with the MMCONFIG table, or an assumption the kernel is
making that it can grow into specific memory addresses that are actually
reserved.

## Overview

_(outdated, see update above)_

LG Gram 17 laptops have trouble booting linux with kernels newer than 4.17.

A `pci=noacpi` kernel parameter can be provided, which does allow booting with
newer kernels, but you lose the ability use the trackpad or suspend.

Older kernels at or before 4.17 can boot without the `pci=noacpi` flag, but
cannot see the trackpad and lack updates to the `lg_laptop` kernel module.

I've identified a workaround that will allow the laptop to run with the latest
linux kernel. I'm still working on identifying the specific line(s) that
trigger the errors so I can submit an upstream patch or file a proper report.


## Errors

When booting a newer kernel, the following messages will be displayed immediately
and repeat indefinitely. A kernel process will take up 100% CPU and, after a
couple of minutes, the system will shut down. If you boot into a working system,
gigabytes of data will be written to the logs.

Logs are not limited to the following GPE numbers, this is just a sample; there
are dozens of different numbers errors for other fixed events:
```
May 05 02:48:07 eugenia systemd-journald[507]: Missed 5 kernel messages
May 05 02:48:07 eugenia kernel: ACPI Error: No handler or method for GPE 27, disabling event (20181213/evgpe-839)
May 05 02:48:07 eugenia systemd-journald[507]: Missed 3 kernel messages
May 05 02:48:07 eugenia kernel: ACPI Error: No handler or method for GPE 54, disabling event (20181213/evgpe-839)
May 05 02:48:07 eugenia systemd-journald[507]: Missed 4 kernel messages
May 05 02:48:07 eugenia kernel: ACPI Error: No handler or method for GPE 63, disabling event (20181213/evgpe-839)
May 05 02:48:07 eugenia systemd-journald[507]: Missed 4 kernel messages
May 05 02:48:07 eugenia kernel: ACPI Error: No installed handler for fixed event - SleepButton (3), disabling (20181213/evevent-257)
May 05 02:48:07 eugenia systemd-journald[507]: Missed 3 kernel messages
May 05 02:48:07 eugenia kernel: ACPI Error: No handler or method for GPE 0A, disabling event (20181213/evgpe-839)
May 05 02:48:07 eugenia systemd-journald[507]: Missed 3 kernel messages
May 05 02:48:07 eugenia kernel: ACPI Error: No handler or method for GPE 0E, disabling event (20181213/evgpe-839)
May 05 02:48:07 eugenia systemd-journald[507]: Missed 5 kernel messages
May 05 02:48:07 eugenia kernel: ACPI Error: No handler or method for GPE 16, disabling event (20181213/evgpe-839)
May 05 02:48:07 eugenia systemd-journald[507]: Missed 2 kernel messages
```


## Issue Identified

_(outdated, see update above)_

Because the laptop is known to boot with older kernels, I was able to track
down the commit that introduced the acpi errors through `git bisecct` and
recompiling a minimal kernel.

The commit that started logging errors is [84c8b58ed3addf17d3beb2e5037b001ffa65c5ef](https://github.com/torvalds/linux/commit/84c8b58ed3addf17d3beb2e5037b001ffa65c5ef):

```
ACPI / hotplug / PCI: Don't scan bridges managed by native hotplug

When acpiphp re-enumerates a PCI hierarchy because of an ACPI Notify()
event, we should skip bridges managed by native hotplug (pciehp or shpchp).
We don't want to scan below a native hotplug bridge until the hotplug
controller generates a hot-add event.
```

The description is consistent with the parts of the kernel raising the errors.


## Patching / "I need a workaround!"

_(outdated, see update above)_

Performing a `git revert 84c8b58ed3addf17d3beb2e5037b001ffa65c5ef` against
the latest linux kernel checkout does not apply cleanly, resulting in a
single conflict. The conflict is minor and easy to finish manually.

### Vanilla Kernel

A patch file has been included in this repo that applies cleanly to the
v5.1 release and HEAD of git master branch (as of 2019-05-10).

### Arch Linux

A PKGBUILD and package config snapshot are available in the aur/ directory.
Running the following command inside the `aur/linux-mainline` directory should
compile a version of the 5.1 mainline kernel with the patch applied:

```
MAKEFLAGS="--jobs=$(nproc)" makepkg -s
```

Three files should be generated with `.tar.xz` extensions that you can then
install using `sudo pacman -U linux-mainline-*.tar.xz`.

This will install a copy of the kernel to `/boot/vmlinuz-linux-mainline`.
Depending on how your bootloader is set up, you may need to perform additional
steps to get the kernel to show up in your boot loader, consult instructions at
[https://wiki.archlinux.org/index.php/Arch_boot_process#Boot_loader].

Pre-built kernel modules are also included in this repo under `arch-pkgs/`,
but I'm a stranger on the internet offering unsigned packages... It's a bad
idea to accept unsigned binaries from strangers, so first try to build your
own using the instructions above if you can.


## Root Cause

I'm still working on identifying the exact cause. Check back later.
