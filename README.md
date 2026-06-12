# SenseCAP T1000-E ~ Meshtastic GPS Tracker Setup

A working reference for setting up a [**SenseCAP T1000-E**](https://www.seeedstudio.com/SenseCAP-Card-Tracker-T1000-E-for-Meshtastic-p-5913.html) as a private-but-publicly-relayed asset tracker on a [Meshtastic](https://meshtastic.org/) network. Written around a real deployment: tracking a **work bag** carried on a daily commute, riding an existing private network that sits on the public US915 frequency for maximum relay reach.

> **Scope check before you build:** A Meshtastic tracker only reports while a node on the same preset/frequency can hear it (directly or via relay). It is **not** a cellular/Tile/AirTag-style recovery device. If the bag leaves all mesh coverage, it goes dark until it returns. This setup is for *known-route / known-location* tracking, not stolen-asset recovery anywhere. Because of this I have and use a [Tracki 4G GPS Tracker](https://tracki.com/) in addition to the T1000-E.

---

## 1. Hardware at a Glance:

| Component | Part | Notes |
|---|---|---|
| MCU | Nordic **nRF52840** | Not ESP32; this matters for power config (see §6) |
| LoRa radio | Semtech **LR1110** (863–928 MHz) | SX126x-family interop; cannot *receive* from legacy SX127x radios |
| GNSS | MediaTek **AG3335** | GPS only in Meshtastic (no WiFi/BT indoor positioning) |
| Battery | 700 mAh LiPo | Magnetic charging cable |
| Sensors | Temp, light, **3-axis accelerometer** | **Accelerometer is NOT used by Meshtastic firmware** |
| Enclosure | IP65, ~6.5 mm thick | Pocket/asset-friendly |

Two consequences worth internalizing up front, because they shape every decision below:

- **No motion gating.** The accelerometer is on the board but unused by Meshtastic. The device cannot tell "moving" from "stationary" except by comparing GPS fixes. A clean "fast while moving / slow while idle" profile is therefore **not achievable**; you pick one cadence.
- **nRF52, not ESP32.** Several power-module sleep timers are ESP32-only and do nothing here (see §6).

---

## 2. Flash / Update Firmware:

1. Open the [**Meshtastic Web Flasher**](https://flasher.meshtastic.org/) (Chrome/Edge).
2. Select target: **Seeed Card Tracker T1000-E**.
3. Enter DFU: hold the button and quickly connect the magnetic charge cable **twice** → a `T1000-E` USB drive appears.
4. Flash the **erase** UF2 first; let the drive drop.
5. Flash the firmware UF2.

> **Firmware version:** The 2.5.x line had a GPS-accuracy regression (reported 2–3 km errors) versus the solid 2.4.2. That was mid-2025, so check the current Meshtastic release notes / Seeed forum for the latest **stable tracker build** and any open GPS issues before taking "latest." For a tracker, accuracy regressions matter more than features. Position-precision-per-channel (§4) requires **2.7.1+**.

---

## 3. Radio / LoRa Config:

This is the layer that decides **who can hear and relay you**. To be relayed by the broad public mesh while staying private, everything here must match the public mesh's radio settings; only the *key* differs.

| Setting | Value | Why |
|---|---|---|
| Region | `US` | Required; sets legal band |
| Modem Preset | **Long Fast** | Must match the public mesh or no one relays you |
| Frequency Slot | **20** | US915 LongFast public slot (906.875 MHz). Pinning this is the whole trick; see below |
| Hop Limit | `3` | Default. Do **not** raise it; higher hops worsen congestion |
| Transmit Enabled | `On` |; |
| Transmit Power | Max | Firmware clamps to legal/hardware ceiling; you want range for pickup |
| Override Duty Cycle | `Off` | EU concern, not US |
| Frequency Offset / Override Frequency | default / blank | The slot system handles frequency |
| Boosted RX Gain | `On` (receiver nodes) | Helps your catch-nodes hear the bag; moot on the bag itself |

### The name vs. slot vs. PSK distinction (the key insight)

Three separate things people conflate:

- **Frequency slot** = the actual RF frequency. When left at `0/auto`, Meshtastic *hashes the primary channel name* to pick a slot. A custom primary name (like `PNW_LoRa`) therefore lands you on a **non-public frequency**, invisible to everyone else. **Pinning the slot to 20 overrides the name hash** and puts you on the public frequency regardless of channel name.
- **Channel name + PSK** → hashed into the 1-byte **channel hash** in each packet header, which tells a receiver *which key to try*. Public LongFast (`LongFast` + `AQ==`) hashes differently than your `PNW_LoRa` + private key.
- **Relay does not require the key.** Nodes rebroadcast any packet sharing their modem settings (preset + slot), *regardless of whether they can decrypt it*. So public nodes on slot 20 relay your encrypted bag packets; only your nodes decode them.

**Net result:** keep your network name (`PNW_LoRa`), keep your private PSK, just pin **slot 20** + **Long Fast** preset. Your existing private network now rides the public mesh.

> **Sanity check:** after applying, confirm all devices (bag + receivers) show the same frequency (~**906.875 MHz**). A mismatch means the slot didn't take and they won't hear each other regardless of keys.

### MQTT flags (on this screen)

| Setting | Value | Why |
|---|---|---|
| OK to MQTT | **Off / false** | Requests public internet gateways not upload your packets to public MQTT/maps. Keeps the commute off public maps even while RF-relayed |
| Ignore MQTT | default |; |

---

## 4. Channel & Privacy Config (PRIMARY Channel):

Telemetry and position **only ride the PRIMARY channel**; so the private key must live on slot 0, not a secondary.

| Setting | Value | Why |
|---|---|---|
| Role | **PRIMARY** | Position/telemetry only broadcast on primary |
| PSK | **256-bit, your `PNW_LoRa` key** | Keep your existing key so receivers still decode. Don't regenerate unless you re-clone to all nodes |
| Name | `PNW_LoRa` | Name no longer drives frequency (slot pinned to 20), so keep your identity |
| Location (position precision) | **Precise Location** | Private channel → you want exact bag coordinates, not the obfuscated tiers (those are for public channels) |
| Uplink Enabled | `Off` | No public MQTT bridging |
| Downlink Enabled | `Off` |; |

> Clone the channel to your receiver nodes via **Export → Import** (or QR) rather than retyping the key.

---

## 5. Device Role:

| Setting | Value | Why |
|---|---|---|
| Role | **TRACKER** | Deep-sleeps between GPS broadcasts; radio powers off during sleep |
| Rebroadcast Mode | **LOCAL_ONLY** | Bag stays a quiet endpoint, doesn't burn battery relaying others' traffic. (Does not stop others from relaying the bag) |

---

## 6. Power Config:

On the **nRF52** T1000-E, only **two** fields here do anything useful. The rest are ESP32-only or for hardware this device lacks.

| Field | Value | Why |
|---|---|---|
| **Enable power saving mode** | **On** | The real lever. In TRACKER role this also sleeps the LoRa radio between broadcasts → days of battery instead of ~1. Safe here because the T1000-E has a wake button |
| **Shutdown on battery delay** | **0** | 0 = run on battery indefinitely (non-zero powers off after losing charge power; the opposite of what a tracker wants) |
| Light Sleep Duration (`ls_secs`) | leave default | **ESP32-only, no effect on nRF52** |
| Minimum Wake Time (`min_wake_secs`) | leave default | Tied to the same ESP32 light-sleep path; inert here |
| Super Deep Sleep Duration (`sds_secs`) | leave default (`0`) | `0` = use default (~1 yr / button-wake), i.e. "stay off until I press the button." Correct for a tracker |
| ADC Multiplier Override | blank / `0` | Only touch if battery % reads visibly wrong; variant default is calibrated |
| No-Connection BT Disabled (`wait_bluetooth_secs`) | default | Low impact on this device |
| INA219 Address | ignore | For boards with an external INA-2XX monitor; T1000-E has none |

> **Don't chase the SDS *duration* to control sleep behavior.** *When* it enters SDS is governed by the comms-timeout (~2 hr of no mesh contact), not this field. Practical effect: a bag left with zero mesh contact for hours can fall into near-off and won't resume until power-cycled; a non-issue on a regular commute, but the reason receiver coverage matters.

---

## 7. Position Profiles:

Because there's no motion gating (§1), this collapses to **one cadence** + a charging strategy. Two profiles, depending on how you charge.

### Profile A; Charge Nightly *(recommended for a bag that's home every night)*

Best tracking; battery is a non-issue since you top up the magnetic cable each evening.

```bash
meshtastic --set position.gps_mode ENABLED
meshtastic --set position.gps_update_interval 60
meshtastic --set position.position_broadcast_secs 120
meshtastic --set position.position_broadcast_smart_enabled true
meshtastic --set position.broadcast_smart_minimum_distance 50
meshtastic --set position.broadcast_smart_minimum_interval_secs 60
```

Fix ~every minute, report every 1–2 min → a real breadcrumb trail of the commute.

### Profile B; Weekend Charge, ~5-day Target Battery Life

Charge Friday night, run Mon–Fri untouched. Single 5-min cadence (idle/moving can't be distinguished).

```bash
meshtastic --set position.gps_mode ENABLED
meshtastic --set position.gps_update_interval 300
meshtastic --set position.position_broadcast_secs 300
meshtastic --set position.position_broadcast_smart_enabled true
meshtastic --set position.broadcast_smart_minimum_distance 100
meshtastic --set position.broadcast_smart_minimum_interval_secs 300
```

**Middle ground:** `gps_update_interval 120` / `position_broadcast_secs 180` ≈ 3 days with still-decent tracks.

### Field Meanings:

| Field | Meaning |
|---|---|
| `gps_update_interval` | How often the GPS powers on to fix. **The dominant power knob** (runs 24/7; no motion gating to suppress it) |
| `position_broadcast_secs` | Guaranteed max interval between broadcasts (the "report at least this often" floor) |
| `position_broadcast_smart_enabled` | Skip redundant broadcasts when not moving (saves airtime, not GPS power) |
| `broadcast_smart_minimum_distance` | Meters moved before a smart broadcast is eligible |
| `broadcast_smart_minimum_interval_secs` | Rate-limit / floor between smart broadcasts |

> **Indoor battery tax:** sitting indoors at a desk, the GPS can't fix and stays powered hunting until timeout *every cycle*. This is the main thing that can drop Profile B below 5 days. Payoff is low indoors anyway (best it reports is the fix from walking in), so "near the office" is usually all you get; which is fine for "is it there / did it leave."

> **Public-slot courtesy:** you're a guest on the congested metro slot 20. Keep the cadence in the few-minutes range; a 30-second beacon spams shared airtime and gets your packets deprioritized in relay.

---

## 8. MQTT Module; OFF

Separate from the channel uplink/OK-to-MQTT toggles. This is the master switch.

```bash
meshtastic --set mqtt.enabled false
```

- Disabling the module also covers the **Map Reporting** sub-toggle (which would push you to the public map).
- On the bag it's effectively a no-op (no IP path without a connected phone), but off is the clean resting state.
- **Re-enable later only on a gateway node** pointed at *your own* broker if you build a private dashboard; never the default public broker.

---

## 9. Module Config:

A battery tracker on a shared slot should transmit as little as possible. Enable almost nothing.

| Module | State | Why |
|---|---|---|
| **Telemetry** (device metrics) | **On**, 30–60 min interval | Read **battery level remotely** without pulling the bag out. Earns its airtime |
| Ext Notif | *Optional* | Triggers the buzzer remotely → "ring my bag" locate. Costs nothing idle |
| Range Test | *Temporary only* | Useful to probe commute coverage on a test drive, then **turn off**; it spams sequenced packets |
| **Neighbor Info** | **Off** | Airtime-heavy, actively discouraged on dense public meshes. You already see link info per-packet (hop count + relay ID) |
| Paxcounter | Off | Scans WiFi/BLE → wrecks battery |
| Audio / Canned / Ambient Lighting / Serial / Detection Sensor | Off | Irrelevant to a tracker |
| Store & Forward | Off | A powered-router feature; a sleeping tracker can't S&F anyway |

---

## 10. Receiver / Catch nodes:

You only need nodes where you want guaranteed pickup; here: **downtown office** (high-value: dense relay zone) and **home (162nd)**. Each must be **Meshtastic** on the identical Region / Long Fast / **Slot 20** + the `PNW_LoRa` channel (name + PSK) to both hear and decode the bag.

- Enable **Boosted RX Gain** on these.
- Mains-powered, so cadence/power don't matter; Telemetry on is fine if you want their stats.
- A `PNW_LoRa` Reticulum/raw-LoRa node will **not** work; must be Meshtastic on matching settings.

---

## 11. Viewing the Tracker:

### On a LilyGO T-Deck (standalone, no phone)

The T-Deck is just another node with a screen. Put it on the same mesh (Region US / Long Fast / Slot 20 + import `PNW_LoRa` channel via QR/URL), then:

- **Nodes list** → the bag shows last position, distance, last-heard, battery. Tap for detail → exact coords, **Request Position** (force a fresh fix), **Traceroute** (see the relay path / whether a stranger's node carried it).
- **Own location:** T-Deck **Plus** has GPS and self-locates (live distance/bearing). **Base** T-Deck has no GPS → set a fixed position; distance readouts are only meaningful while you're at that point.
  ```bash
  meshtastic --set position.fixed_position true --setlat 45.63 --setlon -122.55
  ```
- **Offline map:** the MUI map screen renders from **map tiles on the SD card** (256×256 PNGs under `/map`). Pre-load tiles for the Portland/Vancouver corridor once (e.g. the `tdeck-maps` tool, zoom ~8–14) and the bag plots as a pin; no internet.

### Easy alternative

The phone Meshtastic app paired to any of your nodes has a full online Mesh Map; strictly easier, just not standalone.

---

## Quick reference; consolidated CLI

```bash
# --- Radio / LoRa (apply to bag AND receiver nodes) ---
meshtastic --set lora.region US
meshtastic --set lora.modem_preset LONG_FAST
meshtastic --set lora.channel_num 20          # frequency slot 20 = US915 public
meshtastic --set lora.tx_enabled true
# (set TX power to max, Boosted RX gain on for receiver nodes, in-app or via lora.* )

# --- Channel (PRIMARY): set name + private PSK, Precise location, MQTT off ---
# Easiest via app Export/Import (QR). Set Location = Precise, Uplink/Downlink = off,
# and OK-to-MQTT = false on the LoRa screen.

# --- Role ---
meshtastic --set device.role TRACKER
meshtastic --set device.rebroadcast_mode LOCAL_ONLY

# --- Power ---
meshtastic --set power.is_power_saving true
meshtastic --set power.on_battery_shutdown_after_secs 0

# --- Position (Profile A: charge nightly) ---
meshtastic --set position.gps_mode ENABLED
meshtastic --set position.gps_update_interval 60
meshtastic --set position.position_broadcast_secs 120
meshtastic --set position.position_broadcast_smart_enabled true
meshtastic --set position.broadcast_smart_minimum_distance 50
meshtastic --set position.broadcast_smart_minimum_interval_secs 60

# --- Telemetry (remote battery) ---
meshtastic --set telemetry.device_update_interval 1800

# --- MQTT module off ---
meshtastic --set mqtt.enabled false
```

---

## Deployment notes / verification checklist

- [ ] All devices report **~906.875 MHz** (slot 20 took)
- [ ] Bag appears in a receiver node's node list (proves slot + preset + key align)
- [ ] Bag gets a real GPS fix outdoors (coords populate, not blank)
- [ ] Battery slope checked over the first work week (Mon vs Fri); adjust `gps_update_interval` if it runs ahead of target
- [ ] Traceroute on the bag occasionally shows an unknown relay node = public-mesh relay is working
- [ ] Reminder: off-route (theft) = no coverage. This tracks known routes, not recovery.

---

*Environment for this build: `PNW_LoRa` Meshtastic network, US915 slot 20 (public), private 256-bit PSK; work-bag use, ~M–F 11:00–18:00, home (162nd Ave) ↔ downtown office; receiver nodes at home + office; viewer: T-Deck and/or phone app.*
