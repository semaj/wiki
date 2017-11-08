# Shredding Your Garbage: Reducing Data Lifetime Through Secure Deallocation

## Motivation
Operating systems, word processor, web browsers, and other common pieces of software do not take measure to remove data from memory after usage. In fact, sensitive data such as passwords and keys can remain in memory for days. 

## Terms and Definitions

* **Ideal Lifetime** - the period of time that data is *in use*, from the first write after allocation to the last read before deallocation
* **Natural Lifetime** - the window of time where attackers can retrieve useful information from an allocation, even after it has been freed. This spans from the first write to the first overwrite (of the complete data)
* **Secure Deallocation Lifetime** - as per the authors' system, this is the time between the first write and its deallocation (whether via a managed runtime or via a manual call to `free`). This lies between the first two lifetimes, and is the best the authors think they can achieve.
* **Holes** - Many might say that things will get overwritten eventually, but unfortunately many times compilers and applications allocate much more data than is actually used (on top of possibly sensitive data). This empty space is known as a 'hole', and it leaves sensitive data around. Overwriting is necessary.

## Experimental Findings
The authors placed 64KB of data onto Linux and Windows machines. After 14 days, between 23KB and 3MB of the data remained (the process stopped 14 days ago!) in memory. Soft reboots often does not clear most of RAM (this happens during many restarts). 

## Design
Can we design a system that performs secure deallocation, which essentially means overwriting data once it is deallocated, which does not incur sigificant efficiency or development overhead?

The authors propose instrumenting such secure deallocation at 3 layers:

1. Applications: by instrumenting standard library calls such as `malloc` and `free`, they can ensure that things get overwritten once they are manually deallocated. **I assume this is very challenging for managed runtimes like Python and Ruby, since they can free memory at whim**
1. Compilers: by instrumenting the stack and heap managers in popular compilers, we can ensure pieces of data that the application developers do not explicitly control can be overwritten
1. OS/Kernel: kernels manage application resources. This includes process *pages*, which hold process memory. We need to ensure that process pages are overwritten after the process is killed.

We want multi-layered clearing because applications may be long running, or short running. If they are long running, the kernel can't clear memory. If they are short running, the application might not bother to clear it and it's up to the kernel to do so.

1. Compilers/Libraries: all heap allocated data is zeroed immediately during `free`. For the stack, we periodically zero all data below the stack pointer.
1. Kernel: often we can't clear memory too often, as it will incur penalties. But we can periodically wipe "dirty" pages. This *zeroing daemon* wakes up periodically to zero pages that have been polluted longer than some time limit.

## Results
Many applications allocate much more memory than overwrite. This means things are left over after execution. The authors did not actually instrument applications **(I THINK)**, though they measured the time between allocation/deallocation, providing a theoretical secure deallocation time. They found that the lifetime was much better for most programs.

What about testing to see if *specific* pieces of sensitive data is overwritten?

They instrumented applications with taint analysis (via an x86 simulator) to see if tainted pieces of data were cleared. With an unmodified kernel, many tainted regions were found in kernel/user space after the program terminated (perl CGI). With their modified kernel, all taints were cleared.

The overall overhead for such things was reasonably negligible. Usually within 1%.


