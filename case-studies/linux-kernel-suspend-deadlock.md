# Root-Causing a Silent Linux Kernel Deadlock: A Deep Dive into Asynchronous Power Management & S3 Sleep

![Platform](https://img.shields.io/badge/Platform-HP%20ProBook%20450%20G4-blue)
![OS](https://img.shields.io/badge/OS-Fedora%2045%20(Kernel%207.3)-informational)
![Status](https://img.shields.io/badge/Status-Solved%20Upstream-success)

> **Upstream Community Contribution:** Published on the [Ask Fedora Discussion Forum](https://discussion.fedoraproject.org/t/solved-intermittent-suspend-freeze-at-pm-suspend-entry-deep-on-hp-probook-450-g4-kernel-7-x-fedora/209207).

---

## 📌 Executive Summary
On modern Linux kernels (6.x / 7.x), enterprise and consumer laptops with legacy bus architectures often experience silent, intermittent deadlocks during system sleep (`suspend-to-RAM` / S3). When this occurs, the display goes dark, power rails remain energized without the rhythmic S3 LED pulse, and the kernel produces zero panic or stack trace.

This case study documents the end-to-end diagnostic methodology used to isolate, analyze, and permanently resolve an intermittent suspend hang on an **HP ProBook 450 G4** running **Fedora 45**.

Rather than relying on ad-hoc trial-and-error, this issue was solved systematically by isolating userspace compositors, auditing ACPI DSDT tables with `iasl`, instrumenting the kernel power subsystem via `sysfs`, and enforcing hierarchical synchronous teardown (`pm_async=0`) via `systemd-tmpfiles`.

---

## 💻 Target Architecture
- **Device:** HP ProBook 450 G4
- **Processor:** Intel Core i5-7200U @ 2.50GHz (Kaby Lake-U)
- **Integrated Graphics:** Intel HD Graphics 620 (`i915`)
- **Storage Subsystem:** SATA SSD (`ata1.00`) + secondary SATA drive (`ata3.00`) + PCIe root complex
- **Wireless:** Intel Dual Band Wireless-AC 8265 (`iwlwifi` + `btusb`)
- **Firmware:** HP BIOS P85 Ver. 01.48 (Updated from 01.44)
- **Kernel:** Linux 7.3.4-300.fc45.x86_64

---

## 🔍 The Failure Signature
During a sleep transition (`systemctl suspend` or closing the laptop lid), the machine intermittently halted indefinitely. Power-drain indicators showed the CPU remained halted with hardware buses locked, requiring a forced 10-second power-button cutoff.

Inspecting non-volatile system logs from the hung boot (`journalctl -b -1`) revealed an abrupt termination:

```text
Nov 24 10:47:52 fedora systemd-logind[734]: The system will suspend now!
Nov 24 10:47:52 fedora systemd[1]: Starting systemd-suspend.service - System Suspend...
Nov 24 10:47:52 fedora systemd-sleep[5674]: Performing sleep operation 'suspend'...
Nov 24 10:47:52 fedora kernel: PM: suspend entry (deep)
[ HARD SYSTEM FREEZE — NO FURTHER LOG ENTRIES ]
```

### Why standard debugging failed:
1. **No Kernel Panic Log:** Because storage controllers (`ata1.00`, `ata3.00`) had already entered a low-power state, the kernel could not flush memory buffers to disk.
2. **No Screen Console:** Display planes and backlight had already powered down via `i915` DPM suspend, making `no_console_suspend` invisible without serial hardware.

---

## 🛠️ The Diagnostic Ladder

```
[Layer 1: Userspace]   ---> Rule out Wayland, GDM, and desktop compositors.
[Layer 2: Firmware]    ---> Decompile DSDT ACPI tables; test _OSI compliance.
[Layer 3: Kernel PM]   ---> Isolate subsystems using /sys/power/pm_test.
[Layer 4: Bus Timing]  ---> Identify async teardown concurrency deadlock.
```

### 1. Eliminating Userspace Compositors
To test whether the hang originated from Wayland DRM buffer deadlocks or GDM lock-screen session handling:
```bash
sudo systemctl isolate multi-user.target
systemctl suspend
```
The freeze reproduced identically in pure console text mode. This definitively proved the bug was within Ring 0 (kernel space) or platform firmware.

### 2. Firmware Decompilation & ACPI Auditing
Motherboard ACPI tables were dumped and disassembled using Intel ASL compiler:
```bash
sudo cat /sys/firmware/acpi/tables/DSDT > dsdt.aml
iasl -d dsdt.aml
```
Analysis revealed firmware index limit errors in `_TZ.CHGZ` (`AE_AML_PACKAGE_LIMIT`). To evaluate whether HP firmware was branching into untested ACPI code paths, Windows 10 interface spoofing was evaluated:
```bash
acpi_osi=! acpi_osi="Windows 2015"
```
The suspend lockup still occurred, proving that ACPI OS-interface branching was a secondary symptom rather than the root deadlock mechanism.

### 3. The Root Cause: Asynchronous Teardown Race Condition
Modern Linux kernels default to **asynchronous device power management (`pm_async=1`)**. In this mode, the kernel spawns concurrent worker threads to suspend PCIe endpoints, SATA controllers, graphics pipelines, and CPU cores in parallel to reduce suspend latency.

On the 2016 Intel Kaby Lake chipset, concurrent teardown triggers a **bus arbitration race condition**:
1. SATA controllers (`ata1.00`, `ata3.00`) and PCIe root bridges begin concurrent power state transitions.
2. A parent bridge or clock rail drops power before subordinate devices finish their `noirq` sleep callbacks.
3. The platform attempts to execute `ACPI: PM: Preparing to enter system sleep state S3` while an interrupt remains pending on a stalled bus, resulting in an unrecoverable CPU deadlock.

---

## 💡 The Solution: Enforcing Synchronous Teardown

Setting `pm_async=0` forces the kernel's Device Power Management (`dpm_suspend`) to traverse the device tree **serially in topological order** (child devices first, parent bridges second, master root controllers last).

### Runtime Verification:
```bash
echo 0 | sudo tee /sys/power/pm_async
systemctl suspend
```
The system suspended immediately and resumed cleanly across multiple consecutive runs.

### Upstream-Standard Persistence via `systemd-tmpfiles`:
Runtime writes to `/sys` do not survive reboots. Instead of using an ad-hoc shell script, the configuration was permanently integrated via `systemd-tmpfiles`, which applies the sysfs knob early during `sysinit.target`:

```bash
echo 'w /sys/power/pm_async - - - - 0' | sudo tee /etc/tmpfiles.d/pm_async.conf
sudo systemd-tmpfiles --create /etc/tmpfiles.d/pm_async.conf
```

---

## 📊 Verification & Results

Following a cold reboot, `cat /sys/power/pm_async` confirmed `0`. The journal logs verified a complete, error-free S3 sleep lifecycle:

```text
kernel: PM: suspend entry (deep)
kernel: PM: suspend devices took 0.908 seconds
kernel: ACPI: EC: interrupt blocked
kernel: ACPI: PM: Preparing to enter system sleep state S3
kernel: ACPI: EC: EC stopped
kernel: ACPI: PM: Saving platform NVS memory
kernel: ACPI: PM: Low-level resume complete
kernel: ACPI: EC: EC started
kernel: ACPI: PM: Restoring platform NVS memory
kernel: ACPI: PM: Waking up from system sleep state S3
kernel: ACPI: EC: interrupt unblocked
systemd-sleep: System returned from sleep operation 'suspend'.
```

The system was validated across multiple successive suspend/resume stress cycles with 100% stability.

---

## 🤝 Engineering Collaboration Note
This investigation was conducted collaboratively using an AI research companion (Google Antigravity) to explore low-level kernel PM documentation, inspect DSDT disassembly, and compare systemd persistence architectures, while all diagnostic tests, hardware telemetry, and verifications were executed on physical hardware.

---

## 🔐 Cybersecurity & Systems Engineering Takeaways
1. **Low-Level Hardware Boundaries:** Security begins at Ring 0 and the hardware bus. Concurrency and race conditions in power management mirror the types of timing vulnerabilities exploited in hardware fault injection and bus-level attacks.
2. **Abstractions Hide Failure Modes:** High-level abstractions (Kubernetes nodes, cloud VMs, hypervisors) all run on Linux kernel primitives. When a system hangs during shutdown or node drain, understanding kernel power states and bus dependencies is essential.
3. **Upstream Standards Matter:** Always prefer native systemd facilities (`tmpfiles.d`) over custom shell scripts when modifying kernel sysfs parameters.

---

[← Back to Homepage](/)

