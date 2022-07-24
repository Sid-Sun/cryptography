# Documentation of Cryptography used in [ioctl](https://github.com/Sid-Sun/ioctl) - an E2E Encrypted Pastebin alternative

## The problem:
ioctl provides a pastebin service which stores snippets in URLs like: `https://api.ioctl.ml/r/TrophySandbag` where `TrophySandbag` is an ID created using diceware against the [EFF Large Wordlist](https://www.eff.org/files/2016/07/18/eff_large_wordlist.txt).

Having one id is usually sufficient for fetching data in a system without any encryption or encryption at store / rest; either way it boils down to the same problem: the system can see the user's data. 

To add encryption such that the system can't see the data, it would be ideal to use a different randomly generated key; however, for a pastebin which provides a memorable ID, providing another ID (memorable or not) would be inconvenient.

This specification provides this exact capability: to use one ID for both identifying and encrypting user data such that the system cannot access it without having the ID, as well as decoupling the ID from the system so it does not know which ones have been used, while still being able to provide the required service.

## Specification

### Components

- AES-256: Advanced Encryption Standard with 256 bit key length // our cipher
- GCM (Galois Counter Mode): Symmetric Cipher mode that provides AEAD capabilities (used with AES-256)
- ARGON2 (`id` function): Key Derivation Function which provides parameters for parallelism, memory and rounds and internally uses Blake2b hashing function to provide key derivation with resistance to GPU cracking attacks

Some fundamentals:
- Ciphers simply use raw bytes for plaintext / ciphertext and the key to use for operation (encyption or decryption)
- Hashing is the problem of taking arbitrary data as input and generating a fixed length output such that:
  - the output is deterministic i.e. running the hash with the same data always gives the same output
- Key Derivation is essentially the problem of using hashing such that:
  - the original data cannot be derived from the output
  - the output is essentially arbitrary i.e. there is no pattern relating the output to the input
  - it is untrivial to brute force a key; owing to time constraints
- Key Derivation functions / Hashing functions also support a concept of salts to increase security:
  - since hashing / key derivation functions are deterministic, an attacker could make a list of known input and output mapping and use that to boost their attacks
  - using a salt can prevent this: we append a known value to input everytime we hash which now means our hash for X -> Y is totally different from someone else as we are doing X + Z as X and now Y depends on X and Z

#### Why ARGON2 instead of PBKDF2?

ARGON2 allows you to control several parameters which is useful in preventing GPU bruteforce attacks compared to PBKDF2 

### The trick:

To solve the problem of using one ID to do both identification and cryptography without the system having knowledge about the user's data, we use the ID with ARGON2 and derive two keys, let's say; X and Y.

X key is used for identification and Y is used with cipher. Additionally, we generate a unique salt every time we generate Y; this salt can be stored with the data in plaintext. 

The finer print:
- When we generate X, we use a common salt, else we would have, in effect, created two IDs for user.
- Additionally, we also use a different configuration of ARGON2 for ID and Key

### Run Down of the entire encryption / decryption operation:
***Note:*** To prevent confusion over our diceware ID and argon2's ID mode we denote diceware ID as `ID`

#### Encryption:
1. Generate an `ID` using diceware
2. Do: ARGON2ID(`ID`, ARGON2ConfigOne, CommonSalt) -> `uuid`
3. Generate a random salt of 32 bytes called `keySalt`
4. Do: ARGON2ID(`ID`, ARGON2ConfigTwo, `keySalt`) -> `key`
3. Generate a random salt of 32 bytes called `nonce`
5. Do: AES-256-GCM(`plaintext`, `key`, `nonce`, EncryptMode) -> `ciphertext`
6. Save `nonce`, `keySalt`, `ciphertext` against `uuid`

#### Decryption:
1. Fetch user-provided `ID`
2. Do: ARGON2ID(`ID`, ARGON2ConfigOne, CommonSalt) -> `uuid`
3. Fetch file for `uuid` from server and extract `nonce`, `keySalt`, `ciphertext`
4. Do: ARGON2ID(`ID`, ARGON2ConfigTwo, `keySalt`) -> `key`
5. Do: AES-256-GCM(`ciphertext`, `key`, `nonce`, DecryptMode) -> `plaintext`

