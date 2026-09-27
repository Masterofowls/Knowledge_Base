# How Encryption Works

> **Level:** Advanced · **Related:** [SSL/TLS](ssl-tls.md) · [Web Protocols](../05-networking-and-web/web-protocols.md) · [SSH](../05-networking-and-web/ssh.md) · [VPN](../05-networking-and-web/vpn.md) · [Payment Terminals](payment-terminal.md) · [SIM/eSIM](../04-wireless-and-telecom/sim-esim.md) · [Compression](compression.md) · [Malware](malware.md)

## 1. Goals and vocabulary

Cryptography provides:

| Goal | Meaning | Tools |
|---|---|---|
| **Confidentiality** | Only intended parties can read | Ciphers (AES, ChaCha20) |
| **Integrity** | Tampering is detected | MACs (HMAC), AEAD tags, hashes |
| **Authentication** | You know who sent it | MACs, digital signatures, certificates |
| **Non-repudiation** | Sender can't deny | Digital signatures |

- **Plaintext** → (encrypt with **key**) → **ciphertext** → (decrypt) → plaintext.
- **Kerckhoffs's principle**: security must depend only on the secrecy of the key, never the algorithm.
- **Security level** in bits: "128-bit security" means ~2¹²⁸ operations to break — infeasible.

## 2. Symmetric encryption (one shared key)

### 2.1 Block ciphers: AES
**AES** (Rijndael, 2001) encrypts 128-bit blocks with 128/192/256-bit keys over 10/12/14 **rounds**. Each round on the 4×4 byte state:

1. **SubBytes** — non-linear S-box substitution (confusion)
2. **ShiftRows** — rotate rows (diffusion)
3. **MixColumns** — matrix multiplication over GF(2⁸) (diffusion)
4. **AddRoundKey** — XOR with a round key derived by the key schedule

Modern CPUs have **AES-NI** (x86) / ARMv8 Crypto Extensions: several GB/s per core.

### 2.2 Modes of operation
A block cipher alone encrypts one block; **modes** handle messages:

| Mode | Notes |
|---|---|
| ECB | Each block independently — **insecure** (identical blocks → identical ciphertext; the "ECB penguin") |
| CBC | XOR with previous ciphertext block; needs random IV; padding-oracle pitfalls |
| CTR | Encrypt a counter → keystream → XOR; parallelizable; never reuse nonce |
| **GCM** | CTR + GHASH authentication → **AEAD** (authenticated encryption) — TLS, IPsec, disk |
| XTS | Tweakable mode for **disk encryption** (BitLocker, LUKS, FileVault) |

### 2.3 Stream ciphers: ChaCha20
ChaCha20 generates a keystream from (key, nonce, counter) using add-rotate-xor (ARX) operations — fast in software without AES hardware (phones, older CPUs). **ChaCha20-Poly1305** is the AEAD used in TLS 1.3, WireGuard, SSH.

### 2.4 AEAD: always use authenticated encryption
AEAD = encrypt + authenticate in one primitive; decryption **fails** if a single bit of ciphertext, nonce, or associated data changed.

```python
# pip install cryptography
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os
key = AESGCM.generate_key(bit_length=256)
aes = AESGCM(key)
nonce = os.urandom(12)                         # 96-bit, MUST be unique per key
ct = aes.encrypt(nonce, b"transfer $100 to Bob", b"header-v1")   # AAD authenticated, not encrypted
print(aes.decrypt(nonce, ct, b"header-v1"))    # b'transfer $100 to Bob'
tampered = bytearray(ct); tampered[0] ^= 1
try: aes.decrypt(nonce, bytes(tampered), b"header-v1")
except Exception as e: print("rejected:", type(e).__name__)   # InvalidTag
```

JavaScript (Web Crypto API — browsers and Node.js):

```js
const key = await crypto.subtle.generateKey({ name: "AES-GCM", length: 256 }, true, ["encrypt", "decrypt"]);
const iv = crypto.getRandomValues(new Uint8Array(12));
const ct = await crypto.subtle.encrypt({ name: "AES-GCM", iv }, key, new TextEncoder().encode("secret"));
const pt = await crypto.subtle.decrypt({ name: "AES-GCM", iv }, key, ct);
console.log(new TextDecoder().decode(pt));
```

C++ with libsodium (high-level, misuse-resistant):

```cpp
#include <sodium.h>
#include <cstring>
#include <cstdio>
int main() {
    if (sodium_init() < 0) return 1;
    unsigned char key[crypto_aead_xchacha20poly1305_ietf_KEYBYTES];
    unsigned char nonce[crypto_aead_xchacha20poly1305_ietf_NPUBBYTES];   // 192-bit: random is safe
    crypto_aead_xchacha20poly1305_ietf_keygen(key);
    randombytes_buf(nonce, sizeof nonce);
    const unsigned char msg[] = "hello";
    unsigned char ct[sizeof msg + crypto_aead_xchacha20poly1305_ietf_ABYTES];
    unsigned long long ct_len;
    crypto_aead_xchacha20poly1305_ietf_encrypt(ct, &ct_len, msg, sizeof msg, nullptr, 0, nullptr, nonce, key);
    unsigned char out[sizeof msg]; unsigned long long out_len;
    if (crypto_aead_xchacha20poly1305_ietf_decrypt(out, &out_len, nullptr, ct, ct_len, nullptr, 0, nonce, key) == 0)
        printf("%s\n", out);
}
```

**Nonce reuse** with GCM/CTR/ChaCha20 is catastrophic (reveals XOR of plaintexts; with GCM also enables forgeries). Use random 96-bit nonces with a bounded message count per key, counters, or XChaCha20 (192-bit random nonces).

## 3. Hash functions

A cryptographic hash maps any input to a fixed-size digest, and must be **preimage-resistant**, **second-preimage-resistant** and **collision-resistant**.

| Hash | Output | Status |
|---|---|---|
| MD5 | 128 | **Broken** (collisions in seconds) |
| SHA-1 | 160 | **Broken** for collisions (SHAttered 2017) |
| **SHA-256 / SHA-512** (SHA-2) | 256/512 | Secure, ubiquitous |
| **SHA-3** (Keccak), SHAKE | variable | Secure, sponge construction |
| **BLAKE2 / BLAKE3** | variable | Secure, very fast |

**HMAC** (`HMAC(K, m) = H((K⊕opad) ‖ H((K⊕ipad) ‖ m))`) turns a hash into a MAC.

**Passwords are special** — don't store fast hashes. Use slow, salted, memory-hard **KDFs**: **Argon2id** (recommended), scrypt, bcrypt, PBKDF2 (FIPS).

```python
import hashlib, hmac, os
salt = os.urandom(16)
dk = hashlib.scrypt(b"correct horse battery staple", salt=salt, n=2**14, r=8, p=1, dklen=32)
tag = hmac.new(b"server-key", b"message", hashlib.sha256).hexdigest()
```

## 4. Asymmetric (public-key) cryptography

Solves key distribution: publish a **public key**; keep the **private key**.

### 4.1 RSA
- Pick primes p, q; n = p·q; φ = (p−1)(q−1); public exponent e = 65537; private d = e⁻¹ mod φ.
- Encrypt: `c = mᵉ mod n`; decrypt: `m = cᵈ mod n`; sign: `s = H(m)ᵈ mod n`.
- Security relies on the difficulty of **factoring** n. 2048-bit ≈ 112-bit security; 3072-bit ≈ 128.
- Must use padding: **OAEP** for encryption, **PSS** for signatures (textbook RSA is insecure).

Toy RSA in Python (educational only):

```python
p, q = 61, 53
n, phi = p * q, (p - 1) * (q - 1)       # n = 3233
e = 17
d = pow(e, -1, phi)                      # modular inverse → 2753
m = 65
c = pow(m, e, n)                         # encrypt → 2790
assert pow(c, d, n) == m                 # decrypt
```

### 4.2 Elliptic Curve Cryptography (ECC)
Points on a curve over a finite field form a group; given `Q = k·G`, finding `k` is the **elliptic-curve discrete log problem**. 256-bit ECC ≈ 128-bit security (vs 3072-bit RSA) → small keys, fast.
- **ECDH / X25519** — key agreement.
- **ECDSA (P-256)** and **Ed25519** — signatures.

### 4.3 Diffie–Hellman key exchange
Two parties derive a shared secret over a public channel:

```
Alice: secret a, sends A = a·G        Bob: secret b, sends B = b·G
Shared: a·B = b·A = ab·G   (eavesdropper sees A, B but can't compute abG)
```

**Ephemeral** DH per session gives **forward secrecy**: stealing a server's long-term key later doesn't decrypt past sessions.

```python
from cryptography.hazmat.primitives.asymmetric.x25519 import X25519PrivateKey
from cryptography.hazmat.primitives.kdf.hkdf import HKDF
from cryptography.hazmat.primitives import hashes
a, b = X25519PrivateKey.generate(), X25519PrivateKey.generate()
shared_a = a.exchange(b.public_key()); shared_b = b.exchange(a.public_key())
assert shared_a == shared_b
session_key = HKDF(hashes.SHA256(), 32, salt=None, info=b"demo v1").derive(shared_a)
```

### 4.4 Hybrid encryption
Public-key operations are slow and size-limited, so real systems use them only to **agree on/encrypt a symmetric key**, then use AEAD for data — TLS, PGP, age, Signal, HPKE (RFC 9180).

## 5. Trust: certificates and PKI

How do you know a public key really belongs to `bank.com`? A **Certificate Authority** signs an **X.509 certificate** binding name ↔ public key. Browsers/OSes ship trusted root CAs.

```
Root CA (in OS/browser trust store)
 └─ signs Intermediate CA certificate
     └─ signs bank.com leaf certificate (SAN: bank.com, validity, public key)
```

Verification: signature chain, validity dates, hostname match, revocation (OCSP/CRL), **Certificate Transparency** logs (public append-only logs; browsers require SCTs).

| Task | Windows | Linux |
|---|---|---|
| Trust store | `certlm.msc` / `certmgr.msc`; `Cert:\LocalMachine\Root` in PowerShell | `/etc/ssl/certs`, `update-ca-certificates` (Debian) / `update-ca-trust` (Fedora) |
| Inspect cert | `certutil -dump file.cer` | `openssl x509 -in cert.pem -text -noout` |
| Crypto libraries | **CNG** (bcrypt.dll/ncrypt.dll), SChannel (TLS) | OpenSSL, GnuTLS, NSS, kernel crypto API (`/proc/crypto`) |
| Key protection | DPAPI, TPM via Platform Crypto Provider, Windows Hello | Kernel keyrings, TPM2 (`tpm2-tools`), GNOME Keyring/KWallet |

## 6. Encryption in practice

| Where | Mechanism |
|---|---|
| Web (HTTPS) | TLS 1.3: ECDHE (X25519 / hybrid ML-KEM) + AES-GCM/ChaCha20-Poly1305 + ECDSA/RSA-PSS certs — see [Web Protocols](../05-networking-and-web/web-protocols.md#6-tls-13-encryption-and-identity) |
| VPN | WireGuard (Curve25519, ChaCha20-Poly1305), IPsec — see [VPN](../05-networking-and-web/vpn.md) |
| Messaging E2EE | Signal Protocol: X3DH/PQXDH + **Double Ratchet** (new keys per message) |
| Disk | **BitLocker** (AES-XTS, key sealed in TPM; `manage-bde -status`), **LUKS2/dm-crypt** (`cryptsetup luksFormat`, `cryptsetup status`), fscrypt |
| Files | age, GPG, 7-Zip/ZIP AES-256 (see [Compression](compression.md)) |
| Wi-Fi | WPA3-SAE + AES-CCMP/GCMP — see [Wi-Fi](../04-wireless-and-telecom/wifi.md) |
| Mobile | [SIM](../04-wireless-and-telecom/sim-esim.md) AKA (MILENAGE/AES), 5G NEA/NIA |
| Payments | EMV cryptograms, DUKPT, HSMs — see [Payment Terminals](payment-terminal.md) |
| Passwords | Argon2id/bcrypt hashing; passkeys (WebAuthn, ECDSA P-256 per site) |

## 7. How encryption fails (it's rarely the math)

1. **Bad randomness** (predictable keys/nonces) — use OS CSPRNG: `getrandom()`/`/dev/urandom` (Linux), `BCryptGenRandom` (Windows), `secrets`/`os.urandom` (Python), `crypto.getRandomValues` (JS). Never `rand()` / `Math.random()`.
2. **Nonce reuse** in GCM/CTR.
3. **Rolling your own crypto** or composing primitives incorrectly (encrypt-then-MAC vs MAC-then-encrypt, padding oracles).
4. **Side channels**: timing (compare MACs with constant-time functions: `hmac.compare_digest`, `crypto.timingSafeEqual`, `sodium_memcmp`), power/EM analysis on smart cards, cache attacks.
5. **Key management**: keys hard-coded in source, leaked in logs, not rotated. Use HSMs/KMS (AWS KMS, Azure Key Vault), TPMs, secret managers.
6. **Deprecated algorithms**: DES/3DES, RC4, MD5, SHA-1, RSA-1024, PKCS#1 v1.5 encryption.

## 8. The post-quantum transition

A large quantum computer running **Shor's algorithm** would break RSA, DH and ECC. **Grover's algorithm** only halves symmetric security (AES-256 remains fine).

NIST standards (August 2024):
- **ML-KEM** (FIPS 203, from CRYSTALS-Kyber) — key encapsulation (lattice-based).
- **ML-DSA** (FIPS 204, Dilithium) and **SLH-DSA** (FIPS 205, SPHINCS+) — signatures. (FN-DSA/Falcon to follow.)

Deployment is **hybrid** (classical + PQ, secure if either holds): Chrome, Firefox, Cloudflare and Windows/SChannel previews use **X25519MLKEM768** in TLS; OpenSSH defaults to hybrid `mlkem768x25519-sha256` (OpenSSH 10); Signal uses PQXDH and SPQR. Motivation: "**harvest now, decrypt later**" — recorded traffic could be decrypted in the future.

## Further reading
- Jean-Philippe Aumasson, *Serious Cryptography* (2nd ed.)
- Dan Boneh & Victor Shoup, *A Graduate Course in Applied Cryptography* (free)
- Cryptopals challenges (cryptopals.com) — learn by breaking
- NIST FIPS 197 (AES), 203–205 (PQC); RFC 8439 (ChaCha20-Poly1305), RFC 7748 (X25519)
