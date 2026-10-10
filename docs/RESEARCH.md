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


## 2026-10-10 — why direct parallel TX did not work

The receive path is confirmed working, but commands transmitted by ESP through the parallel MOSFET injection were not accepted by the indoor unit.

A matching investigation of older Midea 5-pin CN3 receiver boards reports the same topology measured on this HitAir board: the CN3 **TX** and **REC** pins are factory-shorted (via bridge/jumper **J3**) when the optional Wi-Fi interface is not installed. In that configuration REC1 feeds the main board directly. To use the optional interface as a real man-in-the-middle, the TX/REC bridge must be opened; the inserted interface then receives the stock IR waveform on the REC side and retransmits it to the main-board TX side while also being able to inject its own commands.

This matches our measurements:
- CN3-2 <-> CN3-3 ~0.2 ohm;
- both reach CN1-6;
- stock-remote Coolix is visible on the shared node;
- passive RX works reliably;
- parallel ESP TX changes HA state but the indoor unit does not react.

### Revised architecture

The target architecture is now:

REC1 -> CN3 REC side -> ESP RX -> ESP forward/inject -> CN3 TX side -> CN1/main board

The factory TX/REC bridge must be separated before this architecture can work correctly.

After separation:
- stock remote must still work because ESP forwards every received raw Coolix waveform;
- Home Assistant commands are injected on the TX/main-board side;
- HA state is updated from the REC side;
- no optical IR LED is required;
- no transistor connection to the REC1 package itself is required.

Until the bridge location on this exact EU-KFR26G/N1Y-AB1.D.01.XP1-1 board is positively identified, do not cut traces or remove jumpers.

## 2026-10-10 — TX self-test clarification

The TX self-test returned `FAIL - CN3 stayed HIGH`, but this was expected because **no TX hardware had been installed yet**. At this stage the ESP was connected only for passive RX through the 47k/68k divider to GPIO4. GPIO6 was not physically connected to CN3 through a MOSFET or any other driver.

Therefore this test does **not** indicate a wiring fault and does not provide evidence about CN3 transmit capability. The next real hardware step is to add a safe open-drain TX driver before testing transmission.


## 2026-10-10 — TX hardware works; strict two-frame Coolix required

After the NPN pull-down driver was physically installed on GPIO6, the ESP received its own transmitted waveform back on GPIO4:

- TX 0xB23FC0 -> RX 0xB23FC0
- TX 0xB27BE0 -> RX 0xB27BE0
- TX 0xB5F5A5 -> RX 0xB5F5A5

Therefore the GPIO6 -> transistor -> CN3 signal path is electrically working.

However ESPHome logged these manually generated test packets as **Received unstrict Coolix: [0x...]**, while the stock HitAir remote is logged as **Received Coolix: 0x...**.

ESPHome's Coolix protocol implementation defines strict Coolix as two identical frames (first == second). A Coolix action with only `first:` sends only one frame. When `second:` is supplied, the encoder sends a second frame after the Coolix inter-frame space.

Working hypothesis: this HitAir/Midea main board requires the strict two-frame Coolix packet used by the stock remote and ignores a single-frame/unstrict packet.

Next firmware revision sends every explicit test and mute command with:
- first: CODE
- second: CODE

The native ESPHome Coolix climate path is also retained for comparison.


## 2026-10-10 — strict Coolix TX confirmed on real unit

With the NPN open-collector driver installed on GPIO6 -> CN3 signal line, the strict two-frame Coolix tests were successful on the actual HitAir unit.

Confirmed:
- `0xB23FC0 / 0xB23FC0` -> unit beeped and entered COOL 25 °C HIGH;
- automatic persistent `0xB5F5A5 / 0xB5F5A5` followed after power-on;
- `0xB27BE0 / 0xB27BE0` -> unit switched OFF;
- OFF was executed silently after the B5F5A5 command, confirming that the Display/Sound toggle affects the buzzer on this HitAir revision;
- explicit B5F5A5 while OFF produced a beep, so mute tests should be performed while the unit is ON.

ESP RX saw every transmitted command back as **STRICT** Coolix, proving the complete TX electrical path and double-frame format.

Next validation stage: control exclusively from the Home Assistant climate entity (temperature, fan speeds, modes, swing, OFF/ON restore) while checking physical response and stock-remote reverse synchronization.
