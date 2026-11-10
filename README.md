# Asynchronous FIFO Memory Buffer

**Overview**
This repository contains a Verilog implementation of an Asynchronous FIFO (First-In, First-Out) buffer. The design is specifically engineered to manage reliable data transfer between hardware modules operating in independent, non-synchronized clock domains, ensuring data integrity and mitigating metastability.

In digital system design, transferring data across different clock domains is a critical challenge. A standard synchronous FIFO is unsuitable in these scenarios due to setup and hold time violations. This architecture implements an Async FIFO utilizing a dual-port RAM backbone, Gray Code counters for pointer management (ensuring only single-bit transitions), and multi-stage synchronizers (2-FF) to safely route pointers across clock boundaries.

## Key Use Cases
* **Peripheral Interfacing:** Connecting high-speed processing units with slower peripheral devices.
* **SoC Communication:** Enabling data exchange between distinct modules in a System-on-Chip (SoC) operating at varying clock frequencies.
* **Data Buffering:** Absorbing variations in flow rates between data producer and consumer components.
* **FPGA Clock Domains:** Bridging isolated clock domains within FPGA designs running at different speeds.

## Design Logic
To accurately distinguish between a "Full" and "Empty" state—since both conditions result in the read and write pointers overlapping—an additional Most Significant Bit (MSB) is integrated into the pointer addresses.
* **Empty Condition:** The FIFO evaluates as empty when both pointers (including the MSB) are perfectly identical. This occurs upon reset or when the read pointer entirely catches up to the write pointer.
* **Full Condition:** The FIFO asserts the full flag when the pointers match but their MSBs differ. This indicates that the write pointer has wrapped around the memory array exactly one cycle ahead of the read pointer.

### Gray Coding
To prevent transient glitches during cross-domain synchronization, binary pointers are converted to Gray Code. This guarantees that even if a clock edge captures a pointer mid-transition, the sampled value will be off by a maximum of one bit, preventing catastrophic logic failures and maintaining reliable synchronization.

## Hardware Modules
The architecture is modularized into five core Verilog files for streamlined synthesis and static timing analysis:
1. **`fifo.v`**: The top-level wrapper instantiating the complete system. In ASIC designs, this helps isolate and group domain-crossing logic.
2. **`fifo_memory.v`**: A dual-port RAM array featuring independent read and write ports. Data is latched on the write clock's rising edge provided the buffer is not full.
3. **`read_ptr.v`**: Manages read-domain operations. Synchronized strictly by the read clock, it handles Gray code conversion and empty flag generation.
4. **`write_ptr.v`**: Manages write-domain operations. Synchronized by the write clock, it updates the write pointer and triggers the full flag.
5. **`two_ff_syn.v`**: A 2-Flip-Flop synchronizer that safely routes pointers between domains, mitigating metastability by allowing the signal to settle before destination sampling.

## Signal Definitions
* **`wclk` / `rclk`:** Independent write and read domain clock signals.
* **`wdata` / `rdata`:** Input and output data buses.
* **`wclk_en` / `rinc`:** Enable signals authorizing write or read pointer increments.
* **`wfull` / `rempty`:** Status flags preventing memory overwrite (overflow) or invalid reads (underflow).
* **`wptr` / `rptr`:** Gray code pointers native to each respective domain.
* **`w_rptr` / `r_wptr`:** Cross-synchronized pointers utilized for Empty/Full condition evaluation.

## Verification and Testing
The included testbench (`fifo_tb.v`) validates the RTL design against three critical stress conditions:
* **Linear Flow:** Standard sequential write operations followed by reads to confirm complete data parity.
* **Overflow Protection:** Attempting continuous writes to a full FIFO to ensure the `wfull` flag correctly halts the operation.
* **Underflow Protection:** Attempting continuous reads from an empty FIFO to ensure the `rempty` flag successfully prevents invalid data output.

## Results
Simulation waveforms confirm that the Full and Empty flags assert precisely at the memory wrap-around boundaries and that data remains consistent despite the asynchronous clock barrier. While behavioral simulations validate functional correctness, physical hardware deployment requires rigorous Static Timing Analysis (STA) to guarantee metastability constraints are satisfied.