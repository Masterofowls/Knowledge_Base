# How Compression (Archives) Works

> **Level:** Advanced · **Related:** [Encryption](encryption.md) · [Web Protocols](../05-networking-and-web/web-protocols.md) · [WebRTC](../05-networking-and-web/webrtc.md) · [Code Compilation](../03-os-and-software/code-compilation.md)

## 1. Why compression is possible

Real data is **redundant**: repeated substrings, skewed symbol frequencies, predictable structure. Compression removes redundancy by **modeling** the data (predicting what comes next) and **coding** it (spending few bits on likely symbols, many on unlikely ones).

- **Lossless** (archives, executables, text, PNG): exact reconstruction. Every archive format — ZIP, 7z, RAR, tar.gz — is lossless.
- **Lossy** (JPEG, MP3, AV1, Opus): discards perceptually unimportant information.

### The theoretical limit: entropy
Shannon entropy of a source with symbol probabilities `pᵢ`:

```
H = −Σ pᵢ · log2(pᵢ)     bits per symbol
```

No lossless code can average fewer bits per symbol than H (for that model). English text ≈ 1–1.5 bits/char with a good context model vs 8 bits stored.

```python
import math
from collections import Counter
def entropy(data: bytes) -> float:
    n = len(data); return -sum(c/n * math.log2(c/n) for c in Counter(data).values())
print(entropy(b"aaaaaaab"))        # ~0.54 bits/byte — highly compressible
print(entropy(bytes(range(256))))  # 8.0 — uniform: incompressible
```

**Pigeonhole principle**: no lossless compressor can shrink *all* inputs — some must grow. Already-compressed or encrypted data (≈ random) doesn't compress. That's why you **compress before encrypting**, never after.

## 2. The two building blocks

Nearly every general-purpose compressor is **dictionary/match modeling (LZ77 family) + entropy coding**.

### 2.1 LZ77: replace repeats with back-references (Lempel & Ziv, 1977)

Keep a **sliding window** of recent data. When the upcoming bytes match something already seen, emit `(distance, length)` instead of the bytes:

```
Input:   "abracadabra abracadabra"
Output:  a b r a c a d (7,4) ␠ (12,11)
          literals…      ↑ "abra" from 7 back   ↑ repeat whole word from 12 back
```

A tiny LZ77 compressor/decompressor in Python:

```python
def lz77_compress(data: bytes, window=4096, min_len=3):
    i, out = 0, []
    while i < len(data):
        best_len = best_dist = 0
        for j in range(max(0, i - window), i):          # naive search (real ones use hash chains)
            k = 0
            while i + k < len(data) and data[j + k] == data[i + k] and k < 258:
                k += 1
            if k > best_len: best_len, best_dist = k, i - j
        if best_len >= min_len:
            out.append((best_dist, best_len)); i += best_len
        else:
            out.append(data[i]); i += 1
    return out

def lz77_decompress(tokens):
    buf = bytearray()
    for t in tokens:
        if isinstance(t, tuple):
            dist, length = t
            for _ in range(length): buf.append(buf[-dist])   # overlapping copies allowed (RLE effect)
        else:
            buf.append(t)
    return bytes(buf)

toks = lz77_compress(b"abracadabra abracadabra")
print(toks); assert lz77_decompress(toks) == b"abracadabra abracadabra"
```

Real implementations find matches with **hash chains** or **binary trees/suffix structures**, and use **lazy / optimal parsing** (try whether a later match is better).

### 2.2 Entropy coding: fewer bits for frequent symbols

**Huffman coding**: build a binary tree by repeatedly merging the two least frequent symbols; frequent symbols get short codes. Prefix-free, so decoding is unambiguous.

```python
import heapq
from collections import Counter
def huffman_codes(data: bytes):
    heap = [[freq, [sym, ""]] for sym, freq in Counter(data).items()]
    heapq.heapify(heap)
    while len(heap) > 1:
        lo, hi = heapq.heappop(heap), heapq.heappop(heap)
        for pair in lo[1:]: pair[1] = "0" + pair[1]
        for pair in hi[1:]: pair[1] = "1" + pair[1]
        heapq.heappush(heap, [lo[0] + hi[0]] + lo[1:] + hi[1:])
    return {chr(s): code for s, code in heap[0][1:]}
print(huffman_codes(b"aaaaabbbccd"))   # e.g. {'a': '0', 'b': '10', 'c': '111', 'd': '110'}
```

Huffman wastes up to ~1 bit/symbol because codes are whole bits. **Arithmetic coding / range coding** encode the whole message as one number in [0,1), achieving fractional bits per symbol. **ANS** (Asymmetric Numeral Systems, Jarek Duda 2014) gives arithmetic-coding efficiency at Huffman-like speed — used in **Zstandard (FSE)**, LZFSE, JPEG XL, and more.

### 2.3 Context modeling (the high end)
Predict each bit/byte using statistics conditioned on preceding context — **PPM**, **context mixing** (PAQ, cmix: mixing many models with neural nets), and the **LZMA** "literal + match + rep-match" state machine with adaptive binary range coding. Better prediction → fewer bits, at higher CPU cost. Modern LLM-based compressors push this further (a language model *is* a predictor).

### 2.4 Transforms that help
- **BWT** (Burrows–Wheeler Transform, bzip2): reorders text so similar contexts cluster → long runs → MTF + RLE + Huffman.
- **Delta / filters**: executables' relative call addresses converted to absolute (**BCJ/x86 filter** in 7-Zip/xz) so repeated calls match; delta for audio/images.
- **Dictionaries**: pre-trained dictionaries for small messages (zstd `--train`, Brotli's built-in web dictionary).

## 3. The major formats and algorithms

| Algorithm | Year | Core | Ratio | Speed | Where |
|---|---|---|---|---|---|
| **DEFLATE** | 1993 | LZ77 (32 KB window) + Huffman | Good | Fast | ZIP, gzip, zlib, PNG, HTTP `gzip` |
| bzip2 | 1996 | BWT + MTF + Huffman | Better | Slow | `.tar.bz2` |
| **LZMA / LZMA2** | 1998 | LZ77 (up to GB window) + range coding + context models | Excellent | Slow compress, OK decompress | **7z**, **xz** (`.tar.xz`), Linux kernel packages |
| **Brotli** | 2015 | LZ77 + Huffman + context modeling + static dictionary | Very good for web | Medium | HTTP `br`, WOFF2 fonts |
| **Zstandard (zstd)** | 2016 | LZ77 + FSE/ANS + Huffman | gzip-to-xz range (levels 1–22) | Very fast decompression | Linux kernel/initramfs, Btrfs, Arch/Fedora packages, HTTP `zstd`, Windows 11 File Explorer support |
| **LZ4** | 2011 | LZ77, no entropy coding | Modest | Extremely fast (GB/s) | Real-time: ZFS, zram, databases |
| RAR5 | 2013 | Proprietary LZ + PPMd options | Excellent | Medium | `.rar` (WinRAR) |

## 4. Anatomy of an archive

An **archive** combines **bundling** (many files + metadata into one) and **compression**. Formats differ in how they combine them.

### 4.1 ZIP — per-file compression with a central directory

```
[Local file header 1][compressed data 1][data descriptor]
[Local file header 2][compressed data 2]
...
[Central directory: entry per file → name, sizes, CRC-32, offset of local header]
[End of Central Directory record (EOCD): where the central directory starts]
```

- Readers start from the **end** (EOCD) to find the central directory → random access to any file without decompressing others.
- Each file compressed independently (usually DEFLATE; also Deflate64, bzip2, LZMA, zstd).
- CRC-32 detects corruption. **Encryption**: legacy ZipCrypto (**broken**), WinZip AES-256 (good; file names remain visible).
- ZIP is also the container for `.docx/.xlsx/.pptx`, `.jar`, `.apk`, `.epub`, `.nupkg`.

```python
import zipfile
with zipfile.ZipFile("demo.zip", "w", compression=zipfile.ZIP_DEFLATED, compresslevel=9) as z:
    z.writestr("hello.txt", "hello " * 1000)
with zipfile.ZipFile("demo.zip") as z:
    for info in z.infolist():
        print(info.filename, info.file_size, "→", info.compress_size, hex(info.CRC))
```

### 4.2 tar + compressor — "solid" compression (UNIX way)
`tar` concatenates files with 512-byte headers (name, mode, owner, mtime — preserves UNIX permissions/symlinks); then the **whole stream** is compressed (`.tar.gz`, `.tar.xz`, `.tar.zst`).
- **Solid**: redundancy *across* files is exploited → better ratio for many similar files.
- No random access: extracting the last file decompresses everything before it.

### 4.3 7z — solid blocks + filters + strong crypto
Solid blocks of files, LZMA2 (or PPMd/BZip2), BCJ2 filters for executables, AES-256 encryption with SHA-256-based key derivation, optional **header encryption** (hides filenames).

### 4.4 Integrity & recovery
CRC-32 (ZIP), CRC-32/CRC-64/SHA-256 (xz), XXH64 (zstd frames); RAR supports **recovery records** (Reed–Solomon parity) to repair damage; PAR2 files do the same externally.

## 5. Command-line practice

**Linux**
```bash
tar -czf site.tar.gz site/              # gzip
tar -cJf site.tar.xz site/              # xz (LZMA2)
tar --zstd -cf site.tar.zst site/       # zstd
zstd -19 --long=27 big.iso               # high ratio, 128 MB window
zstd -T0 -3 logs.tar                      # all cores, fast level
xz -9e -T0 file                           # extreme preset, multithreaded
7z a -t7z -mx=9 -mhe=on -p backup.7z docs/   # 7z with encrypted headers (p7zip / 7zz)
unzip -l archive.zip ; zipinfo archive.zip
```

**Windows**
```powershell
Compress-Archive -Path .\site\* -DestinationPath site.zip -CompressionLevel Optimal
Expand-Archive site.zip -DestinationPath .\out
tar -czf site.tar.gz site          # bsdtar (libarchive) ships in Windows 10+
tar -xf archive.7z                 # Windows 11: libarchive also reads 7z/RAR; File Explorer creates 7z/TAR
compact /c /exe:lzx C:\Games\X\*   # NTFS transparent file compression (XPRESS/LZX)
```

NTFS also supports per-file **LZNT1** compression (`compact /c`), and Linux filesystems offer transparent compression (**Btrfs** `compress=zstd`, **ZFS** `compression=lz4|zstd`, **zram** compressed swap in RAM).

## 6. Compression on the web and in apps

- HTTP negotiates via `Accept-Encoding: gzip, br, zstd` → `Content-Encoding: br` (see [Web Protocols](../05-networking-and-web/web-protocols.md)). Pre-compress static assets at build time at max levels.
- **Streaming APIs** in JS:

```js
// Browser/Node: CompressionStream supports "gzip", "deflate", "deflate-raw" (and "brotli"/"zstd" in newer runtimes)
const stream = new Blob(["hello ".repeat(1000)]).stream().pipeThrough(new CompressionStream("gzip"));
const compressed = await new Response(stream).arrayBuffer();
console.log(compressed.byteLength);   // ~30 bytes instead of 6000

// Node.js zlib
import { brotliCompressSync, constants } from "node:zlib";
const out = brotliCompressSync(Buffer.from("hello ".repeat(1000)),
  { params: { [constants.BROTLI_PARAM_QUALITY]: 11 } });
```

- C++ with zstd:

```cpp
#include <zstd.h>
#include <stdexcept>
#include <vector>
#include <string>
std::vector<char> compress(const std::string& in, int level = 3) {
    std::vector<char> out(ZSTD_compressBound(in.size()));
    size_t n = ZSTD_compress(out.data(), out.size(), in.data(), in.size(), level);
    if (ZSTD_isError(n)) throw std::runtime_error(ZSTD_getErrorName(n));
    out.resize(n); return out;
}
```

- Python standard library: `zlib`, `gzip`, `bz2`, `lzma`, `zipfile`, `tarfile`, and **`compression.zstd`** (Python 3.14+).

## 7. Choosing a compressor

| Need | Choose |
|---|---|
| Universal compatibility | ZIP (Deflate) |
| Best ratio for distribution/backups | xz / 7z LZMA2, or zstd -19 --long |
| Speed with good ratio (logs, pipelines, packages) | **zstd** levels 1–9 |
| Real-time / in-memory / filesystem | LZ4 or zstd -1 |
| Web assets | Brotli (static, q11) / zstd or gzip (dynamic) |

## 8. Security considerations

- **Zip bombs**: tiny archives expanding to petabytes (nested or overlapping entries, e.g., 42.zip). Limit decompressed size and ratio.
- **Zip Slip / path traversal**: entries named `../../etc/cron.d/x` or absolute paths. Validate extraction paths (Python's `tarfile` now has `filter="data"` — use it).
- **Compression + encryption side channels**: CRIME/BREACH exploit compressed-size changes to leak secrets in TLS/HTTP — don't compress secrets alongside attacker-controlled data.
- Malware often hides in password-protected archives to evade scanners — see [Malware](malware.md).

## Further reading
- David Salomon, *Data Compression: The Complete Reference*; Matt Mahoney, *Data Compression Explained* (free)
- RFC 1951 (DEFLATE), RFC 7932 (Brotli), RFC 8878 (Zstandard); PKWARE *APPNOTE.TXT* (ZIP spec)
- Yann Collet's blog (zstd, FSE); Jarek Duda, *Asymmetric numeral systems* (arXiv 2013)
