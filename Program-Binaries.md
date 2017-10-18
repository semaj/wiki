# Program Binaries

## Terms and Definitions
* **Dynamic Linking** - when examining a binary, the symbol is unmapped. The binary expects to know where a symbol maps once the loader "links" a library at runtime. This means the library must be loaded into memory by the loader, and can be compiled independently of the target application.
* **Decompiler** - takes machine binary code and tries to translate it into high-level source code
* **ptrace** - Allows one process to pause, resume, and inspect another process. `ptrace` can pause child before syscall, pause child after single instruction during single-step mode. 
* **Debugging symbols** - metadata in a binary which refers to the higher-level source. DWARF format for ELF binaries. Can increase the size of binaries *significantly*.

## Decompilation
The raw binary application (when not compiled with debugging symbols, usually) is fed into the following process:
1. Disassembly : dumping the assembly instructions
1. Idiom translation : Find sequences of instructions that don't make sense at first, but used by compilers to performed well-known actions. 
  * For example, `xor %eax %eax` is often used to clear eax, and its shorter (bytes-wise) than a `mov`.
1. Type extraction : Recover high level types. For instance, we know you don't perform `and`s on floats.
1. Dataflow analysis : Constructs "basic blocks", which are sequences of instructions without jumps. Tries to translate them to higher-level code. 
1. Control flow analysis : Analyze branches to try to reconstruct `if else` statements. Backwards unconditional jumps usually evidence of a loop.

