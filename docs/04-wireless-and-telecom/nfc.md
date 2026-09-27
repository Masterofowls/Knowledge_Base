# How NFC Works

> **Level:** Advanced · **Related:** [Payment Terminals](../06-security-and-data/payment-terminal.md) · [Radio](radio.md) · [Wireless Charging](wireless-charging.md) · [SIM/eSIM](sim-esim.md) · [Bluetooth](bluetooth.md)

## 1. What NFC is

**Near Field Communication** is a short-range (≤ ~4 cm) wireless technology at **13.56 MHz**, derived from **RFID** and contactless smart cards (ISO/IEC 14443, FeliCa). It enables tap-to-pay, transit cards, access badges, e-passports, digital car keys and tap-to-pair.

| | NFC | UHF RFID | Bluetooth LE |
|---|---|---|---|
| Frequency | 13.56 MHz (HF) | 860–960 MHz | 2.4 GHz |
| Range | ~0–4 cm (by design) | Up to ~10 m | ~10–100 m |
| Coupling | Magnetic (inductive), near field | Radiative (backscatter) | Radiative |
| Data rate | 106–848 kbit/s | ~40–640 kbit/s | 1–2 Mbit/s |
| Passive tags? | Yes (powered by reader field) | Yes | No |

## 2. Physics: near-field inductive coupling

At 13.56 MHz, λ ≈ 22 m. Within a small fraction of a wavelength (the **near field**, `< λ/2π ≈ 3.5 m`), the field is mostly **magnetic** and falls off steeply (~1/r³ in field strength, power ~1/r⁶). NFC works like a **loosely coupled transformer**:

```
     Reader (initiator)                         Tag / card (target)
  ┌────────────────────┐    magnetic field     ┌────────────────────┐
  │ 13.56 MHz oscillator│    ~~~~~~~~~~~~~~     │ Loop antenna + C    │
  │  → loop antenna     │  ))))))  ((((((((     │ (resonant @13.56MHz)│
  │  (primary coil)     │                       │ rectifier → powers  │
  └────────────────────┘                       │ chip (~few mW)       │
                                               └────────────────────┘
```

1. The reader's coil drives an alternating magnetic field (H ≈ 1.5–7.5 A/m).
2. The tag's coil (tuned by a capacitor to resonate at 13.56 MHz) picks up voltage by induction → rectified → powers the chip. **Passive tags have no battery.**
3. **Reader → tag data**: the reader modulates its field amplitude (ASK: 100% modified Miller for Type A, 10% NRZ for Type B).
4. **Tag → reader data**: **load modulation** — the tag switches a load resistor/capacitor on and off, changing how much energy it draws. The reader detects tiny changes in its own antenna voltage — sidebands at 13.56 MHz ± **847.5 kHz** subcarrier.

The short range is physics, not just policy — a key part of NFC's security model (intentional "tap" gesture), though relay attacks can extend it (see §7).

## 3. Standards map

| Layer | Standards |
|---|---|
| RF & anti-collision | ISO/IEC 14443-2/3 (**Type A**: MIFARE; **Type B**), JIS X 6319-4 (**FeliCa**, Type F), ISO 15693 (**NFC-V**, vicinity cards) |
| Transport | ISO/IEC 14443-4 (ISO-DEP, block protocol), NFC-DEP (ISO 18092, peer-to-peer) |
| Application | **ISO 7816-4 APDUs** (smart card commands), EMV contactless, NDEF |
| NFC Forum | Tag types 1–5, NDEF format, Reader/Writer mode, Card Emulation, Peer-to-peer, Wireless Charging (WLC) |

## 4. Operating modes

1. **Reader/Writer** — phone reads a passive tag (smart poster, product tag, amiibo).
2. **Card Emulation** — phone acts as a contactless card (Apple Pay, Google Wallet, transit, badges).
3. **Peer-to-peer** — two active devices exchange data (Android Beam, now removed; still used for pairing handover).

## 5. Protocol flow: reader meets a Type A card

```mermaid
sequenceDiagram
  participant R as Reader (PCD)
  participant C as Card (PICC)
  R->>C: RF field on; REQA (7-bit short frame)
  C-->>R: ATQA (answer to request)
  R->>C: ANTICOLLISION / SELECT (cascade levels)
  C-->>R: UID (4/7/10 bytes) + SAK (capabilities)
  R->>C: RATS (request answer to select)
  C-->>R: ATS → now ISO-DEP (14443-4)
  R->>C: SELECT AID (ISO 7816 APDU)
  C-->>R: FCI + 90 00
  R->>C: Application APDUs (e.g., EMV GPO, READ RECORD)
  C-->>R: Responses + status words
```

**Anti-collision**: if several cards are in the field, the reader resolves UIDs bit by bit (bit-collision detection with Manchester coding in Type A) to select one.

**APDU format** (ISO 7816-4):
```
Command:  CLA | INS | P1 | P2 | Lc | Data | Le
          00    A4    04   00   07   A0000000031010   00      ← SELECT Visa AID
Response: Data | SW1 SW2        (90 00 = success)
```

## 6. NDEF: the NFC Data Exchange Format

Simple interoperable messages for tags: an **NDEF message** is a list of **records**, each with a type (TNF + type field) and payload — URI, text, MIME type, Android Application Record, Wi-Fi credentials, Bluetooth pairing (OOB handover).

```
NDEF record for "https://example.com":
D1        header: MB=1 ME=1 SR=1 TNF=0x01 (well-known)
01        type length = 1
0C        payload length = 12
55        type = 'U' (URI)
04        URI prefix code 0x04 = "https://"
65 78 61 6D 70 6C 65 2E 63 6F 6D   "example.com"
```

**Write a tag from a web page (Web NFC, Chrome on Android):**

```js
if ("NDEFReader" in window) {
  const ndef = new NDEFReader();
  await ndef.write({ records: [{ recordType: "url", data: "https://example.com" }] });
  await ndef.scan();
  ndef.onreading = ({ serialNumber, message }) => {
    for (const r of message.records)
      console.log(serialNumber, r.recordType, new TextDecoder().decode(r.data));
  };
}
```

**Python with a USB reader (e.g., ACR122U, PN532) via `nfcpy` on Linux:**

```python
import nfc, ndef
def on_connect(tag):
    print(tag)                                  # type, UID
    if tag.ndef:
        for record in tag.ndef.records: print(record)
    else:
        tag.format(); tag.ndef.records = [ndef.UriRecord("https://example.com")]
    return True
with nfc.ContactlessFrontend("usb") as clf:
    clf.connect(rdwr={"on-connect": on_connect})
```

**PC/SC APDUs (works on Windows *and* Linux via pcscd)** with `pyscard`:

```python
from smartcard.System import readers
conn = readers()[0].createConnection(); conn.connect()
GET_UID = [0xFF, 0xCA, 0x00, 0x00, 0x00]        # PC/SC pseudo-APDU: get card UID
data, sw1, sw2 = conn.transmit(GET_UID)
print("UID:", bytes(data).hex(), hex(sw1), hex(sw2))
```

On Windows the smart-card stack is **WinSCard** (`SCardEstablishContext`, `SCardTransmit`) + the in-box NFC class driver (NFC CX) and "Smart Card Service" (`SCardSvr`); on Linux it's **pcsc-lite** (`pcscd`) + libccid, or `libnfc`/`nfcpy` directly, and the kernel's NFC subsystem (`net/nfc`, `neard`).

## 7. Card emulation in phones & security

Where do the card's secrets live?

| Approach | Where keys live | Example |
|---|---|---|
| **Embedded Secure Element (eSE)** | Tamper-resistant chip (Common Criteria EAL5+), Java Card applets | Apple Pay |
| **UICC-based SE** | [SIM card](sim-esim.md) | Some carrier wallets, transit |
| **Host Card Emulation (HCE)** | App on the main OS, keys protected by **tokenization** + limited-use keys from cloud | Google Wallet, bank apps (Android) |

The **NFC controller** (NXP, ST, Broadcom) routes incoming APDUs by AID to the SE or to the host app (Android `HostApduService`).

**Security properties & attacks:**
- Short range + user gesture + device authentication (Face ID/PIN) before emulation.
- **Eavesdropping** possible at ~1–10 m with good equipment → application-layer crypto is essential (EMV cryptograms, DESFire AES).
- **Relay attacks** (e.g., NFCGate): forward APDUs over the internet to a distant card — mitigated by transaction limits, **distance bounding** (timing checks), and device-bound cryptograms.
- **Weak legacy cards**: MIFARE Classic's Crypto-1 cipher is broken (clonable) — use MIFARE DESFire EV2/EV3 or other AES-based cards for access control.
- **UID-only access systems** are trivially cloned — never rely on UID as a secret.

## 8. Applications

- **Payments**: EMV contactless — see [Payment Terminals](../06-security-and-data/payment-terminal.md).
- **Transit**: MIFARE DESFire, FeliCa (Suica), EMV open-loop "tap your bank card".
- **Identity**: e-passports (ICAO 9303: BAC/PACE, Active/Chip Authentication), national ID cards, mobile driving licenses (ISO 18013-5 — NFC or QR engagement, BLE data transfer).
- **Access & car keys**: Aliro/CCC Digital Key (NFC + UWB/BLE).
- **Pairing handover**: tap headphones to phone → exchange Bluetooth OOB data via NDEF.
- **NFC Wireless Charging (WLC)**: up to ~1 W for small wearables and styluses — see [Wireless Charging](wireless-charging.md).

## Further reading
- NFC Forum specifications (NDEF, Tag Types, Activity, Digital Protocol)
- ISO/IEC 14443 parts 1–4, ISO/IEC 7816-4
- Klaus Finkenzeller, *RFID Handbook*
- Android *Host-based card emulation* docs; Microsoft Learn *NFC device driver interface*
