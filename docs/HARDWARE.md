# Hardware identification

## Unit

- Brand/model: **HitAir HAC/in-09H/N1_20Y**
- OEM platform: **Midea MSABA-09HRN1-QC2**
- Capacity: **9000 BTU/h**
- Type: **fixed-speed / ON-OFF heat pump**
- Production date on label: **12.20**
- Indoor wiring diagram: **16022000003644**

## Display / receiver board

- PCB marking: **EU-KFR26G/N1Y-AB1.D.01.XP1-1**
- Part number: **17122000007157**
- CN1: 8-pin connection to the indoor main board
- CN2: 5-pad footprint, silk: blank / R- / R+ / GND / +5V
- CN3: 5-pad unpopulated footprint

The board is a simple receiver/display board; no populated UART/Wi-Fi controller IC is visible on this revision.

## Confirmed continuity measurements

Numbering is left-to-right exactly as viewed from the component side in the project photos.

### CN1 relative to CN2 rails

- CN1-7 -> CN2 +5V: ~0.2 Ω
- CN1-8 -> CN2 GND: ~0.2 Ω

Therefore:
- **CN1-7 = +5V rail**
- **CN1-8 = GND rail**

### CN3

- CN3-2 <-> CN3-3: ~0.2 Ω
- CN3-2 <-> CN1-6: ~0.3 Ω
- CN3-3 <-> CN1-6: ~0.2 Ω
- CN3-4 <-> CN1-7 (+5V): ~0.5 Ω
- CN3-5 <-> CN1-8 (GND): ~0.2 Ω
- CN3-1: no direct continuity to CN1 found in the initial matrix

Current working map:

| CN3 pin | Confirmed electrical role |
|---|---|
| 1 | NC / optional rail — not yet confirmed |
| 2 | shared signal node |
| 3 | shared signal node |
| 4 | +5V rail |
| 5 | GND |

Pins 2 and 3 are **not independent UART TX/RX in the factory configuration**. They are electrically bridged and both connect to CN1-6.

## Interpretation

This matches the known older Midea 5-pin CN3 receiver-board architecture: the two middle contacts are commonly described as **TX** and **REC**, but are bridged in the non-Wi-Fi configuration so the IR receiver path passes directly to the indoor main board.

Some Midea variants use a separate Wi-Fi subassembly (for example CE-26GY-2.4G / 17122000046741) which is inserted at this 5-pin interface and exposes the usual 4-wire Wi-Fi UART on the other side.

### Powered measurements (2026-10-09)

With the receiver board connected and powered:

- CN3-5 is the reference ground already confirmed by continuity to CN1-8 / CN2 GND.
- CN3-2 and CN3-3 sit at approximately **+5 V relative to CN3-5 while idle**.
- Reversing the meter leads (black on CN3-2/3, red on CN3-5) therefore reads approximately **-5 V**. This is expected and confirms the polarity; it does not mean CN3-5 is a negative supply.
- Small ~0.2 V movement on a handheld multimeter during commands is not useful for identifying the pulse train because the expected signal changes on a microsecond timescale and the meter only shows an average.

The idle-high ~5 V level on the bridged CN3-2/CN3-3 node strongly supports the working hypothesis that this is the demodulated IR / control signal path.

This interpretation remains provisional until the pulse train is captured with the passive ESP sniffer on this exact HitAir unit.

## Safety / measurement rule

Do not connect a 3.3 V ESP GPIO directly to CN3. The board uses a 5 V rail.

Until the live physical layer is confirmed:
- power ESP separately;
- common ESP GND only with CN3-5 when actively measuring;
- reduce the signal level before GPIO;
- use receive-only firmware;
- do not remove/alter the factory bridge between CN3-2 and CN3-3.
