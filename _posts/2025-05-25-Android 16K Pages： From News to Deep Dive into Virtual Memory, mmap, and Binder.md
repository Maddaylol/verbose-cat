## **1. News**
### **What Does It Mean?**
- On November 1, 2025, Google Play will require all apps targeting Android 15+ to support 16KB page sizes. At first glance, this sounds like a low-level systems change that shouldn’t concern app developers—but it does. This change is rooted in how Android and the Linux kernel manage memory: virtual memory.
- Traditionally, most Android devices have used a 4KB page size. A page is the basic unit of memory management; both virtual and physical memory are split into pages and page frames. When we say “16KB pages,” we’re saying that each virtual page now covers 16KB of address space, and each page table entry (PTE) maps a bigger chunk of memory.
### **The Meaning of the News**
- This change reduces the number of page table entries (PTEs) needed to map memory, which improves TLB (Translation Lookaside Buffer) hit rates and reduces the frequency of page faults. It also simplifies memory management overhead, especially for applications with large memory or file mappings.
- However, using larger pages might increase internal fragmentation—if a program needs only 6KB, a 16KB page wastes 10KB. But overall, the system benefits in cache usage and reduced kernel memory pressure often outweigh these drawbacks.
## **2. Practice About Virtual Memory**
### **Virtual Memory Concepts Refresher**

- **Virtual vs Physical Address:** Programs work with virtual addresses. The OS uses page tables to map them to physical memory (page frames).
    
- **Page Table & PTE:** A page table entry maps a virtual page to a physical frame. One PTE typically holds a page frame number and control bits.
    
- **Page Fault:** When a program accesses a virtual address that isn’t currently mapped, the CPU triggers a page fault. The OS handles this fault by allocating a page and updating the page table.
    
- **TLB:** A cache of recently used page table entries. Bigger pages improve TLB hit rates because more memory is covered per entry.
    
- **Why 16K Matters:** Fewer PTEs, better TLB performance, fewer page faults, but potentially more internal fragmentation.
    
### **mmap**
- Programs traditionally use read() to load data into buffers. With mmap, files or devices are mapped directly into a process’s virtual memory, letting the program access data like a regular memory array.
- **Advantages of mmap:**
	- **Zero-copy:** No need to manually copy data from kernel to user space.
	- **On-demand loading:** Pages are brought into memory only when needed.
	- **Performance:** Reduces syscall overhead and improves cache locality.
- Behind the scenes, accessing an mmap-ed address that isn’t backed by a physical page triggers a page fault. The kernel then loads the data into a page frame and updates the process’s page table.
### **mmap vs read()**

| **Feature**       | **mmap**                 | **read()**                       |
| ----------------- | ------------------------ | -------------------------------- |
| Data transfer     | Memory mapping           | Explicit copy into buffer        |
| Page table update | On-demand via page fault | No change (buffer is pre-mapped) |
| System call count | Fewer (after setup)      | One per read                     |
| Zero-copy         | ✅                        | ❌                                |

### **mmap in Android – Binder**

- Binder is Android’s primary IPC (Inter-Process Communication) mechanism. It lets apps communicate as if calling local methods, but under the hood uses shared memory and system calls.
#### **How mmap powers Binder:**
1. App opens /dev/binder, which is a character device managed by the Binder kernel driver.
2. The app mmap()s a region of memory, creating a binder buffer shared with the kernel.
3. To send a request, the app writes data into the buffer and uses ioctl() to notify the driver.
4. The driver either copies or remaps the buffer to the receiving process’s address space.
5. The receiver reads from its own mapped buffer.
    
This avoids excess copying and allows the kernel to manage security, process lifetimes, and buffer ownership.

#### **Why Binder Must Be in Kernel Space:**

- Only the kernel can safely:
    - Map memory across different processes (manage page tables)
    - Check and enforce permissions
    - Perform thread wakeups and inter-process coordination
    - Handle page faults and context switches
- Userspace cannot access another process’s address space or manage page frames securely.
## **3. How Should We Check About on Device**
To observe mmap and Binder behavior on an Android device:
1. Check if Binder exists:

```
adb shell ls -l /dev/binder
# Or
adb shell ls -l /dev/binderfs/binder
```

2. Confirm that a process uses memory-mapped IO:
    
```
adb shell cat /proc/<pid>/maps | grep binder
```
    
3. Use strace to inspect system calls:
```
adb shell strace -p <pid> -e mmap,ioctl
```

  This confirms that the app interacts with the Binder driver via memory-mapped buffers and control commands.

## **4. Conclusion**
Starting from a policy change about 16KB pages, we explored:
- The basics of virtual memory, page tables, and TLBs
- Why larger pages mean fewer page faults and better TLB efficiency
- How mmap changes the game from copying to mapping
- How Binder leverages mmap to implement Android’s high-performance IPC
- Why user-space can’t (and shouldn’t) do what the kernel does: manage memory mappings, protect isolation, and coordinate processes
**Key takeaways:**
- For the CPU, threads and processes are both just register snapshots + stack pointers.
- The OS uses the same task structure to manage both, but for developers, threads imply shared space and processes imply isolation.
- mmap, page fault handling, and Binder all depend on the kernel’s ability to control page tables and physical memory.
Understanding these foundational mechanisms helps developers write faster, more stable apps—and appreciate the power and elegance of the Linux-based Android system.