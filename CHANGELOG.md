# Changelog

## v0.1.0 — 2026-10-09

- Created dedicated project for **HitAir HAC/in-09H/N1_20Y**.
- Recorded OEM platform **Midea MSABA-09HRN1-QC2**.
- Recorded receiver board **EU-KFR26G/N1Y-AB1.D.01.XP1-1 / 17122000007157**.
- Recorded indoor wiring diagram **16022000003644**.
- Documented measured CN1/CN3 continuity.
- Confirmed CN3-2 and CN3-3 are electrically bridged in the factory configuration and both reach CN1-6.
- Rejected the initial assumption that CN3 can immediately be used as a conventional independent TX/RX UART.
- Added passive receive-only CN3 signal sniffer. No transmission to the air conditioner is possible in this firmware.
