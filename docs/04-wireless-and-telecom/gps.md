# How GPS Works

> **Level:** Advanced · **Related:** [Radio](radio.md) · [3G/4G/5G](cellular-3g-4g-5g.md) · [Sensors](../01-hardware/sensors-and-detectors.md) · [Wi-Fi](wifi.md)

## 1. The idea in one paragraph

Each GPS satellite continuously broadcasts **"I am satellite N, here is exactly where I am, and here is exactly what time it is by my atomic clock."** Your receiver measures *when* each signal arrived. Since radio travels at the speed of light, the delay gives the distance to each satellite. With distances to at least **four** satellites you can solve for your **3D position + your own clock error**. That's it — the rest is engineering to make those measurements precise to meters (or centimeters).

**GNSS** (Global Navigation Satellite Systems) is the general term:

| System | Operator | Satellites (approx.) | Civil signals |
|---|---|---|---|
| **GPS** | USA | ~31 operational, 6 orbital planes, ~20,200 km | L1 C/A (1575.42 MHz), L1C, L2C, **L5** (1176.45 MHz) |
| **GLONASS** | Russia | ~24, 3 planes, ~19,100 km | L1OF/L2OF (FDMA), newer CDMA L3OC |
| **Galileo** | EU | ~28, 3 planes, ~23,200 km | E1, **E5a/E5b**, E6 (High Accuracy Service) |
| **BeiDou (BDS-3)** | China | ~45 incl. GEO/IGSO | B1I, B1C, B2a, B3I |
| QZSS / NavIC | Japan / India | Regional | L1, L5, L6 corrections |

Modern phones are **multi-constellation, often dual-frequency (L1+L5)** receivers tracking 30–60 satellites.

## 2. Architecture: three segments

```
 Space segment            ┌────────────── Satellites: atomic clocks (Rb/Cs), transmitters
                          │
 Control segment          ├── Master Control Station (Schriever SFB, Colorado) + monitor stations + ground antennas
                          │     → compute orbits (ephemeris) & clock corrections, upload to satellites
 User segment             └── Receivers: phones, cars, aircraft, survey equipment, timing receivers
```

## 3. The math: trilateration with a clock bias

Receiver clocks are cheap quartz — off by milliseconds (1 ms = 300 km!). So we treat the receiver clock bias `b` (in meters, `c·Δt`) as a fourth unknown.

For each satellite *i* with known position `(xᵢ, yᵢ, zᵢ)`:

```
ρᵢ = √[(x − xᵢ)² + (y − yᵢ)² + (z − zᵢ)²] + b + (errors)
```

`ρᵢ` is the **pseudorange** (measured distance, "pseudo" because it includes the clock bias). Four unknowns `(x, y, z, b)` → need ≥ 4 satellites; more satellites → least-squares for accuracy. Solved iteratively (linearize, Gauss-Newton):

```python
import numpy as np

def solve_position(sat_pos, pseudoranges, iters=10):
    """sat_pos: (N,3) ECEF meters; pseudoranges: (N,) meters. Returns (x,y,z), clock_bias_m."""
    est = np.zeros(4)                        # start at Earth's center, zero bias
    for _ in range(iters):
        diff = est[:3] - sat_pos
        ranges = np.linalg.norm(diff, axis=1)
        predicted = ranges + est[3]
        residual = pseudoranges - predicted
        H = np.hstack([diff / ranges[:, None], np.ones((len(ranges), 1))])  # geometry matrix
        delta, *_ = np.linalg.lstsq(H, residual, rcond=None)
        est += delta
        if np.linalg.norm(delta) < 1e-4: break
    return est[:3], est[3]

# Synthetic test
truth = np.array([2.8e6, 2.2e6, 5.3e6]); bias = 12345.0
sats = np.array([[15e6, 10e6, 20e6], [-12e6, 18e6, 17e6], [20e6, -5e6, 15e6],
                 [5e6, 22e6, 12e6], [-5e6, -15e6, 21e6]])
rho = np.linalg.norm(sats - truth, axis=1) + bias + np.random.normal(0, 3, len(sats))
pos, b = solve_position(sats, rho)
print(np.round(pos - truth, 1), round(b, 1))   # error of a few meters
```

**Geometry matters**: satellites spread across the sky give a well-conditioned `H`. **DOP** (Dilution of Precision) quantifies this: `DOP = √trace((HᵀH)⁻¹)`. Urban canyons (all satellites overhead in a narrow strip) → high DOP → poor accuracy.

## 4. The signal: spread spectrum at −130 dBm

GPS signals arrive at about **−130 dBm** (10⁻¹⁶ W) — roughly 20 dB **below** the thermal noise floor. How can you receive a signal weaker than noise? **Spread-spectrum correlation.**

- **L1 C/A**: each satellite has a unique 1023-chip **Gold code** (PRN) repeating every 1 ms (1.023 Mchip/s).
- The 50 bit/s navigation message is XOR'd with the code, then **BPSK**-modulates the carrier.
- All satellites share the same frequency (**CDMA**) — separated by their codes.
- The receiver generates a local replica of each PRN code and **correlates** it with incoming samples: when code phase and Doppler match, the correlation peak rises out of the noise (processing gain ≈ 10·log10(1023) ≈ 30 dB, plus integration over many ms).

### Receiver processing chain

```mermaid
flowchart LR
  ANT[Antenna RHCP] --> LNA[LNA + SAW filter] --> RF[Down-convert + ADC<br/>~2–16 MHz sampling]
  RF --> ACQ[Acquisition<br/>2D search: code phase × Doppler ±5 kHz]
  ACQ --> TRK[Tracking loops<br/>DLL (code), PLL/FLL (carrier)]
  TRK --> NAV[Decode nav message<br/>ephemeris, almanac, time]
  TRK --> MEAS[Measurements<br/>pseudorange, Doppler, carrier phase]
  NAV & MEAS --> PVT[PVT solution<br/>position, velocity, time]
  PVT --> OUT[NMEA / OS location API]
```

1. **Acquisition** — search 1023 code phases × Doppler bins (satellite motion ±4 kHz + receiver clock offset). FFT-based parallel code search makes this fast.
2. **Tracking** — a **Delay Lock Loop** keeps the replica code aligned (early/prompt/late correlators); a **Phase Lock Loop** tracks the carrier phase.
3. **Navigation message** — 50 bps: **ephemeris** (precise orbit, valid ~4 h), satellite clock corrections, ionosphere model, **almanac** (coarse orbits of all sats). A full ephemeris takes 18–30 s to download — hence slow **cold starts** (TTFF 30 s+).
4. **Pseudorange** = (receive time − transmit time) × c; transmit time comes from the code phase + navigation data time stamps.
5. **Velocity** from Doppler shifts; **time** as a by-product — GPS is the world's primary time-distribution system (cell towers, power grids, stock exchanges sync to it).

## 5. Error budget and corrections

| Error source | Typical magnitude | Mitigation |
|---|---|---|
| Ionospheric delay | 2–10 m (up to 50 m) | **Dual frequency** (L1+L5): delay ∝ 1/f², so combine to cancel; Klobuchar model on single-freq |
| Tropospheric delay | 2–3 m at zenith | Models (Saastamoinen), elevation mask |
| Satellite clock / ephemeris | ~1 m | Broadcast corrections, precise products (IGS) |
| **Multipath** (reflections off buildings) | 1–10 m+ in cities | Antenna design, L5's faster chipping rate (10.23 Mcps), 3D-mapping-aided GNSS |
| Receiver noise | 0.3–1 m | Averaging, carrier smoothing |

**Physics bonus — relativity:** satellite clocks run **+45 µs/day** fast due to weaker gravity (general relativity) and **−7 µs/day** slow due to orbital speed (special relativity) → net **+38 µs/day**. Uncorrected, that would accumulate ~11 km of error per day. The satellite clocks are deliberately tuned to 10.22999999543 MHz instead of 10.23 MHz to compensate.

### Accuracy tiers

| Method | Accuracy | How |
|---|---|---|
| Standalone single-freq (phone) | 3–10 m | Code pseudoranges |
| Dual-freq phone | 1–3 m | Iono-free combination |
| **SBAS** (WAAS, EGNOS, SDCM, GAGAN) | ~1 m | Geostationary satellites broadcast corrections |
| **DGPS** | sub-meter | Nearby reference station corrections |
| **RTK** (Real-Time Kinematic) | **1–2 cm** | Carrier-phase measurements + base station ≤ ~20–40 km, integer ambiguity resolution |
| **PPP** / PPP-RTK (Galileo HAS) | cm–dm | Precise orbit/clock corrections, no local base |

## 6. Assisted GNSS in phones

Phones fix in ~1–5 s instead of 30+ s because of **A-GNSS**:
- **Assistance data** (ephemeris, almanac, approximate time and location) downloaded over [cellular](cellular-3g-4g-5g.md)/Wi-Fi (SUPL protocol, or Google/Apple servers, or predicted "extended ephemeris" files).
- **Fused location**: GNSS + [Wi-Fi](wifi.md) BSSID databases + cell IDs + [sensors](../01-hardware/sensors-and-detectors.md) (accelerometer, gyro, barometer — dead reckoning in tunnels) via Kalman filtering.
- **Android** exposes raw GNSS measurements (`GnssMeasurement`: pseudorange rate, carrier phase, `AccumulatedDeltaRange`) since Android 7 — enabling research-grade processing on phones.

## 7. Output formats and OS interfaces

Most receivers stream **NMEA 0183** sentences over serial/USB:

```
$GPGGA,123519,4807.038,N,01131.000,E,1,08,0.9,545.4,M,46.9,M,,*47
        time   lat         lon         fix sats HDOP alt(MSL) geoid-sep
```

Parsing in Python:

```python
def parse_gga(s):
    f = s.split(",")
    def dm_to_deg(v, hemi):
        d = int(float(v) / 100); m = float(v) - d * 100
        return (d + m / 60) * (-1 if hemi in "SW" else 1)
    return {"lat": dm_to_deg(f[2], f[3]), "lon": dm_to_deg(f[4], f[5]),
            "fix": int(f[6]), "sats": int(f[7]), "hdop": float(f[8]), "alt_m": float(f[9])}
print(parse_gga("$GPGGA,123519,4807.038,N,01131.000,E,1,08,0.9,545.4,M,46.9,M,,*47"))
```

u-blox receivers also speak binary **UBX**; RTK uses **RTCM 3** correction streams (often via NTRIP over the internet).

| Platform | Stack |
|---|---|
| Linux | **gpsd** daemon (`gpsmon`, `cgps`, `gpspipe -w` JSON), ModemManager location (`mmcli -m 0 --location-get`), GeoClue2 for desktops; **chrony/ntpd** with GPS PPS for precise time; **RTKLIB** for post-processing/RTK |
| Windows | **Windows.Devices.Geolocation** API (`Geolocator`), Location Service (`lfsvc`), GNSS driver framework (GNSS DDI, `GnssAdapter`); Settings → Privacy → Location |
| Browser | `navigator.geolocation` (OS decides GNSS vs Wi-Fi) |

```js
navigator.geolocation.watchPosition(
  p => console.log(p.coords.latitude, p.coords.longitude, "±", p.coords.accuracy, "m"),
  e => console.error(e), { enableHighAccuracy: true });
```

```bash
sudo gpsd /dev/ttyACM0 -F /var/run/gpsd.sock && cgps -s
gpspipe -w | grep -m1 TPV    # JSON: lat, lon, alt, speed, time
```

## 8. Threats: jamming and spoofing

- **Jamming**: weak signals are easily drowned (cheap jammers; widespread military GNSS interference in conflict zones affects aviation).
- **Spoofing**: transmit fake GNSS signals to shift position/time (ships reported "at airports", drone hijack research).
- Defenses: multi-constellation/multi-frequency consistency checks, **Galileo OSNMA** (navigation message authentication, operational 2025), controlled reception pattern antennas, inertial cross-checks, monitoring C/N₀ and AGC anomalies.

## Further reading
- Misra & Enge, *Global Positioning System: Signals, Measurements, and Performance*
- Kaplan & Hegarty, *Understanding GPS/GNSS: Principles and Applications*
- IS-GPS-200 (L1/L2 interface spec); Galileo OS SIS ICD
- GNSS-SDR and RTKLIB open-source projects; `gpsd.io` documentation
