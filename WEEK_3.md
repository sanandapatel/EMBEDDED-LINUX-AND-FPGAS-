# Linux Device Drivers --- Week 3 Exam Notes

> **From Hardware Registers to User-Space Files**\
> Detailed theory, practical Linux workflow, examples, and exam
> revision.

## Source scope

Prepared from the two uploaded course PDFs: `ELFPGA_W3.pdf` (Week 3
Lectures 1--7) and `03-linux-driver-interfaces.pdf` (30-slide summary).
The notes preserve the lecture sequence. Additional explanations are
identified as clarifications where they go beyond the lecture material.

## Contents

1.  [How to use these notes](#how-to-use-these-notes)
2.  [1. System power-on, reset and
    bootstrapping](#1-system-power-on-reset-and-bootstrapping)
3.  [2. Bare-metal programming versus an operating
    system](#2-bare-metal-programming-versus-an-operating-system)
4.  [3. Registers, register maps and MMIO
    recap](#3-registers-register-maps-and-mmio-recap)
5.  [4. Kernel space, user space and system
    calls](#4-kernel-space-user-space-and-system-calls)
6.  [5. Virtual memory and address
    translation](#5-virtual-memory-and-address-translation)
7.  [6. Linux device model and the path from registers to
    files](#6-linux-device-model-and-the-path-from-registers-to-files)
8.  [7. /dev, /sys, sysfs, udev and
    permissions](#7-dev-sys-sysfs-udev-and-permissions)
9.  [8. System calls and common device
    APIs](#8-system-calls-and-common-device-apis)
10. [9. Kernel modules and the practical
    demonstration](#9-kernel-modules-and-the-practical-demonstration)
11. [10. From demonstration to a real MMIO
    driver](#10-from-demonstration-to-a-real-mmio-driver)
12. [11. Practical Linux inspection
    lab](#11-practical-linux-inspection-lab)
13. [12. Exam revision: high-value comparisons and
    explanations](#12-exam-revision-high-value-comparisons-and-explanations)
14. [13. Final mental model](#13-final-mental-model)
15. [Source coverage and clarification
    notes](#source-coverage-and-clarification-notes)

------------------------------------------------------------------------

# How to use these notes

These notes consolidate the two uploaded files: ELFPGA_W3.pdf (Week 3
lecture transcript, Lectures 1--7) and 03-linux-driver-interfaces.pdf
(30-slide summary). The lecture sequence is preserved: boot and bare
metal; register/MMIO recap; kernel and user space; virtual memory and
address translation; Linux device model; /dev, /sys and system calls;
kernel-module demonstration.

The exam focus is to explain not only definitions, but the complete path
from a hardware register to a user-space program. Practical sections
show how to inspect Linux interfaces and build a harmless example
module.

> **IMPORTANT:** Important scope distinction: the lecture's
> demonstration uses simulated/fake registers stored in kernel
> variables. It teaches module loading, character-device registration
> and sysfs attributes; it does not physically drive a PWM peripheral.
> Actual hardware access additionally requires a correctly described
> MMIO resource and proper mapping.

# 1. System power-on, reset and bootstrapping

## 1.1 What happens at reset?

A processor contains state-holding elements such as flip-flops and
internal registers. Reset places important state elements into known
values. These values are not necessarily all zero. The program counter
(PC) is especially important because it determines the address from
which the processor fetches its first instruction.

After reset is released and the clock runs, the CPU fetches an
instruction from the address represented by the PC, decodes it, executes
it and updates the PC. Therefore, the hardware platform must ensure that
the reset PC points to a location that can return valid startup
instructions.

## 1.2 BIOS/firmware and bootstrapping

The lecture describes the BIOS as a small initial program held in
firmware/flash. It performs enough initialization to locate and load a
larger program, commonly from storage into RAM, and then transfers
control to it. The process of starting with a small amount of trusted
code that loads the larger software stack is called bootstrapping.

Why is bootstrapping needed? A disk-resident operating system cannot
load itself into RAM until some earlier code can access the disk and
move the required data. Firmware and early boot code solve this initial
dependency.

## 1.3 Bare-metal microcontroller boot

On a microcontroller such as an Arduino-class MCU, the compiled program
is programmed into non-volatile memory. When reset is released, the CPU
begins executing from its defined reset entry point. The program
directly knows the peripherals and their register maps. No
general-purpose operating system is required.

## 1.4 Exam clarification: PC boot chain

The lecture uses BIOS as the conceptual first stage. On modern PCs the
firmware may be legacy BIOS or UEFI; a bootloader commonly loads the
Linux kernel and initramfs; the kernel initializes hardware and
eventually starts the first user-space process (PID 1, often systemd).
So the phrase "OS is the first program after reset" is a high-level
simplification: firmware and bootloader execute before the Linux kernel.

> **EXAM TAKEAWAY:** Remember: reset establishes a known CPU state; the
> reset PC selects the first instruction; firmware/bootloader solve the
> initial loading problem; the kernel takes over system resource
> management.

# 2. Bare-metal programming versus an operating system

## 2.1 Bare-metal model

In bare-metal programming, one application owns the processor and
directly controls peripherals. The program may read and write fixed
peripheral addresses, poll status bits, and configure interrupts. Timing
and execution order are largely under the programmer's control.

The lecture's PWM example uses a register block with CTRL at offset
0x00, PERIOD at 0x04, DUTY at 0x08 and STATUS at 0x0C. A program asserts
reset, writes period and duty values, enables the PWM, waits for a wrap
flag and clears the flag.

``` c
#define PWM_BASE    0x40000000u
#define REG32(a)    (*(volatile uint32_t *)(a))
#define PWM_CTRL    (PWM_BASE + 0x00)
#define PWM_PERIOD  (PWM_BASE + 0x04)
#define PWM_DUTY    (PWM_BASE + 0x08)
#define PWM_STATUS  (PWM_BASE + 0x0C)

#define CTRL_EN     (1u << 0)
#define CTRL_RST    (1u << 1)
#define ST_WRAP     (1u << 0)

/* Simplified bare-metal sequence */
REG32(PWM_CTRL)   = CTRL_RST;       /* reset pulse */
REG32(PWM_PERIOD) = 0xFF;
REG32(PWM_DUTY)   = 0x20;
REG32(PWM_CTRL)  |= CTRL_EN;
while (!(REG32(PWM_STATUS) & ST_WRAP)) { }
REG32(PWM_STATUS) = ST_WRAP;        /* write-one-to-clear */
```

## 2.2 Limitations of bare metal

-   All required functionality must be built into the application or its
    firmware.
-   There is no general-purpose process scheduler; multiple tasks must
    be coordinated explicitly, for example using a super-loop or an
    RTOS.
-   There is usually no process-level memory protection or isolation.
-   The application must manage peripheral access, timing, interrupts
    and resource conflicts itself.
-   The code is often tied closely to a particular MCU memory map and
    hardware design.

## 2.3 What an operating system adds

An OS takes responsibility for shared resources and offers standard
services to applications. The lecture lists process scheduling, memory
management, storage/filesystems, I/O drivers, networking and security. A
scheduler decides which process runs; memory management provides virtual
memory and protection; filesystems organize persistent data; the I/O
stack coordinates drivers, interrupts, DMA and power management.

  -----------------------------------------------------------------------
  Bare metal                          Operating system
  ----------------------------------- -----------------------------------
  One main application controls CPU   Many processes share CPU time

  Direct physical register access     Drivers mediate hardware access

  Little or no process isolation      Virtual memory and protection

  Programmer controls execution flow  Scheduler manages execution

  Often hardware-specific             More portable interfaces and
                                      resource management
  -----------------------------------------------------------------------

> **EXAM TAKEAWAY:** An OS trades some direct control and simplicity for
> concurrency, isolation, portability and resource sharing.

# 3. Registers, register maps and MMIO recap

## 3.1 Register map

A peripheral exposes a set of control, configuration, data and status
registers. The hardware designer assigns each register an offset within
a peripheral address window. Software uses the peripheral base address
plus the register offset to identify a register.

For the smart PWM timer: CTRL controls enable/reset; PERIOD sets the
counter period; DUTY sets the active portion of the PWM cycle; STATUS
reports events such as counter wrap or update-pending.

## 3.2 Memory-mapped I/O

With memory-mapped I/O (MMIO), peripheral registers occupy locations in
the processor's physical address map. The CPU uses load/store-style
transactions to read or write those locations. On an FPGA SoC, an
AXI-Lite or APB interconnect can decode the address and route the
transaction to the peripheral register bank.

A register write is not the same as writing ordinary RAM. A write may
trigger a hardware action, change an output, clear a flag or start a
conversion. Register semantics must therefore be understood before
writing.

## 3.3 Important register semantics

-   Read/write (RW): software can read the current value and write a new
    value.
-   Read-only (RO): software can observe state but should not modify it.
-   Write-only (WO): writes have meaning; reads may be undefined or
    return no useful value.
-   Write-one-to-clear (W1C): writing a 1 to a status bit clears that
    bit; writing 0 leaves it unchanged. Do not use a read-modify-write
    unless the register specification explicitly permits it.
-   Pulse/self-clearing bit: writing 1 triggers an action and hardware
    clears the bit automatically.

## 3.4 Bit operations

``` c
#define ENABLE_BIT (1u << 0)
#define RESET_BIT  (1u << 1)

ctrl |= ENABLE_BIT;      /* set bit 0; preserve other bits */
ctrl &= ~ENABLE_BIT;     /* clear bit 0 */
ctrl ^= ENABLE_BIT;      /* toggle bit 0 */
if (status & (1u << 0))  /* test bit 0 */ { /* event occurred */ }
```

The lecture uses `volatile` in its bare-metal macro to prevent the
compiler from treating the register like an ordinary variable and
reusing a cached value. `volatile` alone is not a complete substitute
for the Linux kernel's I/O accessors, memory-ordering rules or
device-specific barriers.

> **EXAM TAKEAWAY:** The register map is the hardware/software contract.
> Correct driver code must preserve register width, offset, bit meaning
> and side effects.

# 4. Kernel space, user space and system calls

## 4.1 User space

User space contains ordinary applications such as browsers, terminals,
editors and custom control programs. A process runs with restricted
privileges. It cannot directly execute privileged CPU operations or
freely access arbitrary hardware registers.

## 4.2 Kernel space

Kernel space contains the operating-system core and kernel drivers. It
has privileged access to CPU facilities, memory-management structures
and hardware resources. Drivers run in kernel context because they must
safely coordinate access to devices.

## 4.3 Why separate them?

If every application could write directly to a timer, disk controller or
memory-management register, unrelated applications could interfere with
one another or crash the machine. The kernel provides controlled entry
points, checks permissions and arbitrates access.

## 4.4 System-call path

A system call is a controlled transition from user mode into kernel
mode. The application requests an operation such as opening a device,
reading data, writing data or issuing an I/O control command. The kernel
validates the request, performs the operation through the appropriate
subsystem/driver and returns a result.

-   System call: controlled user-to-kernel request.
-   Interrupt: an event, often from hardware, that causes the CPU to
    enter a kernel handler.
-   Fault/exception: a condition such as an invalid memory access that
    transfers control to the kernel for handling.

The lecture's keyboard example illustrates interrupts: a key press
generates an event; the CPU enters kernel handling code; the OS
processes the event and makes the input available to applications.

## 4.5 CPU privilege levels

The lecture explains the privilege-ring model: ring 0 is the most
privileged level and is commonly used for the kernel on x86; ring 3 is
used for ordinary applications. Other CPU architectures use different
terminology and privilege-level structures, but the central idea is the
same: hardware enforces privilege separation.

## 4.6 What is privileged?

-   Direct access to device registers and configuration of interrupts.
-   Control of DMA-related facilities and memory-management hardware.
-   Modification of page tables and other protected kernel structures.
-   System-wide power-state and device power-management operations.

> **EXAM TAKEAWAY:** Applications request services; the kernel decides
> whether the request is permitted and performs privileged operations.

# 5. Virtual memory and address translation

## 5.1 Why virtual memory?

Virtual memory isolates processes and gives each process its own
address-space view. A process should not be able to read or overwrite
another process's private memory simply by guessing an address. The OS
can also place a process in different physical RAM locations at
different runs without changing the process's virtual addresses.

## 5.2 Virtual address versus physical address

A virtual address is the address generated and used by a process. A
physical address identifies an actual location in RAM or a device's
physical address window. Two processes may use the same virtual address
value while that value maps to different physical frames.

The Memory Management Unit (MMU) performs address translation using page
tables configured by the kernel. Memory is divided into pages in virtual
address space and page frames in physical memory. Page-table entries
also encode access permissions and mapping attributes.

## 5.3 Typical process address-space regions

-   Text/code: executable instructions.
-   Data/BSS: global and static variables.
-   Heap: dynamic allocations such as those requested by malloc;
    commonly grows as allocations increase.
-   Memory-mapped regions: shared libraries, mapped files and possibly
    mapped device regions.
-   Stack: function-call frames, local variables and saved execution
    state.

The exact layout and growth direction depend on architecture and OS
implementation. The key exam concept is that the process sees a virtual
layout; it is not a direct picture of physical RAM.

## 5.4 Address types relevant to drivers

  -----------------------------------------------------------------------
  Address type                        Meaning
  ----------------------------------- -----------------------------------
  User virtual address                Address visible to a particular
                                      user process; translated by the
                                      MMU.

  Kernel virtual address              Address used by kernel code; it is
                                      also virtual, not generally a
                                      physical address.

  Physical address                    Actual system address of RAM or a
                                      peripheral register window.

  Bus/MMIO address                    Address decoded by the interconnect
                                      to select a peripheral/register
                                      region.

  I/O virtual address (IOVA)          Address presented to a DMA-capable
                                      device when an IOMMU is used.
  -----------------------------------------------------------------------

## 5.5 Why a driver uses ioremap

A pointer in a Linux process is not a physical address. A kernel driver
therefore cannot safely copy the bare-metal pattern of casting a fixed
physical address to an ordinary C pointer and dereferencing it. The
driver maps the device's physical resource into kernel virtual address
space using an I/O mapping API. It then accesses the mapped region using
the kernel's I/O accessors.

``` c
/* Conceptual MMIO sequence */
void __iomem *base;

base = devm_ioremap_resource(dev, res);
if (IS_ERR(base))
    return PTR_ERR(base);

u32 ctrl = readl(base + CTRL_OFF);
writel(ctrl | ENABLE_BIT, base + CTRL_OFF);
```

`__iomem` is a kernel annotation that marks a pointer as referring to
I/O memory. `readl()` and `writel()` are the standard 32-bit MMIO
accessors in this example. They express the intended I/O access and
provide architecture-appropriate semantics; they are not interchangeable
with ordinary pointer dereferences.

## 5.6 DMA and IOMMU --- introductory context

DMA allows a device to transfer data to or from memory without the CPU
copying every byte. The lecture notes that devices may not directly use
CPU physical addresses. With an IOMMU, a device can use an I/O virtual
address that is translated and permission-checked. DMA mapping, coherent
memory and cache ownership rules are advanced driver topics and require
dedicated APIs.

> **EXAM TAKEAWAY:** Never assume a C pointer equals a physical address.
> In Linux, user and kernel pointers are virtual; MMIO resources must be
> mapped using the appropriate kernel APIs.

# 6. Linux device model and the path from registers to files

## 6.1 Device, bus and driver

Linux represents hardware and virtual devices in a device model. Devices
are attached to buses; drivers bind to matching devices and implement
their operation. The lecture names platform, PCI and I2C buses. The
platform bus is common for SoC-integrated peripherals described by
firmware/Device Tree; PCI is common for high-throughput expansion
devices; I2C connects addressed low-speed peripherals.

The bus framework helps discover devices and match them with drivers.
Once a driver is bound, it can initialize the device, map resources,
register interrupts and expose user-facing interfaces.

## 6.2 The central flow: registers to files

  -----------------------------------------------------------------------
  Stage                               What happens
  ----------------------------------- -----------------------------------
  1\. Hardware                        Peripheral exposes a register map
                                      over AXI-Lite, APB or another bus.

  2\. Kernel driver                   Driver obtains the hardware
                                      resource and maps MMIO into kernel
                                      virtual address space.

  3\. Driver policy                   Driver validates requests,
                                      sequences register operations and
                                      handles device-specific behavior.

  4\. User API                        Driver exposes an interface such as
                                      a character device, sysfs
                                      attributes or (where appropriate)
                                      ioctl/mmap.

  5\. Application                     User-space program uses
                                      open/read/write/ioctl or
                                      reads/writes a sysfs attribute.
  -----------------------------------------------------------------------

## 6.3 What does "everything is a file" mean?

Linux and Unix-like systems use file-like interfaces as a common
abstraction. A device node can be opened and operated on with familiar
file operations even though the underlying object is hardware. Sysfs
presents kernel objects and attributes as a hierarchy of small text
files. These are interfaces provided by the kernel, not necessarily
ordinary files stored on a disk.

## 6.4 Device types

  -----------------------------------------------------------------------
  Type                    Typical behavior        Examples
  ----------------------- ----------------------- -----------------------
  Character device        Byte-oriented or        /dev/ttyS0, /dev/i2c-1,
                          sequential I/O;         /dev/input/event\*
                          read/write/ioctl.       

  Block device            Block-oriented storage; /dev/sda, /dev/mmcblk0,
                          buffering and page      /dev/nvme0n1
                          cache.                  

  Network interface       Packet interface        eth0; visible in
                          normally accessed       /sys/class/net
                          through sockets and     
                          networking APIs.        

  Pseudo device           Kernel-provided         /dev/null, /dev/zero,
                          interface that may not  /dev/random
                          correspond to a         
                          physical peripheral.    
  -----------------------------------------------------------------------

Character devices are useful for streams and structured device control.
Block devices are designed for random access to blocks and
high-throughput storage. Network interfaces are not normally represented
as ordinary /dev character nodes; applications use sockets.

## 6.5 Character-device identity: major and minor

A device node in /dev is associated with a major and minor number. The
major number identifies the driver or device class association; the
minor number distinguishes individual devices or instances handled by
that driver. A driver registers its device numbers, and the kernel
routes operations on the node to the associated driver.

The lecture's demo uses `misc_register()`, which is a convenient way to
register a simple miscellaneous character device and have the kernel
allocate a minor number. For more complex drivers, character-device
registration can use `alloc_chrdev_region()`, `cdev_init()` and
`cdev_add()`.

> **EXAM TAKEAWAY:** A device node is the user-visible entry point; the
> driver implements what open/read/write/ioctl actually do.

# 7. /dev, /sys, sysfs, udev and permissions

## 7.1 /dev

`/dev` contains device nodes (special files), commonly provided through
devtmpfs and managed/named by udev. These nodes are entry points to
drivers. Examples include serial ports, disks and input devices. They
are not normal data files containing the device's register contents.

`ls -l /dev` shows the file type and major/minor numbers. In long
listing output, `c` indicates a character device and `b` indicates a
block device. The node's owner, group and permission bits determine who
may open it.

## 7.2 /sys and sysfs

`/sys` is a virtual filesystem (sysfs) exposing a live view of kernel
objects: devices, buses, classes and their attributes. The hierarchy
includes `/sys/devices`, `/sys/bus` and `/sys/class`. An attribute is
usually a small text value used to inspect or configure a device.

-   Use sysfs for small scalar values: enable/disable, mode, period,
    duty, status or counters.
-   Do not use sysfs for large binary streams or high-rate data
    transfer; use a suitable character-device or subsystem interface.
-   Sysfs attributes should be simple, well-defined and have clear
    read-only or read/write semantics.

## 7.3 /dev versus /sys

  -----------------------------------------------------------------------
  /dev                                /sys
  ----------------------------------- -----------------------------------
  Device nodes used to perform I/O    Virtual hierarchy exposing kernel
  through file operations.            objects and attributes.

  Common operations:                  Common operations: read/write small
  open/read/write/ioctl.              text attributes.

  Uses major/minor device numbers.    Reflects device, bus, class and
                                      driver relationships.

  Entry point to a driver interface.  Introspection and simple
                                      configuration interface.
  -----------------------------------------------------------------------

## 7.4 udev

udev is the user-space device manager that reacts to kernel device
events (uevents), applies rules and manages device-node names,
ownership, groups and permissions. The lecture notes that udev listens
to sysfs-related events and helps populate/manage `/dev`.

## 7.5 Permissions and safe access

A device may be owned by root and assigned to a group such as dialout or
video. A udev rule can grant the appropriate group access to a device
without running every application as root. Using `sudo` is useful for a
controlled demonstration, but routinely running programs as root
increases risk.

> **EXAM TAKEAWAY:** Use /dev to operate a device; use /sys to inspect
> or change small attributes of kernel objects; udev manages device-node
> policy and naming.

# 8. System calls and common device APIs

## 8.1 open, read, write and close

`open()` asks the kernel to open a path and returns a file descriptor
(an integer) on success. The descriptor is then passed to `read()`,
`write()`, `ioctl()` and `close()`. A file descriptor is a process-local
handle to an open kernel file object.

``` c
int fd = open("/dev/mydev", O_RDWR);
if (fd < 0) {
    perror("open");
    return 1;
}

char buf[64];
ssize_t n = read(fd, buf, sizeof(buf));
/* process n bytes if n > 0 */

close(fd);
```

## 8.2 ioctl

`ioctl` means input/output control. It sends a device-specific command
to a driver when ordinary byte-stream read/write operations are not a
natural fit. For example, resetting a timer or selecting a special
operating mode can be represented as an ioctl command.

``` c
int fd = open("/dev/mydev", O_RDWR);
if (fd >= 0) {
    if (ioctl(fd, MYDEV_RESET) < 0)
        perror("ioctl");
    close(fd);
}
```

A production ioctl interface should define command numbers and data
structures in a UAPI header shared by kernel and user-space code. The
driver must validate pointers, sizes, command values and permissions.
The source lecture introduces ioctl conceptually; it does not fully
specify a production UAPI.

## 8.3 mmap

`mmap()` maps a file or device region into a process's virtual address
space. It can be useful for large buffers or specialized
high-performance interfaces, but it requires careful lifetime,
permission, cacheability and synchronization design. It is not a
replacement for ioremap: `ioremap` is a kernel-side mapping mechanism,
while `mmap` is a user-space mapping API.

> **EXAM TAKEAWAY:** Use read/write for natural data transfer, ioctl for
> device-specific commands, and mmap only when a carefully designed
> mapped-memory interface is justified.

# 9. Kernel modules and the practical demonstration

## 9.1 What is a kernel module?

A kernel module is compiled code that can be loaded into or removed from
a running Linux kernel. Unlike an ordinary C application, it has no
`main()` function. It provides module initialization and cleanup entry
points. A loadable module is commonly built as a `.ko` file.

The lecture demonstrates a simple module that registers a miscellaneous
character device named `vreg_demo` and creates sysfs attributes for
control, period, duty and status. The register values are simulated
using static variables; this allows the interface to be tested on an
ordinary Linux PC without custom FPGA hardware.

## 9.2 Module lifecycle

-   Compile source against the headers/build tree for the running
    kernel.
-   Load with `insmod module.ko` or `modprobe module` (modprobe also
    resolves dependencies by module name).
-   The kernel invokes the module's initialization function. It
    registers the device and creates its sysfs attributes.
-   User space accesses the new device node and attributes.
-   Unload with `rmmod module_name`. The cleanup function removes sysfs
    entries and unregisters the device.

## 9.3 Kernel module structure

``` c
#include <linux/module.h>
#include <linux/init.h>

static int __init demo_init(void)
{
    pr_info("demo: loaded\\n");
    return 0;
}

static void __exit demo_exit(void)
{
    pr_info("demo: unloaded\\n");
}

module_init(demo_init);
module_exit(demo_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Example");
MODULE_DESCRIPTION("Minimal demonstration module");
```

`module_init()` identifies the function called when the module is
loaded. `module_exit()` identifies cleanup code called when it is
unloaded. `pr_info()` writes a message to the kernel log.
`MODULE_LICENSE`, `MODULE_AUTHOR` and `MODULE_DESCRIPTION` provide
module metadata. The kernel's license compatibility rules can affect
whether a module loads and whether it taints the kernel.

## 9.4 Building an external module

``` bash
# Makefile
obj-m += demo.o

KDIR := /lib/modules/$(shell uname -r)/build
PWD  := $(shell pwd)

all:
    $(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
    $(MAKE) -C $(KDIR) M=$(PWD) clean
```

The `uname -r` command prints the running kernel release.
`/lib/modules/$(uname -r)/build` normally points to the matching kernel
build headers. `make -C` runs the kernel build system in that directory,
while `M=$(PWD)` tells it where the external module source is located.
`obj-m += demo.o` asks the build system to build the module from
`demo.c`.

## 9.5 Practical commands (Ubuntu/Debian family)

``` bash
sudo apt update
sudo apt install build-essential linux-headers-$(uname -r)

uname -r                  # running kernel release
make                      # build the module
ls -l *.ko                # inspect generated module
sudo insmod demo.ko       # load module
lsmod | grep demo         # check whether it is loaded
sudo dmesg | tail -n 30   # inspect kernel messages
sudo rmmod demo           # unload module
make clean                # remove generated build files
```

If `insmod` fails, inspect the returned error and `dmesg`. Common causes
include headers for the wrong kernel, a symbol/version mismatch, a
module initialization error or an incorrect module name. `insmod` takes
a path to a `.ko` file; `rmmod` takes the loaded module name, normally
without `.ko`.

## 9.6 What the lecture's demo does

-   Creates fake register variables such as control, period, duty and
    status.
-   Defines file operations so reads from `/dev/vreg_demo` return a
    snapshot of the simulated register state.
-   Registers a misc character device, which results in a device node
    under `/dev`.
-   Creates sysfs attributes under a path similar to
    `/sys/class/misc/vreg_demo/` (the exact hierarchy may vary with
    kernel/device registration details).
-   The `show` callbacks format attribute values for reading; `store`
    callbacks parse user text and update the simulated state.
-   A mutex protects shared state while it is read or changed.
-   On module removal, sysfs attributes are removed and the misc device
    is deregistered.

## 9.7 How sysfs show/store callbacks work

A `show` callback converts an internal value into text in the supplied
buffer and returns the number of bytes written. A `store` callback
receives text written by user space, validates and parses it, updates
state, and returns the number of bytes consumed or an error. Kernel
helpers such as `kstrtou32()` are used to parse numeric strings safely.

The lecture emphasizes bounds checking and string termination. Kernel
code executes with high privilege; an unchecked buffer length can
corrupt kernel memory and crash or compromise the system.

## 9.8 Why mutexes are used

Several processes or kernel execution paths may try to access the same
device concurrently. A mutex provides mutual exclusion: code locks
before accessing shared state and unlocks afterward. Keep the critical
section short. Do not hold a mutex around slow operations such as
lengthy printing or waits unless the design requires it.

## 9.9 Demo interpretation

After loading the module, `dmesg` shows the load message,
`/dev/vreg_demo` appears as a character device, and the sysfs directory
contains attributes such as control, period, duty and status. Writing a
decimal number to period or duty changes the simulated values; reading
them back shows the updated values. Writing to the demo character device
may do nothing if its write callback is intentionally a no-op. After
`rmmod`, the device node and sysfs attributes disappear.

> **IMPORTANT:** Do not mistake the demo for a physical peripheral
> driver. Its fake registers are ordinary kernel variables. To control
> real FPGA/SoC registers, the driver must obtain the physical resource
> (for example through a platform device/Device Tree), map it, use
> readl/writel and implement hardware-specific error handling.

# 10. From demonstration to a real MMIO driver

## 10.1 What changes when real hardware is attached?

The simulated demo has no physical register address. A real driver must
learn the peripheral's address range and size from the platform/firmware
description, claim the resource, map it and then use I/O accessors. The
source lecture points to Device Tree as a mechanism that tells the
kernel which devices exist and where their resources are.

## 10.2 Typical platform-driver outline

``` c
/* Conceptual outline only: not a complete hardware driver */
static int timer_probe(struct platform_device *pdev)
{
    struct resource *res;
    void __iomem *base;

    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    if (!res)
        return -ENODEV;

    base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(base))
        return PTR_ERR(base);

    /* Initialize hardware using readl()/writel(). */
    return 0;
}

static void timer_remove(struct platform_device *pdev)
{
    /* Stop hardware and release non-managed resources if any. */
}
```

The probe function is called when a device matches the driver. It
obtains resources, maps registers and initializes the peripheral.
Managed `devm_` helpers automatically release resources when the device
is detached or probe fails, reducing cleanup errors. The remove path
should stop the hardware and undo any non-managed setup.

## 10.3 Device Tree idea

A Device Tree describes hardware that cannot always be discovered
automatically, especially SoC-integrated peripherals. It can describe
compatible strings, register ranges, interrupts, clocks and other
resources. The kernel uses this information to instantiate devices and
match them with drivers. The lecture introduces Device Tree as a later
topic rather than teaching its full syntax.

## 10.4 Interrupts, DMA and power management

-   Interrupt handling: the device signals an event; the driver
    acknowledges it and handles the event in an interrupt handler or
    threaded handler.
-   DMA: the device transfers data directly to/from memory; the driver
    must use DMA APIs and respect mapping/coherency rules.
-   Power management: drivers coordinate suspend/resume and runtime
    power states so devices can save energy and recover correctly.
-   UAPI and mmap: stable user/kernel interfaces may require shared
    headers and a carefully designed memory mapping.

These are listed as beyond-the-basics topics in the slide deck. They are
important in real drivers but are not developed in detail in this Week 3
material.

# 11. Practical Linux inspection lab

## 11.1 Explore the filesystem and system calls

``` bash
pwd
ls -la
strace ls
ls -l /dev
ls /sys
ls /sys/class
ls /sys/bus
ls /sys/devices
```

`pwd` prints the current working directory. `ls -la` lists normal and
hidden entries. `strace ls` displays system calls made by the `ls`
process, showing the boundary between user-space utilities and kernel
services. The lecture uses this to demonstrate that even a simple shell
command performs many system calls.

## 11.2 Inspect device nodes

``` bash
ls -l /dev/tty*
ls -l /dev/null /dev/zero
ls -l /dev/sd* 2>/dev/null
```

Look at the first character in `ls -l`: `c` means character device and
`b` means block device. The major/minor numbers appear in the
device-node listing. Do not write arbitrary data to raw disk nodes such
as `/dev/sda`; doing so can destroy data or the filesystem.

## 11.3 Inspect sysfs

``` c
ls /sys/class
ls /sys/class/net
ls /sys/class/misc
cat /sys/class/net/lo/operstate
```

Sysfs paths vary with the devices and kernel configuration present on a
machine. Treat sysfs as a live kernel view, not as a fixed directory
tree that is identical on every computer.

## 11.4 Observe kernel messages

``` c
sudo dmesg | tail -n 50
sudo journalctl -k -n 50
```

Kernel messages are especially useful when loading or debugging modules.
A module can load without printing anything to the terminal; inspect the
kernel log for `pr_info()` output or initialization errors.

# 12. Exam revision: high-value comparisons and explanations

## 12.1 Common "differentiate between" questions

  -----------------------------------------------------------------------
  Pair                                Core distinction
  ----------------------------------- -----------------------------------
  Bare metal vs Linux                 Direct application ownership of
                                      hardware versus OS-mediated,
                                      protected access.

  User mode vs kernel mode            Restricted application execution
                                      versus privileged OS/driver
                                      execution.

  Virtual vs physical address         Process-visible address translated
                                      by MMU versus actual system
                                      address.

  ioremap vs mmap                     Kernel mapping of device I/O memory
                                      versus mapping a file/device region
                                      into user space.

  /dev vs /sys                        Device-node I/O interface versus
                                      kernel-object/attribute hierarchy.

  Character vs block device           Sequential/byte-oriented interface
                                      versus block storage with
                                      buffering/page cache.

  read/write vs ioctl                 General data transfer versus
                                      device-specific control command.

  insmod vs rmmod                     Load a module file versus unload a
                                      loaded module.

  Kernel module vs user program       Kernel-loaded code with init/exit
                                      and privileged execution versus
                                      process with main() and restricted
                                      privileges.
  -----------------------------------------------------------------------

## 12.2 Typical long-answer question: Explain the complete path from a user application to a hardware register

1.  1.  A user application opens a device node or accesses a sysfs
        attribute.
2.  2.  The request enters the kernel through a system call.
3.  3.  The VFS/device framework routes the operation to the registered
        driver callback.
4.  4.  The driver validates the request and applies device-specific
        policy.
5.  5.  The driver accesses mapped MMIO using readl()/writel(), or uses
        a bus-specific subsystem/API.
6.  6.  The hardware register transaction travels through the SoC
        interconnect (such as AXI-Lite) to the peripheral.
7.  7.  The driver returns status/data to the kernel, which returns a
        result to the user process.

## 12.3 Likely exam questions

1.  Explain the reset and bootstrapping sequence of a computer. What
    role does the reset PC play?
2.  Compare bare-metal programming with an OS-based system. Give at
    least five differences.
3.  Explain MMIO and how an AXI-Lite peripheral register bank is
    accessed by software.
4.  Why can a user-space application not safely dereference a
    peripheral's physical address in Linux?
5.  Explain user mode, kernel mode, privilege levels and system-call
    transitions.
6.  What is virtual memory? Explain page tables and process isolation.
7.  Differentiate user virtual, kernel virtual, physical, bus and I/O
    virtual addresses.
8.  Explain why Linux drivers use ioremap/devm_ioremap_resource and
    readl/writel.
9.  Describe the Linux device model and the role of buses, devices and
    drivers.
10. Differentiate character devices, block devices, network interfaces
    and pseudo devices.
11. Explain major and minor numbers and how a /dev node is connected to
    driver file operations.
12. Compare /dev and /sys. What is sysfs and what is udev?
13. What is ioctl? When is it preferable to read/write?
14. What is a loadable kernel module? Explain module_init, module_exit
    and .ko files.
15. Explain the build, load, test and unload sequence for a simple
    kernel module.
16. What are sysfs show/store callbacks? Why must input parsing and
    buffer handling be safe?
17. Why are mutexes needed in drivers? Explain race conditions in shared
    register/state access.
18. What additional work is required to convert the fake-register demo
    into a real MMIO platform driver?

## 12.4 Short-answer definitions

-   MMIO: accessing peripheral registers through locations in the
    physical address map.
-   ioremap: maps a physical I/O resource into kernel virtual address
    space.
-   readl/writel: kernel accessors for reading/writing 32-bit MMIO
    registers.
-   sysfs: virtual filesystem exposing kernel objects and small
    attributes.
-   udev: user-space device manager reacting to device events and
    managing device nodes/rules.
-   ioctl: device-specific I/O control operation.
-   DMA: device-initiated data transfer between a device and system
    memory.
-   Device Tree: data structure describing hardware devices and their
    resources to the kernel.
-   Mutex: mutual-exclusion lock protecting shared state from concurrent
    access.
-   Kernel module: loadable kernel code with initialization and cleanup
    entry points.

## 12.5 Common mistakes to avoid

-   Do not equate a virtual address with a physical address.
-   Do not use ordinary pointer dereferences for Linux MMIO; use the
    appropriate I/O mapping and accessors.
-   Do not assume all status registers are ordinary RW registers; W1C
    and self-clearing semantics matter.
-   Do not confuse /dev nodes with regular disk files or sysfs
    attributes with bulk data files.
-   Do not assume the lecture's fake-register module controls actual
    hardware.
-   Do not run arbitrary kernel modules on your primary machine. A
    faulty module can crash or corrupt the running kernel; use a VM for
    experiments.
-   Do not use root privileges unnecessarily, and never write
    experimentally to raw disk device nodes.

# 13. Final mental model

Hardware exposes registers. The kernel knows which device exists and
where its resources are. A driver maps and controls the hardware while
enforcing policy and handling concurrency. Linux exposes a suitable user
API---often a character device in /dev and/or small sysfs attributes in
/sys. A user program uses system calls to access that API. The kernel
then translates the request into safe register transactions or other
device operations.

> **IMPORTANT:** One-line summary for the exam: Linux device drivers are
> the controlled bridge between hardware register maps and user-space
> interfaces.

# Source coverage and clarification notes

Primary source coverage: ELFPGA_W3.pdf, Week 3 Lectures 1--7 (boot
sequence; bare-metal operation; register/MMIO recap; kernel/user space;
privilege levels; virtual memory; address translation; Linux device
model; device types; ioctl; kernel modules; /dev and /sys inspection;
module build/load/unload demo).

Slide cross-reference: 03-linux-driver-interfaces.pdf, slides 1--30.
Particularly useful: slides 7--9 (bare-metal PWM example and
comparison), 11--17 (privilege and address spaces), 19--22 (device model
and MMIO access), 23--28 (device types, ioctl, /dev, /sys, sysfs and
permissions), and 29--30 (advanced topics and summary).

Clarifications added for exam completeness are explicitly presented as
context rather than claims that the lecture fully covered them: modern
UEFI/bootloader/PID 1 sequence; production UAPI considerations; the
distinction between ioremap and mmap; and the outline of a real platform
driver.
