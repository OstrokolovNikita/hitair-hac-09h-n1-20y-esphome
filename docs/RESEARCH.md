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

If the shared node produces demodulated remote pulses, the 5-pin CN3 architecture is confirmed.
