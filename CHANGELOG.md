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


## v0.2.0 — 2026-10-09

- Passive CN3 capture confirmed the stock control path is Coolix.
- Captured/decoded: 0xB23F40 = COOL 24 °C HIGH, 0xB23FC0 = COOL 25 °C HIGH, 0xB27BE0 = OFF.
- RX polarity corrected to active-low/inverted for native Coolix decoding.
- Added native ESPHome Coolix climate with receiver synchronization from the stock remote.
- Added direct CN3 TX through a safe N-MOSFET pull-down on GPIO6.
- Added persistent "Без звука" using B5F5A5 with automatic reapply after OFF -> ON.
- Added remembered fan speed, last active mode, HEAT -> 30 °C and COOL -> 25 °C behavior from the Midea/Castorama project.
