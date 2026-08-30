# Low-latency-configuration-for-Arch-Linux
Collection of parameters and configurations to tune Arch Linux and CachyOS for ultra-low latency, deterministic thread scheduling, and zero OS jitter in competitive gaming.

# zxcv.cpu.partition

A hardcore, kernel-level tuning utility designed to eliminate micro-stutters, bypass hardware interrupts, and minimize jitter for gaming on Linux.
FPS and Computational performance are not a metric considered in this configuration. It may or may not increase or decrease your fps and computational power. Latency and jitter are the primary considerations here.

Built specifically for modern Arch Linux and CachyOS systems, this script reaches into the Linux scheduler, memory manager, power management, and I/O subsystems to force your operating system to prioritize game rendering above everything else.

⚠️ **WARNING:** This script is designed for extreme performance. It disables hardware CPU security mitigations (Spectre/Meltdown) to reclaim raw speed, disables swap subsystems to eliminate page faults, and blinds kernel watchdogs to prevent preemptive game interruptions. Do not run this on a production server without backups.


## 🛑 Why GameMode and Custom Schedulers are Obsolete

* **Feral GameMode (`gamemoderun`):** GameMode is a user-space daemon that alters CPU governors and adjusts process niceness. This script handles CPU governors, memory limits, and I/O policies directly at the root kernel level. Furthermore, games are launched under **Real-Time Round-Robin priority (`chrt -r 50`)**, which strictly outranks and bypasses standard user-space priority queues. Running GameMode simultaneously creates redundant overhead and conflicting thread management.
* **Custom CPU Schedulers (BORE, EEVDF, Cachy, CacULE):** Alternative CPU schedulers balance timeslices among competing `SCHED_OTHER` desktop tasks. When game threads run under `SCHED_RR` on isolated cores (`isolcpus`, `nohz_full`, `rcu_nocbs`), standard scheduler algorithms are completely bypassed—the kernel hands CPU cycles directly to the real-time thread.


## ⚠️ Requirements & Conflicts (CachyOS / Arch Users)

Before running the tuner, disable conflicting user-space tuning daemons that alter CPU affinity, interrupt balancing, or power profiles dynamically:

```bash
sudo systemctl disable --now ananicy-cpp irqbalance tuned
```


## 🚀 Features & Technical Breakdown

### 1. CPU Core Isolation & Thread Sweeping
* **What it does:** Dedicates target CPU cores exclusively to game threads, while constraining all OS services, systemd units, and driver workqueues to designated "Housekeeping" cores.
* **Zero Preemption:** Isolated cores disable scheduler tick interrupts (`nohz_full`) and offload RCU callbacks (`rcu_nocbs`), giving game threads uninterrupted core time.
* **Dynamic Core Sweep:** A dedicated boot service sweeps active and new kernel worker threads back onto housekeeping cores on startup.

### 2. GPU & Direct Memory Access (DMA / IOMMU)
* **What it does:** Disables IOMMU translation layers (`iommu=off`, `intel_iommu=off` / `amd_iommu=off`) and locks PCIe bus power (`pcie_aspm.policy=performance`).
* **Why it matters:** Eliminates virtualization and memory security checkpoint overhead for the GPU and storage controllers, allowing bare-metal DMA access directly to RAM. Also injects NVIDIA DRM modesetting (`nvidia-drm.modeset=1 nvidia-drm.fbdev=1`) to prevent frame sync deadlocks on Optimus laptops.

### 3. Swap Assassination & Unlimited Memory Locking
* **What it does:** Disables physical swap partitions, ZSWAP, and ZRAM (`swapoff -a`, masks `swap.target`), while granting non-root users unlimited real-time and memory locking privileges (`rtprio 99`, `memlock unlimited`).
* **Why it matters:** Mathematically eliminates disk page faults and memory paging latency. Assets remain locked in physical RAM during execution and are freed automatically by the kernel when the game closes.

### 4. Deterministic Power States (C-States & Polling)
* **What it does:** Provides an option for continuous core polling (`idle=poll`) or deep-sleep suppression (`processor.max_cstate=1`, `intel_idle.max_cstate=1`).
* **Why it matters:** Prevents CPU cores from dropping into sleep states (C2–C6), eliminating hardware wake-up latency when processing mouse events or network frames.

### 5. Network Polling & Virtual Memory Sysctl
* **What it does:** Enables Linux socket busy polling (`busy_read=50`, `busy_poll=50`) and tunes dirty memory writeback intervals.
* **Why it matters:** The CPU actively checks network sockets for inbound packets instead of waiting on hardware interrupts, reducing round-trip packet processing delay.

### 6. Storage I/O Scheduler Assignment
* **NVMe:** Bypasses software queuing entirely (`none`).
* **SATA SSDs:** Applies low-latency request prioritization (`kyber`).
* **Mechanical HDDs:** Uses budget-fair queuing to prevent background read starvation (`bfq`).


## 📊 Benchmark Proof (Real-Time Latency & Jitter)

60-second Real-Time kernel jitter benchmark (`cyclictest`) on isolated cores running under **Round-Robin (`SCHED_RR`)** priority:

**The Command:**
```bash
sudo cyclictest --affinity=1-3 --threads=3 --priority=99 --policy=rr --interval=1000 --duration=60s -m
```

**The Output:**
```text
/dev/cpu_dma_latency set to 0us
policy: rr: loadavg: 0.47 1.01 0.76 1/409 4079

T: 0 ( 4063) P:99 I:1000 C:  59996 Min:      1 Act:    1 Avg:    1 Max:       9
T: 1 ( 4064) P:99 I:1500 C:  39997 Min:      1 Act:    1 Avg:    1 Max:       6
T: 2 ( 4065) P:99 I:2000 C:  29998 Min:      1 Act:    1 Avg:    1 Max:       5
```

### Understanding the Metrics:
* **/dev/cpu_dma_latency set to 0us:** Confirms hardware C-state sleep locks are fully enforced.
* **Avg: 1 µs:** Isolated gaming cores respond to real-time wakeups with an average latency of 0.001 ms.
* **Max: 9 / 6 / 5 µs:** Peak worst-case scheduling delay across all cores remained under **0.009 ms** under full load, compared to 200–500+ µs typical on stock desktop configurations.


## 🛠️ Installation

1. Download the script directly from the repository using `curl`:
   ```bash
   curl -LO [https://raw.githubusercontent.com/Zirhael221/Low-latency-configuration-for-Arch-Linux/main/zxcvcpupartition.txt](https://raw.githubusercontent.com/Zirhael221/Low-latency-configuration-for-Arch-Linux/main/zxcvcpupartition.txt)
   ```
2. Move it to your local binary path and make it executable:
   ```bash
   sudo mv zxcvcpupartition.txt /usr/local/bin/zxcv.cpu.partition
   sudo chmod +x /usr/local/bin/zxcv.cpu.partition
   ```

## 🎮 Usage

Run the utility as root:
```bash
sudo zxcv.cpu.partition
```

### Configuration Prompts:
* **Apply or Revert:** Select `A` to apply custom parameters or `R` to scrub changes and restore stock bootloader/sysctl defaults.
* **Vendor:** Choose Intel (`i`) or AMD (`a`) to apply processor-specific register flags.
* **NVIDIA Optimus:** Select `y` if using hybrid graphics on a laptop.
* **Disable All Swap:** Select `y` to purge swap/ZRAM (requires adequate physical RAM).
* **Strict CPU Core Isolation:**
  * **Housekeeping Cores (CRITICAL):** Separate cores with **commas only** (e.g., `0,1`), **never dashes**. The script uses bitwise arithmetic (`1 << cpu`) to calculate the kernel workqueue hex mask; dashes are parsed as subtraction operators and break mask computation.
  * **Isolated Cores:** Enter your game cores (e.g., `2-3` or `2-7`).
  * *Tip:* Do not isolate every core. Leave sufficient housekeeping cores for OS tasks to prevent thread contention.

### Steam Launch Options

Launch games with non-root Real-Time scheduling privileges:

**With Strict Isolation:**
```text
chrt -r 50 taskset -c <YOUR_ISOLATED_CORES> %command%
```
*(Example: `chrt -r 50 taskset -c 2-3 %command%`)*

**Without Strict Isolation:**
```text
chrt -r 50 %command%
```


## ♻️ Uninstallation

To revert all modifications:
1. Run `sudo zxcv.cpu.partition`.
2. Select `R` at the prompt.
3. The script will remove GRUB flags, restore sysctl rules, re-enable swap, remove systemd affinity overrides, delete sweep services, and rebuild your bootloader.


## 🙌 Credits & References
* [Low Latency System Guide](https://lowlatencysystem.com/guide/)
* [SUSE CPU Isolation Guide](https://www.suse.com/c/cpu-isolation-introduction-part-1/)
* [Erik Rigtorp - Low Latency Tuning](https://rigtorp.se/low-latency-guide/)
* [Red Hat Enterprise Linux Real-Time Tuning](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux_for_real_time/9/html-single/optimizing_rhel_9_for_real_time_for_low_latency_operation/index)
* [TuneD Project](https://tuned-project.org/)
* CachyOS and Linux Latency Tuning Communities
