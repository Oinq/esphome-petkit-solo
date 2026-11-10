# Changelog

## 0.2.0 — 2026-11-09

- **Time zone from Home Assistant.** The `homeassistant` time source no longer sets a time zone,
  so Home Assistant (2026.3 or newer) sends its own on every sync, including daylight saving.
  0.1.0 images had `Europe/London` fixed, so meals ran one hour late elsewhere. The `timezone`
  substitution (default `UTC`) now only applies until Home Assistant connects.
- **Rotation pulse timeout raised from 6 s to 10 s.** With food a portion takes ~5.9 s.
- **Optional Bluetooth proxy** (`packages/bluetooth-proxy.yaml`): active proxy, BLE scanning
  paused while the motor runs, *Bluetooth proxy* on/off switch, free-heap diagnostics.
- `before_feed` / `after_feed` script hooks for optional packages (no-op by default).

## 0.1.0 — 2026-11-08

- First release: portion feeding with motor protection, four meals stored on the feeder,
  Wi-Fi onboarding (access point, Improv serial and BLE), Home Assistant adoption.
