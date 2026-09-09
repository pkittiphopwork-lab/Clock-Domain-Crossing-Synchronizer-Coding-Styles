# Clock Domain Crossing Synchronizer Coding Styles

## Introduction

Clock Domain Crossing (CDC) happens when a signal moves from one clock domain to another clock domain.

For example, an asynchronous FIFO normally has two clocks:

```text
Write Clock Domain                    Read Clock Domain

i_w_clk                               i_r_clk
   |                                     |
   |                                     |
Write Pointer                        Read Pointer
   |                                     |
   +------------- CDC -------------------+
```

The two clocks may have different frequencies and different phases. Because of this, the receiving flip-flop can sample a signal close to its clock edge. This can create **metastability**.

A common solution for a one-bit control signal is a two-stage synchronizer.

For an asynchronous FIFO pointer, the pointer is normally converted to **Gray code** before it crosses the clock domain. Gray code is useful because only one bit changes between adjacent counter values.

A basic two-stage synchronizer looks like this:

```text
Source Domain                    Destination Domain

r_r_gray
    |
    | asynchronous crossing
    v
+---------+     +---------+
| Stage 1 | --> | Stage 2 | --> synchronized Gray pointer
+---------+     +---------+
     ^               ^
     |               |
 destination clock   destination clock
```

The first stage can become metastable. The second stage gives the first stage more time to become stable before the synchronized value is used by other logic.

---

## Important Rule: Use the Destination Clock

The synchronizer must always use the clock of the **receiving domain**.

For example, if the read Gray pointer crosses into the write domain:

```text
Read Gray Pointer
      |
      | CDC
      v
Write Clock Domain
```

the synchronizer must use:

```verilog
posedge i_w_clk
```

If the write Gray pointer crosses into the read domain:

```text
Write Gray Pointer
      |
      | CDC
      v
Read Clock Domain
```

the synchronizer must use:

```verilog
posedge i_r_clk
```

The signal name is not enough to decide the clock. The important question is:

> Which clock domain will use this synchronized signal?

That clock is the destination clock.

---

# Style 1: Explicit Synchronizer Registers

This style writes each synchronization register clearly.

```verilog
(* ASYNC_REG = "TRUE" *)
reg [c_ADDR_WIDTH:0] r_r_gray_wsync1;

(* ASYNC_REG = "TRUE" *)
reg [c_ADDR_WIDTH:0] r_r_gray_wsync2;


always @(posedge i_w_clk or negedge i_w_rst_n) begin

    if (!i_w_rst_n)
        r_r_gray_wsync1 <= {(c_ADDR_WIDTH + 1){1'b0}};

    else
        r_r_gray_wsync1 <= r_r_gray;

end


always @(posedge i_w_clk or negedge i_w_rst_n) begin

    if (!i_w_rst_n)
        r_r_gray_wsync2 <= {(c_ADDR_WIDTH + 1){1'b0}};

    else
        r_r_gray_wsync2 <= r_r_gray_wsync1;

end
```

The hardware is:

```text
r_r_gray
    |
    v
r_r_gray_wsync1
    |
    v
r_r_gray_wsync2
```

### Advantages

- Very easy to read.
- Very easy to debug in a waveform.
- Each register has a clear purpose.
- Good for learning CDC.
- Good for small and fixed synchronizers.
- Fits a coding style where one registered signal is written in one clocked block.

### Disadvantages

- More source code.
- Not as convenient when the number of synchronization stages must be configurable.

### Recommended use

This style is a very good choice for a small asynchronous FIFO where two synchronization stages are fixed.

For this project, this is the preferred style because the signals are easy to see:

```text
r_r_gray
r_r_gray_wsync1
r_r_gray_wsync2
```

This makes FIFO FULL and EMPTY debugging easier.

---

# Style 2: Array and For Loop

The same structure can be written as an array.

```verilog
localparam integer c_SYNC_STAGES = 2;

reg [c_ADDR_WIDTH:0] r_r_gray_wsync [0:c_SYNC_STAGES-1];

integer i;


always @(posedge i_w_clk or negedge i_w_rst_n) begin

    if (!i_w_rst_n) begin

        for (i = 0; i < c_SYNC_STAGES; i = i + 1)
            r_r_gray_wsync[i] <= {(c_ADDR_WIDTH + 1){1'b0}};

    end

    else begin

        r_r_gray_wsync[0] <= r_r_gray;

        for (i = 1; i < c_SYNC_STAGES; i = i + 1)
            r_r_gray_wsync[i] <= r_r_gray_wsync[i-1];

    end

end


assign w_r_gray_sync =
    r_r_gray_wsync[c_SYNC_STAGES-1];
```

The hardware is still:

```text
r_r_gray
    |
    v
sync[0]
    |
    v
sync[1]
```

If `c_SYNC_STAGES` becomes 3:

```text
r_r_gray
    |
    v
sync[0]
    |
    v
sync[1]
    |
    v
sync[2]
```

### Advantages

- Easy to change the number of stages.
- Good for reusable CDC modules.
- Less repeated RTL.
- Useful when building a generic synchronizer library.

### Disadvantages

- Slightly harder to read for a beginner.
- Waveform names may be less clear than explicit register names.
- Debugging can be less direct.
- A simple two-stage FIFO synchronizer may become more abstract than necessary.

### Recommended use

Use this style when the synchronizer is intended to be reusable or configurable.

---

# Style 3: Concatenation Assignment

A two-stage synchronizer can also be written using one concatenation assignment.

```verilog
always @(posedge i_w_clk or negedge i_w_rst_n) begin

    if (!i_w_rst_n)

        {
            r_r_gray_wsync2,
            r_r_gray_wsync1
        } <= 0;

    else

        {
            r_r_gray_wsync2,
            r_r_gray_wsync1
        }
        <=
        {
            r_r_gray_wsync1,
            r_r_gray
        };

end
```

This means:

```text
next sync1 = current r_r_gray
next sync2 = current sync1
```

It is functionally the same as:

```verilog
r_r_gray_wsync1 <= r_r_gray;
r_r_gray_wsync2 <= r_r_gray_wsync1;
```

The reason is that non-blocking assignments use the old values on the right side of the assignment at the active clock edge.

### Advantages

- Compact.
- Uses fewer lines of RTL.
- Can clearly describe a register shift chain for an experienced RTL designer.

### Disadvantages

- Harder to understand for a beginner.
- The relationship between Stage 1 and Stage 2 is less obvious.
- Waveform debugging is not improved by the shorter syntax.
- It can hide the CDC structure when reading the source quickly.

### Recommended use

This style is valid, but it is better when the design team already understands the register transfer clearly.

For educational RTL and detailed debugging, the explicit style is easier to maintain.

---

# Comparison of the Three Styles

| Item | Explicit Registers | Array + For Loop | Concatenation |
|---|---|---|---|
| Functionally correct | Yes | Yes | Yes |
| Easy to read | Excellent | Good | Medium |
| Easy to debug | Excellent | Good | Good |
| Easy to parameterize | Medium | Excellent | Low |
| Good for beginners | Excellent | Medium | Medium |
| Good for reusable modules | Good | Excellent | Medium |
| Good for a fixed 2-stage FIFO CDC | Excellent | Good | Good |
| Source code size | Larger | Medium | Small |
| Recommended for this project | **Yes** | Optional | Optional |

All three styles can synthesize into the same basic hardware:

```text
Asynchronous Signal
        |
        v
      FF1
        |
        v
      FF2
        |
        v
Synchronized Signal
```

The coding style changes how humans read and maintain the RTL. It does not change the basic CDC theory.

---

# Why Gray Code Is Important in an Asynchronous FIFO

A normal binary counter can change many bits at the same time.

Example:

```text
Binary:

0111
  |
  v
1000
```

Four bits change.

If this bus crosses directly into another asynchronous clock domain, the destination can possibly observe a mixed value.

Gray code is different.

Adjacent Gray-code values change only one bit.

Example:

```text
Binary     Gray

000        000
001        001
010        011
011        010
100        110
101        111
110        101
111        100
```

Because only one Gray bit changes between adjacent pointer values, Gray code is well suited for asynchronous FIFO read and write pointers.

The common FIFO CDC structure is:

```text
Binary Pointer
      |
      v
Gray Encoder
      |
      v
Registered Gray Pointer
      |
      | asynchronous crossing
      v
Synchronizer Stage 1
      |
      v
Synchronizer Stage 2
      |
      v
FULL / EMPTY Logic
```

However, Gray code is not a general solution for every multi-bit bus.

It is suitable for values such as:

```text
FIFO pointer
counter
monotonic address state
```

It is not suitable as a direct replacement for CDC techniques for:

```text
PCM samples
I/Q samples
arbitrary data buses
payload data
configuration packets
```

These types of data normally need a handshake or an asynchronous FIFO.

---

# Why `ASYNC_REG` Is Useful

For AMD/Xilinx FPGA designs, synchronization registers should normally be marked as asynchronous synchronizer registers.

Example:

```verilog
(* ASYNC_REG = "TRUE" *)
reg [c_ADDR_WIDTH:0] r_r_gray_wsync1;

(* ASYNC_REG = "TRUE" *)
reg [c_ADDR_WIDTH:0] r_r_gray_wsync2;
```

This tells Vivado that these registers form a CDC synchronization chain.

The final design should also be checked with:

```text
report_cdc
```

The CDC report helps find unsafe or unexpected clock-domain crossings.

---

# Reset Behavior

A synchronizer should start from a known state.

For an asynchronous FIFO pointer, zero is normally a useful reset value because the read and write pointers also start at zero.

Example:

```verilog
if (!i_w_rst_n) begin

    r_r_gray_wsync1 <= {(c_ADDR_WIDTH + 1){1'b0}};
    r_r_gray_wsync2 <= {(c_ADDR_WIDTH + 1){1'b0}};

end
```

This creates deterministic simulation behavior and keeps the FIFO pointer state clear after reset.

A design should also consider how reset is released in each clock domain. In larger CDC designs, asynchronous assertion with synchronous deassertion is a common technique.

---

# Physical CDC Constraints

Correct RTL is only one part of a good CDC design.

A Gray-coded bus also has physical routing delays. Different bits can take different paths through the FPGA.

For example:

```text
gray[0] ------------------------->
gray[1] -------------->
gray[2] ---------------------->
gray[3] -------->
```

Large routing differences can reduce the safety of the Gray-code assumption at the destination.

AMD recommends timing constraints for asynchronous Gray-coded buses. Two important methods are:

```text
set_max_delay -datapath_only
```

and:

```text
set_bus_skew
```

`set_bus_skew` is especially useful for Gray-coded buses such as asynchronous FIFO pointers because it limits the capture-time difference between bits of the bus.

The exact constraint values depend on the clock architecture and FPGA project.

---

# Common Mistakes

## 1. Using the source clock inside the synchronizer

Wrong concept:

```text
Read pointer
    |
    v
Synchronizer using read clock
```

This does not move the signal into the write domain.

Correct concept:

```text
Read pointer
    |
    v
Synchronizer using write clock
    |
    v
Write-domain logic
```

---

## 2. Using a two-flip-flop synchronizer for arbitrary data buses

This is not safe:

```text
32-bit PCM
   |
   +--> 2FF for every bit
```

Different bits can arrive at different times.

Use a proper handshake or asynchronous FIFO instead.

---

## 3. Using Gray code for arbitrary data

Gray code is useful when adjacent values follow the Gray-code rule.

It is normally used for:

```text
pointers
counters
```

Do not use it as a general data-transfer protocol.

---

## 4. Using Stage 1 directly

The first synchronizer stage can become metastable.

Bad:

```text
async
  |
  v
 FF1 ------> normal logic
  |
  v
 FF2
```

Better:

```text
async
  |
  v
 FF1
  |
  v
 FF2
  |
  +------> normal logic
```

Only the final synchronization stage should normally be used by the destination-domain logic.

---

## 5. Thinking that `set_false_path` fixes CDC

A timing exception does not remove metastability.

CDC safety comes from the hardware architecture:

```text
synchronizer
handshake
Gray code
asynchronous FIFO
```

Timing constraints and CDC reports support the architecture, but they do not replace it.

---

# Recommended Coding Style for This Project

For the current asynchronous FIFO design, the recommended style is the explicit two-stage implementation:

```verilog
(* ASYNC_REG = "TRUE" *)
reg [c_ADDR_WIDTH:0] r_r_gray_wsync1;

(* ASYNC_REG = "TRUE" *)
reg [c_ADDR_WIDTH:0] r_r_gray_wsync2;


always @(posedge i_w_clk or negedge i_w_rst_n) begin

    if (!i_w_rst_n)
        r_r_gray_wsync1 <= {(c_ADDR_WIDTH + 1){1'b0}};

    else
        r_r_gray_wsync1 <= r_r_gray;

end


always @(posedge i_w_clk or negedge i_w_rst_n) begin

    if (!i_w_rst_n)
        r_r_gray_wsync2 <= {(c_ADDR_WIDTH + 1){1'b0}};

    else
        r_r_gray_wsync2 <= r_r_gray_wsync1;

end
```

The main reasons are:

1. The CDC stages are easy to understand.
2. The destination clock is clear.
3. Each register can be inspected easily in GTKWave or Vivado.
4. It matches the current RTL naming style.
5. It is easy to compare `r_r_gray`, `r_r_gray_wsync1`, and `r_r_gray_wsync2`.
6. It makes FULL-condition debugging easier in the write domain.

For a reusable CDC library, the array-and-for-loop version can be a better choice because it is easier to parameterize.

---

# Final Summary

The three coding styles below can implement the same two-stage synchronizer:

```text
1. Explicit registers
2. Array + for loop
3. Concatenation assignment
```

The most important point is not the syntax.

A safe synchronizer needs the correct architecture:

```text
Registered source
      |
      v
Correct CDC encoding
      |
      v
Destination-clock synchronizer
      |
      v
Final synchronization stage
      |
      v
Destination logic
```

For an asynchronous FIFO Gray pointer:

```text
Binary Pointer
      |
      v
Gray Pointer
      |
      v
2-Stage Synchronizer
      |
      v
FULL / EMPTY Logic
```

The synchronizer must use the destination clock, the Gray pointer must follow valid Gray-code transitions, and the implemented design should be checked with CDC analysis and suitable physical constraints.

---

# References

1. AMD, **XPM_CDC_GRAY — Synchronizer via Gray Encoding**, PG382.  
   https://docs.amd.com/r/en-US/pg382-xpm-cdc-generator/XPM_CDC_GRAY

2. AMD, **Vivado Design Methodology — Constraints on Individual CDC Paths**, UG949.  
   https://docs.amd.com/r/en-US/ug949-vivado-design-methodology/Constraints-on-Individual-CDC-Paths

3. AMD, **Vivado Using Constraints — About Bus Skew Constraints**, UG903.  
   https://docs.amd.com/r/en-US/ug903-vivado-using-constraints/About-Bus-Skew-Constraints

4. AMD, **Xilinx Parameterized Macros**, including XPM_CDC_SINGLE, XPM_CDC_GRAY, XPM_CDC_HANDSHAKE, XPM_CDC_PULSE, and XPM_FIFO_ASYNC.  
   https://docs.amd.com/
