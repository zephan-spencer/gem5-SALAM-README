All source code for these objects are stored in src/hwacc

# CommInterface

The communications interface is the base gem5 component of an accelerator in gem5-SALAM. It provides programmability as well as system memory access to the accelerator. It also provides mechanisms for synchronization  including memory interrupt lines. 

The memory map for the CommInterface object is broken into three sections:

- Flags: Provides runtime status information and switches for invoking the interface
- Config: Currently has no function, but is reserved for a future version 
- Variables: Addresses for runtime variables or values that will be pulled upon invocation. 

## Ports: 

- PIO: Connects to MMRs and provides external devices the ability to program the comm interface
- Local Ports: Provides access to other devices within an accelerator's local cluster 
- ACP Ports: Provides access to devices outside of the accelerator's local cluster.

## Access Control Ports:

- Stream Ports: Implements an AXI-stream like paradigm that limits read and write to data availability. It can be used in producer-consumer schemes with other devices using StreamBuffers or StreamDMAs. 
- SPM Ports: Provides access to scratchpads using the additional synchronization controls provided by the scratchpad memory.

# LLVMInterface

The LLVM Interface represents the data path of the accelerator. It is what parses the LLVM IR file to generate the hardware data path and then generates and executes the LLVM control and data flow graph (CDFG) using runtime data provided by the CommInterface. 

# AccCluster

Optional sim object useful for organizing acclerators as well as resources shared between accs. Provides utilities for connecting accs and other shared acc resources into the larger system. 

# ScratchpadMemory

Custom fast access memory for accelerators that includes access syncronization mechanizems (ready mode).  When access sync is activated accs will not be able to access data that

Additional controls will be placed on reads and writes to the scratchpad in order to implement various sync mechanisms.

# NoncoherentDma

Provides memory to memory dma transfer. Useful copying data to and from system memory and scratchpads. The MMR layout of the noncoherentdma is descriped in noncoherentdma.hh.

# StreamDma

The stream dma provides dma access between traditional memory objects and scratchpads and an AXI-stream like interface.

Supports auto-play features commonly found in video DMAs. Memory map is defined in streamdma.hh 

# StreamBuffer

Small FIFO buffer that enables AXI-Stream like communication between devices. 

