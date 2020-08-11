

Take some pictures from the paper. Object overview for sure. If anything relevant for the gemm example take it.

For GEMM:  File generated from sys_validation. The .dot file 

# Writing the accelerator code

In this example we want to design an accelerator for a generic matrix multiply operation (GEMM). This example has already created in **benchmarks/sys_validation/gemm** and will be referenced throughout this guide. It contains a folder for accelerator code (hw) and a folder for the host code (sw). 

When creating your accelerator there are a few considerations to be made. These include what kind of parallelism are you desiring, power consumption, and target area. 

**how do you want to integrate it into the system, how are you going to feed it data, is it going to be coupled in memory, does it need DMAs? How do you want to control the accelerator?** 

In this example, we are going to design a highly parallel accelerator that is loosely coupled in memory and control. 

For our accelerator code we have two files. top.c manages the GEMM accelerator and DMAs, and gemm.c includes our algorithm with any compiler optimizations that we want. 

## gemm.c 

We will first start with creating the code for our accelerator. 

In gemm.c there is a GEMM loop application written. To expose parallelism for computation and memory access we fully unroll the innermost loop of the application. gem5-SALAM will natively pipeline the other loop instances for us. To accomplish the loop unrolling we can utilize clang compiler pragmas such as those on line 18: 

```c
#pragma clang loop unroll(full)
```

Since the GEMM accelerator is going to pull from a static set of memory accesses that are hard coded, the memory addresses associated with each of the matrices for the GEMM operation will be used again in our system design.

## top.c

Now that the accelerator is written, we are going to move over to our control mechanism for the accelerator and DMAs. In this instance we want to have the DMAs and accelerator controlled by an additional device to reduce overhead on the CPU. 

In the top.c we begin with a declaration having 3 addresses passed to our Top accelerator. These addresses correspond to locations in the accelerator's Memory Mapped Register (MMR) that will be filled by the host CPU later. 

Additionally, we setup a series of static addresses (Lines 8-12) associated with the MMRs of the GEMM accelerator so that the Top accelerator can invoke and control those. 

We then set the MMRs of the DMA to perform the memory copy between DRAM and the scratchpad memory (Lines 16-28). 

After copying our two input matrices we invoke the GEMM accelerator and wait for it to finish computation (Lines 31-33). 

After the computation has completed, we write back the resulting matrix from the scratchpad memory to system memory (Lines 36-41).

## INI files

For each of our accelerators we need to geneerate an ini file feel free to use examples from the existing benchmarks. Here we can define the number of cycles for each IR instruction. As well as provide any limitations on the number of FUs associated with IR instructions. Additionally we have options for setting the FU clock periods, controls for pipelining of the accelerator, importantly, under ACC config, we set MMR specific details such as the size of the flags register, if it has an addit, what its interupt line number will be as well as the accelerator clock. 

Under memory if we want to simplify, you can define memory address of scratchpad, size, response latency and number of ports. 

Additionally, if we want the accelerator to verify that memory exists in the scratchpad prior to accessing the scratchpad we can set ready mode to true. 

Lastly, we need to set the memory address under CommInterface of our MMR for the accelerator as well as the overall size of the MMR that account for all variables that need to be passed, 8 bytes per variable, and flags.

# Writing the host code

In our example we're using the baremetal kernel. So we need to have the load file, assembly file (cpu register setup for things like interupt handlers),  and generate elf file. 

In boot code we provide link with isr (isr.c) to interact with  top module acc

Main host code is in main.cpp where we start by created and filling out our matrices for the gemm operation. We then pass the addresses of those matrices to the acc via its MMRs and invoce the acc. Lines 36-38 we pass addresses of matrices to the acc. Line 40 we invoke the acc. Since this is an interuptting system you can do anything in the meantime, but we have nothing to do until our task is completed. Once done We will fire out of waiting time and read back results for evalutation. 

m5_dumpstats Will tell gem5 to dump statistics it has been tracking

m5_exit closes the sim

The statistics associated with the acc will automatically dump to stdout when the acc finishes executing.

we provide a run script for all of this. With our systemValidation.sh. 

The actual run command is defined under the RUN_SCRIPT variable. 

vadd is good for streaming dma

# Compiling the Benchmark

### Hardware Makefile

Once the two accelerators have been written,  you will want to invoke the clang compiler to generate the LLVM Intermediate Representation (IR). To do this an example Makefile has also been provided. In the Makefile we compile our accelerators to IR, then we pass that through the LLVM optimizer with Level 1 optimizations and disable the subsequent object file to get the human readable IR.

```makefile
TARGET=gemm.ll top.ll
%.o : %.c
	clang -emit-llvm -c -o $@ $<
%.bc : %.o
	opt -O1 -mem2reg -simplifycfg -loops -loop-unroll -o $@ $<
%.ll : %.bc
	llvm-dis -o $@ $<
build : $(TARGET)

clean:
	rm -f *.ll
```

This Makefile is stored in the accelerator code folder (hw). 

### Software Makefile



```makefile
include ../../../common/Makefile

OBJS         = boot.o ../../../common/syscalls.o main.o isr.o

main.elf: $(OBJS) $(LNK_SCRIPT) Makefile
	$(CC) $(LNK_FILE_OPT) -o $@ $(OBJS) $(LNK_OPT)

boot.o: Makefile
	$(CPP) boot.s $(CFLAGS) | $(AS) $(ASFLAGS) -o boot.o

clean:
	rm -f *.o *.elf
```



# Constructing the System

We are going to leverage and modify the example scripts for gem5's full system simulation. In configs/SALAM/sysValidation.py we have a modified version of the configs examples fs.py that is provided with the default gem5. The only real difference is that we have it invoke our own function we have added line 240. This function is to find and validate acc.py.

## validate_acc.py

In order to simplify organization accelerator related resources, we define a accelerator cluster. This accelerator cluster will contain any shared resources between the accelerators as well as the accelertors themsevles. 

tHas several functions associated with it that help with hooking accelertors for it and hooking cluster into the system. Attach+bridges function connects the acc cluster into larger system and connects the memory bus to the cluster which gives devices outside the cluster accesses "Master Access."

We then invoke the connect_caches in order to connect any cache heirarchy that exists in-between the cluster, the memory bus, or l2xbar of the CPU depending on design. 

attach_bridges gives resources outside of accelerator master accesses to cluster resources

connect_caches: This gives the accelerator cluster master access to resources outside of the clutser. As well as establishes coherency between cluster and other resources via caches. If no caches are needed this will meerly attach the cluster to the membus without a cache.

These functions are all visible in acc_cluster.py

Now adding ACCs:

We are going to create a COmmInterface which is the communications portion of acc

We then will configure acc and then generate it's llvm interface by passing comminterface, config file, and IR file, to acc_config. This will generate the LLVM interface configure any hardware limitations or other hardware config and will establish the static cdfg.

We then connect the hardware accelertor to the cluster

This will attach the pio port of the acc to the cluster's local bus that is associated with MMRs. 

For our acc we do the same thing, we create a comminterface for it and configure it using acc config

Since we want the acc to be managed by the top acc we connect the pio directly to the local ports of the top accelerator. Has a direct conneciton no additional buses or ports. 

We then define a scratchpad memory and configure that scratchpad memory based on using the acc_spm config with points to our accelertor config file. Lastly we connect scratchpad memory to the cluster. 

lines 36-63 can be ignore they are just setting different buffer size on dma if you want to impose additional limitations on dma to control how data is transfer

We go ahead and create a dma 

noncoherent dma because it doesn't have any coherent parts pass through cache if you want it

The cluster dmaport, master access within cluster, dma port master accesses to system, pio port associated with mmr and gives other devices control of the dma

The cluster side dma port is connected to the cluster localbus which gives it accesses to the scratchpad memory 

the system dma port is connected to the coherency bus which gives master access to the rest of the system

pio port is connected to top since it is the only one it is interacting with, the dma

***Make own memory space for accelerator cluster.*** 



<img src="https://raw.githubusercontent.com/clonetrooper67/gem5-SALAM-README/master/GEMM_ACC.png" style="zoom: 50%;" />