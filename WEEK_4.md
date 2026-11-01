# Week 4 — Emulation: System-Level Design
## Detailed exam notes with theory, implementation workflow, diagrams and revision questions

**Source basis:** `ELFPGA_W4.pdf` (182-page lecture transcript) and `04-emulation.pdf` (13-slide lecture deck). The notes preserve the lecture's scope and examples. Explanatory additions are marked **Practical interpretation** where useful. The lecture does not provide a conventional AXI/AMBA cycle waveform for this week; therefore the diagrams below are system boot / MMIO interaction sequences, not invented bus timing specifications.

---

## 1. Learning objectives

By the end of this topic, you should be able to:

1. Distinguish simulation from emulation and explain their abstraction levels.
2. Explain Renode's role in system-level emulation.
3. Describe a machine, system bus, CPU, RAM and memory-mapped peripheral.
4. Load and execute a bare-metal ELF in Renode.
5. Inspect the address map and register contents using the monitor.
6. Explain how a stock Linux image is booted on VExpress and Zynq/ZedBoard models.
7. Describe the kernel + BusyBox + initramfs build workflow.
8. Explain why a Device Tree Blob (DTB), boot arguments, reset PC and memory addresses matter.
9. Integrate a custom Linux kernel module with an emulated MMIO peripheral.
10. Explain physical-to-kernel-virtual MMIO mapping and safe driver cleanup.

---

## 2. Simulation versus emulation

### 2.1 Simulation

A **simulation** executes a model of system behaviour. The model captures *what* the system should do; it does not necessarily reproduce the exact physical implementation or internal mechanism.

The lecture's familiar example is Verilog simulation:

- A behavioural model may describe logic using `always` blocks and Boolean expressions.
- A gate-level model may instantiate gates such as AND, OR and NOT.
- The simulator evaluates signal changes and scheduled events.
- Simulation time is an internal model of time. The wall-clock time taken by the simulator is not necessarily related to the hardware's real execution time.

For example, a Verilog testbench may advance simulated time by 10 ns in a fraction of a second, or take much longer than 10 ns to compute it. The simulator's execution speed is not the physical clock frequency.

**Important distinction:** We normally say that a C program is *executed*, while a Verilog description is *simulated*. The latter is a model whose behaviour is evaluated.

### 2.2 Emulation

**Emulation** recreates the behaviour of a target system at a useful abstraction level so that software intended for that target can run against the model.

Examples of abstraction levels discussed in the lecture:

- **Instruction-set-level emulation:** target processor instructions are interpreted/emulated in software. A binary built for the target CPU can execute.
- **Gate-level modelling/emulation:** behaviour is represented at individual logic-gate level.
- **Cycle-accurate emulation:** timing and state progression are modelled cycle by cycle. This matters particularly in hardware/software co-simulation.

The lecture's CPU example is useful: a detailed RTL model may contain flip-flops, adders and gates to update the program counter. A higher-level emulator can represent the architectural effect as `PC = PC + 4` for a simple sequential instruction path. This is faster, but it intentionally abstracts away internal gate activity.

The objective is **functional equivalence** at the chosen abstraction level: software should observe behaviour consistent with the intended target system.

### 2.3 Comparison table

| Aspect | Simulation | System-level emulation |
|---|---|---|
| Main object | Model of behaviour | Model of a target system |
| Typical input | RTL / behavioural model | Target binary plus platform model |
| Abstraction | Behavioural or gate level, depending on model | Often instruction-set and peripheral level |
| Main observation | Signal/state changes | Program execution and system behaviour |
| Time | Simulated time may not match wall-clock time | Execution speed is an emulator implementation property |
| Typical use here | Verify RTL logic | Boot software, Linux and drivers before hardware bring-up |

**Exam nuance:** The words are related and can overlap. A sufficiently detailed RTL simulation can reproduce system behaviour, but in this course *emulation* usually means a higher-level model that makes it practical to run a complete software stack.

### 2.4 Why not boot Linux using a full RTL simulation?

A processor implemented in RTL may require enormous computation to simulate every internal event. Booting a large OS through a gate-level model is often impractically slow.

System-level emulation abstracts processor internals while preserving the architectural behaviour software depends on. It can therefore execute many more target instructions per second and make useful observations about a complete system.

---

## 3. Renode: purpose and capabilities

The lecture uses **Renode**, an open-source development framework from Antmicro.

Its role in this course is to construct a model of a complete embedded system and run software on it before implementing the system in physical hardware.

### 3.1 What can be modelled?

A Renode system can include:

- One or more CPU cores.
- System RAM and other memory regions.
- A system bus.
- Standard peripherals such as UART and SPI.
- Interrupt controllers and timers.
- Custom peripherals implemented as software models.
- RTL components connected through co-simulation (an advanced topic mentioned by the lecture).

Renode supports several processor families. The course examples use ARM processors, including Cortex-A9 / ARMv7-A platforms.

### 3.2 Deterministic execution and debugging

The lecture highlights deterministic execution and debugging:

- The system can be paused and stepped.
- Individual instructions can be inspected.
- Peripheral accesses can be logged.
- The monitor can inspect the address map and memory contents.
- UART output can be displayed in an analyzer window.

Determinism is valuable because a failing software interaction can be reproduced and inspected without relying on a physical board's timing or availability.

### 3.3 Software emulation versus hardware emulation

The Renode demonstrations are **software-level emulation**: a software model of the CPU and peripherals executes target instructions.

The lecture also mentions hardware emulators such as Synopsys ZeBu and Cadence Palladium. These use FPGA-based hardware platforms to emulate a design represented by RTL. They can provide a much more detailed hardware model, but they are a different class of tool and are not the main focus of this week's Renode exercises.

---

## 4. System model: bus, CPU, memory and peripherals

### 4.1 The system bus

The system bus is the common interconnect through which the CPU accesses memory and memory-mapped peripherals.

The lecture uses Renode's `sysbus` abstraction. In a real design, the interconnect may be AXI or another AMBA bus. For the simple Renode model, the important point is the address map and the connections between components; the monitor-level `sysbus` abstraction is not itself a complete electrical/timing description of AXI.

### 4.2 Components

- **CPU:** fetches and executes instructions. Its program counter (PC) identifies the next instruction address.
- **RAM:** stores program instructions, data, kernel images and runtime state.
- **UART:** serial console used to display boot messages and interact with a shell.
- **MMIO peripheral:** a device whose control/status registers occupy addresses in the processor's memory map.
- **Interrupt controller:** routes peripheral interrupt requests to the CPU where the platform model includes one.

### 4.3 Memory-mapped I/O (MMIO)

With MMIO, a peripheral's registers are assigned addresses in the same address space used by the CPU for memory accesses.

A CPU store to a peripheral register address is decoded by the interconnect and delivered to the peripheral. A load from that address returns the peripheral register's value.

The lecture's simplified smart timer is initially represented as a memory-mapped block. This is a stub/fake peripheral for demonstrating the software path; it is not yet a cycle-accurate timer implementation.

### 4.4 Example memory map from the simple lecture platform

The simple platform description places:

- System RAM at `0x80000000`, with a size of `0x00400000` (4 MiB).
- Smart-timer register window at `0x70000000`, with a size of `0x00001000` (4 KiB).

The two regions do not overlap. Addresses outside mapped regions have no device assigned in this simplified model.

![Illustrative memory map](emulation_diagrams/memory_map.png)

**Address-range calculation**

For a region with base \(B\) and size \(S\), the last included byte is:

\[
A_{\mathrm{last}} = B + S - 1
\]

For RAM:

\[
0x80000000 + 0x00400000 - 1 = 0x803FFFFF
\]

Thus the RAM range is `0x80000000` through `0x803FFFFF`.

For the smart timer:

\[
0x70000000 + 0x1000 - 1 = 0x70000FFF
\]

Thus its window is `0x70000000` through `0x70000FFF`.

**Practical interpretation:** If the driver is configured with the wrong base address, it accesses a different region. The Linux module may load successfully but will not communicate with the intended peripheral.

---

## 5. Renode file types and monitor workflow

### 5.1 `.repl` platform description

The lecture refers to platform-description files with the `.repl` extension. They describe the hardware model: CPU, memory, peripherals, bus mappings, interrupt connections and other platform components.

Think of `.repl` as the *hardware inventory and wiring description*.

### 5.2 `.resc` Renode script

A `.resc` script automates the sequence of monitor commands needed to create and configure a machine.

Typical responsibilities include:

1. Create a machine.
2. Load a platform description.
3. Load program or kernel images.
4. Set CPU state and boot arguments.
5. Open a UART analyzer.
6. Start execution.

Think of `.resc` as the *bring-up procedure* for a platform.

### 5.3 Important commands shown in the lecture

| Command / construct | Purpose |
|---|---|
| `mach create` | Create a machine instance |
| `machine LoadPlatformDescription @file.repl` | Load the platform model |
| `include @file.resc` / `i @file.resc` | Execute/include a Renode script |
| `sysbus WhatIsAt <address>` | Identify the component mapped at an address |
| `sysbus ReadDoubleWord <address>` | Read a 32-bit word from the bus address |
| `sysbus WriteDoubleWord <address> <value>` | Write a 32-bit word (use only when appropriate) |
| `start` | Start/resume emulated execution |
| `pause` | Pause execution |
| `showAnalyzer uartX` | Open a UART output analyzer |
| `LoadELF <file>` | Load an ELF image using its ELF load information |

The exact command sequence depends on the platform and script. Use the course's supplied scripts as the authoritative syntax for the installed Renode version.

### 5.4 Bare-metal execution: what happens?

In a bare-metal system, there is no operating system to load the application, create processes or provide `printf()`.

The basic path is:

1. Compile C source for the target CPU.
2. Link it into an ELF executable with suitable load addresses and entry point.
3. Create the Renode machine and load its platform description.
4. Load the ELF into the emulated memory.
5. Ensure the CPU starts at the correct entry point / reset state.
6. Start the machine.
7. Inspect UART output, logs, memory or peripheral register values.

The ELF contains metadata such as loadable segments and entry-point information. Renode can use this to place the program in the appropriate memory locations.

### 5.5 Why does bare-metal code use addresses directly?

Without an OS, there is no driver framework or standard I/O service. A simple program may use a pointer to a known MMIO address and read/write registers directly.

Conceptual example:

```c
#define TIMER_BASE   0x70000000u
#define REG_CONTROL  (*(volatile unsigned int *)(TIMER_BASE + 0x00u))
#define REG_PERIOD   (*(volatile unsigned int *)(TIMER_BASE + 0x04u))
#define REG_DUTY     (*(volatile unsigned int *)(TIMER_BASE + 0x08u))

int main(void)
{
    REG_CONTROL = 1u;
    REG_PERIOD  = 100u;
    REG_DUTY    = 25u;
    REG_CONTROL = 2u;

    for (;;) {
        /* Keep the bare-metal program alive. */
    }
}
```

This is an illustrative C pattern, not a complete portable driver. The actual register offsets and semantics must match the peripheral specification. `volatile` tells the compiler that accesses are observable and must not be optimized away as ordinary unused memory operations; it does not by itself provide synchronization or make a multi-step hardware protocol atomic.

---

## 6. Bus inspection and register debugging

A major learning objective is **peek/poke**: inspect or modify the emulated system's bus-visible state.

### 6.1 `WhatIsAt`

Use `sysbus WhatIsAt <address>` to determine which component owns an address.

This answers: *Is this address RAM, UART, the smart timer, or unmapped space?*

### 6.2 `ReadDoubleWord`

Use `sysbus ReadDoubleWord <address>` to inspect a 32-bit value.

Example from the lecture deck:

```text
sysbus WhatIsAt 0x70000000
sysbus ReadDoubleWord 0x70000004
```

If the peripheral register initially contains zero, the monitor reports zero. After a valid write, reading the same register should show the updated value if the model implements that storage behaviour.

### 6.3 Debugging sequence

1. Confirm the peripheral is mapped at the expected base.
2. Read the register before writing.
3. Write a known test value.
4. Read it back.
5. Compare the result with what the software interface reports.
6. If the values differ, check address, offset, access width, mapping and driver configuration.

**Common mistake:** confusing the base address with the register offset. For example, a register at offset `0x04` is accessed at `BASE + 0x04`, not at `0x04`.

---

## 7. Booting a stock Linux system

The lecture demonstrates two example platforms.

### 7.1 VExpress

The stock VExpress example uses a pre-existing Renode script:

```text
i @scripts/single-node/vexpress.resc
```

The script downloads or locates a kernel and root filesystem, loads them into the appropriate memory locations, configures the CPU and starts execution.

The lecture describes this as a very basic Linux boot. It may show warnings or missing functionality (for example, no modules); the purpose is to demonstrate the overall boot path rather than provide a fully featured board image.

### 7.2 Zynq / ZedBoard

The Zynq example is more representative of a board with a richer peripheral set. The lecture associates it with the ZedBoard / Zynq-7000 family.

The script uses a platform description and loads:

- Linux kernel image (ELF in the described workflow).
- Root filesystem / initramfs.
- Device Tree Blob (DTB).

The DTB describes hardware to Linux, including devices and their resources. The kernel uses it to discover or configure platform devices. The lecture explicitly says DTB details are developed further in later material; here, understand its role in describing the board.

The platform model includes items such as CPU, RAM, UARTs, timers, I²C, SPI, SD-card-related devices, Ethernet and an interrupt controller. The actual set depends on the platform description.

### 7.3 Why platform details matter

A kernel image is not automatically compatible with every emulated machine. The platform model and boot configuration must agree on:

- CPU architecture and supported instruction set.
- RAM base address and size.
- Peripheral addresses and interrupt connections.
- UART used as the console.
- Kernel load address and entry point / reset PC.
- Initramfs location and size.
- DTB location and content.
- Any required address redirects.

A mismatch can cause early boot failure, no console output, memory corruption or a kernel that starts but cannot find required devices.

### 7.4 Boot flow diagram

![Linux boot flow](emulation_diagrams/linux_boot_flow.png)

### 7.5 UART console

The kernel emits early boot messages through a configured console, commonly a serial UART. Renode's UART analyzer lets you observe this output.

The lecture notes that the Zynq example can show output on more than one UART during parts of boot. The final interactive console may be associated with a particular UART (the lecture's demonstration uses UART1 for the login shell). Do not assume that UART0 is always the Linux console; check the kernel command line, DTB and platform mapping.

---

## 8. Building a custom ARM Linux kernel

The second half of the lecture moves from stock images to building a kernel and a minimal user space.

### 8.1 Cross-compilation

The build host may be x86-64, while the target system is ARM. A cross-compiler generates ARM code on the host.

The slide lists packages including:

```bash
sudo apt-get install \
  gcc-arm-linux-gnueabihf \
  g++-arm-linux-gnueabihf \
  bc bison flex libssl-dev make dwarves cpio
```

For bare-metal examples, the slide separately lists:

```bash
sudo apt install gcc-arm-none-eabi
```

These toolchains serve different targets:

- `arm-none-eabi`: typically bare-metal ARM firmware, without a Linux userspace ABI.
- `arm-linux-gnueabihf`: ARM Linux binaries using the hard-float GNU/Linux ABI.

Do not interchange them blindly.

### 8.2 Kernel source and configuration

The slide uses Linux kernel version `6.6.110` as its example and points to kernel.org. It also references BusyBox `1.32.1`.

The broad workflow is:

1. Download and extract the Linux kernel source.
2. Select a suitable baseline configuration.
3. Configure the kernel for the target platform.
4. Build the kernel with the ARM cross-compiler.
5. Build BusyBox for the target architecture.
6. Construct an initramfs containing BusyBox and required files.
7. Boot the kernel with the correct command line and initramfs.

The slide's summary specifically calls out `multi_v7_defconfig` as the starting configuration for the ARMv7 class of system used in the demo.

**Configuration matters:** a kernel may compile successfully but still fail to boot if it lacks support for the target CPU, memory layout, console, interrupt controller, filesystem or required drivers.

### 8.3 Example source preparation commands

The lecture slide gives the following example downloads:

```bash
# Kernel archive (example version used in the lecture)
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.6.110.tar.xz
tar -xf linux-6.6.110.tar.xz

# BusyBox archive (example version used in the lecture)
wget https://www.busybox.net/downloads/busybox-1.32.1.tar.bz2
tar -xf busybox-1.32.1.tar.bz2
```

The course-provided `.config` files are to be copied into the corresponding extracted source directories and renamed to `.config` as required. The exact configuration files and helper scripts are part of the course demo material, not embedded in the two PDFs supplied here.

### 8.4 Kernel build concepts

A simplified example of the configuration/build sequence is:

```bash
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-

make multi_v7_defconfig
# Apply the course/platform-specific configuration.
make -j"$(nproc)" zImage
make -j"$(nproc)" modules
```

This is a general illustration of the build variables and target. The lecture's exact demo may use additional configuration changes and targets. In particular, do not assume that `zImage`, `vmlinux` and an ELF file are interchangeable formats:

- **`vmlinux`:** uncompressed ELF kernel image, useful for symbols/debugging and, in some workflows, direct loading.
- **`zImage`:** compressed ARM kernel image with a boot wrapper.
- **`Image`:** uncompressed kernel image in architectures/workflows that use it.

Use the image format expected by the Renode script and target boot method.

### 8.5 BusyBox

BusyBox provides compact implementations of many common Unix utilities in one executable. It is useful for a small embedded Linux user space.

In this lecture's workflow, BusyBox is compiled for the target ARM system and placed in the initial RAM filesystem. It supplies the shell and basic commands used after the kernel starts user space.

### 8.6 initramfs / initial RAM filesystem

An **initramfs** is an archive of files that the kernel unpacks into a root filesystem in memory during early boot.

Typical contents include:

- `/init` — the first user-space program launched by the kernel.
- BusyBox.
- Required applets / symlinks.
- Device nodes or setup scripts as required.
- Custom kernel module files (`.ko`) for the later demonstration.

A conceptual minimal `/init` script might mount essential virtual filesystems and then launch a shell. The lecture's actual demo files are provided separately in the course repository; their exact contents are not present in the supplied PDFs.

### 8.7 initramfs versus root filesystem

The terms are related but not identical:

- A **root filesystem** is the directory tree that user space sees as `/`.
- An **initramfs** is a mechanism for supplying an initial filesystem archive to the kernel.
- A **RAM disk** traditionally presents a block-device-like disk backed by RAM; the lecture uses this analogy to explain why a filesystem image can be loaded into memory.

The VExpress example refers to a rootfs image loaded at a memory address. The custom kernel demonstration instead prepares a small initramfs with BusyBox and `/init`.

### 8.8 Kernel boot arguments and DTB

Boot arguments communicate important information to the kernel, such as:

- Console device (for example, a serial console).
- Root filesystem location/type.
- Initramfs location and size, where applicable.
- Other platform-specific parameters.

The DTB supplies a structured description of the hardware. The kernel command line and DTB must agree with the emulated platform. If the console argument names the wrong UART, the kernel may be booting but appear silent.

### 8.9 Build-to-boot workflow

![Linux boot sequence](emulation_diagrams/linux_boot_flow.png)

---

## 9. Integrating a custom Linux kernel module

The lecture reuses the smart-timer register interface from the earlier MMIO/register demonstration and connects it to the emulated system.

### 9.1 What is a kernel module?

A Linux kernel module is code that can be loaded into a running kernel, commonly used for device drivers and other kernel extensions.

A loadable module is built against the kernel build tree/configuration it is intended to run with. The module's architecture, kernel version/configuration and exported symbol expectations must be compatible.

The lecture builds a module against the custom kernel tree and places the resulting module in the initramfs. It is then loaded from the Linux shell using `insmod`.

### 9.2 Physical base address parameter

The demonstration module accepts a physical base address:

- If the base is zero, the lecture's module uses its virtual-register behaviour and does not map a physical backend.
- If a physical base is supplied, the module attempts to reserve and map the peripheral register window.

The lecture's smart-timer base address is `0x70000000`.

This parameter is critical. Supplying zero or an incorrect address means the module is not accessing the intended emulated peripheral, even if the module itself loads.

### 9.3 Correct MMIO access pattern

The lecture describes the following kernel-side sequence:

1. Request ownership of the physical I/O memory range.
2. Map the physical register window into kernel virtual address space.
3. Access registers through the mapped pointer using I/O accessors.
4. On module unload, unmap the region and release the reserved range.

Typical Linux APIs include:

```c
request_mem_region(phys_base, window_size, "smart-timer");
ioremap(phys_base, window_size);

/* Register access */
readl(mmio_base + REG_PERIOD);
writel(value, mmio_base + REG_PERIOD);

/* Cleanup */
iounmap(mmio_base);
release_mem_region(phys_base, window_size);
```

This is a conceptual API sequence. Production drivers must check every return value, handle errors and follow the exact API contract for the kernel version.

### 9.4 Physical address versus virtual address

The physical base address identifies the device's location in the platform's physical address map. `ioremap()` creates a kernel virtual mapping that the driver can safely use for MMIO accesses.

Therefore:

\[
\text{physical MMIO address} \ne \text{kernel virtual pointer}
\]

Do not dereference the raw physical address as an ordinary C pointer in a Linux driver. Use the kernel's I/O mapping and access mechanisms.

### 9.5 `readl()` and `writel()`

- `readl(addr)` reads a 32-bit value from an I/O-mapped register.
- `writel(value, addr)` writes a 32-bit value to an I/O-mapped register.

These are kernel-provided accessors designed for MMIO. They are not interchangeable with ordinary `*ptr` memory operations.

### 9.6 Register interface

The lecture's smart timer exposes parameters such as:

- Control.
- Period.
- Duty cycle / offset.
- Status.

The bare-metal demo writes control, period and duty-related values. The Linux module exposes an interface (including a sysfs attribute in the demonstration) so that user space can change a value such as the period.

The precise bit definitions and complete register semantics are not fully specified in the supplied lecture deck; use the actual peripheral/module source when implementing against hardware.

### 9.7 Driver initialization and error unwinding

A driver must undo earlier successful operations if a later initialization step fails.

For example:

- If memory-region reservation fails, return an error.
- If reservation succeeds but mapping fails, release the reserved region.
- If mapping succeeds but device registration fails, unmap and release.
- If all setup succeeds, retain the resources until module exit.
- During exit, unregister the interface, unmap MMIO and release the region.

![Kernel module lifecycle](emulation_diagrams/kernel_module_lifecycle.png)

This is a fundamental driver-writing rule: **cleanup is the reverse of successful setup**.

### 9.8 Demonstration: proving that Linux reaches the peripheral

The lecture demonstrates a cross-check between Linux and the Renode monitor:

1. Boot the custom Linux kernel.
2. Confirm the module is not loaded initially.
3. Use `insmod` to load the module with the intended physical base address.
4. Access the module's sysfs attribute (the lecture uses a period attribute).
5. Write a value such as hexadecimal `0x64` (decimal 100).
6. Inspect the smart-timer address from the Renode monitor.
7. Confirm the same value is visible at the corresponding emulated register.

This is an important system-level test: a write made through a Linux interface is reflected in the bus-visible state of the emulated peripheral.

### 9.9 Out-of-tree module warning

The lecture observes a kernel message indicating an “out of tree” module. This means the module was built separately from the main kernel source tree rather than built into that kernel image.

It is a status/warning worth understanding; it does not by itself prove that the module is faulty. Kernel configuration, module signing and version compatibility can impose additional restrictions in other environments.

---

## 10. MMIO interaction sequence (conceptual timing/sequence diagram)

The supplied Week 4 material does not define AXI channel timing or cycle-level register timing. The following image instead shows the *causal sequence* of a Linux-side register write and the corresponding bus-visible state. It is not a substitute for an AXI waveform.

![MMIO interaction sequence](emulation_diagrams/mmio_interaction_sequence.png)

### Step-by-step interpretation

1. A user-space write reaches a driver interface (for example, a sysfs attribute).
2. The kernel driver validates/converts the value.
3. The driver writes to the mapped MMIO register using an I/O accessor.
4. The bus address is decoded to the smart-timer peripheral.
5. The peripheral model updates its register state.
6. The Renode monitor can inspect the same mapped address and observe the updated value.
7. A later read through the Linux interface can observe the register state, assuming the driver exposes a corresponding read operation.

**Exam point:** Renode monitor access and Linux driver access are two different observation/control paths into the same emulated platform model. Agreement between them is evidence that the MMIO address mapping and software path are connected correctly.

---

## 11. Common failure modes and debugging checklist

| Symptom | Likely cause | What to check |
|---|---|---|
| ELF does not execute | Wrong entry point, load address or CPU state | ELF entry, platform RAM map, reset PC |
| `WhatIsAt` shows no device | Peripheral absent or address wrong | `.repl` mapping and base address |
| Register read is always zero | Wrong offset/base, stub behaviour, or write never reached | Monitor address, driver parameter, register offset |
| Linux boots but no serial output | Wrong UART console selection or UART not modelled as expected | `console=` argument, DTB, UART mapping |
| Kernel cannot find root/init | Missing or misplaced initramfs, incorrect boot arguments | Initramfs address/size, `/init`, archive contents |
| Module fails to load | Kernel/module mismatch, missing symbols, architecture/config issue | Build tree, `vermagic`, target architecture |
| Module loads but device is not affected | Wrong physical base or mapping failure | `request_mem_region`, `ioremap`, base address |
| Driver crashes on access | Invalid pointer, wrong register window/offset, bad cleanup | Use I/O accessors, validate mapping and bounds |
| Init succeeds partially then fails | Resource cleanup incomplete | Unwind successful setup operations in reverse order |

### Practical debug order

1. Verify the platform model first.
2. Verify the address map with `WhatIsAt`.
3. Verify the register directly with `ReadDoubleWord`.
4. Verify the ELF/kernel and boot state.
5. Verify UART output and boot arguments.
6. Verify the kernel module's target compatibility.
7. Verify the physical base and mapping.
8. Compare Linux-visible values with Renode monitor values.

This order isolates hardware-model problems before driver problems.

---

## 12. Exam-focused short answers

### Q1. Define simulation.

Simulation evaluates a model of system behaviour. It focuses on the expected functional response and may use behavioural or gate-level representations. Simulator wall-clock speed need not correspond to physical hardware time.

### Q2. Define emulation.

Emulation recreates the behaviour of a target system at a selected abstraction level so that target software, such as a processor binary or operating system, can execute against it. Its goal is functional equivalence at that abstraction.

### Q3. What is instruction-accurate emulation?

It models the architectural effect of each target instruction sufficiently accurately for target binaries to execute. It does not necessarily reproduce every internal gate transition or physical timing detail.

### Q4. What is Renode?

Renode is an open-source system-level emulation framework from Antmicro. It models processors, memory, buses and peripherals, supports debugging and can run target software such as bare-metal programs and Linux.

### Q5. What is the role of `.repl` and `.resc` files?

A `.repl` file describes the emulated platform hardware. A `.resc` script automates machine creation, platform loading, image loading, CPU setup, analyzer setup and execution.

### Q6. What is MMIO?

Memory-mapped I/O assigns peripheral registers addresses in the processor's address space. CPU loads and stores to those addresses access device registers through the system interconnect.

### Q7. Why is the DTB needed?

The Device Tree Blob describes the hardware platform to Linux, including devices and their resources. It helps the kernel identify and configure the hardware it is running on.

### Q8. What is initramfs?

Initramfs is an archive of initial user-space files unpacked by the kernel into a root filesystem in memory. It can contain `/init`, BusyBox, scripts and modules needed during early boot.

### Q9. Why use BusyBox?

BusyBox supplies compact implementations of common Unix utilities in a single executable, making it suitable for a small embedded Linux user space.

### Q10. Why must a module be built against the target kernel?

A loadable module depends on the target kernel's architecture, configuration, exported symbols and ABI/version information. A mismatch can prevent loading or cause runtime failures.

### Q11. Why use `ioremap()`?

It maps a physical MMIO range into kernel virtual address space. The driver then accesses registers using the mapped address and kernel I/O accessors.

### Q12. Why release resources in reverse order?

If initialization fails after partially acquiring resources, reverse-order cleanup ensures that every successfully acquired resource is released and avoids leaks or conflicting ownership.

### Q13. What does the Week 4 demonstration prove?

It demonstrates a path from a Linux user-space interface through a kernel module to an MMIO register in a Renode-emulated peripheral, and verifies that the changed register value is visible through the Renode monitor.

---

## 13. Likely long-answer questions and answer structure

### A. Explain the complete Renode bare-metal workflow.

Include: target C source → ARM bare-metal cross-compile → ELF → create Renode machine → load `.repl` → load ELF → set/confirm PC → start → inspect UART/registers. Explain why the entry point and RAM map must agree.

### B. Explain how Renode boots Linux.

Include: platform model → CPU/RAM/peripherals → kernel image → initramfs/rootfs → DTB (where required) → boot arguments → PC/register setup → `start` → kernel early boot → console output → `/init` → user-space shell. Discuss compatibility and memory addresses.

### C. Explain custom Linux module integration with MMIO.

Include: build module against kernel tree → include module in initramfs → boot kernel → `insmod` → request physical region → `ioremap` → `readl`/`writel` → expose sysfs/misc interface → verify register through Renode monitor → cleanup on unload. Explicitly distinguish physical and virtual addresses.

### D. Compare simulation, software emulation and hardware emulation.

Compare abstraction, execution speed, timing fidelity, target software execution and typical use. Mention that Renode is software-level system emulation, while FPGA-based hardware emulators execute synthesized RTL on a hardware platform.

---

## 14. Quick revision sheet

- Simulation = evaluate a model of behaviour; wall-clock speed is not hardware timing.
- Emulation = reproduce target-system behaviour at a useful abstraction.
- Functional equivalence is the objective; accuracy depends on abstraction level.
- Renode = Antmicro open-source system-level emulation framework.
- `.repl` = platform/hardware description.
- `.resc` = automation script.
- `sysbus` = system-bus view used for mapping and inspection.
- `WhatIsAt` = identify mapped component.
- `ReadDoubleWord` = read 32-bit bus value.
- Bare metal = no OS; application is loaded and starts at a known entry point.
- ELF contains load/entry metadata.
- Linux boot requires compatible CPU, memory, peripheral map and boot configuration.
- DTB describes hardware to Linux.
- UART analyzer displays serial console output.
- Cross compiler builds ARM code on a different host architecture.
- `multi_v7_defconfig` is the lecture's ARMv7 baseline configuration.
- BusyBox provides compact user-space utilities.
- initramfs supplies early user space in memory; `/init` starts it.
- Module must match the target kernel build.
- `request_mem_region` reserves the physical I/O range.
- `ioremap` maps physical MMIO into kernel virtual address space.
- `readl` / `writel` access mapped I/O registers.
- Cleanup must reverse successful initialization.
- Correct physical base address is essential.
- Cross-check Linux register writes against Renode monitor reads.

---

## 15. Source coverage and boundaries

These notes combine the Week 4 lecture transcript and slide deck:

- **Slides 3–5:** simulation, emulation and Renode.
- **Slide 6:** toolchain and Renode installation references.
- **Slide 7:** bare-metal ELF and bus inspection.
- **Slides 8–9:** stock VExpress and Zynq/ZedBoard Linux boot.
- **Slides 10–11:** kernel/BusyBox preparation and initramfs workflow.
- **Slide 12:** custom module integration.
- **Slide 13:** summary, `multi_v7_defconfig` and Zynq platform-specific considerations.
- **Lecture transcript, Lectures 1–7:** detailed explanation of platform files, memory mapping, Linux boot, custom kernel preparation and smart-timer MMIO module demonstration.

**Limitations of the supplied source:** the slide deck points to course repository configuration files and demo source code, but those files were not among the uploads. Therefore this guide explains the build and integration process without inventing the missing `.config`, `.repl`, `.resc`, `/init` or module source contents. The timing/sequence diagrams are explanatory illustrations, not source-provided AXI timing waveforms.
