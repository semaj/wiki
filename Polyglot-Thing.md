# Polyglot: Automatic Extraction of Protocol Message Format Using Dynamic Binary Analysis

## Motivation
Protocol reverse engineering is the process of extracting the application-level protocol used by an implementation **without access to the protocol specification**.  
Many protocols in use are closed source and it is thus to analyze and reason about them for applications such as fingerprint generation, intrusion detection, and vulnerability detection. For open source protocols, often implementations do not adhere correctly to the specification.  
We'd like an automatic system for performing this verification, since protocol specifications and their implementations change often. Polyglot extracts the following information using Panorama. 

## Terms and Definitions
  * **Shadowing** This paper proposes a **dynamic binary analysis** approach to protocol reverse engineering. Rather than just syntactic information, program binaries contain *semantic* information about how the program operates on protocol data. The dynamic analysis tool "shadows" the program (closely monitors it as it processes input).  
  * **Direction fields:** Fields used to mark the boundary of variable-length fields. This includes length fields (how long the following field is) and pointer fields (where this field starts relative to some other position) and counter fields (the position of an element in a list of items).
  * **Separator:** 1) A constant value 2) A scope, which contains a list of position sequences in the application data where the separator is used. Since a separator could be used to separate fields themselves or values within fields...
  * **Protocol Hierarchy:** Protocols are composed of *sessions* which are composed of a sequence of *messages* which are composed of a sequence of *fields*. 
      \includegraphics[scale=0.5]{polyglot-field-formats}
  * **Format attributes:** Field Length indicates whether it's Fixed or Variable, and the corresponding size. Field Boundary indicates the type and the value (Fixed means a constant boundary). Field keywords are words that (I think) appear in either the key or the value of the field, especially since not all fields are key-value pairs. 
  * **Floating field:** A field that may be ordered differently depending on the specific message.
  * **True comparison:** An equality comparison rather than something fancy like a subtraction or XOR comparison.

## Overview

### Goals
The proposed system uses dynamic binary analysis to extract only the **protocol message format**. The system does not require source code *or debugging information about the binary*. Given a number of messages received by a program binary implementation of a protocol, to individually extract the message format of each of those messages. **Taint information can include the offsets in the memory buffer referred to by the register / data, and also the type of data referred to (field value, direction field).**

### Challenges
The main problem is finding field boundaries in the message, since messages can have fixed and variable length fields. How do we find the correct length for a fixed-length field, and how do we dynamically find direction fields / separators for variable-length fields? How do we find the keywords in each field? If the field is floating, we must analyze multiple messages. But usually we can just analyze one. 

### Solution
\includegraphics[width=\textwidth]{polyglot-system}
The *execution monitor* takes the binary and data as input and produces a trace of all instructions performed by program. It uses **dynamic taint analysis** to taint data received from the network. For example if a string of characters represents a GET request and the method (GET) is moved to EAX, EAX is tainted with 0-3 (the positions in the string). 
% 

## Contributions
* **Extracting PMF using binaries**
* **Detecting direction fields**
Polyglot detects direction fields without assuming encodings.
This trace is used to find direction fields. We know a direction field was used when 1) the program computes the value of the pointer increment from a previous field in the buffer in memory or 2) the program increments the pointer pointing to the buffer by one until it reaches the end of the field (bounded by some previous value, the direction field). 1) A tainted memory destination address is accessed using arithmetic which used tainted data. Mark all consecutive positions used to compute destination as length field. 2) Search loop stop conditions for references to tainted data (such as an integer pulled from a direction field). If so, taint the loop and every time the program uses a new position we check if the closest loop was tainted. If so, flag the direction field.
* **Detecting separators**
Polyglot does not assume anything about the separator values (like assuming it's something common like tab or whitespace). The separator must be compared against all bytes in its scope. A message separator would be compared against all bytes received in a stream. Same for a field separator. An in-field separator would be compared against bytes in the field (scope). We assume we know the message boundaries already.
1. Extract a summary of all of the program's comparisons. Tokens-at-position hash: for each buffer position, the ordered list of tokens that was compared. Tokens-series hash: for each token, all of the positions the token was compared to. (created by including all comparisons that included a tainted byte)
1. Any comparison between tainted and untainted byte may denote a separator. For each token, extract list of consecutive buffer positions it was compared with. Min length 3 and it must appear at least once (to avoid obfuscation). Output: list of byte-long separators with positions used. 
1. For each byte-long separator, we check the previous and next bytes. If they're always the same, and there's a comparison done on those bytes as well, we extend the separator. Don't extend beyond four because intuition.
Note the binary might compare against space and tab (for HTTP this are both valid) but maybe only spaces are used. So the tool doesn't know that tabs are also valid. This is tweakable.

* **Finding multi-byte fixed-length fields**
Polyglot finds the correct length for multi-byte fixed-length fields. Consider each byte received from the network as independent. For each instruction that deals with tainted data, extract a list of positions from which the tainted data came from. Check if these bytes are a direction field. If not, create a fixed field. If later we find a sequence of consecutive tainted positions that overlaps with a  previously defined field, extend the previously defined field to encompass the newly found bytes. 
* **Extracting protocol keywords**
Polyglot does not require the comparison of multiple messages: it can extract keywords given one message. Extracts keywords that are present within the given exchange and supported by the implementation. Create the same tables mentioned in the separators section. Explore each position in the tokens-at-positions hash. For each position, if there's a true comparison made between the character at this position and another character, concatenate the non-tainted token to the current keyword. If no true comparison was performed, store the current keyword and start a new one at that position. Also break if we find a separator. 

## Evaluation

## Questions and Issues
  * What happens when the protocol uses TLS or some other encryption mechanism?
  * Could you perhaps apply Machine Learning to this to improve it?
  * 4.1:  In addition, we mark the smallest position in the destination address as the end of target field. For example, in Figure 3 if the instruction is accessing positions 18-20, and the address of the smallest po- sition (i.e., 18) was calculated using taint data coming from positions 12-13, then we mark position 12 as the start of a direction field with length 2, and position 18 as the end of the target field. >> If the direction field is length 10 (0-3) and 3 + 10 =13, then 4-12 indicate the value that the direction field is pointing to. Pointer + 10 will be used to "skip" past the last value. If 10 includes a tainted value and a constant, the program might be skipping a fixed length field and that will (unknowingly) be included in the variable length field picked up by the analyzer.
  * They don't really discuss how they handle scopes in separator discovery.

# Decompilers

## Terms and Definitions
* **Dynamic Linking** - when examining a binary, the symbol is unmapped. The binary expects to know where a symbol maps once the loader "links" a library at runtime. This means the library must be loaded into memory by the loader, and can be compiled independently of the target application.
* **Decompiler** - takes machine binary code and tries to translate it into high-level source code
* **ptrace** - Allows one process to pause, resume, and inspect another process. `ptrace` can pause child before syscall, pause child after single instruction during single-step mode. 
* **Debugging symbols** - metadata in a binary which refers to the higher-level source. DWARF format for ELF binaries. Can increase the size of binaries *significantly*.

## Overview
The raw binary application (when not compiled with debugging symbols, usually) is fed into the following process:
1. Disassembly : dumping the assembly instructions
1. Idiom translation : Find sequences of instructions that don't make sense at first, but used by compilers to performed well-known actions. 
  * For example, `xor %eax %eax` is often used to clear eax, and its shorter (bytes-wise) than a `mov`.
1. Type extraction : Recover high level types. For instance, we know you don't perform `and`s on floats.
1. Dataflow analysis : Constructs "basic blocks", which are sequences of instructions without jumps. Tries to translate them to higher-level code. 
1. Control flow analysis : Analyze branches to try to reconstruct `if else` statements. Backwards unconditional jumps usually evidence of a loop.

