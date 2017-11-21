# CryptDB: Protecting Confidentiality with Encrypted Query Processing

## Motvation

Databases represent a security risk for applications. Often the data is stored in plaintext, which the compromise of the database server leaks all customer information. Database administrators also have full access to plaintext data. Ideally the data is stored encrypted when not in use such that a compromised DBMS does not allow the attacker to view *any* information. CryptDB takes this a step further: if all of the servers are compromised, the attacker cannot view the information of users who are not logged in.

CryptDB aims to provide application developers with a way to perform certain types of queries on encrypted data by providing modifications to database columns and user keys.

## Terms and Definitions
* **DBMS** - Database Management Server, the server which runs the actual database software like MySQL or PostgreSQL
* **UDF** - User Defined Function, a query defined by the user which will be executed multiple times, presumably.
* **Principal** - an entity with its own key that has ownership over some subset of data

## Design

[[img/cryptdb.png]]

CryptDB consists of the normal, unmodified DBMS. This server's data is now almost entirely encrypted. CryptDB UDFs and the encrypted key table are stored here.

CryptDB adds a database proxy, which contains the active keys and an annotated schema for the DBMS.

### Threat Model
CryptDB ensures data confidentiality. It does not ensure integrity, freshness, or completeness of results.

Threats:
1. DBMS Server Compromise: the attacker has full access to the data stored on the DBMS server. Attacker is passive: only views queries and data.
  * CryptDB guarantees confidentiality for content, column names, and  table names. Cannot hide number of rows, types of columns, approx. size in bytes. CryptDB also reveals *classes of computation* that can be performed on various columns due to its onion encryption nature. If the application requests order checks, the proxy reveals the order of the elements in the column. 

2. Arbitrary Threats: attacker has full access to every part of the application. The solution here is to encrypt different data items with different keys. In order to match keys to data, developers annotate the application's database schema to express finer-grained confidentiality policies. Since the proxy server holds keys in memory, any data these keys are able to decrypt are available to the attacker. Presumably when a user is logged in their key is in memory, so if a user isn't logged in during the *duration of the compromise*, their data is safe. 

### Queries

Processing a query looks like this:
1. Application issues query. Proxy anonymizes each table, column name using secret key `MK`. Encrypts each constant in query with encryption scheme best suited for operation (see later). **Note that this means in threat 2 the table/column names are visible once queries are received.**
1. Proxy checks to see if encryption layers on affected columns need to be updated. If so, proxy issues an update to decrypt these columns. 
1. Proxy invokes encrypted query (the data is encrypted) on DBMS server, occasionally invoking UDFs.
1. DBMS server returns encrypted query result, which proxy decrypts and returns the application. **If the attacker can see queries and results, and these are in the clear, this thteat model is kind of weak...**

If a table looks like this:

```
ID | NAME
----------
23 | Alice
```
CryptDB stores this (at the DBMS):
```
C1-IV | C1-Eq | C1-Ord | C1-Add || C2-IV | C2-Eq | C2-Ord | C2-Search
--------------------------------||-----------------------------------
azafs | 82jr8 | 9j2s0z | 02ksnz || owj23 | jzma1 | uwies9 | mzhfhees
----------
```

Where for `C1-x`, `x` represents the operations allowed at some of the layers for each onion. There are four onion types.

1. Onion Eq: `RND(DET(JOIN(value)))`
1. Onion Ord: `RND(OPE(OPE-JOIN(value)))`
1. Onion Search: `SEARCH(value)`
1. Onion Add: `HOM-add(int)`

We need multiple onion "types" because different schemes are not always strictly ordered. For example, the search onion does not make sense for integers, and not all fields are orderable (like strings). 

So `C2-Ord` in the above example will be an onion and will be decrypted for a given key *as needed*. For each layer of each onion, the proxy uses the same key for encrypting values in the same column, and different keys across tables, columns, onions, and onion layers. All keys are derived from master key *MK* like so:

```
K[t, c, o, l] = PRP[MK](table t, column c, onion o, layer l)
```
Where *PRP* is a psuedorandom permutation like AES.

To decrypt `C1-Ord`:
```
UPDATE Table1 SET C2-Ord = DECRYPT_RND(K, C2-Ord, C2-IV)
```
You can see we use the IV stored in the row.

Where these encryption methods/techniques are as follows:

1. Random (RND) : two equal values will (probably) have different ciphertexts, but no computation can be performed on the ciphertext
1. Deterministic (DET) : deterministically generates same ciphertext for given plaintext. Allows equality checks.
1. Order-preserving Encryption (OPE) : If *x < y*, then *OPE[k](x) < OPE[k](y)*. Allows `ORDER BY`, `MIN`, `MAX`, `SORT`, etc.
1. Homomorphic Encryption (HOM) : Allows server to perform computations on encrypted data with the final result decrypted at the proxy. It's slow. *HOM[k](x) + HOM[k](y) = HOM[k](x + y)*.
1. Join (JOIN & OPE-JOIN) : since we use different keys for DET to prevent cross-column correlations, a separate encryption scheme is needed for joins.  **This is done essentially like so. When a user performs a join, the proxy adds a JOIN-ADJ layer to one of the columns using the key from the other column. The two columns now share a deterministic JOIN-ADJ layer that can be used for equality.**
1. SEARCH : Allows server to search for data using the `LIKE` operator and ciphertext. 

### Principals
Extending to the 2nd threat model...

Assume each user has an account and a password, and belongs to certain groups. 

The developer defines

* **principal types** - user, group, etc
* **principal** - instance of a principal type, like principal 5 of type user

*External* principals correspond to end users who explicitl authenticate themselves to the application using a password. *Internal* principals are internal, database-only entities that obtain privileges through the delegation from external principals. Each principal is associated with a secret, randomly chosen key. If principal B speaks for principal A, A's key is encrypted using B's key and stored in `access_keys` table in the database. This allows principal B to decrypt principal A. 

For example, if physical user speaks for a group, the group's key is encrypted via the physical user. **This is desirable since all columns must be encrypted separately for each key that can access the columns.** CryptDB stores the user's randomly generated key encrypted by the user's password.

The developer specifies which columns in the SQL schema contain sensitive data, as well as principals that should be allows to access this data. 

The developer also specifies rules for how to delegate privileges of one principal to other principals. 


# Why Your Encrypted Database Is Not Secure


