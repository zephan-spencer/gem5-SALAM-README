comm interface

comminterface.py

The comm interface or communications interface is the base gem5 component of an acc in gem5-SALAm. It provides programmability as well as system memory access to the acc. Also provides mechanisisms for sync including memory interupt lines. Memory map for comm interface is broken into three sections

- Flags: Provide runtime status information as well as switches for invoking
- Config: Currenlty do not do anything but are here for extended versions 
- Variable: Addresses for runtime variables or values that will be pulled upon invocation. 

Ports: 

- PIO: PIO port connects to mmrs and provides external devices the ability to program the comm interface
- Local Ports: Provide access to other devices within an ACC's local cluster. 
- ACP Ports: Provide access to devices outside of the ACC's cluster.

Access Control Ports:

- Stream Ports: implement the axi-stream-like paradigm that limits read and write to data avalibility. Used in producer-consumer schemes with other devices using stream-buffers or stream-dmas. 
- SPM Ports: Provide access to scratchpads using the additional sync controls provided by the scratchpad memory.

llvminterface

Represents the datapath of the acc. That is what parses the llvm ir file to generate the hardware datapath and then generates and executes the LLVM cdfg using runtime data provided by the comm interface. 

Acc cluster

Optional sim object useful for organizing acclerators as well as resources shared between accs. Provides utilities for connecting accs and other shared acc resources into the larger system. 

scratchpadmemory

Custom fast access memory for accelerators that includes access syncronization mechanizems (ready mode).  When access sync is activated accs will not be able to access data that

Additional controls will be placed on reads and writes to the scratchpad in order to implement various sync mechanisms.

noncoherentdma 

Provides memory to memory dma transfer. Useful copying data to and from system memory and scratchpads. The MMR layout of the noncoherentdma is descriped in noncoherentdma.hh.

stream dma

The stream dma provides dma access between traditional memory objects and scratchpads and an AXI-stream like interface.

Supports auto-play features commonly found in video DMAs. Memory map is defined in streamdma.hh 

stream buffer

Small FIFO buffer that enables AXI-Stream like communication between devices. 

