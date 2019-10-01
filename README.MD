# Clage Passthrough Heater

## Introduction

At the place I live in, we use a Clage MCX7 passthrough heater for the shower.
It is a great product due to

  * Decentralized positioning: (hot) water loss in pipe can be minimized to approximately 0.3 liters
  * minimal format: electrically controlled blank wire heating system uses minimal space
  * 2 Phase 400V design: For us an advantage as 3x2.5mm² wiring is sufficient, just needs 2x16A
  * shower comfort: Electrical control works perfectly. Admittedly, decentralized hot water with single taps is easy to control
    * With this system design, you only need to turn on the hot water tab as the passthrough heater directly produces shower temperature water
  * energy efficiency: The maximum power of 6.5 kW limits people to either shower not too hot or with not too much water.
    * Power is more than sufficient using water saving shower heads like Gruenspar Ecosavers (which is ultracheap and already lasts over a year with being used for 6 showers daily)
    * "Not too much water" means not unnecessary much water - it is still a comforting shower experience

But I don't write this article to advertise a product, I write this because I want to get everything out of my passthrough heater!

## Wishes

  * Easy selection of shower temperatures: Touch display with "ECO" "Comfort" "MAX" modes is hardly usable with wet fingers, often needs multiple presses and does only allow 35, 38 or 45 degrees
    * Mechanical +/- buttons (or a touch technique that works inside the shower)
    * Single button presets - I want to reach my shower temperature intuitively with 2-3 button presses without having to think about a feedback display
  * Visualisation of shower time/cost
    * using energy consumption
    * using water flow rate
  * Visualisation of water temperature
    * We are using preheated water, it is nice to know the temperature also when passthrough heater does not need to heat (or it is actually warmer than the target temperature)
  * Energy saving reminder
    * Visually, acustically or water-temperature alarm (e.g. colder water for 1 second) after a 90 second (variable) shower block


## Investigation

  * There is the Clage FX remote control
    * two preset keys, +/- keys
    * over-power indicator (when target temperature cannot be reached)
    * wireless module is installed inside the MCX via the TRX port
    * 
 
### General working principle
 * Logic power supply: 5V (12V help supply) for Microcontroller
   * isolated via 400V transformer, 1.5VA
   * 78L05 for 5V
 * Microchip PIC18F25K22 MCU
 * Flow rate sensor (S511A 650, Honeywell SS511AT bipolar hall sensor) suspected to detect water flow and rate; powered via GPIO
 * temperature sensor for outflowing water (some kilo Ohms, so likely NTC/thermistor, did not investigate)
 * 24 Ohms heating element in series with 85 degrees over temperature switch (resettable)
 * Heating element controlled via optotriac isolated switches (MOC3083 + Q8040K5 - one each for L1/L2 - so doing multicycle control)
   * both are independently driven by MCU, one might be controlled by water flow rate sensor as well
 * Ports:
   * TRX
   * ICP
   * SCI
   * IIC
   * TMP

### 4 pin IIC header

The IIC (I2C) header is used for the touch control panel with LED. It provides

  * 1: +5V
  * 2: GND
  * 3: SCL (secured via 100 Ohm series, pullup 4k7 to +5V) Pin 22/RB1 of MCU
  * 4: SDA (secured via 100 Ohm series, pullup 4k7 to +5V) Pin 23/RB2 of MCU

The interesting part here is that the pass through heater might act as an I2C slave or an I2C master - depending on what peripherals are connected here.
When the device is turned on without a control panel, it looks for I2C devices at the addresses (dec): 58, 33, 39, 32. 

When the devices is turned on with the control panel, the control panel becomes an I2C master and communicates with an I2C device at address (dec) 54 (the pass through heater).

#### I2C Slave 54 registers

The passthrough heater I2C slave is configured via registers.
The following information and interpretation is based only on a transfer capture, neither device was tried to be emulated (yet). It might contain errors!
Each register write first writes the register number and then writes 2 bytes.
Each register read first writes the register number and then reads 3 bytes, the first one being the register number being read out.
  * Register 1: Read as (1, 1, 154) on boot, written to the same afterwards
  * Register 2: Temperature configuration. Written (and read back) as (2, 35, 80) or (2, 38, 111) or (2, 45, 79)
  * Register 5: Status register. Normally reads (5, 0, 255), but on boot and sometimes reads (5, 64, 185). When water is running (preheated so no power needed), reads (5, 2, 67)
  * Register 8: Written as 1, 16, 124 on boot
  * Register 9: Written as 1, 0, 74 on boot

#### "CRC":
 * 0x9A: 10011010
 * 0x7C: 01111100
 * 0x4A: 01001010
 * 0x50: 01010000
 * 0xB9: 10111001
 * 0x6F: 01101111

### Investigations 2019-10-01 second passthrough heater
* I2C speed 20 kHz, control panel is master
* Data transfer: Register architecture.
* Read:
  ** W <regnr>
  ** R reads <regnr> <data byte 0> [... <date byte n>] <checksum>
* Write:
  ** W <regnr> <data bytes> <checksum>
* Checksum: <CRC-8/MAXIM of payload including regnr, Polynom 0x31>
##### Device init (on low temperature)

* W 36, NACK every 10 ms, until not NACK anymore
* W 36 01, R 01 01 9A
* W 36 01 01 9A
* W 36 08 01 10 7C
* W 36 09 01 00 4A
* W 36 02, R 02 23 50
* W 36 05, R 05 40 B9
* W 36 02, R 02 23 50
* W 36 05, R 05 00 FF

##### Transition low -> med
* W 36 02, R 36 02 23 50
[...]
* W 36 02 26 6F
* W 36 05 40 B9
* W 36 02 26 6F


I2C address 36
(Standby)
Every 100ms W 05, R 05 00 FF
On temperature change to low:
W 02 2D 4F
W 05
R 05 40 B9
W 02
R 02 2D 4F

On temperature change to med(high?):
W 02
R 02 2D 4F

On temperature change to high:
W 02 23 50
W 05
R 05 40 B9
W 02
R 02 23 50

