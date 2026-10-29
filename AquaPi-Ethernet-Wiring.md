# AquaPi Ethernet Build — Wiring & Peripheral Reference

Reference for the customized **Standard – Inverted Optical (Ethernet)** build
(`aquapi_config_ethernet.yaml`). This file is referenced from the config header
and `common/ethernet.yaml`.

## WIZ850io (W5500) Ethernet module

Power from the AquaPi **3V3 rail only** (the module is 3.3 V — 5 V will kill it).
VSPI bus, broken out on the AquaPi AUX terminal blocks:

| ESP32 pin | AquaPi terminal | WIZ850io pin | Signal |
|-----------|-----------------|--------------|--------|
| GPIO18 | AUX1 | J1-4 | SCLK |
| GPIO23 | AUX3 | J1-3 | MOSI |
| GPIO19 | AUX1 | J2-6 | MISO |
| GPIO25 | DAC1 | J1-5 | SCSn (chip select) |
| GPIO17 | AUX6 | J1-6 | INTn (interrupt) |
| GPIO4  | AUX6 | J2-5 | RSTn (ESPHome drives the reset pulse) |

Bus runs at the default 26.67 MHz (an early 0x00 version-mismatch was a MOSI
miswire — D3 instead of D23 — not signal integrity).

**Consequences of claiming these pins:** Ethernet and Wi-Fi are mutually
exclusive (use `device_base_ethernet.yaml`, not the stock base). The stock
binary-sensor pins 4/17/18/19/23 are taken by SPI, so `binary_ethernet.yaml`
keeps only Binary Sensor 1 (GPIO13) and drops the GPIO27 leak sensor — which
frees GPIO27 for the flow meter below. Cloud HTTP-OTA (`ota_https`) is
intentionally omitted: it flashes stock Wi-Fi firmware and would factory-reset
the device off the network. Update via `git pull && esphome run
aquapi_config_ethernet.yaml`.

## RODI flow meter — GREDIA GR-402B (`common/rodi_flow.yaml`)

Hall-effect turbine, G1/4", mounted on the RODI **feed** line.

| Wire | Connect to | Note |
|------|-----------|------|
| Red | **5 V** (any aux-block 5V terminal) | Changed from 3V3 2026-07-12 — 5V is USB VIN, keeps load off the shared 3V3 rail (I2C + W5500). Signal-safe: output is open-collector (pull-low only), line idles at 3.3 V via the ESP32 pull-up |
| Black | GND (same aux block) | |
| Yellow | GPIO27 (D27 terminal, AUX2) | internal pull-up enabled in config |

Entities: `RODI Flow Rate` (gal/h), `RODI Total Volume` (gal, survives
reboots), `RODI This Run` (gal, resets when a run starts), `RODI Producing`
(on >1.6 gal/h, off <0.8 gal/h, 30 s debounce). Spec K-factor is 38.0
(F(Hz) = 38 × Q(L/min)); bucket-test and tune `flow_k_factor` for accurate
volume — the runtime watchdog doesn't depend on calibration.

**Status (2026-07-12):** firmware with this package flashed; meter not yet
physically installed (GPIO27 pull-up reads 0 gal/h until wired). HA created
the entities as `*.reef_tank_sump_aquapi_rodi_*` (IDs derive from the current
device friendly name, not the historical `aquapi_60fe10` prefix) — they were
renamed in the entity registry to `*.aquapi_60fe10_rodi_*` to match the Reef
Command dashboard, the runaway automation, and the Reef RODI Made Today
utility meter. Expect the same prefix mismatch for any future entities added
to this device.

## Optical level sensors (`common/water_level.yaml`)

Inverted logic (`optical_inverted: true`): HA/ESPHome **on = wet**.

| Sensor | Pin | Lead color |
|--------|-----|------------|
| High | GPIO32 | blue |
| Low | GPIO33 | yellow |

The `Water Level` text sensor derives status: both wet = **High**, low wet +
high dry = **Normal**, both dry = **Low**, high wet + low dry = **Error**
(impossible state — check wiring/mounting). The Reef Command dashboard's Sump
Level card reads this sensor directly.

## Other peripherals (unchanged from stock)

| Peripheral | Pin(s) |
|------------|--------|
| Dallas DS18B20 temp probes (×2) | GPIO16 |
| I2C bus (EZO pH / EC / ORP / DO carriers) | SDA GPIO21 (yellow), SCL GPIO22 (blue) |
| Binary Sensor 1 | GPIO13 (yellow) |

## Non-electrical equipment that moves the readings

Nothing here is wired to the AquaPi. Recorded so a step change in a probe
reading right after an install is not chased as a probe or firmware fault.

### BRS GFO & Carbon Reactor – Deluxe

**Status (2026-10-29):** delivered, **not yet installed**. Update this line with
the install date and the media loaded.

**Initial run mode:** pump on 24/7, flow set by hand with the included ½" QC
ball valve (blue handle). No AquaPi or Home Assistant control yet.

| Spec | Value |
|------|-------|
| Housing | single 10" canister, refillable 10" media cartridge |
| Capacity | 3 cups carbon (~300 g) **or** 2 cups GFO (~290 g) — one media per canister |
| Plumbing | ½" push-connect, ½" tubing (6 ft supplied), ½" QC ball valve for flow |
| Size | 13" H × 6" W × 5" D |
| Pump | Sicce Syncra 1.5 (357 GPH max) on the "with pump" SKU; flow trimmed with the ball valve |
| Extras | built-in pressure-release (air bleed) valve, canister wrench |

What to expect on the AquaPi after it goes online:

- **Carbon:** ORP commonly rises over the first days as carbon strips
  organics/yellowing compounds. A climb after install is the media working, not
  ORP probe drift — don't recalibrate ORP on the strength of it.
- **GFO:** lowers phosphate (not measured here). Fresh GFO can also take a
  little alkalinity at first, which can show as a small pH dip; ramp GFO
  in gradually and test PO₄/alk by hand.
- **Neither** should move temperature, EC/salinity or DO noticeably. A change
  in those around the install points at something else (e.g. the probe was
  disturbed while plumbing in the sump).

**Planned automation (not built yet):**

1. **Pump monitoring and control.** Switch the reactor pump on and off, and
   confirm it is actually running, not just powered. How depends on how it is
   powered (smart plug in Home Assistant or a relay on the AquaPi). Decide
   before install so the pump goes on the right outlet.
2. **Media refills and timing.** Track when the media was loaded and how many
   days it has been running, and remind when it is due for a change.
3. **Phosphate-driven runs (after a Neptune Trident NP is added).** Use the
   Trident NP's nitrate/phosphate results to decide when the GFO reactor runs,
   instead of 24/7. This needs the Trident NP readings in Home Assistant first;
   the Trident NP is not wired to the AquaPi.
