# How Bluetooth Works

> **Level:** Advanced · **Related:** [Radio](radio.md) · [Wi-Fi](wifi.md) · [NFC](nfc.md) · [Encryption](../06-security-and-data/encryption.md) · [Drivers](../03-os-and-software/drivers.md)

## 1. Two radios under one name

"Bluetooth" is actually two different, incompatible radio systems specified by the **Bluetooth SIG**, both in the 2.4 GHz ISM band:

| | **Bluetooth Classic (BR/EDR)** | **Bluetooth Low Energy (LE)** |
|---|---|---|
| Introduced | 1999 (v1.0) | 2010 (v4.0) |
| Channels | 79 × 1 MHz | 40 × 2 MHz (3 advertising + 37 data) |
| Modulation | GFSK (1 Mbps), π/4-DQPSK (2), 8DPSK (3 Mbps EDR) | GFSK 1M / 2M PHY, **Coded PHY** (125/500 kbps long range) |
| Typical use | Legacy audio (A2DP, HFP), serial (SPP) | Wearables, sensors, beacons, **LE Audio**, keyboards/mice, trackers |
| Power | ~100 mW class | ~1–15 mW; coin-cell years |
| Topology | Piconet (1 central + 7 active peripherals), scatternet | Star, broadcast, **Mesh** |

"Dual-mode" chips (phones, laptops) support both; "single-mode" LE devices only LE.

## 2. Radio basics

- **Frequency hopping (FHSS)**: Classic hops 1600 times/s over 79 channels following a pseudo-random sequence derived from the central's address and clock; **AFH** (Adaptive Frequency Hopping) avoids channels busy with [Wi-Fi](wifi.md).
- LE data connections hop once per **connection event** using a channel selection algorithm (CSA #2).
- LE advertising channels 37, 38, 39 (2402, 2426, 2480 MHz) sit between the main Wi-Fi channels 1/6/11.
- Range: ~10 m (class 2) typical; LE Coded PHY up to hundreds of meters to ~1 km line-of-sight.
- **Channel Sounding** (Bluetooth 6.0, 2024): phase-based ranging + round-trip timing for ~10 cm-level distance measurement (secure "find my"/digital keys).

## 3. The protocol stack

```
┌─────────────────────────── Host (OS: BlueZ / Windows Bluetooth stack) ───────────────────────────┐
│  Profiles / Apps: A2DP, HFP, HID, LE Audio (BAP, LC3), Heart Rate, Find My ...                   │
│  Classic: RFCOMM (serial), SDP (service discovery), AVDTP/AVCTP     LE: GATT ← ATT, SM, GAP      │
│  L2CAP — multiplexing, segmentation, channels                                                    │
├──────────── HCI (Host Controller Interface): commands/events over UART, USB, SDIO ───────────────┤
│  Controller (firmware on the BT chip): Link Layer / Link Manager, Baseband, Radio (PHY)          │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

The **HCI** boundary is standardized: the OS host stack sends commands (`LE Set Scan Enable`, `Create Connection`) and receives events. That's why one OS stack works with any vendor's chip.

## 4. Bluetooth Low Energy in depth

### 4.1 GAP — roles and discovery
- **Advertiser/Broadcaster** sends advertising packets (up to 31 bytes legacy; 255+ bytes with extended advertising) every 20 ms – 10 s.
- **Scanner/Observer** listens; can send a scan request for extra data.
- **Central** (formerly master) initiates a connection; **Peripheral** accepts.

Advertising payload = list of **AD structures**: `[length][type][data]` — flags, local name, service UUIDs, manufacturer data (e.g., Apple iBeacon, Google Eddystone), TX power.

### 4.2 Connections
Once connected, both devices wake up at each **connection interval** (7.5 ms – 4 s), exchange packets, and sleep. **Peripheral latency** lets a peripheral skip events when idle — the key to multi-year battery life.

### 4.3 GATT — the data model
Everything on an LE device is exposed as a hierarchical **attribute table**:

```
Profile: Heart Rate Sensor
└── Service: Heart Rate (UUID 0x180D)
    ├── Characteristic: Heart Rate Measurement (0x2A37)   [Notify]
    │   └── Descriptor: CCCD (0x2902)  ← client writes 0x0001 to enable notifications
    └── Characteristic: Body Sensor Location (0x2A38)     [Read]
└── Service: Battery (0x180F)
    └── Characteristic: Battery Level (0x2A19)            [Read, Notify]
```

**ATT** operations: Read, Write, Write Without Response, **Notify** (server push, unacknowledged), **Indicate** (acknowledged). Each attribute has a **handle**, a UUID type (16-bit SIG-assigned or 128-bit custom) and permissions.

### 4.4 Code: talk to a heart-rate sensor

**Python (bleak — works on Windows, Linux (BlueZ/D-Bus) and macOS):**

```python
import asyncio
from bleak import BleakScanner, BleakClient
HR_MEASUREMENT = "00002a37-0000-1000-8000-00805f9b34fb"

def on_hr(_, data: bytearray):
    flags = data[0]
    bpm = int.from_bytes(data[1:3], "little") if flags & 0x01 else data[1]
    print("Heart rate:", bpm)

async def main():
    dev = await BleakScanner.find_device_by_filter(
        lambda d, adv: "0000180d-0000-1000-8000-00805f9b34fb" in adv.service_uuids)
    async with BleakClient(dev) as client:
        await client.start_notify(HR_MEASUREMENT, on_hr)   # writes CCCD for us
        await asyncio.sleep(30)

asyncio.run(main())
```

**JavaScript (Web Bluetooth, Chrome/Edge):**

```js
const device = await navigator.bluetooth.requestDevice({ filters: [{ services: ["heart_rate"] }] });
const server = await device.gatt.connect();
const service = await server.getPrimaryService("heart_rate");
const ch = await service.getCharacteristic("heart_rate_measurement");
ch.addEventListener("characteristicvaluechanged", e => {
  const v = e.target.value; const flags = v.getUint8(0);
  console.log("BPM", flags & 1 ? v.getUint16(1, true) : v.getUint8(1));
});
await ch.startNotifications();
```

## 5. Classic Bluetooth & audio

- **Inquiry/paging** for discovery and connection; **SDP** to find services.
- **A2DP** streams stereo audio encoded with **SBC** (mandatory), AAC, aptX, LDAC; **AVRCP** for remote control; **HFP/HSP** for calls (narrowband CVSD / wideband mSBC; later LC3-SWB).
- **SCO/eSCO** synchronous links for voice.

**LE Audio** (Bluetooth 5.2+) replaces Classic audio over time:
- **LC3** codec: better quality than SBC at half the bitrate.
- **Isochronous channels** (CIS for connected, BIS for broadcast).
- **Auracast** broadcast audio: one transmitter to unlimited receivers (airports, TVs, hearing aids).
- True multi-stream for earbuds (each bud gets its own synchronized stream).

## 6. Pairing & security

**Pairing** = authenticate + generate keys; **Bonding** = store keys for reconnection.

| Method | When | MITM protection |
|---|---|---|
| Just Works | No display/keyboard | ❌ |
| Numeric Comparison | Both have displays + yes/no | ✅ |
| Passkey Entry | One keyboard | ✅ |
| Out of Band (OOB) | Keys exchanged via [NFC](nfc.md) or QR | ✅ (depends on OOB channel) |

- **LE Secure Connections** (4.2+) uses **ECDH P-256** key exchange; derived LTK encrypts links with **AES-CCM**. Legacy LE pairing (4.0/4.1) was breakable by sniffing.
- **Privacy**: **Resolvable Private Addresses (RPA)** rotate the device address (~15 min); bonded peers resolve them with the IRK → prevents tracking.
- Notable attacks: BlueBorne (2017, stack RCE), KNOB (key-length negotiation downgrade), BIAS (impersonation), BLURtooth (CTKD), BLUFFS (2023). Keep firmware/OS updated; disable discoverability when unused.

## 7. Bluetooth Mesh

Managed flooding over LE advertising: messages relayed by relay nodes with TTL; **publish/subscribe** addressing; used for smart lighting and building automation. Two-layer security (network key + application key).

## 8. OS integration

**Linux — BlueZ**
- Kernel: HCI core, L2CAP, RFCOMM, `btusb` driver (HCI over USB), `hci_uart`.
- Daemon: `bluetoothd`, exposes **D-Bus API** (`org.bluez`). Audio via PipeWire/WirePlumber (A2DP, HFP, LE Audio support maturing).

```bash
bluetoothctl                # interactive: power on, scan on, pair XX:XX..., connect ...
btmgmt info                 # controller info
sudo btmon                  # live HCI trace (like Wireshark for Bluetooth)
hciconfig -a                # (deprecated but common)
rfkill list                 # soft/hard block status
```

**Windows**
- Stack: `bthport.sys`, `bthusb.sys` (HCI over USB), `BthLEEnum.sys` (LE), profile drivers (`BthA2dp.sys`, `BthHFEnum.sys`, `HidBth.sys`).
- APIs: **Windows.Devices.Bluetooth** (WinRT, C#/C++/JS): `BluetoothLEAdvertisementWatcher`, `GattDeviceService`; classic Win32 `BluetoothFindFirstDevice`, Winsock `AF_BTH` for RFCOMM.
- Settings → Bluetooth & devices; Device Manager → Bluetooth; `btpair`/`btdiscovery`; HCI logs via ETW (`BTHUSB` tracing) + Wireshark conversion.

C++/WinRT scan snippet:

```cpp
#include <winrt/Windows.Devices.Bluetooth.Advertisement.h>
using namespace winrt::Windows::Devices::Bluetooth::Advertisement;
BluetoothLEAdvertisementWatcher watcher;
watcher.ScanningMode(BluetoothLEScanningMode::Active);
watcher.Received([](auto&&, BluetoothLEAdvertisementReceivedEventArgs const& a) {
    wprintf(L"%llx RSSI %d %s\n", a.BluetoothAddress(), a.RawSignalStrengthInDBm(),
            a.Advertisement().LocalName().c_str());
});
watcher.Start();
```

## 9. Troubleshooting

- 2.4 GHz congestion from Wi-Fi and USB 3.0 ports (USB 3 emits broadband noise near 2.4 GHz — use an extension cable for dongles).
- Audio switching to "headset" quality when mic is used (HFP narrowband) — LE Audio solves this.
- Re-pair after firmware updates; remove stale bonds.

## Further reading
- Bluetooth Core Specification v6.x (bluetooth.com); SIG *Assigned Numbers* (UUIDs)
- Kevin Townsend et al., *Getting Started with Bluetooth Low Energy* (O'Reilly)
- BlueZ docs (`doc/*.rst` D-Bus API); Microsoft Learn *Bluetooth LE* (UWP/WinRT)
