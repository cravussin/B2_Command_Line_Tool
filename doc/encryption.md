# Encryption of Files in B2

This document outlines the goals and the design of encryption of files
in B2, along with the reasons for making various choices

## Goals

### Goals

- In command-line tool, store passphrase in `.b2_account_info`, so the user does not have to entry it each time.
- Encrypt file names and metadata (file info).
- Support for the sync command for encrypted buckets.
- B2 service verifies SHA1 checksum of encrypted data on upload.
- Support for changing passphrase without re-uploading everything.
- Checking if a file with a certain name is in the bucket should be efficient
- "file info" metadata must be encrypted
- Max file name length should not be decreased

### Non-Goals

These are things that we have explicitly decided not to do, at least
for now:

- Per-file encryption status.  For now, the encryption settings are on the bucket.
- SHA1 checksum of original data stored with file, so that original content can be verified after downloading.
 - File authenticity is already guaranteed by the used encryption mode.

### Simplifying Assumptions

???

## Metadata and File Format

Encryption is specified per bucket.  Each bucket has its own master key.

The master key for a bucket is stored in a file called `.MASTER_KEY` in the
bucket.  All encryption steps use 128-bit AES, so every key is 128 bits, or 16
bytes.  The master key is randomly generated when the bucket is created.

The key to decrypt the master key is derived using
[PBKDF2](https://en.wikipedia.org/wiki/PBKDF2) with SHA256 as the hash function
and 500000 Iterations.  The high iteration count makes bruteforcing weak
passphrases much more time consuming.

The Python code to create a key looks like this, using the
cryptography library:

    kdf = PBKDF2HMAC(
        algorithm=hashes.SHA256(),
        length=16,
        salt=salt,
        iterations=500000,
        backend=default_backend()
    )
    file_key = kdf.derive(passphrase)



### Per-file Key Generation

A different key is used to encrypt each file.  File encryption keys are derived
from the master key using PBKDF2 again, but with a single iteration and the
master key as the passphrase. A single iteration is already enough, since the
master key is random and therefore not susceptible to a bruteforce attacks.

Note: While this is not an intended use case for PBKDF2, it's still secure.
PBKDF2 with a single iteration and requested key sizes smaller than block size
of the underlying hash function is almost identical to the expand phase of
[HKDF](https://tools.ietf.org/html/rfc5869). We choose PBKDF2 to keep the number
of required key derivation functions small.

### Per-file Initialization Vector

Each file gets its own 12-byte initialization vector, which is
randomly generated, and then stored in the header on the file.

### Encrypted File Format

Each encrypted file is prefixed with a header before being stored.
The header contains:

- The 128-bit salt used to generate the key for the file.
- The number of sections (blocks) in the file.
- The initialization vector for the first section.

### File Encryption

Files are encrypted in sections, each one NNN bytes long. Every section is
encrypted using the [Galois/Counter Mode
(GCM)](https://en.wikipedia.org/wiki/Galois/Counter_Mode), which also
authenticates the encrypted data. Additionally the total number of sections is
also authenticated. After every section the 128-bit authentication tag is
appended. The initialization vectors for each section are just ascending
numbers, starting with the value given in the header.
