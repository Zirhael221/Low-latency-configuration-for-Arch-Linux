A hardcore, kernel-level tuning utility designed to eliminate micro-stutters, bypass hardware interrupts, and minimize jitter for gaming on Linux.

Built specifically for modern Arch Linux and CachyOS systems, this script reaches into the Linux scheduler, memory manager, power management, and I/O subsystems to force your operating system to prioritize game rendering above everything else.

**WARNING:** This script is designed for extreme performance. It disables hardware CPU security mitigations (Spectre/Meltdown) to reclaim raw speed, disables swap subsystems to eliminate page faults, and blinds kernel watchdogs to prevent preemptive game interruptions. Do not run this on a production server without backups.

## Why GameMode and Custom Schedulers are Obsolete

* **Feral GameMode (`gamemoderun`):** GameMode is a user-space daemon that alters CPU governors and adjusts process niceness. This script handles CPU governors, memory limits, and I/O policies directly at the root kernel level. Furthermore, games are launched under **Real-Time Round-Robin priority (`chrt -r 50`)**, which strictly outranks and bypasses standard user-space priority queues. 
  * *Note:* If you still wish to use GameMode for GPU clock locking or screensaver inhibition, you must add `disable_cpu_governor=1` under the `[general]` section in `~/.config/gamemode.ini` to prevent conflicts.
* **Custom CPU Schedulers (BORE, EEVDF, Cachy, CacULE):** Alternative CPU schedulers balance timeslices among competing `SCHED_OTHER` desktop tasks. When game threads run under `SCHED_RR` on isolated cores (`isolcpus`, `nohz_full`, `rcu_nocbs`), standard scheduler algorithms are completely bypassed—the kernel hands CPU cycles directly to the real-time thread.

## Requirements & Conflicts (CachyOS / Arch Users)

Before running the tuner, disable conflicting user-space tuning daemons that alter CPU affinity, interrupt balancing, or power profiles dynamically:

```bash
sudo systemctl disable --now ananicy-cpp irqbalance tuned
```

VM / VFIO Users: This script disables IOMMU (iommu=off) to reduce DMA latency. This will break PCI Passthrough (VFIO) for Virtual Machines. Do not use this script on a host system dedicated to VM GPU passthrough.

### Features & Technical Breakdown
1. Official CachyOS Bootloader Integration (GRUB, Limine, systemd-boot)
What it does: Seamlessly parses and updates GRUB, Limine, and systemd-boot using native CachyOS utilities.
How it works: You explicitly select your bootloader. For systemd-boot, the script hooks directly into /etc/sdboot-manage.conf and executes sdboot-manage gen. For Limine, it edits KERNEL_CMDLINE arrays in /etc/default/limine and natively triggers limine-mkinitcpio. (Fallback logic is included for standard Arch users without these utilities).

3. CPU Core Isolation & Thread Sweeping
What it does: Dedicates target CPU cores exclusively to game threads, while constraining all OS services, systemd units, and driver workqueues to designated "Housekeeping" cores.
Zero Preemption: Isolated cores disable scheduler tick interrupts (nohz_full) and offload RCU callbacks (rcu_nocbs), giving game threads uninterrupted core time.
Dynamic Core Sweep: A dedicated boot service sweeps active and new kernel worker threads back onto housekeeping cores on startup.

5. GPU & Direct Memory Access (DMA / IOMMU)
What it does: Disables IOMMU translation layers (iommu=off, intel_iommu=off / amd_iommu=off) and locks PCIe bus power (pcie_aspm.policy=performance).
Why it matters: Eliminates virtualization and memory security checkpoint overhead for the GPU and storage controllers, allowing bare-metal DMA access directly to RAM. Also injects NVIDIA DRM modesetting (nvidia-drm.modeset=1 nvidia-drm.fbdev=1) to prevent frame sync deadlocks on Optimus laptops.

7. Swap Assassination & Unlimited Memory Locking
What it does: Disables physical swap partitions, ZSWAP, and ZRAM (swapoff -a, masks swap.target), while granting non-root users unlimited real-time and memory locking privileges (rtprio 99, memlock unlimited).
Why it matters: Mathematically eliminates disk page faults and memory paging latency. Assets remain locked in physical RAM during execution and are freed automatically by the kernel when the game closes.

9. Deterministic Power States (C-States & Polling)
What it does: Provides an option for continuous core polling (idle=poll) or deep-sleep suppression (processor.max_cstate=1, intel_idle.max_cstate=1).
Why it matters: Prevents CPU cores from dropping into sleep states (C2–C6), eliminating hardware wake-up latency when processing mouse events or network frames.

10. Network Polling & Virtual Memory Sysctl
What it does: Enables Linux socket busy polling (busy_read=50, busy_poll=50) and tunes dirty memory writeback intervals.
Why it matters: The CPU actively checks network sockets for inbound packets instead of waiting on hardware interrupts, reducing round-trip packet processing delay.

11. Storage I/O Scheduler Assignment
NVMe: Bypasses software queuing entirely (none).
SATA SSDs: Applies low-latency request prioritization (kyber).
Mechanical HDDs: Uses budget-fair queuing to prevent background read starvation (bfq).

### Core Topology Guide (SMT, HT, & AMD CCX)
For the absolute lowest latency, you must understand how your physical CPU cores are wired together.

### Modern Topology Warning
### Modern architectures (Intel 12th-14th Gen P-Cores/E-Cores and AMD Ryzen 7000/9000 series CCD layouts) have complex physical layouts. The old rule of "physical cores first, virtual second" no longer strictly applies. You MUST use lscpu -e or lstopo to verify your exact physical mapping before isolating cores to avoid accidentally isolating an E-core or splitting an L3 cache.

### Hyperthreading (Intel) & SMT (AMD)

Recommendation: Disable HT/SMT in your motherboard BIOS. Virtual threads share the same execution pipeline and L1/L2 cache as their physical counterpart. Disabling them grants your game 100% exclusive access to the physical core's cache and execution engine.
If you cannot disable HT/SMT (Laptops): You MUST pair a physical core and its virtual sibling together. Never put a physical core in Housekeeping and its virtual sibling in Isolated (or vice versa), as they will fight over the same cache.

### AMD Ryzen Architecture (CCX & L3 Cache)
AMD processors group cores into Core Complexes (CCX). Cores inside the same CCX share a massive, ultra-fast L3 cache. If a game has to communicate across two different CCXs, the data travels over the slower Infinity Fabric, introducing latency and micro-stutters.

Recommendation: When choosing Housekeeping and Isolated cores on AMD, isolate an entire CCX for the game and leave the other CCX for the operating system.
Example (Ryzen 9 5900X): This CPU has 12 cores split across two 6-core CCXs. Assign CCX0 (Cores 0,1,2,3,4,5) to Housekeeping. Assign CCX1 (Cores 6-11) to Isolated. Your game now has a massive L3 cache entirely to itself with zero cross-CCX delay.

### Benchmark Proof (Real-Time Latency & Jitter)
60-second Real-Time kernel jitter benchmark (cyclictest) on isolated cores running under Round-Robin (SCHED_RR) priority. (Requires the rt-tests package: sudo pacman -S rt-tests)
   ```
   sudo cyclictest --affinity=1-3 --threads=3 --priority=99 --policy=rr --interval=1000 --duration=60s -m
   ```

Understanding the Metrics:
/dev/cpu_dma_latency set to 0us: Confirms hardware C-state sleep locks are fully enforced.
Avg: 1 µs: Isolated gaming cores respond to real-time wakeups with an average latency of 0.001 ms.
Max: 9 / 6 / 5 µs: Peak worst-case scheduling delay across all cores remained under 0.009 ms under full load, compared to 200–500+ µs typical on stock desktop configurations.

Installation
Download the script directly from the repository using curl:
```
   curl -LO https://raw.githubusercontent.com/Zirhael221/Low-latency-configuration-for-Arch-Linux/main/zxcvcpupartition.txt
```

Move it to your local binary path and make it executable:
```
   sudo mv zxcvcpupartition.txt /usr/local/bin/zxcv.cpu.partition
   sudo chmod +x /usr/local/bin/zxcv.cpu.partition
```
Run the utility as root:
```
sudo zxcv.cpu.partition
```

### Configuration Prompts:
Bootloader Selection: Choose explicitly between GRUB, Limine, or systemd-boot. The script natively uses sdboot-manage and limine-mkinitcpio if running on CachyOS.

Apply or Revert: Select A to apply custom parameters or R to scrub changes and restore stock bootloader/sysctl defaults.

Vendor: Choose Intel (i) or AMD (a) to apply processor-specific register flags.

NVIDIA Optimus: Select y if using hybrid graphics on a laptop.

Disable All Swap: Select y to purge swap/ZRAM (requires adequate physical RAM).

Strict CPU Core Isolation:

Housekeeping Cores **(CRITICAL): Separate cores with commas only (e.g., 0,1), never dashes. The script uses bitwise arithmetic (1 << cpu) to calculate the kernel workqueue hex mask; dashes are parsed as subtraction operators and break mask computation. Isolated cores (which the script hands directly to the kernel) can use dashes (e.g., 1-3,5-7).**

Isolated Cores: Enter your game cores.
Tip: Do not isolate every core. Leave sufficient housekeeping cores for OS tasks to prevent thread contention.

Steam Launch Options
Launch games with non-root Real-Time scheduling privileges:
With Strict Isolation:
```
chrt -r 50 taskset -c <YOUR_ISOLATED_CORES> %command%
```
### Uninstallation
To revert all modifications:
Run sudo zxcv.cpu.partition.
Select your bootloader.
Select R at the tuning prompt.
The script will selectively remove flags from your chosen bootloader, restore sysctl rules, re-enable swap, remove systemd affinity overrides, delete sweep services, and automatically rebuild your boot menu.

### Credits & References
Low Latency System Guide
SUSE CPU Isolation Guide
Erik Rigtorp - Low Latency Tuning
Red Hat Enterprise Linux Real-Time Tuning
TuneD Project
CachyOS and Linux Latency Tuning Communities
