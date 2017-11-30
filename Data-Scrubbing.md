# Shredding Your Garbage: Reducing Data Lifetime Through Secure Deallocation

## Motivation
Operating systems, word processor, web browsers, and other common pieces of software do not take measure to remove data from memory after usage. In fact, sensitive data such as passwords and keys can remain in memory for days. 

## Terms and Definitions

* **Dead Store** A write that is never read by subsequent code. Gcc will try to remove memsets that appear to represent dead stores.
* **Ideal Lifetime** - the period of time that data is *in use*, from the first write after allocation to the last read before deallocation
* **Natural Lifetime** - the window of time where attackers can retrieve useful information from an allocation, even after it has been freed. This spans from the first write to the first overwrite (of the complete data). This is a problem because an attacker could possibly grep through pages files while their in memory, or even after they've been paged to disk.
* **Secure Deallocation Lifetime** - as per the authors' system, this is the time between the first write and its deallocation (whether via a managed runtime or via a manual call to `free`). This lies between the first two lifetimes, and is the best the authors think they can achieve.
* **Holes** - Many might say that things will get overwritten eventually, but unfortunately many times compilers and applications allocate much more data than is actually used (on top of possibly sensitive data). This empty space is known as a 'hole', and it leaves sensitive data around. Overwriting is necessary.

## Experimental Findings
The authors placed 64KB of data onto Linux and Windows machines. After 14 days, between 23KB and 3MB of the data remained (the process stopped 14 days ago!) in memory. Soft reboots often does not clear most of RAM (this happens during many restarts). 

Windows has a background thread that zeroes out pages.

Many systems declare a memset variant that compilers can't optimize away.

## Design
Can we design a system that performs secure deallocation, which essentially means overwriting data once it is deallocated, which does not incur sigificant efficiency or development overhead?

The authors propose instrumenting such secure deallocation at 3 layers:

1. Applications: by instrumenting standard library calls such as `malloc` and `free`, they can ensure that things get overwritten once they are manually deallocated. **I assume this is very challenging for managed runtimes like Python and Ruby, since they can free memory at whim**
1. Compilers: by instrumenting the stack and heap managers in popular compilers, we can ensure pieces of data that the application developers do not explicitly control can be overwritten. Stack and heap will be overwritten.
1. OS/Kernel: kernels manage application resources. This includes process *pages*, which hold process memory. We need to ensure that process pages are overwritten after the process is killed.

We want multi-layered clearing because applications may be long running, or short running. If they are long running, the kernel can't clear memory. If they are short running, the application might not bother to clear it and it's up to the kernel to do so.

1. Compilers/Libraries: all heap allocated data is zeroed immediately during `free`. For the stack, we periodically zero all data below the stack pointer. When a function returns, the popped stack frame should immediately (or periodically) be zeroed out.
1. Kernel: often we can't clear memory too often, as it will incur penalties. But we can periodically wipe "dirty" pages. This *zeroing daemon* wakes up periodically to zero pages that have been polluted longer than some time limit. Specifically, when a user-level process dies, the kernel must zero-out memory pages. Sensitive data could live in internal kernel-level buffers like network packets, keyboard input, etc. These buffers should be securely deallocated.

## Results
Many applications allocate much more memory than overwrite. This means things are left over after execution. The authors did not actually instrument applications **(I THINK)**, though they measured the time between allocation/deallocation, providing a theoretical secure deallocation time. They found that the lifetime was much better for most programs.

What about testing to see if *specific* pieces of sensitive data is overwritten?

They instrumented applications with taint analysis (via an x86 simulator) to see if tainted pieces of data were cleared. With an unmodified kernel, many tainted regions were found in kernel/user space after the program terminated (perl CGI). With their modified kernel, all taints were cleared.

The overall overhead for such things was reasonably negligible. Usually within 1%.

# CleanOS: Limiting Mobile Data Exposure with Idle Eviction

## Motivation
Prevent somone who stole your phone from finding cleartext sensitive data in RAM or persistent storage (SSD).

## Terms and Definitions

* **Sensitive Data Object (SDO)** - a collection of Java objects, files, database items that contain sensitive information

## Design
How do programmers define sensitive data?
* Programmers can explicitly add objects to an SDO. 
* CleanOS predefines certain sources as sources of SDOs. For example, data that comes from the camera, keyboard, microphone etc. CleanOS is built atop [[TaintDroid|Taint-Analysis]], which uses IFC to taint all data derived from SDOs.

When an application as been idle for one minute, CleanOS will encrypt the application's SDOs. 

In a managed language like Java, a GC is responsible for periodically walking the heap and identifying dead objects and freeing then. In CleanOS, if GC finds a live object, GC will check whether the objecth as been recently accessed. If the answer is no, CleanOS encrypts the object in place.

Once GC sweeps finishes, CleanOS sends encryption key to a cloud server. It then securely deallocates the local copy of the key.

Dalvik interpreter detects when executing a bytecould would access an encrypted object. Dalvik fetches decryption key from cloud, decrypts object and that continues as normal.

# Stopping a Program from Reading Its Own Pages

In ROP attacks, the attacker is going to find gadgets in the executable code of a vulnerable application. It will string these gadgets together to create arbitrary code.

* ASLR prevents attackers from having a priori knowledge of gadget locations
* Memory disclosure vulnerabilities allow the attacker to read the contents of virtual memory for the program that is being exploited and find gadgets
* In Firefox, there was a recent vulnerability involving malformed gif files. Malicious page would run JS code that tries to load a malformed gif file inside a canvas tag. Firefox had a gif bug and didn't use secure deallocation, so the bitmap belonging to the canvas tag might leak contents of memory to the malicious JS code in the page. This could be used as a first step for a ROP attack, since it discolses memory.

Preventing a process from reading its own code pages is hard, since on commodity processors, execute *implies* read. These two permissions are conflated by most processors.

* **TLB** - Hardware cache that maps virtual page numbers (upper bits of a virtual address) to physical page numbers (upper bits of physical address). When the hardware needs to translate a virtual address to a physical one, it checks the TLB for a mapping before checking the page tables. This TLB lives in hardware, above memory.
* **Split TLB** - One cache, set of mapping structures, just for instructions AKA code pages. There's another region of the TLB is dedicated to data. Normally, a virtual address that is both read and executed will have the same mapping in both of the TLB halves.

```
virt_addr 0x400000 : 1TLB [0x400] -> [0x100]
virt_addr 0x400000 : 0TLB [0x400] -> [0x100]
both map to same place in physical memory
```

**TLB desynchronization:** the OS will make the instruction TLB point to code pages but the data TLB points to a page that is filled with zeroes for virtual addresses that correspond to code pages.

```
virt_addr 0x400000 : 1TLB [0x400] -> [0x100] : a code page
virt_addr 0x400000 : 0TLB [0x400] -> [0x000] : bunch of zeroes
both map to same place in physical memory
```

When a process tries to read its code, it should read all 0s, not the actual instructions that correspond with the code.

* Initially, OS marks all code pages as not present in memory. When a process tries to access a code page either in reading or executing, the hardware is going to generate a page-fault and invoke the OS.
* In the page fault handler, the OS compares the saved instruction pointer to the virtual address that causes the page fault to occur. If those two addresses are equal, the process is trying to execute its own code. OS sets 1TLB as normal. 

```
0x40000 add %eax, %ebx
// saved %eip: 0x40000
// faulting addr: 0x40000
```
* On the other hand, if the two addresses are not equal, then the process is trying to read its own code. (Since the `eip` is somewhere else, we aren't reading for execution, we're reading for reading!) In this case, we point to 0TLB.
