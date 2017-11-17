# AES-CBC (Cipher Block Chaining)

Encrypted value of a block depends on encrypted value of preceding block,

```
Ci = Ek(Pi xor C[i-1])
C0 = IV // initialization vector
Pi= Dk(Ci) xor C[i-1]
C0 = IV
```
