# Research log

## 2026-10-09 — initial CN3 investigation

### Goal

Determine whether the unpopulated CN3 footprint can be used as a native Midea Wi-Fi UART so the project can avoid optical IR hardware and retain a clean ESPHome architecture.

### Result of continuity tests

The two middle CN3 pads are electrically the same node:

- CN3-2 <-> CN3-3: ~0.2 Ω
- CN3-2 <-> CN1-6: ~0.3 Ω
- CN3-3 <-> CN1-6: ~0.2 Ω

Power rails:

- CN3-4 -> +5V rail
- CN3-5 -> GND
- CN3-1 not yet identified

### Current conclusion

The factory CN3 on this specific receiver board cannot be treated as a normal independent TX/RX UART.

The measurements match reports for older Midea receiver/display boards with a 5-pin CN3:
- the two center contacts are labeled conceptually as TX and REC;
- in non-Wi-Fi builds they are bridged so the IR receiver output passes directly to the main board;
- an optional Wi-Fi subassembly breaks/intercepts this path and provides a conventional 4-wire Wi-Fi UART to the actual Wi-Fi dongle.

This means there are two realistic future architectures.

## Architecture A — official-style Wi-Fi subassembly + ESP UART

Use the Midea/Carrier-style adapter board (example family CE-26GY-2.4G, part 17122000046741) between this receiver board and an ESP UART dongle.

Advantages:
- gives a conventional Midea Wi-Fi UART interface;
- likely allows the standard ESPHome `midea` component;
- no custom raw IR timing code in the ESP project;
- simplest route to reliable feature parity.

Disadvantages:
- needs an additional adapter board;
- compatibility with this exact receiver revision still needs confirmation.

## Architecture B — direct CN3 man-in-the-middle

Use ESP directly on the REC/TX path:
- receive the demodulated IR signal from the stock receiver;
- forward it to the main board;
- inject commands from Home Assistant on the same electrical path;
- decode stock-remote commands to keep HA state synchronized.

Advantages:
- no external Midea adapter board;
- stock remote remains supported;
- no optical IR LED is needed.

Disadvantages:
- still relies on the Midea IR command protocol electrically;
- requires separating/intercepting the factory REC/TX bridge for true man-in-the-middle operation;
- state is inferred from commands unless additional main-board feedback is reverse-engineered.

## Decision gate

Before altering the board, first prove what CN3-2/3 carries.

Next measurements:
1. CN3-4 to CN3-5 DC voltage while powered.
2. CN3-2/3 to CN3-5 idle DC voltage.
3. Passive capture of CN3-2/3 while pressing the stock remote.

### Powered DC check

The shared CN3-2/CN3-3 node was measured at approximately **+5 V relative to CN3-5** in the idle state.

A handheld multimeter showed only small / unstable changes (~0.2 V scale) during commands. This is expected: an IR receiver output is a fast pulse train, so a DC multimeter averages the waveform and cannot reliably display it.

**Important:** the next pulse test must be performed with the physical stock HitAir remote pointed at REC1. A voice command through Alice is not a valid test unless it is known to drive this exact onboard IR receiver.

### Next step — passive waveform capture

Use the receive-only ESP32-C3 firmware:
`esphome/hitair-cn3-passive-sniffer-v0.1.0.yaml`

Physical connection:
- CN3-5 -> ESP GND;
- CN3-2 (or CN3-3, they are the same node) -> 47k -> GPIO4;
- GPIO4 -> 68k -> CN3-5 / ESP GND;
- ESP powered from an isolated USB power bank;
- no ESP output connected to the air conditioner;
- no USB cable to a PC while the ESP is electrically connected to the powered air conditioner.

Capture this stock-remote sequence:
1. OFF / idle for ~10 s;
2. POWER ON;
3. COOL 24 °C;
4. COOL 25 °C;
5. FAN LOW;
6. FAN MEDIUM;
7. FAN HIGH;
8. FAN AUTO;
9. HEAT 24 °C;
10. SWING;
11. OFF.

If the shared node produces demodulated remote pulses, the 5-pin CN3 architecture is confirmed.


## 2026-10-09 — passive capture confirmed Coolix on CN3

The passive GPIO4 capture produced three clean 200-symbol frames on the shared CN3-2/CN3-3 node.

Manual decoding of the captured Pronto timings gives:

| Time | Decoded Coolix | Meaning |
|---|---|---|
| 20:06:45 | `0xB23F40` | COOL 24 °C, fan HIGH |
| 20:06:53 | `0xB23FC0` | COOL 25 °C, fan HIGH |
| 20:06:59 | `0xB27BE0` | OFF |

This is decisive confirmation that CN3-2/CN3-3 carries the demodulated Coolix control waveform used by the indoor main board.

The stock receiver output is idle-high at approximately 5 V and pulls low for marks.

### Architectural decision for v0.2

For the next stage we do **not** touch REC1 itself and we do **not** use an optical IR LED.

- RX: passive read of CN3-2/3 through the existing 47k/68k divider to GPIO4.
- TX: open-drain style pull-down of the same CN3-2/3 node using an N-MOSFET driven by GPIO6.
- ESP32-C3 power: CN3-4 (+5 V) / CN3-5 (GND), after the separate power test succeeded.
- Native ESPHome Coolix climate gets `receiver_id`, so the stock remote updates Home Assistant state.
- `0xB5F5A5` is used for the persistent Display/Sound toggle, matching the already proven Midea/Castorama project.

A direct ESP GPIO must not be connected to the 5 V signal node. The MOSFET is the safe level-isolated pull-down driver; it is connected at CN3, not at the IR receiver package.
