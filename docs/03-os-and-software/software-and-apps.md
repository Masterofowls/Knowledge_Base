# How Software / Apps Work

> **Level:** Advanced · **Related:** [OS](operating-system.md) · [Compilation](code-compilation.md) · [Lexical Environment](lexical-environment.md) · [CPU](../01-hardware/cpu.md) · [Web Server](../05-networking-and-web/web-server.md)

## 1. What "an app" really is

An application is a **file (or bundle of files)** containing machine code or bytecode, data and metadata, which the OS **loads into a process**, and which then runs by executing instructions on the CPU and requesting services from the OS through system calls.

```mermaid
flowchart LR
  SRC[Source code<br/>C++ / Python / JS] --> BUILD[Build<br/>compile, bundle, link]
  BUILD --> ART[Artifact<br/>.exe/.dll, ELF/.so, .pyc, .js bundle, APK]
  ART --> INST[Install / package<br/>MSI, MSIX, winget, apt, rpm, Flatpak]
  INST --> LOAD[Loader creates process<br/>maps code, links libs]
  LOAD --> RUN[Runtime<br/>main(), event loop, GC, JIT]
  RUN --> OS[System calls → kernel → hardware]
```

## 2. Execution models

| Model | How it runs | Examples |
|---|---|---|
| **Native AOT** | Compiled ahead of time to CPU machine code | C, C++, Rust, Go |
| **Bytecode + VM (+JIT)** | Compiled to portable bytecode; VM interprets and JIT-compiles hot code | Java (JVM), C# (.NET CLR), Python (CPython interprets `.pyc`; 3.13+ experimental JIT) |
| **Source + JIT engine** | Engine parses source, interprets bytecode, JIT-tiers to machine code | JavaScript (V8, SpiderMonkey, JavaScriptCore) |
| **WebAssembly** | Portable binary bytecode compiled by the host (browser, Wasmtime) | Rust/C++ in browser, plugins |

Details of each pipeline: [Code Compilation](code-compilation.md).

## 3. Executable file formats

| | Windows | Linux |
|---|---|---|
| Format | **PE/COFF** (`.exe`, `.dll`, `.sys`) | **ELF** (executables, `.so`, `.ko`) |
| Header starts with | `MZ` DOS stub → `PE\0\0` | `\x7fELF` |
| Sections | `.text` (code), `.rdata`, `.data`, `.idata` (imports), `.reloc`, `.rsrc` (icons, manifests) | `.text`, `.rodata`, `.data`, `.bss`, `.dynamic`, `.got`, `.plt` |
| Imports | Import Address Table (IAT) | GOT/PLT + `DT_NEEDED` entries |
| Inspect | `dumpbin /headers /imports app.exe`, PE-bear, CFF Explorer | `readelf -a`, `objdump -d`, `ldd`, `nm` |

## 4. Loading: from double-click to `main()`

**Linux (`execve`)**
1. Kernel reads the ELF header, maps `PT_LOAD` segments with the right permissions (R-X code, RW- data), sets up the stack (argv, envp, **auxv**).
2. If the binary has `PT_INTERP`, the kernel starts the **dynamic linker** `/lib64/ld-linux-x86-64.so.2` first.
3. `ld.so` loads dependencies (`libc.so.6`, …) found via `RPATH`/`LD_LIBRARY_PATH`/`/etc/ld.so.cache`, applies relocations, resolves symbols (lazily through the PLT unless `-z now`).
4. Runs initializers (`.init_array`: C++ global constructors), then jumps to `_start` → `__libc_start_main` → **`main()`**.

**Windows (`CreateProcess`)**
1. Kernel creates the process object and maps the image as a **section** + `ntdll.dll`.
2. The initial thread runs `LdrInitializeThunk` in ntdll: the **loader** walks the import table, loads DLLs (search order: app directory → System32 → … ; KnownDLLs cached), applies base relocations if ASLR moved the image, fills the IAT.
3. Runs `DllMain(DLL_PROCESS_ATTACH)` of each DLL, TLS callbacks.
4. Calls the CRT entry (`mainCRTStartup`/`WinMainCRTStartup`) → **`main`/`WinMain`**.

Load-time vs run-time linking:

```cpp
// Explicit run-time loading — Windows
#include <windows.h>
HMODULE h = LoadLibraryW(L"user32.dll");
auto msgBox = (int (WINAPI*)(HWND, LPCWSTR, LPCWSTR, UINT))GetProcAddress(h, "MessageBoxW");
msgBox(nullptr, L"Loaded at run time", L"Demo", MB_OK);

// Linux equivalent
#include <dlfcn.h>
void* lib = dlopen("libm.so.6", RTLD_NOW);
auto cosine = (double(*)(double))dlsym(lib, "cos");
```

Python does the same through `ctypes`:

```python
import ctypes, sys
libc = ctypes.CDLL("msvcrt" if sys.platform == "win32" else "libc.so.6")
libc.printf(b"Hello from C's printf, called from Python\n")
```

## 5. Anatomy of a running process

```
┌──────────── Process ────────────┐
│ Code (.text, shared libs)       │  read + execute, shared across processes
│ Globals (.data / .bss)          │
│ Heap (malloc/new, GC heap)      │  grows on demand
│ Thread 1 stack │ Thread 2 stack │  local variables, return addresses
│ Memory-mapped files             │
│ Handles / file descriptors      │  files, sockets, pipes, windows
│ Environment, args, cwd, token   │
└─────────────────────────────────┘
```

- **Stack**: function frames (return address, saved registers, locals). Fast, automatically freed, limited (1 MB default Windows main thread; 8 MB Linux).
- **Heap**: dynamic allocation — `malloc` (glibc ptmalloc, jemalloc, mimalloc), Windows Heap / Segment Heap, or a garbage-collected heap in managed runtimes.
- **Calling conventions** define how arguments pass: System V AMD64 (Linux: RDI, RSI, RDX, RCX, R8, R9) vs Microsoft x64 (RCX, RDX, R8, R9 + 32-byte shadow space).

## 6. Memory management strategies

| Strategy | Language | Mechanism |
|---|---|---|
| Manual | C | `malloc`/`free` |
| RAII / ownership | C++, Rust | Destructors run at scope exit; smart pointers (`unique_ptr`, `shared_ptr`) |
| Reference counting (+ cycle GC) | Python (CPython), Swift, Obj-C | Free when count hits 0; Python's `gc` finds cycles |
| Tracing GC | JavaScript, Java, C#, Go | Mark reachable objects from roots; generational (young/old), concurrent/incremental |

```cpp
#include <memory>
#include <vector>
void f() {
    auto buf = std::make_unique<std::vector<int>>(1'000'000);  // heap allocation
    // ... use buf ...
}   // destructor frees memory here — deterministic, no GC
```

```js
// JS: objects live as long as they're reachable; V8's Orinoco GC (generational,
// parallel scavenger for young gen, concurrent mark-compact for old gen) reclaims them.
let cache = new Map();
cache.set("big", new Array(1e6).fill(0));
cache = null; // now unreachable → collectable
```

## 7. Concurrency and the event loop

- **Threads** — true parallelism on multiple cores (C++ `std::thread`, Java, C#). CPython's **GIL** historically allowed only one thread to run Python bytecode at a time; the **free-threaded** build (PEP 703, 3.13+/3.14 supported) removes it.
- **Processes** — isolation (Chrome's multi-process architecture; Python `multiprocessing`).
- **Async I/O / event loop** — one thread multiplexes thousands of I/O operations using the OS readiness/completion APIs (epoll/io_uring, IOCP, kqueue).

The JavaScript event loop (browser & Node.js/libuv):

```js
console.log("1 sync");
setTimeout(() => console.log("4 macrotask (timer)"), 0);
Promise.resolve().then(() => console.log("3 microtask"));
queueMicrotask(() => console.log("3b microtask"));
console.log("2 sync");
// Output: 1, 2, 3, 3b, 4 — run-to-completion; drain microtasks after each task
```

Python asyncio equivalent:

```python
import asyncio
async def fetch(i):
    await asyncio.sleep(1)       # yields to event loop (non-blocking)
    return i
async def main():
    results = await asyncio.gather(*(fetch(i) for i in range(1000)))  # ~1 s total, not 1000 s
    print(len(results))
asyncio.run(main())
```

## 8. GUI applications

GUI apps are event-driven: the OS delivers input events to a **message queue**; the app's loop dispatches them and repaints.

**Win32 message loop (C++):**
```cpp
MSG msg;
while (GetMessageW(&msg, nullptr, 0, 0)) {   // blocks until an event arrives
    TranslateMessage(&msg);
    DispatchMessageW(&msg);                   // calls WndProc(hwnd, WM_PAINT/WM_KEYDOWN/...)
}
```

| Layer | Windows | Linux |
|---|---|---|
| Windowing | `user32` / `win32k`, DWM compositor | Wayland compositor (or X11 server) |
| Toolkits | WinUI 3, WPF, WinForms, Qt, Electron | GTK, Qt, Electron, Flutter |
| Rendering | Direct2D / DirectWrite / D3D | Cairo, Skia, OpenGL/Vulkan via Mesa |

**Electron/Tauri** apps embed a web engine: UI in HTML/CSS/JS, a native backend (Node.js / Rust). **Mobile**: Android apps are APKs/AABs running on ART (ahead-of-time + JIT compiled Dex bytecode) in sandboxed Linux processes; iOS apps are signed Mach-O binaries in sandboxed containers.

## 9. Packaging, installation, updates

| | Windows | Linux |
|---|---|---|
| Package formats | MSI, **MSIX**, EXE installers | `.deb`, `.rpm`, **Flatpak**, **Snap**, AppImage |
| Package managers | `winget`, Microsoft Store, Chocolatey, Scoop | `apt`, `dnf`, `pacman`, `zypper` |
| Install locations | `C:\Program Files`, `%LOCALAPPDATA%` | `/usr/bin`, `/usr/lib`, `/opt`, `~/.local` |
| Config | Registry (`HKCU\Software\…`), `%APPDATA%` | `/etc`, `~/.config` (XDG) |
| Services/daemons | Windows Services (SCM) | systemd units |
| Code signing | Authenticode (SmartScreen reputation) | GPG-signed repos/packages |

## 10. Anatomy of a request in a typical modern app

```
User clicks "Buy" in a React SPA (browser JS)
  → fetch() → HTTP/2 over TLS → load balancer → web server (nginx)
  → app server (Node.js / Python FastAPI) → validates, queries PostgreSQL
  → calls payment API → returns JSON → React updates virtual DOM → browser repaints
```

Each hop is itself "software": processes, threads, event loops, system calls, network stacks — see [Web Server](../05-networking-and-web/web-server.md) and [Web Protocols](../05-networking-and-web/web-protocols.md).

## 11. Observability & debugging tools

| Need | Windows | Linux | Language-level |
|---|---|---|---|
| Debugger | Visual Studio, WinDbg | gdb, lldb | `pdb` (Python), Chrome DevTools / `node --inspect` |
| Syscall/file activity | Process Monitor | `strace`, `ltrace` | |
| Profiling | WPR/WPA, VS Profiler, VTune | `perf`, flame graphs, `valgrind --tool=callgrind` | `cProfile`, `py-spy`, DevTools Performance |
| Memory errors | Application Verifier, ASan (MSVC) | Valgrind memcheck, ASan/UBSan | `tracemalloc`, heap snapshots |

## Further reading
- Bryant & O'Hallaron, *Computer Systems: A Programmer's Perspective* (linking, loading, processes)
- John Levine, *Linkers and Loaders*
- Microsoft Learn: *PE Format*, *Dynamic-Link Library Search Order*; `man 8 ld.so`, `man 5 elf`
