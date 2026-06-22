# AXI DMA CONTROLLER WITH SCATTER-GATHER ENHANCEMENT

## RTL DESIGN REPORT

---



---

## Team Members

* Prayag V T (Team Lead)
* Lasim
* Chinchina
* Shafin V
* Isac
* Lakshmi

---

### Design Tool

**Xilinx Vivado 2023.2**

### Hardware Description Language

**SystemVerilog / Verilog HDL**

### Project Type

RTL Design

---
# CHAPTER 1

# INTRODUCTION

## 1.1 Background

Modern System-on-Chip (SoC) architectures require high-speed movement of large amounts of data between memory and peripherals. If the processor performs every data transfer, a significant portion of its execution time is spent copying data instead of executing application software. This limits overall system performance and reduces processing efficiency.

A Direct Memory Access (DMA) Controller addresses this problem by transferring data directly between memory locations without continuous processor involvement. The processor only configures the DMA controller by providing the source address, destination address, and transfer size. Once the transfer begins, the DMA controller autonomously performs all memory transactions and notifies the processor after completion.

The AXI Direct Memory Access (AXI DMA) Controller follows the AMBA AXI4 protocol, enabling high-bandwidth communication with memory while supporting burst-based transfers for improved throughput.

---

## 1.2 Motivation

The original AXI DMA controller supports only a single memory transfer for each software configuration.

For every new transfer, the processor must:

* Configure the source address
* Configure the destination address
* Configure the transfer size
* Enable the DMA
* Start the transfer
* Wait until completion
* Repeat the entire process for the next transfer

This repetitive software intervention introduces unnecessary processor overhead and creates idle periods between consecutive DMA operations. As the number of transfers increases, overall system throughput decreases.

Therefore, an enhancement was required to automate multiple DMA transfers while reducing processor involvement.

---

## 1.3 Project Objectives

The primary objective of this project is to enhance the existing AXI DMA Controller by introducing Scatter-Gather functionality.

The specific objectives are:

* Study the architecture of the existing AXI DMA controller.
* Understand the operation of the AXI Read and Write Engines.
* Design a Scatter-Gather architecture using descriptor-based transfers.
* Implement a Descriptor Fetch Engine for automatic descriptor loading.
* Develop a Scatter-Gather Finite State Machine (FSM).
* Maintain backward compatibility with the existing DMA architecture.
* Improve overall DMA efficiency by reducing processor intervention.

---

## 1.4 Scope of the Project

The scope of this work is limited to the RTL design and architectural enhancement of the AXI DMA controller.

The project includes:

* Study of the existing DMA architecture.
* Analysis of current design limitations.
* Design of the Scatter-Gather architecture.
* Development of descriptor-based transfer logic.
* Integration of the Scatter-Gather FSM with the existing DMA controller.
* RTL implementation using Verilog/SystemVerilog.
* RTL synthesis and structural validation using Vivado 2023.2.

This report focuses on the architectural design and RTL implementation of the enhanced DMA controller. Functional verification and verification environment development are outside the scope of this report.

---

## 1.5 Project Overview

The enhanced AXI DMA Controller consists of two modes of operation:

### Simple DMA Mode

The processor configures every transfer manually through Control and Status Registers (CSR). After receiving the start command, the DMA performs one memory-to-memory transfer and returns to the idle state.

### Scatter-Gather DMA Mode

Instead of configuring every transfer manually, the processor stores a list of transfer descriptors in memory.

Each descriptor contains:

* Source Address
* Destination Address
* Transfer Length
* Control Information
* Status Information
* Address of the Next Descriptor

The Scatter-Gather FSM automatically fetches each descriptor, configures the DMA core, performs the transfer, updates the descriptor status, and proceeds to the next descriptor until the descriptor chain is completed.

This enhancement significantly reduces processor overhead while improving overall data transfer efficiency.

---

# CHAPTER 2

# AXI DMA ARCHITECTURE

## 2.1 Introduction

The AXI DMA Controller is a hardware module that performs high-speed memory-to-memory data transfers without continuous processor intervention. It communicates with the processor through an AXI Slave interface for configuration and uses an AXI Master interface to access system memory.

The architecture is divided into two major sections:

* **Control Path** – Responsible for configuration and status monitoring.
* **Data Path** – Responsible for transferring data between memories.

This separation improves modularity and simplifies the overall design.

---

## 2.2 Existing AXI DMA Architecture

The existing AXI DMA controller consists of the following major functional blocks:

* AXI Slave Interface
* Control and Status Registers (CSR)
* DMA Controller
* AXI Master Read Engine
* Internal FIFO Buffer
* AXI Master Write Engine
* Interrupt Generation Logic

### Architecture Diagram


<img width="584" height="285" alt="image" src="https://github.com/user-attachments/assets/7c565ce4-eadd-4b8b-979c-cd4dabe9dcfc" />


---

## 2.3 Functional Block Description

### 2.3.1 AXI Slave Interface

The AXI Slave Interface provides communication between the processor and the DMA controller.

Its primary functions are:

* Accept configuration commands from the processor.
* Store DMA parameters into the Control and Status Registers.
* Return DMA status information during register read operations.

The processor accesses this interface only during DMA configuration.

---

### 2.3.2 Control and Status Registers (CSR)

The CSR block stores all information required to perform a DMA transfer.

Typical information stored includes:

* Source Address
* Destination Address
* Transfer Length
* DMA Enable
* Start Command
* Transfer Status

The DMA Controller continuously monitors these registers to determine when a new transfer should begin.

---

### 2.3.3 DMA Controller

The DMA Controller acts as the central control unit of the design.

Its responsibilities include:

* Monitoring the Control Registers.
* Starting DMA transfers.
* Coordinating Read and Write Engines.
* Monitoring transfer completion.
* Generating interrupt requests after successful completion.

The DMA Controller does not transfer data directly. Instead, it controls the operation of the Read and Write Engines.

---

### 2.3.4 Read Engine

The Read Engine performs AXI Master Read transactions.

Its responsibilities include:

* Generating AXI Read Address transactions.
* Receiving read data from memory.
* Handling burst transfers.
* Supporting aligned and unaligned memory accesses.
* Writing received data into the Internal FIFO.

The Read Engine continuously transfers data until the requested transfer length has been completed.

---

### 2.3.5 Internal FIFO

The Internal FIFO acts as a temporary storage buffer between the Read Engine and the Write Engine.

Functions of the FIFO include:

* Temporary storage of read data.
* Decoupling read and write operations.
* Preventing data loss during bus latency.
* Supporting pipelined DMA operation.

Without the FIFO, both engines would have to operate at exactly the same speed, significantly reducing system performance.

---

### 2.3.6 Write Engine

The Write Engine retrieves data from the FIFO and performs AXI Write transactions.

Its responsibilities include:

* Generating AXI Write Address transactions.
* Sending write data.
* Generating byte strobes for partial writes.
* Monitoring write responses.
* Completing burst write operations.

The Write Engine continues until all FIFO data has been written to the destination memory.

---

### 2.3.7 Interrupt Logic

Once the DMA transfer has completed successfully,

the DMA controller:

* Sets the DONE status bit.
* Clears the BUSY status bit.
* Generates an interrupt if enabled.

The processor uses this interrupt to determine that the transfer has completed successfully.

---

# 2.4 Top-Level RTL Architecture

The top-level RTL module connects all internal DMA modules together.

```
dma_axi_simple
│
├── dma_axi_simple_csr_axi
│
├── dma_axi_simple_csr
│
└── dma_axi_simple_core
      │
      ├── dma_axi_simple_core_read
      ├── dma_axi_simple_fifo_sync_small
      └── dma_axi_simple_core_write
```     

# 2.5 DMA Operation

The DMA transfer follows a fixed sequence of operations.

### Step 1

The processor writes the Source Address into the SOURCE register.

↓

### Step 2

The processor writes the Destination Address into the DESTINATION register.

↓

### Step 3

The processor writes the transfer length and chunk size into the NUM register.

↓

### Step 4

The DMA is enabled through the CONTROL register.

↓

### Step 5

The processor sets the GO bit.

↓

### Step 6

The DMA Controller activates the Read Engine.

↓

### Step 7

The Read Engine reads data from Source Memory.

↓

### Step 8

The received data is stored in the Internal FIFO.

↓

### Step 9

The Write Engine retrieves FIFO data.

↓

### Step 10

The Write Engine writes data into Destination Memory.

↓

### Step 11

After completion,

* DONE bit is set.
* BUSY bit is cleared.
* Interrupt is generated (if enabled).

---

### Figure 2.3 – DMA Operation Flow
---------------------------------------------
<img width="240" height="372" alt="image" src="https://github.com/user-attachments/assets/d7c76c04-3ca3-4a45-a0d3-1d4ced39c455" />

---

# 2.6 Control and Status Registers

The DMA controller is programmed through memory-mapped Control and Status Registers.

---

## CONTROL Register (0x30)

| Bit | Name | Description           |
| --- | ---- | --------------------- |
| 31  | EN   | Enables DMA operation |
| 1   | IP   | Interrupt Pending     |
| 0   | IE   | Interrupt Enable      |

---

## NUM Register (0x40)

| Bit   | Name  | Description                 |
| ----- | ----- | --------------------------- |
| 31    | GO    | Starts DMA Transfer         |
| 30    | BUSY  | DMA currently active        |
| 29    | DONE  | Transfer Completed          |
| 23:16 | CHUNK | Burst transfer size         |
| 15:0  | BYTES | Number of bytes to transfer |

---

## SOURCE Register (0x44)

Stores the starting source memory address.

---

## DESTINATION Register (0x48)

Stores the starting destination memory address.

# 2.7 DMA Controller FSM

The DMA Controller operates as a Finite State Machine (FSM).

The major states are:

### IDLE

* Waits for DMA Enable.
* Waits for GO signal.

↓

### CONFIGURED

* DMA parameters are loaded.
* Controller verifies register values.

↓

### READ

* Read Engine begins AXI Read transactions.

↓

### WRITE

* Write Engine performs AXI Write transactions.

↓

### DONE

* Transfer completed.
* DONE bit asserted.
* Returns to IDLE.

---

### Figure 2.5 – DMA Controller FSM

<img width="293" height="299" alt="image" src="https://github.com/user-attachments/assets/6f747fa8-d24e-4e85-b02c-4487c5f6d9ea" />

---

# 2.8 Read Engine FSM

The Read Engine controls AXI Read transactions.

States include:

* RD_IDLE
* SEND_AR
* WAIT_RDATA
* STORE_FIFO
* RD_DONE

The Read Engine continues until all requested bytes have been received.

---

### Figure 2.6 – Read Engine FSM

<img width="228" height="341" alt="image" src="https://github.com/user-attachments/assets/b94dd38f-bce3-4e87-b1a9-6b554e2f7fa5" />

---

# 2.9 Write Engine FSM

The Write Engine controls AXI Write transactions.

States include:

* WR_IDLE
* SEND_AW
* SEND_WDATA
* WAIT_BRESP
* WR_DONE

The Write Engine continues until all FIFO data has been written successfully.

---

### Figure 2.7 – Write Engine FSM

<img width="314" height="315" alt="image" src="https://github.com/user-attachments/assets/1a4bafaf-a3d4-4a62-a0f2-a0f9cbdc1d9d" />


---

## Chapter Summary

The existing AXI DMA Controller provides a modular architecture consisting of separate control and data paths. The processor configures the DMA through Control and Status Registers, while the Read Engine, FIFO, and Write Engine cooperate to perform autonomous memory-to-memory transfers. Although the architecture efficiently supports single DMA transfers, it requires processor intervention for every new transfer. This limitation motivates the Scatter-Gather enhancement discussed in the next chapter.

---




# CHAPTER 3

# SCATTER-GATHER ENHANCEMENT

## 3.1 Introduction

Although the existing AXI DMA controller efficiently performs memory-to-memory transfers, it is designed to execute only one transfer for each software configuration. For every new transfer, the processor must configure the DMA registers again and restart the controller. This repetitive process increases processor overhead and limits the overall throughput of the system.

To overcome these limitations, the DMA controller is enhanced with **Scatter-Gather (SG) capability**. Scatter-Gather DMA allows the controller to execute multiple transfer operations automatically by reading a list of descriptors stored in memory. Once the first descriptor is provided, the DMA autonomously processes all remaining descriptors without requiring further processor intervention.

This enhancement significantly improves system performance and enables continuous high-speed data movement.

---

# 3.2 Limitations of the Existing DMA

The original DMA architecture supports only **single-transfer operation**.

The operation sequence is shown below.

```
CPU
 │
 │ Configure DMA Registers
 ▼
DMA Transfer
 │
 ▼
Transfer Complete
 │
 ▼
CPU Configures Again
 │
 ▼
Next Transfer
```

For every transfer, the processor performs the following operations:

* Write Source Address
* Write Destination Address
* Write Transfer Length
* Enable DMA
* Set GO bit
* Wait until transfer completes
* Repeat the same procedure for the next transfer

This repeated configuration consumes processor time and reduces system efficiency.

---

## Existing DMA Drawbacks

The major limitations are:

### Single Transfer Operation

Only one transfer can be executed after each configuration.

---

### High CPU Intervention

The processor must configure every transfer individually.

---

### Idle Time Between Transfers

After one transfer finishes, the DMA waits until the processor starts another transfer.

---

### Reduced Throughput

Continuous data streaming is not possible because of repeated software interaction.

---

### Limited Scalability

Applications requiring hundreds or thousands of transfers become inefficient.

---


# 3.3 Scatter-Gather DMA Concept

Scatter-Gather DMA eliminates continuous processor involvement by introducing **Descriptor-Based Data Transfers**.

Instead of programming every transfer separately, the processor prepares a list of descriptors in memory.

Each descriptor completely defines one DMA transfer.

The processor only provides the address of the first descriptor.

After that, the DMA controller performs the following operations automatically:

* Fetch descriptor
* Decode descriptor
* Execute DMA transfer
* Update descriptor status
* Fetch next descriptor
* Repeat until no descriptor remains

This allows multiple transfers to be completed using a single processor command.

---

## Scatter-Gather Working Principle

```
CPU
 │
 │ Write First Descriptor Address
 ▼
Scatter-Gather DMA
 │
 ▼
Descriptor 1
 │
 ▼
Transfer 1
 │
 ▼
Descriptor 2
 │
 ▼
Transfer 2
 │
 ▼
Descriptor 3
 │
 ▼
Transfer 3
 │
 ▼
Done
```

The processor is free to execute other tasks while the DMA controller processes all descriptors independently.

---

### Figure 3.2 – Scatter-Gather Architecture

<img width="1600" height="775" alt="image" src="https://github.com/user-attachments/assets/f3c631fb-4e7d-4c7a-86ae-62f2abf17e40" />


---

# 3.4 Descriptor Format

A descriptor is a small data structure stored in memory that completely defines one DMA transfer.

Each descriptor contains the following fields.

| Field                   | Description                            |
| ----------------------- | -------------------------------------- |
| Next Descriptor Address | Address of the next descriptor         |
| Source Address          | Starting address of source memory      |
| Destination Address     | Starting address of destination memory |
| Transfer Length         | Number of bytes to transfer            |
| Control                 | Transfer control information           |
| Status                  | Completion and error information       |

---

## Descriptor Structure

```
-------------------------------------------------
| Next Descriptor Address                       |
-------------------------------------------------
| Source Address                                |
-------------------------------------------------
| Destination Address                           |
-------------------------------------------------
| Transfer Length                               |
-------------------------------------------------
| Control                                       |
-------------------------------------------------
| Status                                        |
-------------------------------------------------
```

Each descriptor occupies six 32-bit words in memory.

The **Next Descriptor Address** links the current descriptor to the next descriptor, creating a linked list of DMA operations.

---

---

# 3.5 Scatter-Gather FSM

To support automatic descriptor execution, a dedicated Scatter-Gather Finite State Machine (FSM) is introduced.

The Scatter-Gather FSM controls the complete descriptor processing sequence.

---

## State 1 – IDLE

The controller waits until:

* DMA Enable = 1
* GO Bit = 1

After receiving the start command, the FSM begins descriptor processing.

---

## State 2 – FETCH_DESCRIPTOR

The Descriptor Fetch Engine generates AXI Read transactions.

The descriptor stored in memory is transferred into internal registers.

---

## State 3 – DECODE_DESCRIPTOR

The descriptor fields are decoded.

The following information is extracted:

* Source Address
* Destination Address
* Transfer Length
* Control Information
* Next Descriptor Address

These values are loaded into the DMA Controller registers.

---

## State 4 – EXECUTE_TRANSFER

The DMA core begins normal operation.

The existing Read Engine, FIFO and Write Engine perform the complete data transfer.

This stage is identical to the original DMA architecture.

---

## State 5 – UPDATE_STATUS

After successful transfer,

the Status field inside the descriptor is updated.

Typical information includes:

* Transfer Completed
* Error Status
* Descriptor Executed

---

## State 6 – NEXT_DESCRIPTOR

The controller checks the **Next Descriptor Address**.

### If another descriptor exists

The FSM returns to **FETCH_DESCRIPTOR**.

### If no descriptor exists

The Scatter-Gather operation finishes and returns to the IDLE state.

---

## Scatter-Gather FSM Flow

```
IDLE

↓

FETCH_DESCRIPTOR

↓

DECODE_DESCRIPTOR

↓

EXECUTE_TRANSFER

↓

UPDATE_STATUS

↓

NEXT_DESCRIPTOR

↓

More Descriptor?

YES ───────────────► FETCH_DESCRIPTOR

NO

↓

DONE

↓

IDLE
```

---


---

# 3.6 RTL Architectural Changes

To support Scatter-Gather functionality, several RTL modifications were introduced.

## Newly Added Modules

### Descriptor Fetch Engine

Reads descriptors directly from system memory using the AXI Master interface.

---

### Scatter-Gather FSM

Controls descriptor execution and sequencing.

---

### Descriptor Registers

Temporarily store descriptor information before initiating DMA transfers.

---

## Modified Modules

The following existing RTL modules were updated:

* dma_axi_simple
* dma_axi_simple_core
* dma_axi_simple_csr
* dma_axi_simple_csr_axi

The Read Engine, FIFO and Write Engine required only minor interface modifications because the existing DMA data path remains unchanged.

---

# 3.7 Advantages of Scatter-Gather DMA

The Scatter-Gather enhancement provides several improvements over the original DMA architecture.

| Existing DMA                    | Scatter-Gather DMA                  |
| ------------------------------- | ----------------------------------- |
| One transfer per configuration  | Multiple transfers automatically    |
| High CPU involvement            | Minimal CPU involvement             |
| Processor starts every transfer | Descriptor chain controls transfers |
| Idle time between transfers     | Continuous operation                |
| Lower throughput                | Higher throughput                   |
| Limited scalability             | Easily scalable                     |

---

## Chapter Summary

The Scatter-Gather enhancement transforms the original single-transfer DMA into a descriptor-based autonomous DMA controller. By introducing a Descriptor Fetch Engine and Scatter-Gather FSM, the DMA is capable of executing multiple transfers without repeated processor intervention. The original Read Engine, FIFO and Write Engine remain unchanged, while the Scatter-Gather controller provides intelligent sequencing of transfer operations through linked descriptors. This enhancement significantly improves throughput, reduces processor overhead, and provides a scalable architecture suitable for high-performance embedded systems.

---


# CHAPTER 4

# RTL DESIGN IMPLEMENTATION

## 4.1 Introduction

The Scatter-Gather enhancement was implemented by extending the existing AXI DMA architecture while preserving its original data path. Instead of redesigning the complete DMA controller, additional RTL modules were introduced to support descriptor-based operation.

The enhanced architecture consists of:

* Existing AXI DMA Core
* Descriptor Fetch Engine
* Scatter-Gather Controller FSM
* Enhanced Control Registers
* Descriptor Storage Registers

The Read Engine, FIFO, and Write Engine continue to perform the actual data movement, while the newly added Scatter-Gather Controller manages descriptor execution.

---

# 4.2 RTL Design Methodology

The design methodology followed during implementation is illustrated below.

```
System Requirements
        │
        ▼
Architecture Design
        │
        ▼
RTL Module Development
        │
        ▼
Module Integration
        │
        ▼
RTL Synthesis
        │
        ▼
RTL Validation
```

The implementation follows a modular design approach where each module performs an independent function, making the overall architecture easier to understand, modify, and maintain.

---

# 4.3 RTL Module Hierarchy

The Scatter-Gather DMA architecture is organized as shown below.

```
dma_axi_simple
│
├── dma_axi_simple_csr_axi
│
├── dma_axi_simple_csr
│
├── dma_axi_simple_core
│     │
│     ├── dma_axi_simple_core_read
│     ├── dma_axi_simple_fifo_sync_small
│     └── dma_axi_simple_core_write
│
├── dma_axi_simple_sg_fsm
│
├── dma_axi_simple_desc_fetch
│
└── dma_axi_simple_descriptor
```

---

# 4.4 RTL Module Description

## 4.4.1 dma_axi_simple

This is the top-level module of the complete DMA controller.

Its responsibilities include:

* Connecting all RTL modules
* Interfacing with AXI Slave and AXI Master buses
* Selecting between Simple DMA mode and Scatter-Gather mode
* Managing interrupt generation

---

## 4.4.2 dma_axi_simple_csr_axi

This module interfaces with the processor through the AXI Slave interface.

Functions:

* Decodes AXI register accesses
* Reads configuration registers
* Writes configuration registers
* Controls DMA configuration

---

## 4.4.3 dma_axi_simple_csr

This module stores all DMA configuration information.

Stored parameters include:

* Source Address
* Destination Address
* Transfer Length
* DMA Enable
* GO bit
* Scatter-Gather Mode
* Descriptor Base Address

---

## 4.4.4 dma_axi_simple_core

This is the data transfer engine.

It contains:

* Read Engine
* FIFO
* Write Engine

The DMA core remains almost identical to the original implementation.

Scatter-Gather only changes how these registers are loaded.

---

## 4.4.5 dma_axi_simple_core_read

The Read Engine generates AXI Read transactions.

Responsibilities:

* Read source memory
* Handle burst transfers
* Receive AXI read data
* Store received data into FIFO

---

## 4.4.6 dma_axi_simple_fifo_sync_small

The FIFO temporarily stores transferred data.

Functions:

* Buffer data
* Prevent data loss
* Decouple Read and Write Engines
* Improve DMA throughput

---

## 4.4.7 dma_axi_simple_core_write

The Write Engine generates AXI Write transactions.

Responsibilities:

* Read data from FIFO
* Generate AXI Write Address
* Send write data
* Handle write responses

---

## 4.4.8 dma_axi_simple_desc_fetch

This newly added module is responsible for reading descriptors from memory.

Its functions include:

* Generate AXI Read requests
* Read descriptor contents
* Store descriptor fields into internal registers
* Notify Scatter-Gather FSM after descriptor reception

---

## 4.4.9 dma_axi_simple_sg_fsm

The Scatter-Gather FSM controls descriptor execution.

Responsibilities include:

* Descriptor sequencing
* Descriptor decoding
* DMA configuration
* Descriptor completion
* Next descriptor selection

---

## 4.4.10 dma_axi_simple_descriptor

Stores descriptor information after it is fetched.

Typical fields include:

* Source Address
* Destination Address
* Transfer Length
* Control Information
* Status Information
* Next Descriptor Address

---

# 4.5 Module Connectivity

The connectivity between the modules is shown below.

```
Processor
      │
      ▼
AXI Slave Interface
      │
      ▼
CSR Registers
      │
      ▼
Scatter-Gather FSM
      │
      ▼
Descriptor Fetch Engine
      │
      ▼
DMA Core
      │
 ┌────┴─────┐
 │          │
 ▼          ▼
Read      Write
Engine    Engine
      │
      ▼
     FIFO
      │
      ▼
AXI Master Interface
      │
      ▼
System Memory
```

---



---

# 4.6 Design Flow

The Scatter-Gather DMA performs the following sequence.

```
GO Bit

↓

Fetch Descriptor

↓

Decode Descriptor

↓

Load DMA Registers

↓

Start DMA Core

↓

Read Engine

↓

FIFO

↓

Write Engine

↓

Update Descriptor Status

↓

Next Descriptor?

↓

YES

↓

Repeat

↓

NO

↓

DMA Complete
```

---

# 4.7 Scatter-Gather Integration

One of the important objectives of this work was to integrate Scatter-Gather support without redesigning the existing DMA architecture.

Instead of replacing the DMA core,

the Scatter-Gather Controller simply loads new values into the existing DMA registers.

Therefore,

the following modules remain unchanged:

* Read Engine
* FIFO
* Write Engine
* AXI Master Interface

Only the method of supplying DMA parameters changes.

This significantly reduces implementation complexity.

---

# 4.8 Design Advantages

The RTL implementation provides several architectural advantages.

### Modular Design

Each functional block performs an independent operation.

---

### Reusable Architecture

The original DMA core remains reusable without modification.

---

### Scalability

Additional descriptor fields can easily be added in future versions.

---

### Backward Compatibility

Simple DMA mode continues to operate exactly as before.

---

### Reduced Processor Overhead

The processor only supplies the first descriptor address.

All remaining transfers are executed automatically.

---

# 4.9 Design Summary

The Scatter-Gather enhancement was implemented by adding a Descriptor Fetch Engine and Scatter-Gather Controller while preserving the existing DMA data path. The modular RTL architecture simplifies future expansion and maintains complete compatibility with the original AXI DMA controller. The enhanced controller is capable of autonomous descriptor processing and continuous memory-to-memory data transfers without repeated processor intervention.

---


# CHAPTER 5

# RTL DESIGN ANALYSIS

## 5.1 Introduction

After integrating the Scatter-Gather enhancement into the AXI DMA controller, the overall operation of the design was analyzed using RTL simulation in Vivado 2023.2. The analysis focuses on the sequence of descriptor processing, DMA data movement, and the interaction between the newly added Scatter-Gather modules and the existing DMA core.

The purpose of this chapter is to explain the operation of the RTL implementation and illustrate how data flows through the enhanced DMA architecture.

---

# 5.2 Overall Design Flow

The Scatter-Gather DMA performs data transfer through the following sequence of operations.

```text
Processor
      │
      ▼
Write First Descriptor Address
      │
      ▼
Scatter-Gather FSM
      │
      ▼
Descriptor Fetch Engine
      │
      ▼
Descriptor Decode
      │
      ▼
Load DMA Registers
      │
      ▼
Read Engine
      │
      ▼
Internal FIFO
      │
      ▼
Write Engine
      │
      ▼
Destination Memory
      │
      ▼
Update Descriptor Status
      │
      ▼
Check Next Descriptor
```

Each stage performs a dedicated task while maintaining compatibility with the original DMA architecture.

---

# 5.3 Descriptor Fetch Operation

The first operation performed by the Scatter-Gather controller is fetching the descriptor from memory.

The processor provides only the address of the first descriptor through the Scatter-Gather control register.

The Descriptor Fetch Engine performs an AXI Read transaction to retrieve the descriptor.

The descriptor contains all information required for one DMA transfer.

These fields include:

* Source Address
* Destination Address
* Transfer Length
* Control Information
* Status Information
* Next Descriptor Address

After receiving the descriptor, the Scatter-Gather FSM decodes each field and loads the required values into the DMA controller.

---

## Descriptor Fetch Flow

```text
Processor

↓

Descriptor Address

↓

Descriptor Fetch Engine

↓

AXI Read Transaction

↓

Descriptor Memory

↓

Descriptor Data

↓

Scatter-Gather FSM

↓

DMA Registers
```

---

# 5.4 DMA Data Transfer Operation

Once the descriptor has been decoded, the Scatter-Gather FSM configures the existing DMA core.

The actual data transfer is performed by the original DMA architecture.

The Read Engine generates AXI Read requests to the source memory.

The received data is temporarily stored inside the Internal FIFO.

The Write Engine continuously reads data from the FIFO and transfers it to the destination memory.

Because the Read and Write Engines operate independently, the FIFO acts as an intermediate buffer that prevents data loss and improves transfer efficiency.

---

## DMA Data Flow

```text
Source Memory

↓

Read Engine

↓

Internal FIFO

↓

Write Engine

↓

Destination Memory
```


---

# 5.5 Descriptor Chaining Operation

One of the most significant advantages of Scatter-Gather DMA is automatic descriptor chaining.

After completing one transfer, the Scatter-Gather FSM checks the Next Descriptor Address field.

Two situations are possible.

### Case 1 – Next Descriptor Exists

The controller automatically performs another descriptor fetch and starts the next DMA transfer.

No processor interaction is required.

---

### Case 2 – Last Descriptor

If the Next Descriptor Address is NULL,

the Scatter-Gather controller:

* Updates the status register
* Generates the DONE signal
* Returns to the IDLE state

This completes the Scatter-Gather operation.

---

## Descriptor Chain Example

```text
Descriptor 1

↓

Transfer 1

↓

Descriptor 2

↓

Transfer 2

↓

Descriptor 3

↓

Transfer 3

↓

Done
```

The processor remains free throughout the complete operation.

---

# 5.6 Scatter-Gather State Analysis

The Scatter-Gather FSM consists of six major operational stages.

### IDLE

Waits for DMA Enable and GO signal.

---

### FETCH_DESCRIPTOR

Reads one descriptor from memory.

---

### DECODE_DESCRIPTOR

Extracts transfer information.

---

### EXECUTE_TRANSFER

Starts the existing DMA core.

---

### UPDATE_STATUS

Stores completion information.

---

### NEXT_DESCRIPTOR

Determines whether another descriptor is available.

If another descriptor exists,

the FSM repeats the entire sequence automatically.

---

---

# 5.7 RTL Waveform Analysis

The operation of the Scatter-Gather DMA can be observed using RTL simulation waveforms generated in Vivado.

<img width="1600" height="654" alt="image" src="https://github.com/user-attachments/assets/c898d249-0a9f-42a5-8062-7e2d444a2229" />
<img width="1600" height="662" alt="image" src="https://github.com/user-attachments/assets/57f29832-f846-4620-8eb3-3d4a07e87ff4" />



# 5.8 Vivado RTL Analysis

After completing the RTL implementation, the design hierarchy and module connectivity can be viewed using the Vivado RTL elaborated design.

The RTL schematic demonstrates:

* Modular architecture
* Descriptor Fetch integration
* Scatter-Gather FSM connectivity
* Existing DMA Core reuse
* AXI Master and AXI Slave interfaces

---

### Figure 5.8 – Vivado RTL Schematic

<img width="519" height="413" alt="image" src="https://github.com/user-attachments/assets/d927a65a-95ff-4f94-b2cc-3407d84c7a5c" />


# 5.9 Chapter Summary

The RTL analysis demonstrates that the Scatter-Gather enhancement successfully integrates with the existing AXI DMA architecture while preserving the original data path. The Descriptor Fetch Engine and Scatter-Gather FSM automate descriptor processing, enabling continuous execution of multiple DMA transfers. The Read Engine, FIFO, and Write Engine remain unchanged, ensuring modularity and backward compatibility. The waveform analysis and RTL hierarchy further illustrate the interaction between the newly introduced Scatter-Gather modules and the original DMA controller.

---

# CHAPTER 6: CONCLUSION

## 6.1 Design Summary

The objective of this project was to enhance the existing AXI DMA Controller by introducing **Scatter-Gather (SG)** functionality while preserving the original DMA architecture. The existing controller was capable of performing only a single memory-to-memory transfer for each software configuration, requiring continuous processor intervention for every subsequent transfer. This limitation increased processor overhead and reduced the overall throughput of the system.

To overcome this limitation, a descriptor-based Scatter-Gather architecture was proposed and integrated into the existing DMA controller. Instead of manually configuring every transfer, the processor initializes the DMA by providing the address of the first descriptor. The Scatter-Gather Controller then autonomously fetches each descriptor, configures the DMA core, executes the transfer, updates the descriptor status, and proceeds to the next descriptor until the descriptor chain is completed.

One of the major advantages of the proposed enhancement is that it preserves the original DMA data path. The existing Read Engine, Internal FIFO, and Write Engine continue to perform memory transfers, while the Scatter-Gather Controller manages descriptor sequencing and transfer control.

---

## 6.2 Design Achievements

The major achievements of this project are summarized below.

### Study of Existing DMA Architecture

The existing AXI DMA controller was analyzed to understand the operation of the AXI Slave Interface, Control and Status Registers (CSR), DMA Controller, Read Engine, Internal FIFO, and Write Engine. This analysis served as the foundation for the proposed Scatter-Gather enhancement.

### Scatter-Gather Architecture

A descriptor-based Scatter-Gather architecture was designed to support automatic execution of multiple DMA transfers. The proposed architecture eliminates repeated processor configuration by allowing linked descriptors stored in memory to control the complete transfer sequence.

### Descriptor Fetch Engine

A dedicated Descriptor Fetch Engine was introduced to retrieve descriptors directly from system memory using the AXI Master interface. After fetching a descriptor, its contents are decoded and loaded into the DMA Controller.

### Scatter-Gather Controller FSM

A dedicated Finite State Machine (FSM) was designed to manage descriptor processing. The controller sequentially performs descriptor fetching, descriptor decoding, DMA configuration, transfer execution, descriptor status updates, and automatic traversal of the descriptor chain.

### RTL Integration

The Scatter-Gather enhancement was integrated with the existing DMA architecture while preserving the original data path. Only the control logic was extended, allowing the existing Read Engine, FIFO, and Write Engine to remain unchanged.

---

## 6.3 Advantages of the Proposed Design

The proposed Scatter-Gather enhancement provides several improvements over the existing DMA architecture.

| Existing DMA                      | Scatter-Gather DMA                  |
| --------------------------------- | ----------------------------------- |
| Single transfer per configuration | Multiple automatic transfers        |
| High CPU intervention             | Minimal CPU intervention            |
| Processor starts every transfer   | Descriptor chain controls transfers |
| Idle periods between transfers    | Continuous DMA operation            |
| Lower throughput                  | Improved throughput                 |
| Limited scalability               | Highly scalable architecture        |

The modular architecture also simplifies future expansion without affecting the existing DMA core.

---

## 6.4 Applications

The proposed Scatter-Gather DMA architecture can be applied in a wide range of embedded and high-performance computing systems.

Typical application areas include:

* Memory-to-memory data transfer
* Embedded Linux systems
* FPGA-based accelerators
* Video and image processing
* Network packet processing
* High-speed storage controllers
* Data acquisition systems
* System-on-Chip (SoC) platforms

These applications require continuous movement of large amounts of data while minimizing processor involvement.

---

## 6.5 Future Scope

Although the proposed architecture significantly improves the existing DMA controller, several enhancements can be considered for future development.

### Multi-Channel DMA

Support multiple DMA channels operating simultaneously for parallel data transfers.

### 64-bit Addressing

Extend the architecture to support larger memory spaces through 64-bit addressing.

### Dynamic Burst Optimization

Implement adaptive burst-size selection based on transfer size and memory alignment.

### Advanced Error Handling

Introduce descriptor validation, timeout detection, and automatic recovery mechanisms for fault-tolerant operation.

### Performance Monitoring

Add hardware performance counters to monitor:

* Number of transfers
* Bytes transferred
* Average transfer latency
* AXI bus utilization

---

## 6.6 Overall Conclusion

The AXI DMA Controller was successfully enhanced with Scatter-Gather functionality through RTL design using Verilog/SystemVerilog. By introducing a Descriptor Fetch Engine and a dedicated Scatter-Gather Controller FSM, the enhanced architecture enables autonomous execution of multiple DMA transfers using linked descriptors stored in memory.

Unlike the original DMA controller, which required repeated processor intervention, the proposed architecture automatically processes descriptor chains, thereby reducing software overhead and improving system efficiency.

The modular implementation preserves the original DMA data path while extending the control logic, making the design scalable, reusable, and suitable for modern high-performance embedded systems.

The proposed architecture provides a strong foundation for future enhancements such as multi-channel DMA, intelligent burst optimization, larger address spaces, and advanced scheduling mechanisms.

---

# APPENDIX A: RTL MODULE LIST

| Module                                   | Description                  |
| ---------------------------------------- | ---------------------------- |
| `dma_axi_simple`                         | Top-level AXI DMA Controller |
| `dma_axi_simple_core`                    | Main DMA Data Path           |
| `dma_axi_simple_core_read`               | AXI Master Read Engine       |
| `dma_axi_simple_core_write`              | AXI Master Write Engine      |
| `dma_axi_simple_fifo_sync_small`         | Internal FIFO Buffer         |
| `dma_axi_simple_csr`                     | Control and Status Registers |
| `dma_axi_simple_csr_axi`                 | AXI Slave CSR Interface      |
| `dma_axi_simple_csr_read`                | CSR Read Logic               |
| `dma_axi_simple_csr_write`               | CSR Write Logic              |
| `dma_axi_simple_defines`                 | Common AXI Definitions       |
| `dma_axi_simple_desc_fetch` *(Proposed)* | Descriptor Fetch Engine      |
| `dma_axi_simple_sg_fsm` *(Proposed)*     | Scatter-Gather Controller    |
| `dma_axi_simple_descriptor` *(Proposed)* | Descriptor Storage Registers |

---

# APPENDIX B: TEAM CONTRIBUTION

| Team Member | Contribution                                           |
| ----------- | ------------------------------------------------------ |
| Prayag V T  | Project planning and architecture design               |
| Lasim       | Descriptor Fetch Engine design                         |
| Chinchina   | Scatter-Gather FSM design                              |
| Shafin V    | RTL integration, architectural analysis, documentation |
| Isac        | AXI interface integration                              |
| Lakshmi     | Documentation and report preparation                   |

---

# APPENDIX C: SOFTWARE TOOLS

| Tool                 | Purpose                           |
| -------------------- | --------------------------------- |
| Xilinx Vivado 2023.2 | RTL design and synthesis          |
| Verilog HDL          | Hardware description language     |
| SystemVerilog        | RTL implementation                |
| GitHub               | Version control and documentation |

---


# END OF REPORT

---

**Project Status:** RTL Design Completed

**Development Tool:** Vivado 2023.2

**Hardware Description Language:** Verilog / SystemVerilog
