# Bitlocker: AES-CBC + Elephant Diffuser

## Motivation & Goals
An estimated 1-2% of laptops are lost each year. Often the data on these laptops is more valuable than the laptops themselves. Microsoft wanted to implement a solution into the Windows OS that would enable encryption of the disk without requiring too much human intervention (no passwords or physical keys). 

## Threat Model

* Bitlocker does not defend against hardware based attacks
* The attacker has many known (but not chosen!) plaintext/ciphertext pairs for *different sectors*.
* The attacker has the ciphertexts for a large number of chosen plaintexts for different sectors. The plaintexts are chosen before the attacker gets access to the laptop.
* The attacker has access to a slow decryption function for some of the sectors
* The attacker gets several ciphertexts of plaintexts for the same sector with a known (but not chosen) difference

The attacker succeeds if he can modify a ciphertext such that the corresponding plaintext change has some non-random property.

## Overview
Bitlocker encrypts almost all data on the hard-disk. It encrypts all data on the OS volume. It protects confidentiality, but unfortunately it's possible for the attacker to modify the encrypted code in order to weaken the OS code. 

* Encryption is done on a per-sector basis, because applications need guarantees that if they are reading or writing to a given sector, they cannot corrupt other sectors on the harddrive. 
* Making the ciphertext larger than the plaintext is not feasible, (MACs) because it would create a lot of space overhead.

As a result, Bitlocker uses "poor man's authentication", which essentially just trusts that it's difficult to modify the ciphertext in a way that creates semantic changes in the plaintext.

Bitlocker uses the [[TPM|Trusted-Hardware]] on the laptop. The Bitlocker encryption key is sealed under the configuration of the user's laptop. This ensures that the disk can only be decrypted on the laptop the disk was encrypted on. Since TPM unsealing is brittle (any changes to the OS change the PCR and thus prevent unsealing of the key) Bitlocker runs at the earliest possible moment. I assume this moment is immediately after the user logs in?

### Properties

* Encrypts and decrypts disk sectors from 512 - 8192 bytes
* It takes the sector number as an extra parameter (the tweak) and implements different encryption / decryption algorithms for each sector
* It protects the confidentiality of the plaintext
* It is faster than 40 clock cycles/byte read from the HD
* It has been validated by public scrutiny
* The attacker cannot control or predict any aspect of the plaintext changes if he modifies or replaces the ciphertext of a sector. **We don't want the attacker to be able to just switch certain sectors**

### Execution
Bitlocker uses AES in CBC mode. This is a block cipher and operates on the given sectors. Unfortunately, AES-CBC suffers from poor *diffusion* in the decryption (each bit in the ciphertext should depend on several parts of the key). If the attacker introduces a change in the ciphertext block *i*, the plaintext block *i* is randomized but plaintext block *i+1* is changed according to the original change. In other words, the attacker can flip arbitrary bits in one block at the cost of randomizing another. 

In order to mitigate this, Bitlocker adds *diffuser* to the plaintext side. It is proven (?) to be just as secure as AES-CBC, though diffuser is an unproven algorithm...

[[img/bitlocker-aes-cbc-diffuser.png]]

The IV used in the encryption:

```
IV[s] = AESEncrypt(AESKey, e(s))
```

`e` is an encoding function from sector number to unique 16 byte value. `s` is the sector number.

Sector key:

```
K[s] = AESEncrypt(SectorKey, e(s)) ++ AESEncrypt(SectorKey, e'(s))
```
`e'` is the same as `e` except the last byte of the result has the value 128. The Sector key is repeated as many times as necessary to get a key size of the block, and the result is xorred into the plaintext.

# Defeating Encrypted and Deniable File Systems

## Motivation
Deniable file systems (see TrueCrypt) are useful for activists, criminals, and security conscious individuals everywhere. Even if the file system is deniable in the mathematical sense, the encironment surrounding that file system can undermine its deniability as well as its contents.

## Terms and Definitions

* **Deniable File System (DFS)** - a file system where the existence of a portion of the file system can be hidden from view. This is different from an encrypted file system, where files and directories are visible yet unintelligible. In a DFS, the very existence of certain files and directories cannot be ascertained by the attacker. 
* **hidden volumes** - TrueCrypt creates DFSes called hidden volumes, which live inside of the normal regular file systems and are protected by a password. Unless Alice reveals this password, its impossible for the adversary to determine whether Alice's computer contains a hidden volume or not. The adversary should not be able to determine if the random data at the end of an encrypted volume is really random data or an encrypted volume.

## Model

We assume that the adversary has one-time access to the computer, meaning a single snapshot of the disk image. We want it to be impossible for the adversary to even *detect* that there is a hidden volume on the disk.

Note that even the TrueCrypt documentation allows that *wear leveling* (some property of data is stored on USB sticks) can reveal information about the existence of a hidden volume on a USB. These are recommendations about the media *underlying* TrueCrypt volumes, whereas here we focus on usages of TrueCrypt volumes at a higher level.

All of the following examples share the following traits: the information is leaked out about the hidden volumes and the files contained therein after the hidden volume is mounted, and the information is not securely destroyed after the hidden volume is unmounted.

## Through the OS

* The Windows registry reveals that TrueCrypt was used, but not that it was used for hidden volumes.
* Windows does create shortcuts for files as they are used (Recent Items). These shortcuts contain the real file's name, location, attributes, etc.
* If the adversary forces Alice to mount and identify all volumes on her machine and he sees that one of these shortcuts contains a volume missing, the adversary can be confident a hidden volume is used.
* The problem is that these volume unique identifiers are stored in the shortcut files. 


## Through Primary Applications

* Microsoft word creates auto-recovery files *locally* in case of crashes. 
* Using a simple disk recovery utility, we can recover pieces of files that were copied into these caches.
* The files in the hidden volume are no longer hidden.

## Through Non-Primary Applications

* Google Desktop caches snapshots of files and stores them for later viewing as users open these files. If a user opens a file from a hidden volume, Google Desktop will cache it. 
* To defend against this, Google Desktop must be shut down while reading files from the hidden volume, or Google Desktop will index these files and cache them locally.

## Solutions

* TrueCrypt could use the same serial number for all volumes, or random serial numbers each time.
* A file system filter that disallows a process any write access to a non-hidden volume once that process reads information from a hidden volume. This would break many applications, but that may be okay. This also doesn't stop leakage over the network, etc.
* A TrueCrypt bootloader could boot into two modes: deniable and non-deniable, keeping these file systems isolated.

Note that many of these problems apply to encrypted file systems as well, since applications cache unencrypted versions of these files.
