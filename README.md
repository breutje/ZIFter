# ZIFter
ZIFter is a simple, low-cost IC tester for sifting through collections of old and new-old-stock (NOS) logic and memory chips.
It also serves as a development platform for software concepts that may eventually find their way into Orterax.

## Status
⚠️ **Preliminary / In Active Development**

## Implementation
It is implemented as a shield for an Arduino Mega 2560 tand can test IC's up to 32 pins.
Both 300 mil and 600 mil ICs, requiring a ZIF-32 socket that can accommodate both widths.
All compatible SRAMs have +5V (Vcc) opposite of pin-1.
I.e. ICs are aligned to the same end of the socket, so pin-1 is always at the same position.
The Arduino Mega 2560 is chosen because it has 5V I/O as well as enough I/O pins to avoid I/O expanders.
This will increase the testing speed.
Early calculations for the AS6C4008 (628512) reduce the test time from 10 minutes to less than 60 seconds.
It is severely lacking in protection circuits and will not feature discovery algorithms.

## Switching Vcc
Vcc is switched through a P-channel MOSFET: SI2301.
The Proof of Concept (POC) switches through a PAN CHAN SIP-1A05 reed relay as there are no through-hole SI2301's.
It does need a (1N4148) flyback diode, however.
As the Arduino has CMOS outputs that are rail-to-rail, we can switch the gate directly.
There is no real need to have a pull-up (10k) resistor to Vcc or a series resistor in the gate control (if the SI2301 is used)
But if someone powers-up with a chip in the socket that may save the day.
The Vcc pin is always aligned with pin 32 of the ZIF-32 socket.
In addition, a 3mm led (red) will be in parallel to the Vcc to show if the socket is powered.

## Switching GND
GND is switched through a N-channel MOSFET: SI2302.
The POC will use a through-hole ALJ2302 (SI3202 in TO-92 package)
Again, there is no need for pull-down or series resistors.
The pin switched to ground is always the number of pins divided by 2.
This is the pin diagonally opposite of Vcc.
Several package sizes can been used: 16, 18, 20, 22, 24, 28 and 32 pins.
This requires 7 connections to GND at pin 8, 9, 10, 11, 12, 14 and 16.
This is a constraint of the design.
ICs with exotic pin layout, or non-standard Vcc and/or GND connections cannot be used by this tester.
Fortunately, all ICs I wanted to test actually fit these constraints.
As memories got bigger, DIP packages were abandoned as well as parallel I/O.
There may have been 40-pin parallel SRAMs, but they cannot be tested with this tester.

## Decoupling
A 100nF ceramic capacitor is used between the permanent +5V rail and permanent GND rail.
It is specifically not switched as its charge could potentially discharge to the chip if Vcc and GND are swiched-off.
Probably just theoretical damage a chip, but better safe than sorry.
A 10 μF capacitor (10V) is used to stabilize the Vcc.

## Indication LEDs
| LED | Colour | Function     | Remarks                                    |
| :-: | :----- | :----------- | :----------------------------------------- |
| 🟡  | Yellow  |DUT power    | Powered from Vcc/GND DUT                    |
| 🟢  | Green   | Testing/Pass | Blinking while testing, steady when PASS   |
| 🔴  | Red     | Error        | Steady if FAIL                             |

* **Yellow**: This is the power of device under test (DUT). Anode goes to switched +5V. GND is GND bus through 4K7 Ω series resistor.
* **Green**: Blinking during testing, steady when test passed.
* **Red**: If the test failed, steady.

### Implementation
In order to have leave the MCU free for testing, the Green blinking is implemented with an independent blinker circuit.
As we needed some logic, I decided to use a CD4093 quad 2-input NAND Schmitt trigger.
One gate is used as an simple RC oscillator, about 2 Hz.
This requires a capacitor of 1 μF to GND and a feedback resistor of 390 kΩ to 470 kΩ.

![Oscillator circuit](./oscillator.png)

This is gated with the `BLINK` output pin to `BLINK_CLOCK`,
The `GREEN` output of the MCU is gated with `BLINK_CLOCK` to output the signal for the green LED.
The NAND output has a series resistor of 4K7 Ω to Vcc.
The `RED` output pin is inverted using the last NAND in the CD4093.
Again this output has a series resistor of 4K7 Ω to Vcc.

![4093 logic](./blinkenlights.svg)

The inverters are not in the final circuit and are there just to duplicate the effect that the LEDs are tied to Vcc and not to GND.
The [Digital](https://github.com/hneemann/Digital) simulation file is [here](./blinkenlights.dig).


| Code | BLINK | GREEN | RED   | State                         |
| :--: | :---: | :---: | :---: | :---------------------------- |
| 0x00 |   0   |   0   |   0   |  both red and green are off   |
| 0x01 |   0   |   0   |   1   |  only red is on               |
| 0x02 |   0   |   1   |   0   |  only green is on             |
| 0x03 |   0   |   1   |   1   |  both red and green are on    |
| 0x04 |   1   |   0   |   0   |  both red and green are off   |
| 0x05 |   1   |   0   |   1   |  only red is on               |
| 0x06 |   1   |   1   |   0   |  green is blinking            |
| 0x07 |   1   |   1   |   1   |  red is on, green is blinking |

Only codes 0x00, 0x01, 0x02 and 0x06 make sense for ZIFter.


## Device list
| Type     | Generic | Manufacturer | Pins | bits  | Words | bit | Comments            |
| -------- | ------- | ------------ | ---- | ----- | ----- | --- | ------------------- |
| CY62128  | 62128   | Cypress      | 32   | 1024k | 128k  | 8   |                     |
| 628512   | 628512  | Various      | 32   | 4096k | 512k  | 8   |                     |
| AS6C4008 | 628512  | Alliance     | 32   | 4096k | 512k  | 8   |                     |
| CY62512V | 628512  | Cypress      | 32   | 4096k | 512k  | 8   |                     |
| 62256    | 62256   | Various      | 28   | 256k  | 32k   | 8   |                     |
| CY62256N | 62256   | Cypress      | 28   | 256k  | 32k   | 8   |                     |
| 6264     | 6264    | Various      | 28   | 64k   | 8k    | 8   |                     |
| MCM6264  | 6264    | Motorola     | 28   | 64k   | 8k    | 8   |                     |
| CY7C199  | 62256   | Cypress      | 28   | 256k  | 32k   | 8   |                     |
| UM6164   | 6164    | UMC          | 28   | 64k   | 8k    | 8   |                     |
| CY7C185  | 6264    | Cypress      | 28   | 64k   | 8k    | 8   |                     |
| CY7C196  |         | Cypress      | 28   | 256k  | 64k   | 4   |                     |
| CY7C195  |         | Cypress      | 28   | 256k  | 64k   | 4   |                     |
| CY7C194  |         | Cypress      | 24   | 256k  | 64k   | 4   |                     |
| CY7C128A |         | Cypress      | 24   | 16k   | 2k    | 8   |                     |
| 6116     | 6116    | Various      | 24   | 16k   | 2k    | 8   |                     |
| UM6116   | 6116    | UMC          | 24   | 16k   | 2k    | 8   |                     |
| SR64K4   |         | Lattice      | 11   | 64k   | 16k   | 4   |                     |
| UM6167   | 6167    | UMC          | 20   | 16k   | 1k    | 1   | separate Din/Dout   |
| IDT6167  | 6167    | IDT          | 20   | 16k   | 1k    | 1   | separate Din/Dout   |
| P4C167   | 6167    | Pyramid      | 20   | 16k   | 1k    | 1   | separate Din/Dout   |
| IS61C67  | 6167    | ISSI         | 20   | 16k   | 1k    | 1   | separate Din/Dout   |
| 6168     | 6168    | IDT          | 20   | 16k   | 4k    | 4   |                     |
| 2148     | 2148    | AMD          | 18   | 4k    | 1k    | 4   |                     |
| 2149     | 2149    | AMD          | 18   | 4k    | 1k    | 4   |                     |
| P2114    | 2114    | Intel        | 18   | 4k    | 1k    | 4   |                     |
| MK4114   | 2114    | Mostek       | 18   | 4k    | 1k    | 4   |                     |
| NM2114   | 2114    | NS           | 18   | 4k    | 1k    | 4   |                     |
| TMS2114  | 2114    | TI           | 18   | 4k    | 1k    | 4   |                     |
| μPD2114  | 2114    | NEC          | 18   | 4k    | 1k    | 4   |                     |
| 2114     | 2114    | Various      | 18   | 4k    | 1k    | 4   |                     |
| MB8114   | 2114    | Fujitsu      | 18   | 4k    | 1k    | 4   |                     |
| HM6114   | 2114    | Hitachi      | 18   | 4k    | 1k    | 4   |                     |
| M5M2114  | 2114    | Mitsubishi   | 18   | 4k    | 1k    | 4   |                     |
| TC5514   | 2114    | Toshiba      | 18   | 4k    | 1k    | 4   |                     |
| SAB 2114 | 2114    | Siemens      | 18   | 4k    | 1k    | 4   |                     |
| M2114    | 2114    | SGS          | 18   | 4k    | 1k    | 4   |                     |
| HM6514   | 2114    | Harris       | 18   | 4k    | 1k    | 4   |                     |
| MCM2148  | 2114    | Motorola     | 18   | 4k    | 1k    | 4   |                     |
| 2125A    | 2125    | Intel        | 16   | 1k    | 1k    | 1   |                     |
| 2115A    | 2115    | Intel        | 16   | 1k    | 1k    | 1   |                     |
| 2102A    | 2102    | Intel        | 16   | 1k    | 1k    | 1   | separate Din/Dout   |
| HM6508   | 6805    | Intersil     | 16   | 1k    | 1k    | 1   | different from 2102A|
 
The most important to me now, are the Lattice SR64K4 and Cypress CY7C194-20PC.
I'll start with the following device list:

| Type     | Generic | Manufacturer | Pins | bits  | Words | bit | Comments            |
| -------- | ------- | ------------ | ---- | ----- | ----- | --- | ------------------- |
| SR64K4   |         | Lattice      | 11   | 64k   | 16k   | 4   |                     |
| CY7C194  |         | Cypress      | 24   | 256k  | 64k   | 4   |                     |
| AS6C4008 | 628512  | Alliance     | 32   | 4096k | 512k  | 8   |                     |


Specifically not supported:

| Type     | Generic | Manufacturer | Pins | bits | Words | bit | Comments             |
| -------- | ------- | ------------ | ---- | ---- | ----- | --- | -------------------- |
| TC5501   |         | Toshiba      | 22   | 1k   | 256   | 4   | non standard Vcc/GND |
| 2602     |         | Signetics    | 16   | 1k   | 1024  | 1   | non standard Vcc/GND |


```
# ZIF pin 1..32 -> Arduino Mega digital pin. Wire once, never touch again.
const uint8_t ZIF_TO_ARDUINO[33] = { /* [0] unused, [1..32] = pin */ };

enum PinRole : uint8_t {
  NC, VCC, 
  GND_P8, GND_P9, GND_P10, GND_P11, GND_P12, GND_P14, GND_P16, // Explicit GND pins
  A0,A1,A2,A3,A4,A5,A6,A7,A8,A9,A10,A11,A12,A13,A14,A15,A16,A17,A18,
  D0,D1,D2,D3,D4,D5,D6,D7,
  DIN, DOUT,                                             # Split I/O (e.g., 2102, 6167)
  WE, OE, CE, CE2
};

struct ChipPin { 
  uint8_t zifPin; 
  PinRole role; 
};

struct ChipDef {
  const char* name;
  uint8_t     pinCount;   # 16, 18, 20, 22, 24, 28, 32
  uint8_t     dataWidth;  # 8, 4, or 1
  const ChipPin* pins;    # Only active pins
  uint8_t     numPins;
};
```
