0. Introduction
===============

This document describes the software setup steps and usage hints that can be
helpful in setting up a system to use tiered memory for evaluation.

The document will go over:
1. Any kernel config options required to enable the tiered memory features
2. Any additional userspace tooling required, and related instructions
3. Any post-boot setup for configurable or tunable knobs

Note:
Any instructions/settings described here may be tailored to the branch this
is under. Setup steps may change from release to release, and for each
release branch, the setup document accompanying that branch should be
consulted.

1. Kernel build and configuration
=================================

a. The recommended starting point is a distro-default kernel config. We
   use and recommend using a recent Fedora config for a starting point.

b. Ensure the following:
   CONFIG_DEV_DAX_KMEM=m
   CONFIG_MEMORY_HOTPLUG_DEFAULT_ONLINE=n
   NUMA_BALANCING=y


2. Tooling setup
================

a. Install 'ndctl' and 'daxctl' from your distro, or from upstream:
   https://github.com/pmem/ndctl
   This may especially be required if the distro version of daxctl
   is not new enough to support the daxctl reconfigure-device command[1]

   [1]: https://pmem.io/ndctl/daxctl-reconfigure-device.html

b. Assuming that persistent memory devices are the next demotion tier
   for system memory, perform the following steps to allow a pmem device
   to be hot-plugged system RAM:

   First, convert 'fsdax' namespace(s) to 'devdax':
     ndctl create-namespace -fe namespaceX.Y -m devdax

c. Reconfigure 'daxctl' devices to system-ram using the kmem facility:
     daxctl reconfigure-device -m system-ram daxX.Y.

   The JSON emitted at this step contains the 'target_node' for this
   hotplugged memory. This is the memory-only NUMA node where this
   memory appears, and can be used explicitly with normal libnuma/numactl
   techniques.

d. Ensure the newly created NUMA nodes for the hotplugged memory are in
   ZONE_MOVABLE. The JSON from daxctl in the above step should indicate
   this with a 'movable: true' attribute. Based on the distribution, there
   may be udev rules that interfere with memory onlining. They may race to
   online memory into ZONE_NORMAL rather than movable. If this is the case,
   find and disable any such udev rules.

3. Post boot setup
==================

a. Enable node-reclaim for cold page demotion
   After the device-dax instances are onlined node-reclaim needs to be
   enabled to start migrating “cold” pages from DRAM to PMEM.
    # echo 15 > /proc/sys/vm/zone_reclaim_mode

b. Enable 'NUMA balancing' for promotion
    # echo 2 > /proc/sys/kernel/numa_balancing
    # echo 30 > /proc/sys/kernel/numa_balancing_rate_limit_mbps

4. Promotion/demotion statistics
==================

   The number of promoted pages can be checked by the following counters in
   /proc/vmstat or /sys/devices/system/node/node[n]/vmstat:
      pgpromote_success

   The number of pages demoted can be checked by the following counters:
      pgdemote_kswapd
      pgdemote_direct

   The page number of failure in promotion could be checked by the
   following counters:
      pgmigrate_fail_dst_node_fail
      pgmigrate_fail_numa_isolate_fail
      pgmigrate_fail_nomem_fail
      pgmigrate_fail_refcount_fail
