# [[Advanced Metamorphic Techniques in Computer Viruses|pdfs/metamorphic-viruses.pdf]]

## Motivation
1. The first antivirus software simply scanned for binary signatures in [[malware|Malware]].
1. Viruses were written to mutate their code and their decryption method upon replication.
1. Antivirus software allowed viruses to decrypt themselves and then scanned for signatures.
1. This lead to metamorphic viruses, which mutate their code after decryption.

These viruses often create new pages which contain the decrypted version of code, since they can't write to NX bits.

This article describes both simple and complex metamorphic techniques.

## Terms and Definitions
* **Polymorphic virus** - a virus which changes its encryption key upon duplication such that the encrypted code is different for every duplication.
* **Metamorphic virus** - a virus which changes its decryption key but also the decrypting portion of the code upon duplication such that its difficult to detect the virus via decryption signature.
* **Form analysis** - A identifying a virus by a sequence of (not necessarily consecutive) bytes whose detection inside a program indicates possible infection.
* **Detection by code emulation** - static analysis was unreliable, so the binary is run in a sandboxed environment with limited instructions. Periodically the affected memory was analyzed to detect decrypted viral code.
* **Entry Point Obscuring** - Running the virus code at the middle or end of the host binary execution rather than the start.

## Overview

1. Viruses commonly appended themselves to the end of executable files, then changed the entry point to point at the virus code. 
1. Form analysis solved this, but virus detection is undecidable.
1. Viruses then encrypted their code, added a decryption routine, and changed the entry point to the decryption routine.
1. Form analysis can detect decryption methods.
1. Viruses like *CHAMELEON* emerged, which mutated the code of their decryption method. The *WHALE* virus emerged, which mutated its *mutation function*. Polymorphic *engines* then appeared, which could be linked to a virus to produce polymorphic variants. After that, virus creation toolkits appeared.
1. Kaspersky worked out **detection by code emulation**. It's quite CPU intensive.
1. Anti-emulation techniques:
    * Using unusual instructions
    * Inserting dead code that will loop for a long time
    * Random canceling of decryption (running the virus randomly)
    * Entry Point Obscuring
    * Using several encryption layers.
    * Decrypting and running the code chunk-by-chunk.
    * Metamorphic techniques including transforming the encrypted code.
1. Virus detection is determined to be NP-complete.
1. Viruses start shipping with their own compilers (Tiny Mutation Compiler). On execution, it decrypts itself, inserts dead code, shuffles things, and recompiles everything. 

### MetaPHOR
The first ever polymorphic, metamorphic Linux virus. Most viruses encrypt themselves using keys either stored within the program, keys derived from the program itself or the environment, or no key at all (the decryption must be brute forced). 

* MetaPHOR used a random key and normal XOR/ADD/SUB encryption.
* Used an **EPO**: it changed all calls to exit into jumps to the decryption routine.
* 70% metamorphic code, 20% infection routine, 10% decryptor creation routine 

#### Branching 
Innocuous programs will usually sequentially test several conditions, and depending on the result finally branch on distinct paths. MetaPHOR creates several random tests until a desired recursivity level that will define an execution tree with leaves containing *distinct* but *equivalent* decryption code. Rather than looping (which would set off heuristics) it jumps to a previous node in the tree, and executes. Since all leaves do the same thing, it will decrypt the program.

#### PRIDE
Heuristics will often check for the following routine:

1. Address of buffer inside data section
1. A *sequential* read of the buffer as well as a creation of a new buffer (the decrypted code)
1. Control given to the new buffer

With Psuedo-Random Index DEcryption, MetaPHOR decrypts the buffer in non-sequential, random order. 

#### Metamorphism
MetaPHOR creates an intermediate representation which it operates on for manipulation and obfuscation. This dissociates it from the complexity of the underlying processor's instruction set. 

1. Disassembly / Depermutation
  * x86 disassembled into IR. All dead code is removed, and we are left with only the bare-bones IR. 
1. Compression
  * Remove expansion that was done previously. (extraneous instructions)
1. Variable reorganization
1. Permutation
1. Expansion
1. Reassembly

MetaPHOR contains its own psuedo-random number generator.

### Detection
* The viral code's encryption can be identified by a statistical analysis of the code. You can compare "entropy profiles". Antivirus will consider suspicious any application with lots of encrypted data...
* Antivirus can monitor memory for the compression signatures.
* Behavior analysis: load malware in a sanbox and look for suspicious patterns (writing to memory page and jumping into a new page)
