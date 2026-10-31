# MMIO and AXI-Lite — Detailed Lecture Notes
### Exam preparation + practical RTL and simulation guide

**Primary source material:** `01-mmio-axi.pdf` (26-slide lecture) and `ELFPGA_W1.pdf` (lecture transcript and practical demonstration).  
**Scope:** memory-mapped peripherals, register banks, AXI-Lite channels and handshakes, Verilog implementation, Cocotb testing, Verilator simulation and GTKWave waveform inspection.

> **How to use these notes.** Read Sections 1–7 for the theory, Sections 8–12 for implementation and verification, and Section 13 for exam revision. The explanations follow the supplied lectures. Where a detail is a general AXI/RTL clarification rather than an explicit statement in the slides, it is labelled **Engineering note**.

---

## Contents
1. Digital-system foundations
2. CPU, memory and memory map
3. Memory-mapped I/O (MMIO)
4. Peripheral registers and register-bank design
5. AXI architecture and the five channels
6. VALID–READY handshake
7. AXI-Lite write and read transactions
8. Address decoding, alignment and WSTRB
9. Implementing a register bank in Verilog
10. Cocotb, Verilator and GTKWave workflow
11. Practical verification plan and debugging
12. Worked examples
13. Exam-focused questions and answers
14. Quick revision sheet and glossary

---

# 1. Digital-system foundations

## 1.1 Combinational logic

A combinational circuit produces an output determined by its **present inputs**. It does not intentionally retain a previous value.

Examples:
- AND, OR, XOR and inverter gates
- Multiplexers and decoders
- Adders and comparators
- FPGA lookup tables (LUTs), which can implement Boolean functions

For a combinational block:

\[
Y(t)=f(X_1(t),X_2(t),\ldots,X_n(t))
\]

If an input changes and propagation delay has elapsed, the output changes according to the Boolean function.

**Practical example — address decoder:** a decoder examines an address and asserts a register-select signal when the address matches a particular register offset.

## 1.2 Sequential logic and flip-flops

Sequential logic stores state. In the lecture, the main storage element is the edge-triggered D flip-flop.

At the active clock edge, the flip-flop samples D and updates Q. Between active edges, Q retains its value.

- Positive edge: clock transition from 0 to 1 (`posedge`).
- Negative edge: clock transition from 1 to 0 (`negedge`).

For a positive-edge D flip-flop:

\[
Q_{next}=D \quad \text{at the active rising edge}
\]

This is the hardware basis of registers, counters, state machines and transaction-control flags.

## 1.3 Datapath and control

A useful architectural decomposition is:

- **Datapath:** arithmetic/logic resources, registers, muxes and data-routing logic.
- **Control:** logic that decides which operation occurs and when values are captured.

The datapath may contain an adder or ALU with multiple possible inputs. Multiplexer select signals choose the operands and operation. The control logic often uses a finite-state machine (FSM).

An FSM uses:
1. Current state
2. Current inputs
3. Next-state logic
4. Output/control logic

The FSM controls datapath enables and selects. For an AXI peripheral, handshake flags and response-valid state are part of this control.

### Why this matters for AXI-Lite

An AXI-Lite register bank is a sequential system:
- input address/data are accepted on clock edges;
- accepted values may be stored in internal registers;
- address decoding and read-data selection are combinational;
- response-valid flags and handshake state are retained until the receiver accepts them.

## 1.4 Synchronous and asynchronous reset

The supplied Verilog example uses:

```verilog
always @(posedge ACLK or negedge ARESETn)
```

This is an **active-low asynchronous reset**:
- `ARESETn = 0` asserts reset immediately, independent of the clock.
- `ARESETn = 1` releases reset; normal state updates occur on `posedge ACLK`.

The suffix `n` conventionally indicates active-low polarity. Reset should place internal state and response flags into known values.

**Exam point:** distinguish reset polarity from reset timing. Active-low is about the logic level; asynchronous is about whether assertion requires a clock edge.

---

# 2. CPU, memory and memory map

## 2.1 CPU datapath model

A simplified CPU contains:
- **Registers:** small, fast storage inside the processor.
- **ALU:** performs arithmetic and logic operations.
- **Control unit:** determines operand selection, operation and destination.
- **Instruction storage:** supplies instructions that guide control.

The ALU needs operands from registers. Results are written back to registers. Since the number of CPU registers is limited, the CPU also accesses larger external memory.

## 2.2 Basic memory read and write

Think of memory as a collection of addressable locations.

A read conceptually supplies:
- an address;
- a read request/enable;
- a returned data word.

A write conceptually supplies:
- an address;
- data to store;
- a write enable.

The address identifies *where*; the data identifies *what*.

## 2.3 What is a memory map?

A **memory map** assigns address ranges to different resources. The CPU issues loads and stores; address-decode/interconnect logic decides which target should respond.

A simplified system:

```text
                 address + control + write data
+------+       ------------------------------------>
| CPU  |              Address decoder / interconnect
|      |       <------------------------------------
+------+                 read data / response
                               |
              +----------------+----------------+
              |                |                |
          +--------+       +--------+       +---------+
          | SRAM   |       | Timer  |       | GPIO    |
          | block  |       | regs   |       | regs    |
          +--------+       +--------+       +---------+
```

The CPU does not need a fundamentally different instruction to access a peripheral register than to access memory. It uses a load/store to an address that the system has assigned to the peripheral.

## 2.4 Address decoding

Suppose a peripheral occupies a region selected by the upper address bits. The interconnect decodes the region and forwards the lower address bits to the peripheral.

The lecture's small example uses a 4-bit local address. The interconnect can decode the larger system address and pass the low four bits to the register bank. That gives 16 byte-address offsets in the local region.

Address regions are often aligned to powers of two because upper-bit decoding is simple.

**Example:** if a peripheral base is `0x4000_0000`, and a 32-bit register is at offset `0x04`, its full address is:

\[
0x4000\_0000 + 0x04 = 0x4000\_0004
\]

The base address is system integration information; the offset is local peripheral design information.

## 2.5 Alignment and address units

A 32-bit word contains four bytes. In a byte-addressed system, consecutive word registers are commonly separated by four byte addresses:

| Register | Byte offset | Meaning |
|---|---:|---|
| Control | `0x00` | First 32-bit register |
| Data | `0x04` | Second 32-bit register |
| Status | `0x08` | Third 32-bit register |
| Reserved | `0x0C` | Unused/reserved |

For a 32-bit word-aligned register interface, the two least-significant address bits identify a byte position within a word. A simple register bank may therefore ignore/drop those bits when selecting a word register.

**Important:** dropping `ADDR[1:0]` is only correct for the intended aligned word-register map. A design that supports sub-word or unaligned accesses must define those semantics explicitly.

---

# 3. Memory-mapped I/O (MMIO)

## 3.1 Definition

**Memory-mapped I/O** is a system organization in which peripheral control/status registers are assigned addresses in the processor's address space.

The CPU accesses a peripheral by reading or writing its mapped address. An interconnect routes the transaction to the peripheral, which exposes a small register bank.

## 3.2 Memory versus MMIO

| Ordinary memory | MMIO register |
|---|---|
| Stores program/data values | Controls or reports hardware state |
| Read usually returns stored data | Read may return live status or have side effects |
| Write usually updates memory contents | Write may start/stop hardware or clear an event |
| Often cacheable | Usually mapped as device/uncached memory |
| Bulk data storage | Small control/status interface |

The same load/store mechanism can reach both, but the *meaning and side effects* differ.

**Engineering note:** exact cacheability, ordering and access attributes depend on the processor architecture and memory system. Software must use the platform's device-memory rules; ordinary cached pointers are not automatically safe for MMIO.

## 3.3 Why MMIO is useful

MMIO gives software a standard way to configure hardware:
- set a PWM period or duty cycle;
- enable or disable a timer;
- load a counter;
- read a present count;
- check whether an event occurred;
- configure transfer length and source/destination for a DMA engine.

The peripheral performs the time-critical or repetitive work in hardware. Software configures it and checks status.

## 3.4 Why registers are not a high-speed data path

The lecture emphasizes that register access is primarily for configuration and control, not high-throughput bulk transfer. For example, a CPU might program a DMA engine through registers and then let DMA move a large data block.

---

# 4. Peripheral registers and register-bank design

## 4.1 What is a register bank?

A register bank is a collection of addressable hardware registers exposed by a peripheral. Each register has:
- an address/offset;
- a width;
- access permissions;
- a defined reset value;
- a functional meaning.

The lecture's counter example illustrates common register types:

| Register | Width | Access | Purpose |
|---|---:|---|---|
| Enable | 1 bit | Read/write | Start or stop counting |
| Up/Down | 1 bit | Read/write | Select count direction |
| Load | 32 bits | Write | Load an initial count |
| Present count | 32 bits | Read | Observe current count |
| Threshold | 32 bits | Write | Program comparison threshold |
| Trigger | 1 bit | Write | Trigger an action/event |

The exact behavior of each register is a peripheral design decision. The table is the conceptual example from the lecture.

## 4.2 Control registers

A control register contains software-written configuration bits.

Example:
- bit 0: enable;
- bit 1: count direction;
- bit 2: interrupt enable.

Software writes a value to change hardware behavior. The hardware logic consumes those bits.

## 4.3 Status registers

A status register reports hardware state to software.

Example:
- counter overflowed;
- transfer completed;
- peripheral is busy/idle;
- threshold reached;
- input event detected.

A status register is often read-only from the software side, though some designs use write-one-to-clear (W1C) semantics. W1C is a general design pattern, not a behavior specified by the supplied counter example.

## 4.4 Read-only, write-only and read/write

- **Read/write (R/W):** software can read the current stored value and update it.
- **Read-only (R/O):** software can inspect a value generated by hardware.
- **Write-only (W/O):** software sends a command or one-shot value; reading may be undefined or return a fixed value.

The lecture uses the counter's present count as a read-only example and the load value as a write-only example.

## 4.5 Register design questions

For every register, define:
1. What is its offset?
2. What is its width?
3. What is its reset value?
4. Which bits are writable?
5. Which bits are readable?
6. Does a write have a side effect?
7. Does hardware update it independently?
8. What happens on reserved/invalid accesses?

A register map should be documented before RTL is written.

---

# 5. AXI architecture and the five channels

## 5.1 What is AXI?

AXI (Advanced eXtensible Interface) is an Arm-defined on-chip communication protocol used to transfer information between managers and subordinates. The lecture introduces AXI as a standardized way to communicate with peripherals and other hardware blocks.

The supplied material focuses on **AXI-Lite**, a simpler memory-mapped interface suitable for control/status registers. AXI-Lite does not use burst transactions in the way full AXI does; each access is a single-beat transfer.

Terminology:
- **Manager**: initiates a transaction (historically called master).
- **Subordinate**: responds to a transaction (historically called slave).
- **Source**: drives a particular channel's VALID and payload.
- **Destination**: receives that channel and drives READY.

The source/destination role is channel-specific. For example, on AW the manager is source; on B the subordinate is source.

## 5.2 Five independent channels

AXI has five channels:

| Channel | Full name | Direction for a manager-initiated access | Payload/control |
|---|---|---|---|
| AW | Write Address | Manager → Subordinate | `AWADDR`, `AWVALID`, `AWREADY` |
| W | Write Data | Manager → Subordinate | `WDATA`, `WSTRB`, `WVALID`, `WREADY` |
| B | Write Response | Subordinate → Manager | `BRESP`, `BVALID`, `BREADY` |
| AR | Read Address | Manager → Subordinate | `ARADDR`, `ARVALID`, `ARREADY` |
| R | Read Data | Subordinate → Manager | `RDATA`, `RRESP`, `RVALID`, `RREADY` |

The write address and write data are separate channels. They can be accepted in different clock cycles and in either order. The response is sent after the subordinate has processed the write.

A read has an address phase followed by a read-data/response phase.

## 5.3 Payload and handshake signals

For every channel:
- `VALID` is driven by the source and indicates that payload is available.
- `READY` is driven by the destination and indicates that it can accept the payload.
- Payload is transferred only when `VALID` and `READY` are both high at the same active clock edge.

For AXI-Lite:
- AW payload: address.
- W payload: write data and byte strobes.
- B payload: write response code.
- AR payload: read address.
- R payload: read data and read response code.

## 5.4 Response codes

The supplied lecture uses:
- `2'b00`: `OKAY` — successful access.
- `2'b10`: `SLVERR` — subordinate error.

**Engineering note:** AXI response encodings also define `EXOKAY` and `DECERR` in the wider AXI family. The simple register-bank example only needs OKAY and SLVERR. An interconnect may use DECERR for an unmapped address; the peripheral example uses SLVERR for an address it does not implement.

---

# 6. VALID–READY handshake

## 6.1 The fundamental rule

A transfer occurs on a rising clock edge for which:

\[
\boxed{VALID \land READY = 1}
\]

This is the most important AXI handshake rule.

- If `VALID=1` and `READY=0`, no transfer occurs.
- If `VALID=0` and `READY=1`, no transfer occurs.
- If both are 1 at the sampling edge, exactly one transfer occurs.

## 6.2 Responsibilities of source and destination

The source:
1. Asserts VALID when it has a transfer to send.
2. Must not wait for READY before asserting VALID.
3. Keeps VALID asserted until a handshake occurs.
4. Keeps the associated payload stable while VALID is high and READY is low.

The destination:
1. Asserts READY when it can accept a transfer.
2. May wait for VALID before asserting READY.
3. May assert READY before VALID.
4. Deasserts READY when it cannot accept more data.

The lecture explicitly highlights the rule: a source cannot wait for READY before asserting VALID; a destination may wait for VALID before asserting READY.

## 6.3 Timing diagram: READY already high

![VALID and READY handshake when READY is already asserted](images/valid_ready_early_ready.png)

Interpretation:
- Cycle 1: VALID becomes high while READY is already high.
- At the next active rising edge, the transfer is accepted.
- The source may then remove VALID or present the next payload according to the interface's transaction flow.

## 6.4 Timing diagram: delayed READY / backpressure

![VALID held while READY is low](images/valid_ready_backpressure.png)

Interpretation:
- The source asserts VALID and presents a payload.
- READY remains low for several cycles. No transfer has happened.
- VALID and payload remain stable.
- When READY becomes high, the next rising edge completes the transfer.

This is called **backpressure**: the receiver temporarily prevents further transfers because it is not ready.

## 6.5 Common handshake mistakes

1. **Dropping VALID too early:** the source removes VALID before READY is high at a sampling edge. The transfer is lost.
2. **Changing payload while stalled:** payload changes while VALID=1 and READY=0. The receiver may sample the wrong information.
3. **Waiting for READY before asserting VALID:** this violates the source rule and can create deadlock if both sides wait.
4. **Treating VALID alone as a transfer:** VALID means “available,” not “accepted.”
5. **Treating READY alone as a transfer:** READY means “can accept,” not “data accepted.”

---

# 7. AXI-Lite write and read transactions

## 7.1 Write transaction: conceptual sequence

A write consists of three channel transfers:
1. AW: manager sends write address.
2. W: manager sends write data and byte strobes.
3. B: subordinate returns a write response.

AW and W are independent. The subordinate must handle the address arriving before data, data arriving before address, or both being accepted in the same cycle.

![AXI-Lite write transaction with independent AW and W channels](images/axi_write_transaction.png)

### Step-by-step

1. Manager asserts `AWVALID` and supplies `AWADDR`.
2. Subordinate asserts `AWREADY` when it can accept the address.
3. On an edge with `AWVALID && AWREADY`, the address is captured.
4. Independently, manager asserts `WVALID` and supplies `WDATA` and `WSTRB`.
5. On an edge with `WVALID && WREADY`, data and strobes are captured.
6. Once both address and data have been captured, the subordinate decodes the address and updates the selected register.
7. Subordinate asserts `BVALID` and supplies `BRESP`.
8. Manager asserts `BREADY` when it can accept the response.
9. On an edge with `BVALID && BREADY`, the response transfer completes. The subordinate can clear BVALID and accept the next write according to its design.

**Critical point:** do not assume AW and W handshakes occur simultaneously.

## 7.2 Write-address capture

The slide's example tracks whether the AW handshake has completed:

```verilog
if (!aw_hs_done && saxi_awvalid && saxi_awready) begin
    aw_hs_done <= 1'b1;
    awaddr_q   <= saxi_awaddr;
end

if (saxi_bvalid && saxi_bready) begin
    aw_hs_done <= 1'b0;
end
```

Meaning:
- capture the address only on the AW handshake;
- mark the address as received;
- keep the state until the write response is accepted;
- clear the flag to prepare for another write.

The reset branch initializes the flag and stored address.

## 7.3 Write-data capture

The corresponding W-channel logic captures both data and byte strobes:

```verilog
if (!w_hs_done && saxi_wvalid && saxi_wready) begin
    w_hs_done <= 1'b1;
    wdata_q   <= saxi_wdata;
    wstrb_q   <= saxi_wstrb;
end

if (saxi_bvalid && saxi_bready) begin
    w_hs_done <= 1'b0;
end
```

`WDATA` is the payload. `WSTRB` says which bytes of that payload are valid for writing.

## 7.4 Timing diagram: AW and W arrive independently

![AW and W channels complete on different cycles](images/axi_aw_w_independent.png)

In this example:
- AW handshake happens first.
- W handshake happens later.
- The subordinate waits until both have been accepted before committing the register write.
- The response is then presented on B.

A correct design must also handle the reverse order (W before AW).

## 7.5 Timing diagram: write response backpressure

![BVALID remains asserted until BREADY](images/axi_b_response.png)

The subordinate asserts BVALID after processing the write. If BREADY is low, BVALID and BRESP must remain stable. The response is complete only when both are high at a rising edge.

## 7.6 Read transaction

A read consists of:
1. AR: manager sends the read address.
2. R: subordinate returns read data and response code.

![AXI-Lite read address and read response timing](images/axi_read_transaction.png)

### Step-by-step

1. Manager asserts `ARVALID` with `ARADDR`.
2. Subordinate asserts `ARREADY` when it can accept the address.
3. On `ARVALID && ARREADY`, the address is captured.
4. Subordinate decodes the address and selects the corresponding register value.
5. Subordinate places the value on `RDATA`, sets `RRESP`, and asserts `RVALID`.
6. Manager asserts `RREADY` when it can accept the result.
7. On `RVALID && RREADY`, the read-data/response transfer completes.

The R channel carries **both** data and response status.

## 7.7 Read response backpressure

If `RVALID=1` while `RREADY=0`, the subordinate must keep `RVALID`, `RDATA` and `RRESP` stable until the handshake. It must not replace the read result with a different address's result while the current response is stalled.

## 7.8 Transfer versus transaction

The lecture distinguishes:
- **Transfer:** one VALID/READY handshake on one channel.
- **Transaction:** the complete operation made up of the required channel transfers.

For a write, there are three channel transfers (AW, W, B). For a read, there are two (AR, R). AXI-Lite uses single-beat data transfers.

---

# 8. Address decoding, alignment and WSTRB

## 8.1 Example register map from the lecture

The practical Verilog example has two 32-bit registers:

| Local word select | Byte offset | Register |
|---|---:|---|
| `2'b00` | `0x0` | `reg_control` |
| `2'b01` | `0x4` | `reg_data` |
| `2'b10` | `0x8` | Invalid in this example |
| `2'b11` | `0xC` | Invalid in this example |

The local address is four bits wide. The lecture uses the upper two local address bits as the word select after dropping the bottom two byte-offset bits.

For 32-bit words:

```verilog
word_addr = awaddr_q[3:2];
```

This maps offsets 0 and 4 to word indices 0 and 1.

## 8.2 Why are the lowest two address bits dropped?

A 32-bit word contains four bytes. Word-aligned addresses are multiples of four:
- `0x0`, `0x4`, `0x8`, `0xC`, ...

The lowest two address bits select a byte within a 4-byte word. If the design only supports aligned 32-bit registers, they are not needed to choose the register.

## 8.3 WSTRB — write strobes

For a 32-bit data bus, there are four byte lanes, so `WSTRB` is four bits wide.

| WSTRB bit | Data byte lane | Meaning when 1 |
|---|---|---|
| `WSTRB[0]` | `WDATA[7:0]` | Update byte 0 |
| `WSTRB[1]` | `WDATA[15:8]` | Update byte 1 |
| `WSTRB[2]` | `WDATA[23:16]` | Update byte 2 |
| `WSTRB[3]` | `WDATA[31:24]` | Update byte 3 |

A strobe bit of 0 means that byte lane must not be modified.

For a 32-bit register `reg_data`, the byte-wise update is conceptually:

```verilog
for (int i = 0; i < 4; i++) begin
    if (wstrb_q[i])
        reg_data[i*8 +: 8] <= wdata_q[i*8 +: 8];
end
```

This preserves bytes whose strobe bits are zero.

**Engineering note:** the lecture's Cocotb partial-write test writes a two-byte value into the low two byte lanes and checks that the upper two bytes remain unchanged. Endianness determines how the byte sequence is interpreted as an integer; it does not change the meaning of the byte-lane strobe bits.

## 8.4 Invalid addresses

The example supports offsets 0 and 4. Other local word-select values produce a subordinate error response:
- write: set `BRESP = SLVERR`;
- read: set `RDATA` to zero and `RRESP = SLVERR`.

This makes unsupported accesses observable rather than silently writing an unrelated register.

---

# 9. Implementing a register bank in Verilog

## 9.1 Interface direction

When the register bank is the subordinate:
- AW address and AWVALID are inputs; AWREADY is an output.
- WDATA, WSTRB and WVALID are inputs; WREADY is an output.
- BRESP and BVALID are outputs; BREADY is an input.
- ARADDR and ARVALID are inputs; ARREADY is an output.
- RDATA, RRESP and RVALID are outputs; RREADY is an input.

The manager initiates requests; the subordinate returns READY and responses.

## 9.2 Internal state

A simple implementation typically needs:
- `awaddr_q`: latched write address;
- `wdata_q`: latched write data;
- `wstrb_q`: latched byte strobes;
- `aw_hs_done`: address accepted flag;
- `w_hs_done`: data accepted flag;
- register storage such as `reg_control` and `reg_data`;
- response registers/flags such as `bvalid`, `bresp`, `rvalid`, `rdata`, `rresp`.

These are sequential state elements because they must retain information between clock edges.

## 9.3 READY generation

The lecture's compact implementation drives AWREADY and WREADY low while the corresponding handshake has already been accepted and the subordinate is waiting to finish the response. During reset, READY is held low.

A simplified conceptual expression is:

```verilog
assign saxi_awready = ARESETn && !aw_hs_done;
assign saxi_wready  = ARESETn && !w_hs_done;
```

This is a simplified explanation of the lecture's control approach, not a complete AXI-Lite implementation template. A production design must also consider response buffering and simultaneous channel activity.

## 9.4 Write commit condition

The register write should occur only after both independent inputs have arrived:

```verilog
wire do_write = aw_hs_done && w_hs_done && !saxi_bvalid;
```

The exact timing depends on how flags are updated and whether the implementation uses registered or combinational control. In the lecture, `do_write` is used to indicate that the address and data handshakes have completed and the response has not yet been issued.

## 9.5 Write decode and response

Conceptually:

```verilog
case (awaddr_q[3:2])
  2'b00: begin
      // Update reg_control using WSTRB
      bresp_q <= 2'b00; // OKAY
  end

  2'b01: begin
      // Update reg_data using WSTRB
      bresp_q <= 2'b00; // OKAY
  end

  default: begin
      bresp_q <= 2'b10; // SLVERR
  end
endcase

bvalid_q <= 1'b1;
```

Then clear BVALID only when the response handshake occurs:

```verilog
if (bvalid_q && saxi_bready)
    bvalid_q <= 1'b0;
```

The code above is an instructional fragment. It omits reset assignments, signal declarations, and the complete arbitration/flag sequencing needed for a synthesizable module.

## 9.6 Read decode and response

The read side captures the address on AR handshake, decodes it and prepares a response:

```verilog
case (saxi_araddr[3:2])
  2'b00: begin
      rdata_q <= reg_control;
      rresp_q <= 2'b00; // OKAY
  end

  2'b01: begin
      rdata_q <= reg_data;
      rresp_q <= 2'b00; // OKAY
  end

  default: begin
      rdata_q <= 32'b0;
      rresp_q <= 2'b10; // SLVERR
  end
endcase

rvalid_q <= 1'b1;
```

Hold RVALID and the associated data/response until `RVALID && RREADY`.

## 9.7 Nonblocking assignments

Sequential Verilog uses nonblocking assignments (`<=`) for state updated on clock edges. All right-hand sides are evaluated using the pre-edge values, then updates take effect together.

This matters when setting handshake flags, response-valid signals and register data in the same clocked process. Do not reason about nonblocking assignments as if each line immediately updates the variable.

## 9.8 Design checklist

- Reset all state to known values.
- Capture address only on AW handshake.
- Capture data and WSTRB only on W handshake.
- Do not assume AW and W arrive together.
- Commit the write only when both are available.
- Hold BVALID and BRESP until BREADY handshake.
- Capture read address only on AR handshake.
- Hold RVALID, RDATA and RRESP until RREADY handshake.
- Decode only documented offsets.
- Return an error for unsupported offsets.
- Respect every WSTRB bit.
- Avoid changing payload while VALID is asserted and READY is low.

---

# 10. Cocotb, Verilator and GTKWave workflow

## 10.1 Why use a Python testbench?

Cocotb is a Python-based co-simulation/testbench framework. It allows a test to:
- drive DUT input signals;
- generate or wait for clocks;
- apply reset;
- perform bus transactions;
- check outputs with assertions;
- organize tests into separate cases.

The lecture uses Cocotb with Verilator to simulate a Verilog register bank.

## 10.2 Roles of the tools

| Tool | Role |
|---|---|
| Verilog/SystemVerilog | Describes the hardware design (DUT) |
| Cocotb | Python testbench and stimulus/checking |
| Verilator | Compiles and simulates the RTL; the lecture notes its C++-based compiled simulation flow |
| GTKWave | Opens waveform dumps so signals can be inspected over time |
| `make` | Runs the project build/test flow configured by the project |

Cocotb provides clock/time triggers so Python can coordinate with the HDL simulator. Python code is not inherently clocked; Cocotb's triggers synchronize testbench actions with simulation time.

## 10.3 Typical testbench structure

```python
import cocotb
from cocotb.clock import Clock
from cocotb.triggers import RisingEdge, Timer

@cocotb.test()
async def basic_register_test(dut):
    # Start clock
    cocotb.start_soon(Clock(dut.aclk, 10, units="ns").start())

    # Apply reset
    dut.aresetn.value = 0
    await Timer(30, units="ns")
    dut.aresetn.value = 1
    await RisingEdge(dut.aclk)

    # Drive/check DUT signals or use an AXI-Lite bus helper.
```

The exact API spelling can vary by Cocotb version; check the installed version's documentation. The lecture demonstrates a bus helper that discovers the DUT signals by a common prefix (`saxi_`) and exposes read/write methods.

## 10.4 AXI-Lite bus helper

In the lecture, the testbench creates an AXI-Lite bus object using the signal prefix. The helper takes care of:
- driving AW address and VALID;
- driving W data and VALID;
- observing READY;
- waiting for the B response;
- performing the read address and read-data handshakes.

This reduces repetitive low-level signal manipulation. It does not remove the need to understand the protocol; the waveform should still be inspected to confirm the handshakes.

## 10.5 Example test strategy from the lecture

1. Reset the DUT.
2. Write a value to register offset `0x00`.
3. Write a value to register offset `0x04`.
4. Read offset `0x04`.
5. Convert the returned bytes to an integer and compare with the expected value.
6. Perform a partial write to offset `0x04`.
7. Read back the complete word and verify only the strobed byte lanes changed.
8. Run the simulation and inspect the waveform.

The lecture reports two tests passing in its demonstration and uses GTKWave to inspect AW, W and B activity.

## 10.6 Verilator and waveform inspection

The lecture explains that Verilator compiles the RTL into a C/C++ simulation model and builds the required Python interface. The test command runs the tests and can generate waveform output.

GTKWave is then used to inspect:
- clock and reset;
- AWVALID/AWREADY/AWADDR;
- WVALID/WREADY/WDATA/WSTRB;
- BVALID/BREADY/BRESP;
- ARVALID/ARREADY/ARADDR;
- RVALID/RREADY/RDATA/RRESP;
- internal handshake flags and register values.

---

# 11. Practical verification plan and debugging

## 11.1 Minimum test matrix

| Test | Stimulus | Expected result |
|---|---|---|
| Reset | Assert reset | Flags clear; registers at defined reset values |
| Basic write | Write `0x12345678` to offset `0x04` | Register stores value |
| Readback | Read offset `0x04` | Returns `0x12345678`, OKAY |
| AW before W | Delay W after AW | Address retained; write completes after W |
| W before AW | Delay AW after W | Data retained; write completes after AW |
| READY stall | Hold BREADY low | BVALID/BRESP remain stable |
| Read stall | Hold RREADY low | RVALID/RDATA/RRESP remain stable |
| Partial write | Assert only selected WSTRB lanes | Only selected bytes change |
| Invalid write | Write unsupported offset | SLVERR response |
| Invalid read | Read unsupported offset | Zero data and SLVERR in the lecture example |

## 11.2 What to inspect in GTKWave

For each channel, identify the clock edge where VALID and READY are both high. That is the transfer edge.

**Write debug sequence**
1. Did AWVALID rise?
2. Was AWADDR stable until AW handshake?
3. Did AWVALID and AWREADY overlap at a rising edge?
4. Did WVALID and WREADY overlap at a rising edge?
5. Did the subordinate wait for both address and data?
6. Did the selected register update correctly?
7. Did BVALID assert with the expected BRESP?
8. Did BVALID remain asserted until BREADY?

**Read debug sequence**
1. Was ARADDR stable while ARVALID was high?
2. Did ARVALID and ARREADY handshake?
3. Was the correct register selected?
4. Did RDATA and RRESP match the expected values?
5. Did RVALID remain high during a stall?
6. Did the R handshake occur only when RREADY became high?

## 11.3 Common failure patterns

| Symptom | Likely cause | What to inspect |
|---|---|---|
| Write never completes | AW or W handshake missing | VALID/READY and reset |
| Wrong register changes | Address decode/offset error | Address bits and word select |
| Partial write corrupts other bytes | WSTRB ignored or lane mapping wrong | WSTRB and byte slices |
| Response disappears too early | VALID cleared before READY | BVALID/BREADY or RVALID/RREADY |
| Read returns old/unknown value | Register reset or update timing issue | reset values and sequential logic |
| Invalid address looks successful | Default decode branch missing | BRESP/RRESP |
| Simulation hangs | Testbench waiting for a handshake that never occurs | clock, reset, VALID, READY |
| Waveform difficult to read | Too many signals or poor grouping | group signals by AXI channel |

---

# 12. Worked examples

## 12.1 Register offset calculation

A peripheral base address is `0x4000_0000`. The data register is at local offset `0x04`.

\[
\text{Address} = \text{Base} + \text{Offset}
\]

\[
=0x4000\_0000+0x04=0x4000\_0004
\]

A software store to this address is routed by the interconnect to the peripheral and decoded locally as the data register.

## 12.2 Determine the handshake cycle

Assume at a rising edge:
- VALID = 1
- READY = 0

No transfer occurs.

At the next rising edge:
- VALID = 1
- READY = 1

The transfer occurs at this second edge. If READY had become high between edges, the transfer still occurs at the next active sampling edge.

## 12.3 Partial write example

Suppose the existing register value is:

`0x11223344`

A write updates only the lower two byte lanes with `0xAABB`, and `WSTRB = 4'b0011`.

Byte lanes:
- lane 0 gets `0xBB`;
- lane 1 gets `0xAA`;
- lanes 2 and 3 are unchanged.

New register value:

`0x1122AABB`

The upper two bytes remain `0x11` and `0x22`.

## 12.4 Invalid register offset

The example register bank supports only `0x00` and `0x04`. An access to `0x08` is not mapped in this implementation.

Expected response in the lecture's example:
- write: `BRESP = 2'b10` (SLVERR);
- read: `RDATA = 0`, `RRESP = 2'b10` (SLVERR).

## 12.5 Explain why AW and W need separate flags

A manager may send the write address and write data independently. If the address arrives first, the subordinate must remember it while waiting for data. If data arrives first, it must remember the data and strobes while waiting for the address. Two flags (`aw_hs_done`, `w_hs_done`) represent these independent events. The write can be committed only when both are complete.

---

# 13. Exam-focused questions and answers

### Q1. Define MMIO.
MMIO is a method in which peripheral registers are assigned addresses in the processor's address space. The CPU uses load/store operations, and an interconnect routes the access to the selected peripheral.

### Q2. Why are peripheral registers used?
They allow software to configure hardware, issue commands and inspect status. They are primarily control/status interfaces, not high-speed bulk-data paths.

### Q3. Differentiate combinational and sequential logic.
Combinational outputs depend on present inputs. Sequential logic stores state and updates state in response to clock edges (for edge-triggered elements).

### Q4. What are the five AXI channels?
AW (write address), W (write data), B (write response), AR (read address), and R (read data/response).

### Q5. When does an AXI transfer occur?
At an active clock edge where VALID and READY are both high.

### Q6. Can the source wait for READY before asserting VALID?
No. The source must assert VALID when it has a valid transfer. It then holds VALID and payload stable until the handshake.

### Q7. Can READY be asserted before VALID?
Yes. The destination may assert READY in advance or wait until VALID appears.

### Q8. Why are AW and W independent?
Address and data are carried on separate channels and may handshake in different cycles. The subordinate must store each independently and combine them to perform the write.

### Q9. What is WSTRB?
WSTRB is a byte-enable vector. Each bit controls whether the corresponding byte lane of WDATA is written.

### Q10. What is the purpose of BVALID and BREADY?
BVALID indicates that the subordinate has a write response available. BREADY indicates that the manager can accept it. The response transfers when both are high at a rising edge.

### Q11. What is the difference between a transfer and a transaction?
A transfer is one channel-level VALID/READY handshake. A transaction is the complete operation, consisting of the required channel transfers.

### Q12. What does SLVERR indicate in the lecture's example?
It indicates that the subordinate could not perform the requested access, such as an unsupported register offset.

### Q13. Why are the bottom two address bits ignored for a 32-bit word register map?
A 32-bit word is four bytes and word-aligned addresses are multiples of four. The two low bits select byte position within a word, not the word register.

### Q14. What is the role of Cocotb?
It provides a Python testbench environment to drive the DUT, synchronize with simulation time, perform transactions and check expected results.

### Q15. What is GTKWave used for?
It displays simulation waveforms so the designer can inspect signal timing, handshakes, state transitions and data values.

### Q16. Why must RDATA remain stable while RVALID=1 and RREADY=0?
The receiver has not accepted the read response. The subordinate must preserve the offered payload until the handshake occurs.

### Q17. What does reset do in the example?
It initializes handshake flags and stored state to known values and prevents the peripheral from accepting transactions while reset is active.

### Q18. What happens if AW completes before W?
The subordinate stores the address and sets its AW-complete flag. It waits for the W handshake, then performs the write and issues the B response.

---

# 14. Quick revision sheet and glossary

## 14.1 Five channels — memorize this table

| Channel | Direction | VALID | READY | Payload |
|---|---|---|---|---|
| AW | Manager → Subordinate | AWVALID | AWREADY | AWADDR |
| W | Manager → Subordinate | WVALID | WREADY | WDATA, WSTRB |
| B | Subordinate → Manager | BVALID | BREADY | BRESP |
| AR | Manager → Subordinate | ARVALID | ARREADY | ARADDR |
| R | Subordinate → Manager | RVALID | RREADY | RDATA, RRESP |

## 14.2 Essential equations and rules

\[
\text{Handshake} = VALID \land READY
\]

\[
\text{Write complete} \Rightarrow AW\ handshake + W\ handshake + B\ response\ handshake
\]

\[
\text{Read complete} \Rightarrow AR\ handshake + R\ response\ handshake
\]

- Source must not wait for READY before asserting VALID.
- VALID and payload remain stable until handshake.
- AW and W can arrive in either order.
- A response remains valid until accepted.
- For 32-bit data, WSTRB has four byte-enable bits.
- The lecture example maps local offsets `0x00` and `0x04`.
- `OKAY = 2'b00`; example `SLVERR = 2'b10`.

## 14.3 Glossary

| Term | Meaning |
|---|---|
| MMIO | Memory-Mapped Input/Output |
| AXI | Advanced eXtensible Interface |
| AXI-Lite | Lightweight AXI memory-mapped interface for simple accesses |
| Manager | Transaction initiator |
| Subordinate | Transaction responder |
| DUT | Design Under Test |
| FSM | Finite State Machine |
| LUT | Lookup Table |
| AW | Write Address channel |
| W | Write Data channel |
| B | Write Response channel |
| AR | Read Address channel |
| R | Read Data channel |
| VALID | Source indicates payload is available |
| READY | Destination indicates it can accept payload |
| WSTRB | Write byte strobe/byte enable |
| BRESP | Write response code |
| RRESP | Read response code |
| Cocotb | Python-based HDL verification framework |
| Verilator | RTL simulator/compiler |
| GTKWave | Waveform viewer |

---

## Source alignment and scope notes

- `01-mmio-axi.pdf` supplies the Week 1 outline, digital-system review, register examples, MMIO fundamentals, five-channel AXI overview, handshake rules, AW/W capture fragments, transfer-versus-transaction distinction and summary.
- `ELFPGA_W1.pdf` supplies the extended lecture explanation and practical walkthrough: CPU/memory map, counter register semantics, Cocotb testbench structure, register-bank signals, two-register address map, independent AW/W handshakes, WSTRB partial writes, OKAY/SLVERR handling, read/write tests, Verilator execution and GTKWave inspection.
- The timing diagrams in this guide are newly drawn teaching diagrams that illustrate the handshake rules described in the supplied material. They are not copied from the source slides.
- The code fragments are instructional excerpts/pseudocode. A complete peripheral must include all declarations, reset behavior, state sequencing and interface details before synthesis.
