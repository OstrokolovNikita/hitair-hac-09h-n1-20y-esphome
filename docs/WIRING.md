# Wiring — v0.2 direct CN3 control

Target: **HitAir HAC/in-09H/N1_20Y** / receiver board **EU-KFR26G/N1Y-AB1.D.01.XP1-1**.

## Power

- CN3-4 (+5 V) -> ESP32-C3 **5V/VBUS**
- CN3-5 (GND) -> ESP32-C3 **GND**

Do not use the ESP 3V3 pin for the 5 V rail.

## Receive path

The shared CN3-2/CN3-3 line is approximately 5 V when idle.

```text
CN3-2 (or CN3-3) ---- 47k ----+---- GPIO4
                               |
                              68k
                               |
CN3-5 / GND -------------------+---- ESP GND
```

The divider gives approximately 3.0 V at GPIO4 from a 5 V idle level.

## Transmit path

A GPIO must **not** be connected directly to the 5 V CN3 signal.

Use the same safe open-drain pull-down idea already proven in the Midea/Castorama project:

```text
GPIO6 ---- 1k ---- Gate  AO3400 / A09T
                    |
                   100k
                    |
GND ----------------+

MOSFET Source ------ CN3-5 / GND
MOSFET Drain  ------ CN3-2 OR CN3-3
```

Only one of CN3-2/CN3-3 is needed because continuity measurements confirmed they are factory-bridged on this board.

The MOSFET is **not connected to REC1 itself** and the receiver remains untouched. The MOSFET only pulls the existing CN3 baseband control line low, exactly like the demodulated receiver output.

## Optional supply decoupling

A 100 nF capacitor marked `104` may be placed between ESP 5V and GND near the ESP board. Add a larger 100–220 µF electrolytic later only if power instability/brownout is actually observed.
