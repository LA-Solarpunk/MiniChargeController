# MiniChargeController

A small solar charge controller for single-cell lithium batteries, built around the TI BQ25792. It tracks the panel's maximum power point, charges either LiPo or LiFePO4, and runs a load at the same time it charges. Target cost is under $15 per unit in small quantities. 

**Status: schematic design. Not yet routed, not yet built, not yet tested. Do not fabricate from this repo.**

## Why this exists

Cheap all-in-one solar charger boards have no usable low-voltage cutoff and many don't have load path management. Boards that do have a healthy cutoff cost $20 and up, and most of those are sized for loads far bigger than ours. Charger ICs that do MPPT properly, like the CN3791, have no power path, so a load hanging off the battery corrupts charge termination and leaves the cell parked at 4.2V which will kill the lifetime of the cell. 

The BQ25798 covers all of it in one and we add a CH32V cheap RISC micro to handle setup, manual MPPT, low voltage cutoff, and charge source selection. It can be reprogrammed to support other chemistries as well like LiFePo4. The low power mode is 10uA coupled with the 24uA draw of the BQ25792 means potentially months of runtime when no solar is present.

## Requirements

- Solar input from 5V to 18V nominal panels, open-circuit voltage limited to 22V
- Charge a single-cell LiPo or LiFePO4 
- Maximum power point tracking
- Power a load while charging, without disturbing charge termination
- Configurable low-voltage cutoff that disconnects the load and leaves the battery to recover
- Raw battery voltage on the output, for a boost converter downstream
- USB-C as a secondary input, for bench charging or for running the device with no panel

### Charger

TI BQ25792, a buck-boost charger with four integrated switching FETs, an integrated BATFET, current sensing, and NVDC power path management. Input range is 3.6V to 24V with a 30V absolute maximum. MPPT works by periodically halting the converter, measuring panel open-circuit voltage, and setting the input voltage regulation point to a configurable fraction of it. The BQ25798 has builtin MPPT, but is more expensive so this saves a buck because we already have an MCU.

Power path means the system rail is regulated separately from the battery, so load current does not flow through the charge termination sense path. The load runs off SYS, the cell charges independently, and neither interferes with the other.

### Inputs

Solar arrives on VAC2, USB-C on VAC1. Each input has its own pair of back-to-back N-channel FETs driven from the charger's ACDRV pins, so the charger selects which source feeds the converter. Back-to-back is required because a single FET's body diode conducts regardless of gate state, and the mux has to block in both directions.

Solar input has reverse polarity protection: a P-channel FET with its drain at the connector and source at VAC2, gate pulled to ground through 100k. A 10V zener clamps gate-to-source, since a 22V panel would otherwise exceed the FET's 12V gate rating. Normal polarity forward-biases the body diode, the channel enhances, and the drop collapses to a few millivolts. Reversed polarity leaves the gate at zero and nothing conducts.

The MCU selects the input, so the current plan is to use USB when no solar is present and the battery is below some threshold so that USB can be used as a backup power supply if desired.

### Microcontroller

A CH32V003J4M6 in SOP-8, powered from SYS, configures the charger over I2C at power-on and then sleeps. It is required rather than optional, because several things the design depends on are off by default:

- MPPT doesn't work on this chip, but we can fake it with some simple logic
- Input source selection is host-controlled
- The I2C watchdog reverts every register to its power-on default when it expires, so it has to be turned off
- The power-on charge voltage is 4.2V per cell from the PROG resistor, which overcharges a LiFePO4 cell
- The MCU shuts off load power when under voltage

`CE` is held high by a pull-up out of reset, so charging stays off until the MCU has written the configuration registers and read them back. This helps ensure the MCU is functioning before charging begins.

### Low voltage cutoff

The working cutoff runs in firmware. The CH32V003's programmable voltage detector raises an interrupt at a threshold set in software, the handler opens the load switch, and the part sleeps. The MCU wakes on the charger's interrupt line when an input source appears, or on a long timer, and reads battery voltage with the load still disconnected before closing the switch again. 

The independent watchdog covers firmware that hangs while the cell is still healthy.

### Battery temperature

There's a thermistor on the board for thermal cutoff, but there's a PCB jumper that can be cut and an external or battery thermistor can be used.

Fitting the thermistor is recommended for outdoor deployment. This can help prevent thermal runaway especially when operating outdoors in hot/sunny weather.

### Input current limit

The ILIM_HIZ pin is voltage-programmed. It becomes a hard ceiling that the host cannot program past. It's currently set to support up to 3A.

The ceiling also matters because the reverse-protection FET is a 50mΩ SOT-23 part. At 5A it would dissipate 1.25W which is a lot of losses for panels that will probably be in the 5W-10W range. Practical input ceiling for this board is set by that FET rather than by the charger.

## Cost

This board is built with jlcpcb parts and mfg in mind, currently around $10 per unit at volume 50. The charger IC is about $2 of that. Most of the rest is the four-layer board, assembly, the inductor, the connectors, and thirteen bulk capacitors.

## Known issues


## Hardware notes


## Repository layout

```
MiniChargeController/     KiCad project
```

## Toolchain

- KiCad 10 for schematic and layout
- ch32fun for firmware, once firmware exists
