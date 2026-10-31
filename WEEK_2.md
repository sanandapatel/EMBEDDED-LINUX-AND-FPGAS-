# Register Banks and AXI4-Lite — Exam-Oriented Lecture Notes

> **Course material:** `02-register-banks.pdf` (Week 2 slides) and `ELFPGA_W2.pdf` (lecture transcript).
> **Focus:** register-bank design, MMIO programmer's model, UART datasheet interpretation, smart-timer/PWM example, byte strobes, register side-effects, shadow/active configuration, and RTL verification.
>
> **How to use these notes:** Read Sections 1–7 for concepts, Sections 8–12 for implementation, then revise the timing diagrams, worked examples, and exam questions. The lecture's own Smart Timer register map and terminology are retained. Additional protocol explanations are included where needed to make the implementation understandable.

---

## 1. The big picture: why a register bank exists

A hardware datapath performs the actual work: counting, filtering, packet processing, PWM generation, data conversion, and so on. A CPU needs a controlled way to configure that datapath and observe what it is doing. A **register bank** provides this software-visible interface.

The CPU accesses the peripheral through **memory-mapped I/O (MMIO)**. Each register is assigned an address (usually an offset from a peripheral base address). A software load reads a register; a software store writes one. The register bank translates those transactions into internal control signals and state.

### 1.1 Analogy

Think of a washing machine:

- The **datapath** is the motor, pump, drum, and control machinery.
- The **register bank** is the control panel: wash mode, spin speed, start, stop, status lights.
- The **CPU/software** is the person selecting settings and checking the display.
- The **bus interface** is the wiring and communication protocol between the control panel and the machine.

The person does not need to know the motor's internal commutation sequence. Likewise, software should not need to know the internal RTL implementation. It needs a documented set of registers and rules.

### 1.2 Main design goals

1. **A small, discoverable API:** a documented address map and bit definitions tell firmware how to use the peripheral.
2. **Decoupling:** software transaction timing should not dictate the datapath's cycle-by-cycle operation.
3. **Observability/debug:** status flags, counters, and error bits expose internal events to software.
4. **Predictable side-effects:** reading or writing a register must have explicitly documented consequences.
5. **Extensibility:** reserved address space and reserved bits can allow future revisions, but software must not depend on them.

### 1.3 Functional description versus programmer's model

A datasheet generally has two complementary viewpoints:

| Datasheet part | Main question | Typical contents |
|---|---|---|
| Functional description | What does the hardware do, and what are its limitations? | Features, data path, supported rates, FIFO depth, interfaces, operating conditions |
| Programmer's model | How does software operate it? | Register summary, offsets, access types, reset values, bit fields, usage sequences, side-effects |

**Exam point:** The functional description helps decide whether a peripheral is suitable. The programmer's model tells a firmware developer exactly how to configure and use it.

---

## 2. MMIO address map and register anatomy

### 2.1 Base address plus offset

A peripheral is assigned a **base address** by the system address map. Individual registers are selected using fixed offsets:

\[
\text{Register address} = \text{Peripheral base address} + \text{Register offset}
\]

For example, if a peripheral is mapped at `0x4000_0000`, and a register has offset `0x08`, its absolute address is `0x4000_0008`.

The base address is platform-dependent; the offsets are defined by the peripheral's programmer's model. This is why portable driver code often receives the peripheral base address from a platform header/device tree rather than assuming it is universal.

### 2.2 Byte addresses and word alignment

A 32-bit register occupies four byte lanes. In a conventional word-aligned map, consecutive 32-bit registers are placed at offsets:

- `0x00`
- `0x04`
- `0x08`
- `0x0C`

The address bus is byte-addressed, even though the register width is 32 bits. Therefore, an address increment of four selects the next 32-bit register.

For a 32-bit data bus, the low two address bits select a byte position within a 4-byte word. In the Week 2 Smart Timer design, the word index is decoded with:

```verilog
wire [1:0] word_sel_w = awaddr_q[3:2];
wire [1:0] word_sel_r = araddr_q[3:2];
```

Thus:

| `ADDR[3:2]` | Offset | Register |
|---:|---:|---|
| `2'b00` | `0x00` | CTRL |
| `2'b01` | `0x04` | PERIOD |
| `2'b10` | `0x08` | DUTY |
| `2'b11` | `0x0C` | STATUS |

**Important distinction:** Selecting a word with `ADDR[3:2]` does not by itself implement byte writes. Byte-level write qualification is handled by `WSTRB`.

### 2.3 Address aliasing and alignment

If hardware ignores `ADDR[1:0]`, addresses `0x00`, `0x01`, `0x02`, and `0x03` may all select the same 32-bit register. This is called **aliasing**.

The lecture recommends avoiding accidental confusion by enforcing/documenting word-level access. A robust peripheral should define what it does with misaligned accesses: accept and alias them, ignore them, or return an error. Do not leave the behavior accidental.

### 2.4 What a register definition should document

For every register, document:

- Offset and name.
- Width (e.g., 32 bits).
- Access type (RW, RO, WO, W1C, W1P, etc.).
- Reset value or explicitly unspecified reset state.
- Bit-field names and meanings.
- Reserved bits and required read/write behavior.
- Side-effects of reads and writes.
- Whether byte strobes are supported.
- Any sequencing requirement (for example, disable before changing configuration).

### 2.5 Reserved and unused locations

A reserved address or bit is **not spare storage for application software**. The implementation may change in a future revision. A read may return an unspecified value, and a write may have an undocumented effect.

Common safe implementation policy:

- **RAZ:** Read-As-Zero.
- **WI:** Writes-Ignored.
- Together, **RAZ/WI** means reserved bits read as zero and writes to them do not alter state.

Even when a particular datasheet says RAZ/WI, software should still avoid using reserved fields. Do not rely on reserved behavior unless the specification explicitly guarantees it.

---

## 3. Register access semantics and side-effects

The letters in a register summary are not merely labels. They define the software contract.

| Type / pattern | Meaning | Typical use |
|---|---|---|
| RW | Read and write | Configuration such as period, divider, mode |
| RO | Read only | Status, counters, identification |
| WO | Write only | Command/data push register |
| W1C | Write 1 to clear | Sticky interrupt/error/event flags |
| W1S | Write 1 to set | Set bits without read-modify-write |
| W1T | Write 1 to toggle | Toggle control bits |
| W0C / W0S | Write 0 to clear / set | Less common; must be explicitly documented |
| RC | Read to clear | Event/status registers where a read acknowledges an event |
| W1P | Write 1 to pulse | Start, reset, trigger, kick-watchdog command |
| Sticky-until-reset | Once set, remains set until specified clear/reset | Fault/event capture |
| Lock / write-once | Configuration becomes immutable after a write or lock event | Secure/boot configuration |
| SET/CLR aliases | Separate addresses set or clear selected bits | Avoid read-modify-write races |
| Shadowed register | Software writes a staging value; hardware applies it later | Glitch-free updates at a safe boundary |

### 3.1 RW, RO, and WO

- **RW:** A write updates the stored value; a later read normally returns that value.
- **RO:** Software can observe the state but cannot directly modify it. The hardware changes it.
- **WO:** Software can write a command or data item, but readback may be undefined or may expose a different meaning. A UART transmit data register is a common conceptual example: writing pushes a byte into the transmit path.

A single address can have different read and write meanings. The UART data register discussed in the lecture is an example: writes supply transmit data, while reads return received data (and, in FIFO mode, associated receive status bits).

### 3.2 W1C: write-one-to-clear

A W1C flag is cleared by writing a one to its bit position. Writing zero leaves it unchanged.

Example: `STATUS.WRAP` is sticky.

| Value written to bit 0 | Result |
|---:|---|
| 0 | Do not clear the flag |
| 1 | Clear the flag |

Why W1C is useful:

- Hardware can set a flag when an event occurs.
- Software can clear only the event(s) it has handled.
- Software does not need to read a register, modify a bit, and write the whole value back.
- This reduces read-modify-write races, especially when hardware can set flags asynchronously to software's sequence.

**RTL sketch:**

```verilog
if (write_status && wstrb_q[0] && wdata_q[0])
    status_wrap <= 1'b0;

if (wrap_pulse)
    status_wrap <= 1'b1;
```

If both clear and set happen in the same clock cycle, the assignment priority determines the result. In the sketch above, the later assignment has priority in a single procedural block. A production design must consciously specify event-versus-clear priority; do not leave it to accidental statement ordering.

### 3.3 W1P: write-one-to-pulse

A W1P bit is a command, not a stored state bit. Writing one generates a one-clock-cycle pulse. Reading the bit returns zero.

```verilog
// Default every cycle
ctrl_rst_pulse <= 1'b0;

// Reassert only on a qualifying write
if (write_ctrl && wstrb_q[0])
    ctrl_rst_pulse <= wdata_q[1];
```

Use W1P for actions such as soft reset, start capture, or kick watchdog. Do not use it for a condition that must remain asserted. For a sustained reset, use a normal state bit or a dedicated reset protocol.

### 3.4 Read-to-clear (RC) versus W1C

With RC, the act of reading clears the event. This can be dangerous when multiple software contexts, interrupt handlers, or diagnostic tools may read the same register: one reader can clear information before another reader observes it.

W1C is often preferable for shared status because software explicitly acknowledges the bits it has handled. RC can still be appropriate when the peripheral specification requires it and the ownership/reading rules are clear.

### 3.5 Set/clear aliases and read-modify-write hazards

Suppose a control register contains independent bits. Software performs:

```c
REG |= (1u << 3);   // read, modify, write
```

If hardware changes another bit between the read and write, the stale value may overwrite that change. Dedicated `REG_SET` and `REG_CLR` addresses avoid this class of race by applying a bit mask directly.

---

## 4. Byte enables: AXI4-Lite WSTRB

A 32-bit write-data bus contains four byte lanes. `WSTRB[3:0]` tells the slave which lanes contain valid write bytes.

| `WSTRB` | Byte lane 3 | Byte lane 2 | Byte lane 1 | Byte lane 0 |
|---|---|---|---|---|
| `4'b0001` | Keep | Keep | Keep | Write |
| `4'b0010` | Keep | Keep | Write | Keep |
| `4'b0100` | Keep | Write | Keep | Keep |
| `4'b1000` | Write | Keep | Keep | Keep |
| `4'b1111` | Write | Write | Write | Write |
| `4'b0000` | Keep | Keep | Keep | Keep |

For a 32-bit register, byte lane `i` maps to bits `[8*i +: 8]`.

```verilog
integer i;
for (i = 0; i < DATA_WIDTH/8; i = i + 1) begin
    if (wstrb_q[i])
        period_shadow[i*8 +: 8] <= wdata_q[i*8 +: 8];
end
```

### 4.1 Worked example

Assume:

- Current `PERIOD = 32'h1122_3344`
- `WDATA = 32'hAABB_CCDD`
- `WSTRB = 4'b0101`

The enabled lanes are lane 0 and lane 2:

- Lane 0 replaces bits `[7:0]`: `44 → DD`
- Lane 2 replaces bits `[23:16]`: `22 → BB`
- Lanes 1 and 3 are preserved.

Result:

\[
\text{PERIOD}_{new} = \texttt{32'h11BB\_33DD}
\]

### 4.2 Short fields inside 32-bit registers

In the Smart Timer, CTRL uses only byte 0:

- `WSTRB[0]` must be one for CTRL bits 0 and 1 to be acted upon.
- `WSTRB[1]`, `[2]`, and `[3]` should not accidentally change unrelated state.
- Reserved CTRL bits should be RAZ/WI.

**Exam point:** `WSTRB` is a byte mask, not a bit mask. Each strobe controls one 8-bit lane.

---

## 5. AXI4-Lite transaction mechanics needed for the register bank

AXI4-Lite is a lightweight memory-mapped, single-beat interface. It uses independent channels for write address, write data, write response, read address, and read data. The Week 2 lecture builds on the earlier AXI handshake material and uses the stored handshakes to update the register bank.

### 5.1 VALID/READY handshake rule

For any AXI channel, a transfer occurs on a rising clock edge when:

\[
\boxed{\text{Handshake} = VALID \land READY}
\]

- The **source** asserts `VALID` when its information is available.
- The **destination** asserts `READY` when it can accept that information.
- The source must keep `VALID` and the associated payload stable until the handshake.
- A transfer does not occur merely because VALID is high or READY is high; both must be high at the sampling edge.

This mechanism allows either side to introduce wait cycles without losing data.

### 5.2 Write channels

| Channel | Signals | Purpose |
|---|---|---|
| AW | `AWVALID`, `AWREADY`, `AWADDR` | Write address |
| W | `WVALID`, `WREADY`, `WDATA`, `WSTRB` | Write data and byte enables |
| B | `BVALID`, `BREADY`, `BRESP` | Write response |

The address and data channels are **independent**. The address may handshake before the data, or the data may handshake before the address. The slave must remember each accepted item until it has both pieces and can perform the write.

A simplified register-bank implementation may use:

- `aw_hs_done`: an address handshake has occurred and the address is stored.
- `w_hs_done`: a data handshake has occurred and data/strobes are stored.
- `do_write`: both pieces are available and no previous B response is blocking a new write.

The slide's expression is:

```verilog
wire do_write = aw_hs_done && w_hs_done && ~saxi_bvalid;
```

This is a design-specific internal condition, not an AXI signal. It says the register update is performed only after both write address and write data have been captured, while avoiding a new write during an outstanding response in this simple implementation.

### 5.3 Write timing diagram

![AXI-Lite write timing](images/axi_write_timing.png)

**How to read it:**

1. AWVALID and WVALID are allowed to rise at different times.
2. AW is accepted at the edge where AWVALID and AWREADY are both high.
3. W is accepted at the edge where WVALID and WREADY are both high.
4. Once the slave has both address and data, it updates the selected register.
5. The slave asserts BVALID with BRESP. The response is consumed when BVALID and BREADY are both high.

The drawing intentionally shows address acceptance before data acceptance. A correct design must also handle the reverse ordering.

### 5.4 Read channels

| Channel | Signals | Purpose |
|---|---|---|
| AR | `ARVALID`, `ARREADY`, `ARADDR` | Read address |
| R | `RVALID`, `RREADY`, `RDATA`, `RRESP` | Read data and response |

The slave accepts the address when `ARVALID && ARREADY`. It then selects the register value, presents it on RDATA, and asserts RVALID. The master accepts the read result when `RVALID && RREADY`.

![AXI-Lite read timing](images/axi_read_timing.png)

### 5.5 Responses and error behavior

AXI response encodings commonly include:

- `OKAY`: successful access.
- `SLVERR`: slave detected an error.
- `DECERR`: address decode error (often generated by an interconnect/default slave).

The Week 2 slide checklist explicitly says to respond consistently with OKAY/SLVERR. The exact policy for unmapped addresses, misalignment, and unsupported accesses must be defined by the peripheral/system design.

### 5.6 AXI-Lite implementation checklist

- Capture address and data independently.
- Do not assume AW and W arrive in the same cycle.
- Hold response VALID until the response handshake.
- Keep payload stable while VALID is high and READY is low.
- Apply WSTRB to every writable byte.
- Do not perform a register write twice while waiting for BREADY.
- Ensure read data remains stable while RVALID is high and RREADY is low.
- Define behavior for unsupported addresses and access types.

---

## 6. Case study: ARM PrimeCell UART programmer's model

The lecture uses an ARM UART to demonstrate how to read a real peripheral datasheet. The point is not to memorize every UART register; it is to learn how to interpret the programmer's model.

### 6.1 UART hardware blocks

A UART may contain:

- Transmit and receive logic.
- Baud-rate generator.
- Transmit and receive FIFOs.
- FIFO control and interrupt generation.
- APB/register-bus interface.
- DMA request/interface signals.
- Reset and clock interfaces.

The actual external serial interface may be only TX and RX, but the CPU-facing control interface is much larger. The lecture emphasizes that control/management signals can outnumber the functional I/O pins.

### 6.2 FIFO intuition

A FIFO is a first-in, first-out queue:

- Software can enqueue multiple bytes.
- Hardware transmits them serially in the background.
- Software need not wait for every individual bit/byte to finish.
- Receive hardware can queue incoming bytes until software or DMA reads them.

FIFO status bits tell software whether the queue is empty, full, or has data available. They prevent writes into a full transmit FIFO and reads from an empty receive FIFO.

### 6.3 Register summary: how to interpret columns

A typical register summary contains:

- Offset
- Register name
- Access type
- Reset value
- Width
- Brief description

The summary is a navigation aid. The detailed per-register description defines bit meanings and side-effects.

### 6.4 UART data register example

The lecture describes the UART data register as having different read and write behavior:

- A write supplies transmit data, typically in the low eight bits.
- A read returns received data.
- When receive FIFOs are enabled, the read value may include the 8-bit data plus four receive error/status bits (break, framing, parity, overrun).
- Reserved upper bits must not be used.

The lecture's key lesson: **one address can be read/write but have different semantic meanings in each direction.**

### 6.5 Flag/status register example

A UART flag register can expose:

- TX FIFO empty.
- TX FIFO full.
- RX FIFO empty.
- RX FIFO full.
- Busy.
- Modem/handshake status such as CTS, DSR, DCD, RI (depending on the UART).

For example:

- TX FIFO empty generally indicates that more transmit data can be queued.
- TX FIFO full means software should not enqueue another byte.
- RX FIFO empty means there is no received byte waiting.
- RX FIFO full indicates no more receive data can be accepted without draining the FIFO.
- Busy indicates that transmission or another UART operation is in progress.

The exact bit positions and reset values must be taken from the specific UART revision's TRM.

### 6.6 Reset values are meaningful

A FIFO-empty flag may reset to one because the FIFO is empty after reset. A FIFO-full flag should normally reset to zero. Therefore, not every register bit must reset to zero.

If a datasheet marks a reset value as unspecified/don't-care, software must not assume it is zero. Read the field description and mask reserved/unused bits before using a register value.

### 6.7 Practical firmware pattern (illustrative)

```c
#define UART_BASE       (/* platform-defined base address */)
#define UART_DR         (*(volatile unsigned int *)(UART_BASE + 0x00))
#define UART_FR         (*(volatile unsigned int *)(UART_BASE + 0x18))

#define UART_FR_TXFE    (1u << 7)
#define UART_FR_RXFE    (1u << 4)

void uart_putc(unsigned char ch)
{
    while ((UART_FR & UART_FR_TXFE) == 0u) {
        /* Wait until the transmit FIFO is empty.
           Some drivers instead wait for TX FIFO not-full. */
    }
    UART_DR = ch;
}
```

This is an explanatory pattern, not a complete PL011 driver. In a production driver, waiting for FIFO-not-full is generally more efficient than waiting for FIFO-empty when multiple bytes can be queued. Always follow the specific UART's TRM.

---

## 7. Smart Timer: understand the datapath first

The Smart Timer is a PWM peripheral. Its core is independent of the bus interface; the AXI-Lite wrapper makes its parameters software-programmable.

The slide's core interface is:

```verilog
module pwm_core (
    input  wire        clk,
    input  wire        rstn,
    input  wire        en,
    input  wire [31:0] period,
    input  wire [31:0] duty,
    output reg         pwm_out,
    output reg         wrap
);
```

### 7.1 PWM operation

The core:

1. Counts upward while enabled.
2. Compares the current count against `duty`.
3. Drives PWM high while `count < duty`.
4. Produces a wrap indication when the counter reaches its terminal count and restarts from zero.

A simplified conceptual implementation is:

```verilog
always @(posedge clk or negedge rstn) begin
    if (!rstn) begin
        count   <= 0;
        pwm_out <= 0;
        wrap    <= 0;
    end else begin
        wrap    <= 0;                 // default: one-cycle pulse
        if (!en) begin
            count   <= 0;
            pwm_out <= 0;
        end else begin
            pwm_out <= (count < duty);
            if (count >= period) begin
                count <= 0;
                wrap  <= 1;
            end else begin
                count <= count + 1;
            end
        end
    end
end
```

This is a teaching sketch consistent with the slide's stated behavior; exact edge ordering and whether the terminal count is `period` or `period-1` must be checked against the actual RTL. That distinction changes the period by one clock.

### 7.2 PWM equations

If the counter visits values 0 through `P` inclusive, then:

\[
N = P+1 \quad \text{clock cycles per PWM period}
\]

\[
f_{\mathrm{PWM}} = \frac{f_{\mathrm{clk}}}{P+1}
\]

For a conventional compare rule `count < D`, with `D` values in the range 0 through `P+1`:

\[
D_{\mathrm{fraction}} \approx \frac{D}{P+1},
\qquad
\text{Duty percentage} \approx 100\frac{D}{P+1}\%
\]

These equations assume the counter includes zero and the terminal count, and that the output is high for exactly `D` counter states. Confirm the RTL's terminal-count convention before calculating exact frequency or duty.

### 7.3 Illustrative timing diagram

![PWM counter and output](images/pwm_counter_timing.png)

The example uses a terminal count of 4 and duty threshold 3. The output is high for counter values 0, 1, and 2, and low for 3 and 4. The counter wraps after the terminal count.

### 7.4 Edge cases to reason about

- `duty = 0`: `count < duty` is never true, so output remains low.
- `duty` greater than the counter's maximum/period: output may remain high for the entire cycle.
- `period = 0`: depending on RTL, the counter may wrap every clock.
- `en = 0`: the lecture's conceptual core uses enable to stop/reset the counter; check exact RTL.
- Changing period or duty in the middle of a count can create a shortened/extended cycle unless updates are staged.

---

## 8. Smart Timer V1 register map

The register map is the software API for the PWM datapath.

| Offset | Register | Access | Bits / behavior | Reset |
|---:|---|---|---|---|
| `0x00` | CTRL | RW + W1P | bit 0 = EN; bit 1 = RST pulse; other bits RAZ/WI | EN=0; pulse reads 0 |
| `0x04` | PERIOD | RW | Terminal count; software writes shadow value | `0x000000FF` |
| `0x08` | DUTY | RW | PWM compare threshold; software writes shadow value | `0x00000000` |
| `0x0C` | STATUS | RO + W1C | bit 0 = sticky WRAP; bit 1 = UPD_PENDING | flags cleared |

### 8.1 CTRL (`BASE + 0x00`)

- Bit 0 `EN`: persistent state. Write 1 to enable the PWM core; write 0 to disable it.
- Bit 1 `RST`: W1P command. Writing 1 creates a one-cycle reset pulse; readback is always zero.
- Other bits: RAZ/WI.
- The lecture's implementation checks byte lane 0 before interpreting either control bit.

### 8.2 PERIOD (`BASE + 0x04`)

- 32-bit RW configuration register.
- Reset value: `0x000000FF` (255).
- Software writes the shadow copy.
- If the PWM is disabled, the active configuration can be updated immediately.
- If enabled, the update is marked pending and committed at wrap.

### 8.3 DUTY (`BASE + 0x08`)

- 32-bit RW configuration register.
- Reset value: `0x00000000`.
- Defines the compare threshold for `count < duty`.
- Uses the same shadow/active update policy as PERIOD.

### 8.4 STATUS (`BASE + 0x0C`)

- Bit 0 `WRAP`: sticky status. It is set by a wrap event and remains set until software writes one to clear it.
- Bit 1 `UPD_PENDING`: indicates that new shadow configuration is waiting for the next safe commit boundary. It is cleared after the active registers are updated.
- Other bits: reserved/RAZ/WI in this design.

### 8.5 Why shadow values are echoed on read

The readback path returns `period_shadow` and `duty_shadow`, not necessarily the active values. This lets software verify what it most recently programmed, even while the old values remain active until the next wrap.

This is an important distinction:

- **Shadow value:** what software has requested.
- **Active value:** what the datapath is currently using.

If software needs to inspect the active configuration, the design would need separate active-value readback registers or another documented mechanism.

---

## 9. Shadow and active registers: coherent updates

### 9.1 The problem with immediate writes

Suppose the active PWM period is 1000 and the counter is currently at 700. Software writes a new period of 100.

If the active period changes immediately, the current count is already greater than the new terminal count. The core may wrap unexpectedly or produce a truncated pulse. The behavior can be confusing and may violate the requirement that the current PWM cycle finish normally.

### 9.2 Shadow-register solution

Maintain two copies of configuration:

- `period_shadow`, `duty_shadow`: written by software.
- `period_active`, `duty_active`: consumed by the PWM datapath.

Policy:

1. If disabled, copy shadow values to active values immediately.
2. If enabled, update the shadow values and set `UPD_PENDING`.
3. Wait for the natural wrap boundary.
4. At wrap, copy both shadow values to active values in the same clocked update.
5. Clear `UPD_PENDING`.

This ensures PERIOD and DUTY change as one coherent configuration at a known boundary.

![Shadow configuration timing](images/shadow_commit_timing.png)

### 9.3 Why this is atomic/coherent

The period and duty settings are related. Updating one without the other can temporarily create an inconsistent PWM configuration. Copying both at the same boundary ensures the next PWM cycle uses a matched pair.

The same technique generalizes to:

- Timer reload values.
- Clock/rate dividers.
- Video timing registers applied at vertical blanking (vblank).
- Motor-control parameters applied at a commutation/PWM boundary.
- Packet/descriptor configuration applied between transactions.

### 9.4 UPD_PENDING versus WRAP

Do not confuse these two status bits:

- `UPD_PENDING` is a temporary hardware state. It clears automatically when the shadow values are committed.
- `WRAP` is an event history flag. It is sticky and remains set until software explicitly clears it using W1C.

---

## 10. RTL implementation: from bus handshake to datapath control

A clean register-bank implementation can be understood in layers.

### 10.1 Layer 1 — Capture bus payloads

The slave captures the address on the AW handshake and captures data plus strobes on the W handshake. Because the channels are independent, the captured values may arrive on different cycles.

Conceptually:

```verilog
if (s_axi_awvalid && s_axi_awready) begin
    awaddr_q   <= s_axi_awaddr;
    aw_hs_done <= 1'b1;
end

if (s_axi_wvalid && s_axi_wready) begin
    wdata_q   <= s_axi_wdata;
    wstrb_q   <= s_axi_wstrb;
    w_hs_done <= 1'b1;
end
```

This is pseudocode: a complete slave also needs to clear the capture flags at the correct time, generate BVALID, and manage backpressure.

### 10.2 Layer 2 — Decode the register

For this four-register map:

```verilog
wire [1:0] word_sel_w = awaddr_q[3:2];

case (word_sel_w)
    2'b00: /* CTRL   */
    2'b01: /* PERIOD */
    2'b10: /* DUTY   */
    2'b11: /* STATUS */
endcase
```

The address decode should remain small and combinationally simple. The lecture emphasizes that bus-facing logic should be straightforward and fast.

### 10.3 Layer 3 — Apply write semantics

A register write is not always a simple `reg <= WDATA`:

- CTRL bit 0 is stored state.
- CTRL bit 1 generates a pulse.
- PERIOD and DUTY update shadow registers, with byte enables.
- STATUS bit 0 is W1C.
- Reserved bits are ignored.

The meaning of each write depends on address, byte strobe, and data bit.

### 10.4 Layer 4 — Update active datapath configuration

The active registers are updated based on enable state and wrap boundary. This is separate from the AXI bus handshake. The CPU can complete a write transaction before the datapath applies the new configuration.

### 10.5 Layer 5 — Readback mux

The read path selects the value associated with the read address:

```verilog
case (s_axi_araddr[3:2])
    2'b00: s_axi_rdata <= {30'd0, 1'b0, ctrl_en};
    2'b01: s_axi_rdata <= period_shadow;
    2'b10: s_axi_rdata <= duty_shadow;
    2'b11: s_axi_rdata <= {30'd0, upd_pending, status_wrap};
endcase
```

Notice:

- CTRL bit 1 reads as zero because it is a pulse command, not stored state.
- PERIOD and DUTY read back their shadow values.
- STATUS returns current status state.
- Reserved bits read as zero.

### 10.6 Nonblocking assignments and procedural priority

In clocked Verilog, nonblocking assignments (`<=`) schedule register updates for the end of the time step. Multiple `always` blocks execute concurrently; do not assume one block runs before another.

Within one procedural block, if the same register is assigned more than once in the same clock event, the last executed assignment generally determines the scheduled value. This is why default assignments (such as clearing a pulse) are often placed near the start of a clocked block, and a later qualifying condition reasserts the pulse.

**Design discipline:** Make priority explicit for simultaneous conditions such as hardware event set versus software W1C clear. Avoid assigning the same state register from multiple clocked processes.

---

## 11. Practical firmware usage sequence

Assume the peripheral base address is supplied by the system. The following sequence illustrates the software-visible contract.

### 11.1 Configure while disabled

1. Disable the peripheral (`CTRL.EN = 0`).
2. Write PERIOD.
3. Write DUTY.
4. Read back PERIOD and DUTY to verify the shadow values.
5. Clear any stale WRAP status by writing one to STATUS bit 0.
6. Set `CTRL.EN = 1`.

### 11.2 Update while running

1. Write the new PERIOD value.
2. Write the new DUTY value.
3. Read STATUS until `UPD_PENDING` clears.
4. The new configuration has been committed at a wrap boundary.
5. If required, clear sticky WRAP using W1C.

### 11.3 C-style pseudocode

```c
#define REG32(addr) (*(volatile unsigned int *)(addr))

#define TIMER_BASE       (/* assigned by platform */)
#define TIMER_CTRL       REG32(TIMER_BASE + 0x00)
#define TIMER_PERIOD     REG32(TIMER_BASE + 0x04)
#define TIMER_DUTY       REG32(TIMER_BASE + 0x08)
#define TIMER_STATUS     REG32(TIMER_BASE + 0x0C)

#define CTRL_EN          (1u << 0)
#define CTRL_RST_W1P     (1u << 1)
#define STATUS_WRAP      (1u << 0)
#define STATUS_UPD_PEND  (1u << 1)

void timer_configure(unsigned int period, unsigned int duty)
{
    TIMER_CTRL = 0u;                 // disable
    TIMER_PERIOD = period;
    TIMER_DUTY = duty;
    TIMER_STATUS = STATUS_WRAP;     // W1C: clear sticky WRAP
    TIMER_CTRL = CTRL_EN;            // enable
}

void timer_request_reset(void)
{
    TIMER_CTRL = CTRL_EN | CTRL_RST_W1P; // EN state + one-cycle reset command
}
```

**Important:** The example assumes a full-word write to CTRL and the peripheral's defined behavior for EN and RST. In a real driver, ensure byte-lane behavior and concurrent updates are handled correctly. Do not use generic read-modify-write on W1C or command registers unless the register specification explicitly permits it.

### 11.4 Polling pending status

```c
while ((TIMER_STATUS & STATUS_UPD_PEND) != 0u) {
    /* Wait for the next wrap boundary and shadow-to-active commit. */
}
```

A production driver may prefer an interrupt or timeout rather than an unbounded busy loop.

---

## 12. Verification and simulation strategy

The lecture describes walking through `smart_timer_axil.v` and simulating it, then inspecting the waveform in GTKWave. The verification objective is to test both the bus protocol and the peripheral behavior.

### 12.1 Directed tests

| Test | Stimulus | Expected result |
|---|---|---|
| Reset | Assert/deassert reset | EN=0, PERIOD shadow reset to `0xFF`, DUTY shadow=0, status cleared |
| CTRL enable | Write bit 0 = 1 | PWM starts counting |
| CTRL reset pulse | Write bit 1 = 1 with byte 0 enabled | One-cycle reset pulse; readback bit 1 = 0 |
| PERIOD write | Write a 32-bit value | Shadow period updates |
| DUTY write | Write a 32-bit value | Shadow duty updates |
| Byte strobe | Write selected byte lanes only | Only enabled bytes change |
| W1C | Set WRAP, write 1 to bit 0 | WRAP clears |
| W1C with zero | Write 0 to bit 0 | WRAP remains unchanged |
| Shadow update | Change PERIOD/DUTY while EN=1 | UPD_PENDING sets; active values wait for wrap |
| Commit | Cause wrap while update pending | Both active values update; pending clears |
| Readback | Read each offset | Correct value and reserved/pulse bits read as specified |
| Backpressure | Hold BREADY/RREADY low | Response remains valid and stable until accepted |
| Channel skew | AW and W arrive on different cycles | Exactly one correct write occurs |

### 12.2 What to inspect in GTKWave

Group related signals so the waveform tells a story:

- Clock/reset.
- AW channel: AWVALID, AWREADY, AWADDR.
- W channel: WVALID, WREADY, WDATA, WSTRB.
- B channel: BVALID, BREADY, BRESP.
- AR channel: ARVALID, ARREADY, ARADDR.
- R channel: RVALID, RREADY, RDATA, RRESP.
- Internal capture flags and decoded write enable.
- CTRL.EN and CTRL reset pulse.
- PERIOD/DUTY shadow and active values.
- Counter, PWM output, wrap pulse.
- STATUS.WRAP and UPD_PENDING.

Do not inspect only the bus pins. A bus write can be accepted correctly while the internal register update or datapath commit is wrong.

### 12.3 Assertions / properties worth checking

- A handshake occurs only when VALID and READY are simultaneously high at a clock edge.
- Once VALID is asserted, payload remains stable until handshake.
- A W1P output is one cycle wide.
- A W1C bit clears only when the relevant write strobe and data bit are both one.
- A disabled timer can apply configuration immediately (as specified by this design).
- An enabled timer applies pending configuration only at wrap.
- PERIOD and DUTY active values change together at commit.
- A single accepted write causes exactly one register update.
- Read data is stable while RVALID=1 and RREADY=0.

---

## 13. Common design mistakes and how to avoid them

1. **Assuming AW and W arrive together.** They are independent channels. Capture each independently.
2. **Ignoring WSTRB.** Partial writes can corrupt unrelated bytes or command bits.
3. **Treating a pulse as stored state.** A W1P command should default low and be asserted only for the intended cycle.
4. **Using read-to-clear for shared flags without considering races.** Prefer W1C where appropriate.
5. **Changing active PWM parameters mid-cycle.** Use shadow registers and a safe commit boundary.
6. **Returning shadow values while documenting them as active values.** State exactly what readback represents.
7. **Forgetting reserved bits.** Define and implement RAZ/WI where appropriate.
8. **Leaving reset values unspecified accidentally.** Every state element needs a deliberate reset policy.
9. **Not defining simultaneous event/clear priority.** Specify whether a new hardware event wins over software clear.
10. **Assuming the terminal count means period+1 in every implementation.** Inspect the actual counter comparison and derive timing from it.
11. **Testing only nominal transactions.** Include backpressure, split AW/W timing, partial writes, and status races.
12. **Overcomplicating the address decode.** Keep the register map compact and decode only the necessary address bits.

---

## 14. Exam-focused revision sheet

### 14.1 Definitions to memorize

- **MMIO:** CPU accesses peripheral registers through the system address space.
- **Register bank:** A set of software-visible registers that configure and observe a hardware block.
- **Programmer's model:** Register offsets, access permissions, reset values, bit fields, and usage rules.
- **WSTRB:** Byte-lane write-enable mask.
- **W1C:** Writing one clears the corresponding status bit.
- **W1P:** Writing one generates a one-cycle pulse.
- **Sticky flag:** Remains asserted until explicitly cleared or reset.
- **Shadow register:** Staging copy written by software; applied to active datapath state at a safe boundary.
- **VALID/READY handshake:** Transfer occurs at a rising edge where both are high.
- **RAZ/WI:** Read-as-zero, writes-ignored.

### 14.2 Important equations

For a 32-bit data bus:

\[
\text{Number of byte lanes} = \frac{32}{8}=4
\]

\[
\text{Register address} = \text{base address}+\text{offset}
\]

For a counter visiting 0 through terminal count \(P\):

\[
N=P+1,\qquad f_{\mathrm{PWM}}=\frac{f_{\mathrm{clk}}}{P+1}
\]

For duty threshold \(D\), assuming `count < D`:

\[
\text{Duty fraction}\approx \frac{D}{P+1}
\]

### 14.3 Short-answer questions

**Q1. Why are register banks used?**  
They provide a documented MMIO interface through which software configures a datapath, observes status, and debugs hardware without knowing its internal implementation.

**Q2. Differentiate functional description and programmer's model.**  
The functional description explains what the peripheral does and its limitations. The programmer's model explains how software controls it through registers.

**Q3. Why is the peripheral base address not necessarily fixed?**  
The system address map assigns the base address. The register offsets remain fixed within the peripheral's programmer's model.

**Q4. What is WSTRB?**  
A byte-enable vector. Each bit controls whether the corresponding 8-bit lane of WDATA is written.

**Q5. Explain W1C with an example.**  
A status bit is cleared by writing one to that bit. Writing zero leaves it unchanged. It allows software to acknowledge selected events without a read-modify-write sequence.

**Q6. Why use W1P for reset?**  
A reset action may need only a single clock pulse. W1P turns a software write into a one-cycle command and reads back as zero.

**Q7. What is a sticky status bit?**  
A bit that remains set after the event pulse ends, allowing software to observe the event later. It is cleared by a defined action such as W1C.

**Q8. Why use shadow registers in PWM?**  
To avoid changing period/duty in the middle of a PWM cycle. The new values are applied together at wrap.

**Q9. Why must AW and W be handled independently?**  
AXI4-Lite write address and write data are separate channels and may handshake on different cycles.

**Q10. When does an AXI VALID/READY transfer occur?**  
At a rising clock edge where VALID and READY are both high.

### 14.4 Longer design questions

1. Draw and explain a four-register MMIO map for a PWM peripheral.
2. Explain the implementation of a W1C sticky status bit, including event/clear priority.
3. Draw an AXI4-Lite write timing diagram in which AW and W handshakes occur on different cycles.
4. Explain WSTRB using a 32-bit register and a partial-byte write example.
5. Explain how shadow and active registers prevent glitches in a PWM generator.
6. Describe the readback behavior of CTRL, PERIOD, DUTY, and STATUS in the Smart Timer.
7. Propose a verification plan for the Smart Timer AXI-Lite peripheral.
8. Explain how a UART's data and flag registers expose FIFO state to software.

---

## 15. Source alignment and important cautions

These notes are based primarily on the two supplied Week 2 materials:

- `02-register-banks.pdf`: especially slides 2–7 (design goals and UART programmer's model), 9–17 (PWM core, Smart Timer register map, address decode, WSTRB, W1P, W1C, shadow/active, readback), and 19–22 (register behavior patterns and checklist).
- `ELFPGA_W2.pdf`: lecture transcript covering the UART datasheet example, address offsets and reserved fields, Smart Timer/PWM behavior, register semantics, RTL implementation, and simulation/waveform inspection.

**Cautions when applying the notes to a real design:**

- The diagrams in this README are explanatory timing diagrams generated for these notes; they are not screenshots of a measured simulation waveform.
- The PWM equations depend on the exact terminal-count comparison in the RTL. Confirm whether the counter wraps at `count == period` or another condition.
- The Smart Timer slide gives a concise behavioral specification. A production implementation must define simultaneous W1C/event priority, misaligned access policy, unsupported-address response, and all reset values precisely.
- The UART example is used to teach datasheet reading. Exact PL011 bit positions, FIFO modes, and register semantics must be checked against the exact ARM TRM revision used by the target system.

---

## 16. Final mental model

A good peripheral has three cleanly separated pieces:

1. **Datapath:** performs the actual function (the PWM counter and comparator).
2. **Register bank:** defines the software contract (CTRL, PERIOD, DUTY, STATUS and their side-effects).
3. **Bus interface:** transports register reads/writes reliably (AXI4-Lite VALID/READY channels).

The register bank turns a hardware datapath into a usable software-controlled peripheral. The quality of the design depends not only on whether the counter works, but also on whether the register semantics, byte enables, status handling, timing boundaries, reset behavior, and bus handshakes are all precise and predictable.
