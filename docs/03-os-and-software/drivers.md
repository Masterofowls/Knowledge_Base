# How Drivers Work (System Drivers)

> **Level:** Advanced · **Related:** [OS](operating-system.md) · [Motherboard](../01-hardware/motherboard.md) · [GPU](../01-hardware/gpu.md) · [Sensors](../01-hardware/sensors-and-detectors.md) · [Malware](../06-security-and-data/malware.md)

## 1. What a driver is

A **device driver** is code — usually running inside the kernel — that translates the OS's generic requests ("read 4 KB from this block device", "send this packet") into the **device-specific** register writes, DMA setup and interrupt handling a particular piece of hardware understands. It is the adapter between an OS abstraction and silicon.

```
 Application:  fopen/ReadFile/send()
      │  (syscall)
 Kernel subsystem: VFS/I-O Manager, block layer, network stack, input, DRM/WDDM
      │  (driver interface: function tables / IRPs)
 Driver:  "nvme", "e1000e", "amdgpu" / "stornvme.sys", "e1d.sys", "nvlddmkm.sys"
      │  (MMIO register access, DMA descriptors, interrupts)
 Hardware
```

Not all drivers talk to hardware: **filter drivers** (antivirus file filters, encryption), **virtual drivers** (loopback, TAP/TUN for [VPNs](../05-networking-and-web/vpn.md), virtual audio), **software bus drivers** also exist.

## 2. How software talks to hardware

1. **Discovery / enumeration** — buses (PCIe, USB, ACPI, Device Tree on ARM) report devices by IDs:
   - PCIe: vendor/device IDs, e.g., `PCI\VEN_8086&DEV_15F3` (Windows hardware ID), `8086:15f3` (Linux `lspci -nn`).
   - USB: VID/PID + class codes.
2. **Matching** — the OS finds a driver that declares support for that ID (Windows: INF files; Linux: `MODULE_DEVICE_TABLE` → modalias → `udev`/`modprobe`).
3. **Resource assignment** — BAR memory ranges, interrupt vectors (MSI/MSI-X), DMA masks.
4. **Register access (MMIO/PIO)** — the device's control registers are mapped into physical address space; the driver maps them and reads/writes with special accessors (`readl`/`writel`; `READ_REGISTER_ULONG`) that prevent compiler/CPU reordering.
5. **DMA** — the driver builds **descriptor rings** in RAM (address, length, flags); the device reads/writes RAM directly, then raises an interrupt. The **IOMMU** restricts which memory a device may touch.
6. **Interrupts** — device signals completion; the driver's ISR acknowledges quickly and defers work.

```
        Driver (CPU)                           NIC (device)
  ┌─────────────────────────┐            ┌───────────────────────┐
  │ TX ring in RAM:         │   DMA read │                       │
  │ [desc0][desc1][desc2]...│ ─────────▶ │ fetch descriptor+data │
  │ write tail register ────┼──MMIO────▶ │ "new work up to N"    │
  │                         │ ◀──MSI-X───│ interrupt: sent!      │
  └─────────────────────────┘            └───────────────────────┘
```

## 3. Linux driver model

Key abstractions: **bus** (pci, usb, platform, i2c), **device**, **driver**, bound in `/sys/bus/*/`. Drivers are usually **loadable kernel modules** (`.ko`).

Driver classes:
- **Character devices** (`/dev/ttyS0`, custom devices) – `file_operations` (open/read/write/ioctl/mmap)
- **Block devices** – via `blk-mq` multi-queue layer
- **Network devices** – `net_device` + NAPI polling
- Subsystem frameworks: DRM/KMS (GPU), ALSA (audio), V4L2 (video), input, IIO (sensors), hwmon

A minimal character-device module (C, as required by the Linux kernel):

```c
// hello_dev.c  — build with a Kbuild Makefile: obj-m += hello_dev.o
#include <linux/module.h>
#include <linux/miscdevice.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

static const char msg[] = "hello from kernel\n";

static ssize_t hello_read(struct file *f, char __user *buf, size_t len, loff_t *off) {
    return simple_read_from_buffer(buf, len, off, msg, sizeof(msg) - 1);
}

static const struct file_operations hello_fops = {
    .owner = THIS_MODULE,
    .read  = hello_read,
};

static struct miscdevice hello_misc = {
    .minor = MISC_DYNAMIC_MINOR,
    .name  = "hello",            // appears as /dev/hello
    .fops  = &hello_fops,
};

module_misc_device(hello_misc);  // registers on load, deregisters on unload
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Minimal misc char device");
```

```bash
make -C /lib/modules/$(uname -r)/build M=$PWD modules
sudo insmod hello_dev.ko && cat /dev/hello && sudo rmmod hello_dev
dmesg | tail                    # kernel log
```

A PCI driver skeleton registers IDs and a `probe()`:

```c
static const struct pci_device_id ids[] = { { PCI_DEVICE(0x8086, 0x15f3) }, { } };
MODULE_DEVICE_TABLE(pci, ids);

static int my_probe(struct pci_dev *pdev, const struct pci_device_id *id) {
    int err = pcim_enable_device(pdev);                 // managed (auto-cleanup)
    if (err) return err;
    pci_set_master(pdev);                               // allow DMA
    dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));
    void __iomem *regs = pcim_iomap(pdev, 0, 0);        // map BAR0
    u32 status = readl(regs + 0x08);                    // read a device register
    /* allocate rings, request_irq / pci_alloc_irq_vectors, register netdev... */
    return 0;
}
static struct pci_driver my_driver = { .name = "mydrv", .id_table = ids, .probe = my_probe };
module_pci_driver(my_driver);
```

Also relevant: **Rust for Linux** (drivers in Rust are now merged, e.g., the NVIDIA "Nova" and Apple AGX efforts), **eBPF** (verified programs attached to kernel hooks), and user-space drivers (**VFIO**, **UIO**, DPDK, libusb).

Useful commands: `lspci -k` (which driver is bound), `lsmod`, `modinfo e1000e`, `udevadm monitor`, `/sys/class/`, `echo 1 > /sys/bus/pci/devices/…/remove`.

## 4. Windows driver model

| Framework | Use |
|---|---|
| **WDM** (Windows Driver Model) | Original low-level model: IRPs, dispatch routines, complex PnP/power |
| **KMDF** (Kernel-Mode Driver Framework, part of WDF) | Object-based wrapper over WDM — recommended for most kernel drivers |
| **UMDF 2** | Same WDF API but in user mode (crash doesn't bluescreen) — USB/HID/sensors |
| Class/miniport models | NDIS (network), Storport (storage), WDDM (display), PortCls/AVStream (audio/video), HID minidrivers |
| **Minifilters** (Filter Manager) | File system filters: antivirus, backup, encryption |

**IRP flow:** the I/O Manager creates an **I/O Request Packet** and sends it down a **device stack** (filter DO → function DO (FDO) → bus/physical DO (PDO)). Each driver handles or passes it down (`IoCallDriver`) and completes it (`IoCompleteRequest`).

Minimal KMDF driver:

```c
#include <ntddk.h>
#include <wdf.h>

DRIVER_INITIALIZE DriverEntry;
EVT_WDF_DRIVER_DEVICE_ADD EvtDeviceAdd;

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    WDF_DRIVER_CONFIG config;
    WDF_DRIVER_CONFIG_INIT(&config, EvtDeviceAdd);
    KdPrintEx((DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL, "Hello KMDF\n"));
    return WdfDriverCreate(DriverObject, RegistryPath, WDF_NO_OBJECT_ATTRIBUTES,
                           &config, WDF_NO_HANDLE);
}

NTSTATUS EvtDeviceAdd(WDFDRIVER Driver, PWDFDEVICE_INIT DeviceInit) {
    UNREFERENCED_PARAMETER(Driver);
    WDFDEVICE device;
    NTSTATUS status = WdfDeviceCreate(&DeviceInit, WDF_NO_OBJECT_ATTRIBUTES, &device);
    // Next: WdfIoQueueCreate(...) with EvtIoRead/EvtIoWrite/EvtIoDeviceControl handlers
    return status;
}
```

Packaging: `.sys` binary + **INF** (installation instructions: hardware IDs, services, registry) + **CAT** (signed catalog). Installed to `C:\Windows\System32\drivers\`, registered as a service under `HKLM\SYSTEM\CurrentControlSet\Services\<name>`, staged in the **Driver Store** (`C:\Windows\System32\DriverStore\FileRepository`).

Tools: Visual Studio + **WDK**, Device Manager, `pnputil /enum-drivers`, `pnputil /add-driver x.inf /install`, `driverquery /v`, `sc query type= driver`, **Driver Verifier** (`verifier.exe`), WinDbg kernel debugging, `devcon`/`pnputil` for device ops.

**Signing:** 64-bit Windows requires kernel drivers to be signed by Microsoft (attestation or WHQL via the Hardware Dev Center). With **HVCI / Memory Integrity** on, drivers must also be compatible (no W+X memory), and Microsoft's **vulnerable driver blocklist** blocks known-exploitable signed drivers.

## 5. Kernel-mode constraints (both OSes)

- **No crash isolation**: a null dereference = kernel panic (Linux "Oops"/panic) or **BSOD** (bug check). The July 2024 CrowdStrike Falcon incident — a faulty content file read by a boot-start kernel driver crashed ~8.5 million Windows machines — is the canonical example of why kernel code is risky.
- **Context rules**: can't sleep/page-fault in interrupt context (Linux atomic context; Windows IRQL ≥ DISPATCH_LEVEL — touching pageable memory there → `IRQL_NOT_LESS_OR_EQUAL`).
- **Concurrency**: ISRs, DPCs/softirqs and multiple CPUs run simultaneously → spinlocks, careful memory ordering.
- **Never trust user pointers**: use `copy_from_user` / `ProbeForRead` + `__try/__except`; unchecked IOCTLs are a top source of privilege-escalation CVEs.
- **Resource cleanup** on every error path (managed APIs `devm_*`/`pcim_*`, WDF object parenting help).

## 6. Talking to a driver from user space

**Linux — ioctl from Python:**

```python
import fcntl, struct, os
# Example: get terminal window size via TIOCGWINSZ ioctl on the tty driver
TIOCGWINSZ = 0x5413
buf = fcntl.ioctl(os.open("/dev/tty", os.O_RDONLY), TIOCGWINSZ, b"\0" * 8)
rows, cols, _, _ = struct.unpack("HHHH", buf)
print(rows, cols)
```

**Windows — DeviceIoControl from C++:**

```cpp
#include <windows.h>
#include <winioctl.h>
#include <cstdio>
int main() {
    HANDLE h = CreateFileW(L"\\\\.\\PhysicalDrive0", 0, FILE_SHARE_READ | FILE_SHARE_WRITE,
                           nullptr, OPEN_EXISTING, 0, nullptr);
    DISK_GEOMETRY_EX g{}; DWORD ret;
    if (DeviceIoControl(h, IOCTL_DISK_GET_DRIVE_GEOMETRY_EX, nullptr, 0, &g, sizeof g, &ret, nullptr))
        printf("Disk size: %lld bytes\n", g.DiskSize.QuadPart);   // served by disk.sys/stornvme
    CloseHandle(h);
}
```

## 7. Plug and Play & power management

1. Bus driver detects new device (hot-plug interrupt, USB enumeration).
2. Kernel creates device object → emits event (Linux **uevent** → `udev` loads module/creates `/dev` node; Windows **PnP Manager** searches Driver Store / Windows Update).
3. Driver's probe/`AddDevice` + start routines run.
4. Power: runtime PM (Linux `pm_runtime_*`), Windows device power states **D0–D3**, system states S0 (Modern Standby) / S3 / S4; drivers must save/restore hardware state.

## 8. Driver updates and troubleshooting

| Symptom | Linux | Windows |
|---|---|---|
| Device not working | `dmesg | grep -i error`, `lspci -k` (no driver bound?) | Device Manager yellow bang, Code 10/43, `pnputil /enum-devices /problem` |
| Roll back | Boot older kernel, DKMS, `modprobe -r` | Device Manager → Roll Back Driver, System Restore |
| Blacklist | `/etc/modprobe.d/blacklist.conf` | Device installation restrictions (GPO) |
| Out-of-tree driver | **DKMS** rebuilds modules per kernel | Vendor installers, Windows Update |

## Further reading
- Corbet, Rubini, Kroah-Hartman, *Linux Device Drivers* (3rd ed., dated but foundational) + kernel `Documentation/driver-api/`
- Microsoft Learn: *Windows Driver Kit (WDK)*, *Getting started with Windows drivers*, WDF samples on GitHub (`microsoft/Windows-driver-samples`)
- Pavel Yosifovich, *Windows Kernel Programming*
