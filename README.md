# Advanced Tech Knowledge Base

A deep, cross-linked guide to **how modern technology actually works**, from transistors and radio waves to compilers, encryption and AI.

Every article follows the same structure:

1. **Core idea.** The one-paragraph mental model.
2. **Architecture and diagrams.** Mermaid and ASCII diagrams of the real mechanism.
3. **Deep dive.** Protocols, algorithms, formulas and data structures.
4. **Hands-on code.** Runnable examples in **JavaScript, Python or C++**.
5. **OS references.** **Windows and Linux** commands, APIs and internals for anything OS-related.
6. **Security, pitfalls and further reading.** Standards, RFCs and books to go deeper.

---

## Index (in the requested order)

| # | Topic | Domain |
|---|---|---|
| 1 | [How SIM / eSIM work](docs/04-wireless-and-telecom/sim-esim.md) | Wireless & Telecom |
| 2 | [How GPS works](docs/04-wireless-and-telecom/gps.md) | Wireless & Telecom |
| 3 | [How a GPU works](docs/01-hardware/gpu.md) | Hardware |
| 4 | [How a CPU works](docs/01-hardware/cpu.md) | Hardware |
| 5 | [How an NPU works](docs/01-hardware/npu.md) | Hardware |
| 6 | [How RAM works](docs/01-hardware/ram.md) | Hardware |
| 7 | [How an OS works](docs/03-os-and-software/operating-system.md) | OS & Software |
| 8 | [How BIOS / UEFI works](docs/01-hardware/bios.md) | Hardware |
| 9 | [How 3G / 4G / 5G work](docs/04-wireless-and-telecom/cellular-3g-4g-5g.md) | Wireless & Telecom |
| 10 | [How a web server works](docs/05-networking-and-web/web-server.md) | Networking & Web |
| 11 | [How a payment terminal works](docs/06-security-and-data/payment-terminal.md) | Security & Data |
| 12 | [How NFC works](docs/04-wireless-and-telecom/nfc.md) | Wireless & Telecom |
| 13 | [How computer graphics work](docs/02-graphics/computer-graphics.md) | Graphics |
| 14 | [How software / apps work](docs/03-os-and-software/software-and-apps.md) | OS & Software |
| 15 | [How sensors and detectors work](docs/01-hardware/sensors-and-detectors.md) | Hardware |
| 16 | [How (system) drivers work](docs/03-os-and-software/drivers.md) | OS & Software |
| 17 | [How shaders work](docs/02-graphics/shaders.md) | Graphics |
| 18 | [How frame generation works](docs/02-graphics/frame-generation.md) | Graphics |
| 19 | [How DLSS works](docs/02-graphics/dlss.md) | Graphics |
| 20 | [How ray tracing works](docs/02-graphics/ray-tracing.md) | Graphics |
| 21 | [How Wi-Fi works](docs/04-wireless-and-telecom/wifi.md) | Wireless & Telecom |
| 22 | [How Bluetooth works](docs/04-wireless-and-telecom/bluetooth.md) | Wireless & Telecom |
| 23 | [How radio works](docs/04-wireless-and-telecom/radio.md) | Wireless & Telecom |
| 24 | [How a motherboard works](docs/01-hardware/motherboard.md) | Hardware |
| 25 | [How AI image detection works](docs/07-ai/image-detection.md) | AI |
| 26 | [How wireless charging works](docs/04-wireless-and-telecom/wireless-charging.md) | Wireless & Telecom |
| 27 | [How malware works](docs/06-security-and-data/malware.md) (defensive perspective) | Security & Data |
| 28 | [How web protocols work](docs/05-networking-and-web/web-protocols.md) | Networking & Web |
| 29 | [How a VPN works](docs/05-networking-and-web/vpn.md) | Networking & Web |
| 30 | [How a proxy works](docs/05-networking-and-web/proxy.md) | Networking & Web |
| 31 | [How WebRTC works](docs/05-networking-and-web/webrtc.md) | Networking & Web |
| 32 | [How code compilation works](docs/03-os-and-software/code-compilation.md) | OS & Software |
| 33 | [How the lexical environment works](docs/03-os-and-software/lexical-environment.md) | OS & Software |
| 34 | [How encryption works](docs/06-security-and-data/encryption.md) | Security & Data |
| 35 | [How compression (archives) works](docs/06-security-and-data/compression.md) | Security & Data |

---

## Browse by domain

```
docs/
├── 01-hardware/              CPU · GPU · NPU · RAM · BIOS/UEFI · Motherboard · Sensors & Detectors
├── 02-graphics/              Computer Graphics · Shaders · Ray Tracing · DLSS · Frame Generation
├── 03-os-and-software/       Operating System · Drivers · Software/Apps · Code Compilation · Lexical Environment
├── 04-wireless-and-telecom/  Radio · Wi-Fi · Bluetooth · NFC · 3G/4G/5G · SIM/eSIM · GPS · Wireless Charging
├── 05-networking-and-web/    Web Protocols · Web Server · Proxy · VPN · WebRTC
├── 06-security-and-data/     Encryption · Compression · Malware · Payment Terminals
└── 07-ai/                    AI Image Detection
```

## Suggested learning paths

The topics build on each other. If you're starting fresh, these orders work well:

**Path 1: The machine, bottom-up**
[Motherboard](docs/01-hardware/motherboard.md) → [BIOS/UEFI](docs/01-hardware/bios.md) → [CPU](docs/01-hardware/cpu.md) → [RAM](docs/01-hardware/ram.md) → [OS](docs/03-os-and-software/operating-system.md) → [Drivers](docs/03-os-and-software/drivers.md) → [Software/Apps](docs/03-os-and-software/software-and-apps.md)

**Path 2: Programming languages**
[Code Compilation](docs/03-os-and-software/code-compilation.md) → [Lexical Environment](docs/03-os-and-software/lexical-environment.md) → [Software/Apps](docs/03-os-and-software/software-and-apps.md) → [CPU](docs/01-hardware/cpu.md)

**Path 3: Graphics and games**
[GPU](docs/01-hardware/gpu.md) → [Computer Graphics](docs/02-graphics/computer-graphics.md) → [Shaders](docs/02-graphics/shaders.md) → [Ray Tracing](docs/02-graphics/ray-tracing.md) → [DLSS](docs/02-graphics/dlss.md) → [Frame Generation](docs/02-graphics/frame-generation.md)

**Path 4: Wireless world**
[Radio](docs/04-wireless-and-telecom/radio.md) → [Wi-Fi](docs/04-wireless-and-telecom/wifi.md) → [Bluetooth](docs/04-wireless-and-telecom/bluetooth.md) → [NFC](docs/04-wireless-and-telecom/nfc.md) → [Wireless Charging](docs/04-wireless-and-telecom/wireless-charging.md) → [3G/4G/5G](docs/04-wireless-and-telecom/cellular-3g-4g-5g.md) → [SIM/eSIM](docs/04-wireless-and-telecom/sim-esim.md) → [GPS](docs/04-wireless-and-telecom/gps.md)

**Path 5: The internet and the web**
[Web Protocols](docs/05-networking-and-web/web-protocols.md) → [Web Server](docs/05-networking-and-web/web-server.md) → [Proxy](docs/05-networking-and-web/proxy.md) → [VPN](docs/05-networking-and-web/vpn.md) → [WebRTC](docs/05-networking-and-web/webrtc.md)

**Path 6: Security and data**
[Encryption](docs/06-security-and-data/encryption.md) → [Compression](docs/06-security-and-data/compression.md) → [Payment Terminals](docs/06-security-and-data/payment-terminal.md) → [Malware](docs/06-security-and-data/malware.md)

**Path 7: AI hardware and vision**
[NPU](docs/01-hardware/npu.md) → [Sensors (cameras)](docs/01-hardware/sensors-and-detectors.md) → [AI Image Detection](docs/07-ai/image-detection.md) → [DLSS](docs/02-graphics/dlss.md)

## Conventions

- **OS references:** OS-related articles list the relevant **Windows** tools and APIs (PowerShell, `netsh`, Device Manager, WDK, Win32/WinRT) alongside their **Linux** equivalents (`/proc`, `/sys`, `ip`, `systemctl`, kernel subsystems).
- **Code:** examples use **JavaScript** (browser/Node.js), **Python 3.10+** or **C++17/20**. Kernel drivers, UEFI and shader code are shown in the languages those platforms actually require (C, HLSL/GLSL/WGSL).
- **Diagrams:** written in Mermaid, which GitHub renders natively, or plain ASCII.
- **Security content:** the malware and interception material is written from a **defender's perspective** and contains no working malicious code.
- **Currency:** facts reflect the state of the field as of 2026. Fast-moving areas (5G/6G, Wi-Fi 8, DLSS, post-quantum cryptography, AI detectors) are marked with versions and years so you can tell how current each claim is.
