# Taint Analysis

## Panorama
Panorama tracks Information Flow Control at the x86 level. It runs applications within QEMU, an x86 emulator. 

* To assign taint, label every register and every byte of memory that is touched
* Define taint sources as HDD, keyboard, network
* Looks at every CPU instruction propagating taint across registers and memory
* Whenever tainted pointer is *dereferenced*, result is also tainted

### Taint Explosion
If it's so easy to taint things, everything could get tainted. For example, `%ebp` is used *by convention* to store old stack frame pointer, but it can be used as a general purpose register, especially by the compiler. 

If:
```
foo() calls bar()
foo() uses %ebp in the standard way
bar() uses %ebp as a general purpose register
```

in `bar()`, `%ebp` is tainted. When `bar()` returns, the compiler resets `%ebp` for foo like ` add some_constant %ebp`. `%ebp` is still tainted upon return to `foo()`. `foo`'s local variables are tainted because they are accessed via *deferences* of offsets of `%ebp`. If the kernel state is accidentally tainted via system call, GAME OVER.

## Implicit Flows
If `IMEI` is tainted:

```
if (IMEI > 42) {
  x = 0
} else {
  x = 1
}
```
`x`'s value is dependent upon a tainted value, but since it's value is set via control flow rather than assignment, `x` is not tainted. 

Solution:

1. Assign label to PC
1. Update w/ taint of each branch test
1. Assign PC's taint to values in each clause

# TaintDroid: An Information-Flow Tracking System

## Motivation
We'd like a way to track data within and between Android applications. By tracking data, we can prevent secret data from being externalized. TaintDroid prevents sensitive information from being sent over the network. 

Existing Android permissions are very course-grained: can this application access your location or not? If it can, it has cart blanche on what it can do with that data. What is it actually doing?

## Terms and Definitions
* **Source** - a sensor or other piece of the phone which contains and can emit sensitive data, from which we want to track
* **Sink** - Exposes sensitive data. Usually the network.
* **Register-based language** - a language which reasons using actual registers rather than a stack and stack operations
* **Native methods** - usually C/C++ methods which expose primitive functionality within the Linux kernel. Can also reference Java internals. Android has VM methods and JNI (Java Native Interface) methods.
* **Binder** - the interface for IPC (inter-process communication)

## Overview
TaintDroid runs on Android, which uses the Dalvik VM interpreter. Its language, DEX, is a **register-based language** in which registers *loosely* correspond to local variables in the Java method. Android programs can also execute **native methods** which expose Linux functionality.

TaintDroid instruments the Dalvik VM interpreter. TD maintains a 32-bit label which indicates the taint status of a given object. `[0, 0, 1, ... 0]` indicates that the 3rd bit (whatever that represents) is tainted.

In:
```java
x = gps.getLatitude();
```
`x` would be tainted. 

In reality, this happens at the bytecode level.
```x86
mov-op dst, src         # dst gets taint of src
bin-op dist, s0, s1     # union of s0 and s1's taint
```

* TaintDroid assigns a universal (unioned) taint value to arrays to save space.
* TD uses a similar approach for IPC packages and files: one taint value for all bytes in IPC or file.

How does TD assign taint for **native methods**, since they execute outside of the interpreter VM? 

* Authors went through every native method and assigned unique taint-assignment rules for each method.

There are five types of data that require taint-tracking:
1. Local variables in methods
1. Method arguments
1. Object instance fields
1. Static class fields
1. Arrays

Basic approach: store the taint-flag for a variable *close* to that variable in memory. This improves caching behavior and random access. Caches often pull **cache lines**, so if you keep your taint label next to your memory, your label will probably be pulled when the data is pulled.

Since Java doesn't have raw pointers, attackers can't just overwrite taint values in memory.

## Performance
Memory overhead: 3 - 5% increase
CPU: 3 - 29% increase

## Now
Android (post 2013) applications are downloaded in Dalvik bytecode form, installed via compilation to native code. It's faster, and it only pays JIT overhead once.
