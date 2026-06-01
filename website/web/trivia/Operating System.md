# Operating systems

1. What is the difference between process and thread?

| THREAD   | PROCESS |
| -------- | ------- |
| part of a process. Therefore, going to be more lightweight: less time for creation and termination, context switching,  | Program under execution. Ready state process will be scheduled by CPU for execution. |
| 1. All the threads in the same program share the same memory space. If an object is accessible to various threads then these threads share access to that object's data member and thus communicate each other. 2. Thread control methods such as yield, join   | Process is isolated, but IPC is possible by message queue or shared memory buffer |
| created via API | System call is needed |
| PC1    | PC1    |  
<br />

2. What is a process control block?
Process Control Block (PCB) is a data structure used by the operating system to manage information about a process. It tracks information such as
- Pointer: It is a stack pointer that is required to be saved when the process is switched from one state to another to retain the current position of the process.
- Process state: It stores the respective state of the process.
- Process number: Every process is assigned a unique id known as process ID or PID which stores the process identifier.
- Program counter: Program Counter stores the counter, which contains the address of the next instruction that is to be executed for the process.
- Register: Registers in the PCB, it is a data structure. When a processes is running and it’s time slice expires, the current value of process specific registers would be stored in the PCB and the process would be swapped out. When the process is scheduled to be run, the register values is read from the PCB and written to the CPU registers. This is the main purpose of the registers in the PCB.
- Memory limits: This field contains the information about memory management system used by the operating system. This may include page tables, segment tables, etc.
- List of Open files: This information includes the list of files opened for a process.
- Interrupt Handling: The PCB also contains information about the interrupts that a process may have generated and how they were handled by the operating system.
- Context Switching: The process of switching from one process to another is called context switching. The PCB plays a crucial role in context switching by saving the state of the current process and restoring the state of the next process.
-Virtual Memory Management: The PCB may contain information about a process’s virtual memory management, such as page tables and page fault handling.
-Inter-Process Communication: The PCB can be used to facilitate inter-process communication by storing information about shared resources and communication channels between processes.
-Fault Tolerance: Some operating systems may use multiple copies of the PCB to provide fault tolerance in case of hardware failures or software errors.

<br />

3. What is virtual memory and why is it important?
Virtual memory is a memory management technique used by operating systems to give the appearance of a large, continuous block of memory to applications, even if the physical memory (RAM) is limited. It allows the system to compensate for physical memory shortages, enabling larger applications to run on systems with less RAM.

Memory references (virtual addresses) used in each process maps to some phyiscal memory address at runtime. Process can be swapped in and out of main memory during execution. Process can be broken down into multiple pieces and each occupies a non contiguous part of the physical memory. 2 main types of virtual memory are paging and segmentation.
- **Paging** -> divides memory into fixed size blocks called pages. pages that are not currently in use will be swapped to disk. When a page that is not in ram is needed, CPU generates an interrupt indicating memory access fault, and the OS will bring the page back into RAM.
- **Segmentation** -> divides virtual memory into segments of different sizes. segments that are not needed can be moved into disk. segments are only mapped into a process address space when needed.
<br />

4. What is the difference between Cache and Ram?
Cache is usually located near the CPU, the main use is to increase access speed of CPU. Main memory is the capacity of memory to run processes, etc. Some data on RAM can be fetched and be in cache memory for faster access.
<br />

5. Difference between Spatial Locality and Temporal Locality?
Spatial Locality means that instructions that are stored nearby to the executed instruction have higher chance of execution. Temporal Locality means that instruction that are recently executed have higher chances of execution again.
<br />

6. Cache, TLB, paging / segmentation ...
7. Thread yield, Thread join
8. What is the difference between mutex and semaphore
9. Deadlocking
10. What is buffering?
The buffer is an area in the main memory used to store or hold the data temporarily. A buffer may be used when moving data between processes within a computer. Load data slowly to a buffer, and one shot do a fast write operation for example. Types of buffer includes single, double and circular.