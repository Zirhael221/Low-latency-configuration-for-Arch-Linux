# Low-latency-configuration-for-Arch-Linux
Collection of parameters and config to tweak Arch linux for low latency 

# zxcv.cpu.partition

A hardcore, kernel-level tuning utility designed to eliminate micro-stutters, bypass hardware interrupts, and minimize jitter for gaming on Linux. 

Built specifically for modern Arch/CachyOS systems, this script reaches deep into the Linux scheduler, memory manager, and I/O subsystems to force your operating system to prioritize game rendering above everything else.

⚠️ **WARNING:** This script is designed for extreme performance. It disables hardware CPU security mitigations (Spectre/Meltdown) to reclaim raw speed, and it blinds kernel watchdogs to prevent preemptive game interruptions. Do not run this on a production server or without setting up a system restore incase things go sideways. There is a revert option but best to be safe.

---

## Features & What It Does

### 1. CPU Core Isolation & Thread Management (Smart Toggle)
* **What it does:** Dedicates specific cores purely for gaming, while forcing the OS, background apps, and drivers onto "Housekeeping" cores.
* **Why it matters:** Games never have to pause for a background process (like a Discord update or network interrupt). A custom background service actively sweeps stray threads off the gaming cores. 
* **The Smart Toggle:** If you have a quad-core laptop, strict isolation can actually starve modern games. The script asks if you want to enable isolation. If you decline, it safely applies all other network/memory tweaks while letting the OS dynamically manage the 4 cores.

### 2. GPU & Direct Memory Access (IOMMU)
* **What it does:** Bypasses kernel security checks for hardware memory access (`iommu=pt`) and applies NVIDIA sync fixes.
* **Why it matters:** Tells the CPU to stop auditing the memory requests of trusted hardware. Your GPU, NVMe drive, and Network Interface can read/write directly to physical RAM without waiting in line, heavily reducing input lag. It also bakes in NVIDIA DRM modesetting to prevent "Async Page Flip" hard-freezes on Optimus laptops.

### 3. Memory Locking & Swap Assassination
* **What it does:** Permanently disables physical disk swap, ZRAM, and ZSWAP, while granting all users `mlock` privileges.
* **Why it matters:** Prevents the OS from temporarily moving game assets (textures, audio) to your hard drive. Games and Proton can lock their data permanently into high-speed physical RAM, completely eliminating traversal stutters.

### 4. Kernel Polling & Power States
* **What it does:** Disables dynamic clock scaling (`intel_pstate=disable` / `amd_pstate=disable`) and prevents the CPU from going into deep sleep (`idle=poll`).
* **Why it matters:** When a CPU core sleeps to save power, waking it up takes microseconds. `idle=poll` forces the cores to stay awake and spin, ready to execute a mouse click or network packet the exact nanosecond it arrives.

### 5. Network & Sysctl Latency Tweaks
* **What it does:** Edits the kernel's virtual memory and TCP/IP stack rules.
* **Why it matters:** Uses **Busy Polling** (`busy_read=50`) so the CPU actively checks network sockets for new game packets instead of waiting for a hardware interrupt. It also tweaks `dirty_ratio` to force the OS to write smaller data chunks in the background, preventing massive I/O stutters.

### 6. Storage I/O Schedulers
* **What it does:** Applies the perfect storage algorithm based on the physical type of drive.
* **Why it matters:** NVMe drives bypass software schedulers entirely (`none`). SATA SSDs prioritize tiny, urgent reads like loading a game texture (`kyber`). Mechanical HDDs use a budget-based algorithm so background updates don't lock up the read/write needle (`bfq`).

### 7. Stripping Kernel "Bloat"
* **What it does:** Passes a massive string of parameters to GRUB (e.g., `mitigations=off`, `audit=0`, `nowatchdog`).
* **Why it matters:** Claws back 5-15% of raw CPU performance by disabling security patches. Disables kernel watchdogs so they don't accidentally panic and reboot the PC when a game maxes out a CPU core for an extended period.

## 📊 Benchmark Proof (Real-Time Latency & Jitter)

To prove how effectively this script clears out OS bloat and prioritizes raw hardware speed, here is a 60-second Real-Time kernel benchmark (`cyclictest`) run on isolated cores using `chrt -r 99`.

/dev/cpu_dma_latency set to 0us
policy: fifo: loadavg: 0.47 1.01 0.76 1/409 4079

T: 0 ( 4063) P:99 I:1000 C:  59996 Min:      1 Act:    1 Avg:    1 Max:       9
T: 1 ( 4064) P:99 I:1500 C:  39997 Min:      1 Act:    1 Avg:    1 Max:       6
T: 2 ( 4065) P:99 I:2000 C:  29998 Min:      1 Act:    1 Avg:    1 Max:       5

**The Command:**
```bash
sudo cyclictest --affinity=1-3 --threads=3 --priority=99 --interval=1000 --duration=60s -m

What do these numbers mean?
cyclictest measures OS Jitter—the delay between when hardware asks the CPU to do something (like a mouse click) and when the CPU actually executes it. It is measured in microseconds (µs).

/dev/cpu_dma_latency set to 0us: Proves C-state and power-polling tweaks are working. The CPU is not sleeping; it is held completely awake to await instructions.

Avg: 1: The isolated gaming cores are executing instructions with an average delay of exactly 1 microsecond (0.001 milliseconds).

Max: 9 / 6 / 5: This is the most important metric. Over a full 60 seconds of hammering the CPU, the absolute worst-case delay before a core responded was 9 microseconds (0.009 milliseconds).

A standard desktop Linux kernel often spikes to 200–1000+ microseconds of jitter because the kernel pauses your game to handle background tasks. This script achieves near-perfect bare-metal latency. For gaming, this translates to absolutely flawless 1:1 mouse tracking and flatline frametimes with zero OS-level micro-stutters.

Installation

1. Create the script file in your local bin directory:
   ```bash
   sudo nano /usr/local/bin/zxcv.cpu.partition

Paste the script contents into the editor, save, and exit.

Make the script executable:
sudo chmod +x /usr/local/bin/zxcv.cpu.partition

sudo zxcv.cpu.partition


The Prompts:
You will be asked a series of simple questions to tailor the script to your exact hardware:

Apply or Revert: Choose A to apply tuning, or R to completely restore your system to factory OS defaults.

Vendor: Select Intel (i) or AMD (a) to apply the correct microcode and IOMMU flags.

NVIDIA Optimus: If you use an NVIDIA GPU on a laptop with an Intel iGPU, select y to apply critical X11 display deadlock fixes.

Swap Purge: Select y to permanently kill ZRAM, ZSWAP, and physical disk swap (requires 16GB+ of RAM).

Strict Isolation:

If you want to isolate cores for game threads: Select y, and input your desired Housekeeping (e.g., 0,1) and Isolated (e.g., 2-7) cores.
  (Please dont set all cores as isolated. Leave as many for housekeeping as you can)
  (Eg. For a 4 core cpu if you hit your desired fps on 2 core leave other 2 for housekeeping)

Launching Your Games
Once the script finishes and you reboot, your system is fully primed. Because the script grants non-root Real-Time privileges, you can launch games directly via Steam using this launch option:

If you enabled Strict Isolation:

chrt -r 50 taskset -c yourisolatedcores %command%

Uninstallation
If you ever want to return your system to completely stock settings, simply run the script again and select R (Revert).

The script will scrub your GRUB bootloader, delete the core-sweep services, restore swap files, and reset your network/sysctl rules to factory defaults automatically.

Feedback welcome!


Credits:

https://lowlatencysystem.com/guide/
https://www.suse.com/c/cpu-isolation-introduction-part-1/
https://rigtorp.se/low-latency-guide/
https://tuned-project.org/
https://docs.redhat.com/en/documentation/red_hat_enterprise_linux_for_real_time/9/html-single/optimizing_rhel_9_for_real_time_for_low_latency_operation/index
Latency and Gaming server members
Cachyos server members 
Others across reddit and forums

