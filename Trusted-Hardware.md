# Why?
Protecting sensitive code from untrusted code is difficult. Traditionally the OS will be responsible for isolating processes. (Page tables, system calls). This keeps the hardware simple, but 
* the OS is the trusted computing base and the OS is huge! You can't assume the OS is trusted since chances are, there are bugs. 
* Hardware is vulnerable to physical attacks like side channels, memory buses, etc 

## Goal
* Add hardware-level support for improving isolation. If we can encrypt RAM using a key stored in tamper proof hardware, someone with physical access to the RAM can't do anything.
* Shrink the software-level Trusted Computing Base to just the security-conscious code.
* Hopefully keep the additional complexity of hardware small

# TPM (Trusted Platform Module)
The TPM spec outlines a dedicated security chip designed to enhance software security. 

## Background

The TPM contains at least 16 **Platform Configuration Registers** (PCRSs). These are essentially internal memory slots. At boot time, PCRs are initialized to a known value (0 for PCRs 1 - 16 and -1 for PCRs 17 - 22). 

Software may change the value of a PCR via:

```
PCRExtend(index, data)
```

This updates the value of the PCR indicated by index with a SHA1 hash of the previous value, plus the new value.

```
PCR[index] = H(PCR[index] ++ data)
```

Data must be 20 bytes. 

## Classic Remote Attestation

```
firmware: extend(10, hash(firstLevelBootloader)
firstlevelbootloader: extend(10, hash(secondLevelBootloader)
secondlevelbootloader: extend(10, hash(OS)
OS: extend(10, hash(executable binary)
```

* Attestor says hello
* Verifier sends a nonce
* Attestor sends a list of hases for extended components, certificate for public key, and signed `<PCR[10], nonce>`
* Verifier validates the signature, ensures that PCR[10] is cumulative hash of specific components
* Checks a hash database to see whether binaries are trusted

Deficiencies
* Boot order is non-deterministic
* Attestation only captures load-time integrity of a file
* Difficult to reason about machine-specific customizations for a config file
* Attestation allows verifier to learn the exact software stack that the attestor runs

## Sealed Storage

The TPM provides two operations:

```
SEAL(indices, data) = (C, MAC[k]((0, PCR[0]), (1, PCR[1])...))

UNSEAL(C, MAC[k]((0, PCR[0]), (1, PCR[1])..)) = data
```

* `C` : ciphertext `= {data}key`
* `k` : root secret key stored on the TPM that never leaves

The TPM verifies the integrity of the MAC provided by checking to see if the contents of the PCRs in the MAC match the current values of those PCRs. If so, the TPM decyrpts the ciphertext.

## Secure Booting

The TPM can compute:

```
PCR[5] = H(0 ++ B ++ L ++ O ++ A)
```

Where:
* `B` = the BIOS code
* `L` = the bootloader code
* `O` = the OS code
* `A` = the application code

The application can then generate a secret like so:

```
SEAL(5, secret) = (C, MAC[k]((5, PCR[5]))
```

Note that an attacker cannot access this secret unless it is running all of the correct software. If it is, it can access this secret and prove to others that it is potentially running correct software. Note this does not help against bugs *in* the software itself.

# SGX
The problem: how can we execute software on a remote computer owned and operated by an *untrusted third party*? How can we improve the security of the x86 architecture without performing major changes to the architecture?

* Backwards compatibility is the reason for much of SGX's complexity
* **Enclave** - a secure computation that belongs to an enclosing but untrusted process
  * Has a region of memory for enclave's code, data, and CPU
  * SGX only wants to attest to the enclave's computation. In contrast, a TPM chip is used to attest the entire software stack.
  * Enclave in SGX world treats all code outside the enclave as untrusted. That include the OS.

Intel's Software Guard Extensions (SGX) aims to place trusted hardware on the remote computer. 

The SGX, via *software attestation*, proves that the computer is running specific software. SGX will always provide an honest answer about the software running, so the user can refuse to provide the remote computer with data if it is not running the correct software.

The user does so by verifying the *attestation key* provided by the server against an *endorsement certificate* created by the hardware manufacturer. This certificate states that the attestation key is only known to the trusted hardware. 

SGX does this by defining an *enclave*. SGX can attest to the software running inside of the enclave, and all computations performed inside the enclave are protected.

The enclave relys on the untrusted OS to do things like 
* Manage page tables
* Interact with I/O devices
* However, SGX verifies the integrity of memory pages (encryption and hashes)
* The enclave only exchanges encrypted data with remote clients (OS never sees the encryption keys)

* **Processor Reserved Memory (PRM)** - region of **physical** DRAM that is reserved for enclave code and data: only enclave code or SGX hardware itself can access PRMs

## Creating an Enclave

1. The untrusted host process invokes the `ECREATE` instruction to initialize SGX metadata for new enclave.
  * Starting virtual address and lendth of enclave's code and data
  * Hash of the enclave's code and data (hash value is unset at first)
1. Host process copies the enclave's code and data from regular untrusted host memory into the PRM region for the enclave
  * `EADD` instruction specifies source virtual address in untrusted host process and the destination virtual address in the enclave's PRM. Works at the granularity of pages. Copies memory page to enclave's PRM. 
  * `EADD` does not extend the hash that is associated with the enclave. 
  * So we must call `EEXTEND`, which asks the SGX hardware to use 256 bytes of data at a starting virtual addreess in host to add to cumulative hash. If a page size is 4k bytes, we must call `EEXTEND` multiple times to extend the hash to include the code and data
  * We split up instructions because if you allow interrupts you must be willing to restart operations
1. Host process calls `EINIT` to tell the SGX hardware that the enclave setup is done.
  * SGX hardware finalizes hash value in enclave's metadata page in PRM. After that host process can't call `EADD` or `EEXTEND` any more. 
  * Enclave and untrusted host process use same page tables and in fact share same address space
  * SGX hardware can tell if the currently executing code is running in enclave mode or regular mode. If the mode is equal to regular, the enclave region is mapped to ABORT pages. Although its encrypted, we don't want code to be able to operate on it at all.

SGX allows the untrusted OS to control page tables.
  * Why can't the OS change the mappings for the virtual pages in the `ELRANGE` to non-PRM memory? The `ELRANGE` is the set of *virtual* addresses that correspond to enclave code and data. This would allow the OS to map the sensitive `ELRANGE` pages to OS-controlled non-PRM memory. This would allow other processes to read non-PRM sections of the physical memory.
  * SGX binds each virtual page in the enclave to the physical PRM page at `EADD` time, these bindings will be stored in PRM memory that only SGX hardware can access. For every memory access that the enclave generates, SGX hardware can ensure that the TLB mappings are the ones set by the initial `EADD`. 

When a core is running in enclave mode, the SGX hardware is going to transparently encrypt writes to the PRM and decrypt reads from the PRM. A physical attacker can't look at the RAM hardware and see cleartext PRM data.

To enter the enclave code, untrusted host invokes the `EENTER` instruction. 
  * Switch the CPU mode to enclave mode and jump to the enclave's entrypoint.
  * `EENTER` can only be called by ring 3 code (lowest x86 privilege level). While the enclave code executes, it stays at ring 3 privilege. It's not executing at OS level privilege. The enclave is dependent upon the untrusted OS to perform I/Os.

`EEXIT` - switches mode from enclave to normal

## SGX Remote Attestation

* Client sends $g^c mod p$
* Server sends certificate, ($g^client mod p, g^server mod p, hash(initialEnclaveState)$)signed by SGX private key
* Client sends some secret data encrypted by DH key
* Server sends back result of computation encrypted by DH key
