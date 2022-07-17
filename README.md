# PCE_Controller_Info

Information about controller signalling on the PC Engine

## Electrical

The PC Engine is based on CMOS 5V logic, so the voltage supply is 5V and the logic levels are set as such.

There are 6 active lines (besides power and ground) on the connector, with two being driven by
the console (SEL and CLR), and 4 being return signals from the joypad/peripheral.

Electrically, the signals of SEL and CLR are generally pulled up by a 47K resistor on the peripheral
to prevent issues on the internals of the peripherals if power is supplied but CLR/SEL arr floating;
this is not a likely case, but this is a protective measure.

Similarly, the controller inputs also use 47K resistors as pullups, and return signals from the
peripheral are genreally sent back through current-limiting 300-ohm resistors in order to limit
the amount of current driven by the internal chips, in the event of an unexpected short circuit
on the console side of the controller.

### Controller Schematics and Connector Pinout

This schematic shows the wiring diagram for a basic PC Engine controller. Schematics of additional
configurations can be found here:
https://console5.com/wiki/NEC_PC_Engine#Controller_Schematics

(Many thanks to console5.com for sourcing, preserving and improving console information like this.)

As can be seen in the schematic:
 1) All inputs default to high, unless they are specifically driven low
 2) The 74HC163 creates a divide-by-2/4/8 counter based on the CLR signal, which is then used by the
'turbo' switches to modulate auto-repeat keying.

![https://console5.com/wiki/File:PC-Engine---TG16-Controller-Schematic.png](https://console5.com/techwiki/images/9/9f/PC-Engine---TG16-Controller-Schematic.png)


## Signalling Protocols - Overview

The program will scan the joypad controls with code similar to the below.
Notice that normally:
1) The CRL bit is set only briefly, once per scan across all joypads
2) SEL is kept HIGH for the initial read of joypad values, and toggled LOW to read the second group
3) There is normally a brief delay - although the size of this can vary - between changing the value
of the SEL line, and reading back return values from the joypad

One further point which is not highlighted in the code below is that subsequent joypads are read
simply by toggling SEL high again to advance to the next joypad in the multitap. Toggling CLR resets
the sequence to the first port all over again.

Normal reading of joypads in this way is generally carried out as part of the VSYNC interrupt, or
as a direct result of its completion, so that values are always available, always 'fresh', and always
based on a regular timebase.

```
LDA #1     ; SEL High (Least-significant bit), CLR low (next-least significant bit)
STA $1000  ; output to joypad port
LDA #3     ; both SEL and CLR high
STA $1000
LDA #1     ; Keep SEL high; drive CLR low again
STA $1000
           ; the following instructions are a delay to allow transition of the joypad device
PHA        ; 3 cycles
PLA        ; 4 cycles
NOP        ; 2 cycles  total = 9 cycles, roughly 1.25 microseconds (CPU clock is normally 7.16 MHz)

LDA $1000  ; read from Joypad port
AND #$0F   ; strip the unnecessary bits

LDA #0     ; Keep SEL high; drive CLR low again
STA $1000
           ; the following instructions are a delay to allow transition of the joypad device
PHA        ; 3 cycles
PLA        ; 4 cycles
NOP        ; 2 cycles  total = 9 cycles, roughly 1.25 microseconds (CPU clock is normally 7.16 MHz)

LDA $1000  ; read from Joypad port
AND #$0F   ; strip the unnecessary bits
```

### 2-button Controller Protocol

A 2-button controller is keyed primarily on the SEL signal, and sends 4 key values when SEL is high,
and another 4 key values when SEL is low.

A 2-button controller doesn't care about the CLR line, but due to the way it is connected to the 74HC157,
all 4 outputs will be LOW when CLR is HIGH.

| SEL value | Bit Number | Key |
|:------:|:--------------:|:--------:|
| HIGH | bit 3 (ie $8) | LEFT |
| HIGH | bit 2 (ie $4) | DOWN |
| HIGH | bit 1 (ie $2) | RIGHT |
| HIGH | bit 0 (ie $1) | UP |
| LOW | bit 3 (ie $8) | RUN |
| LOW | bit 2 (ie $4) | SELECT |
| LOW | bit 1 (ie $2) | II |
| LOW | bit 0 (ie $1) | I |

### Multitap Protocol

Initially, multitap devices were built only for 2-button joypads, so subsequent devices which
were deisnged to be used with them needed to be built to accept the multitap design.

The multitap is essentially a multiplexer which takes CLR and SEL values from the console, and
creates CLR and SEL values on each of the ports; likewise, it takes the output values from the
active port and routes those back to the console.

| CLR | SEL | Active Port | Port 1 CLR | Port 1 SEL | Port 2 CLR | Port 2 SEL | Port 3 CLR | Port 3 SEL | Port 4 CLR | Port 4 SEL | Port 5 CLR | Port 5 SEL |
|-----|-----|-------------|------------|------------|------------|------------|------------|------------|------------|------------|------------|------------|
| 0  | 1 | None | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 |
| 1 | 1 | None | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 |
| 0  | 1 | 1 | 0 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 |
| 0  | 0  | 1 | 0 | 0 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 |
| 0  | 1 | 2 | 1 | 1 | 0 | 1 | 1 | 1 | 1 | 1 | 1 | 1 |
| 0  | 0  | 2 | 1 | 1 | 0 | 0 | 1 | 1 | 1 | 1 | 1 | 1 |
| 0  | 1 | 3 | 1 | 1 | 1 | 1 | 0 | 1 | 1 | 1 | 1 | 1 |
| 0  | 0  | 3 | 1 | 1 | 1 | 1 | 0 | 0 | 1 | 1 | 1 | 1 |
| 0  | 1 | 4 | 1 | 1 | 1 | 1 | 1 | 1 | 0 | 1 | 1 | 1 |
| 0  | 0  | 4 | 1 | 1 | 1 | 1 | 1 | 1 | 0 | 0 | 1 | 1 |
| 0  | 1 | 5 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 0 | 1 |
| 0  | 0  | 5 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 0 | 0 |

### 6-button Controller Protocol

| Scan number | SEL value | Bit Number | Key |
|:-----------:|:------:|:--------------:|:--------:|
| 1 | HIGH | bit 3 (ie $8) | LEFT |
| 1 | HIGH | bit 2 (ie $4) | DOWN |
| 1 | HIGH | bit 1 (ie $2) | RIGHT |
| 1 | HIGH | bit 0 (ie $1) | UP |
| 1 | LOW | bit 3 (ie $8) | RUN |
| 1 | LOW | bit 2 (ie $4) | SELECT |
| 1 | LOW | bit 1 (ie $2) | II |
| 1 | LOW | bit 0 (ie $1) | I |
| 2 | HIGH | bits 3-0  | value 0000 |
| 2 | LOW | bit 3 (ie $8) | VI |
| 2 | LOW | bit 2 (ie $4) | V |
| 2 | LOW | bit 1 (ie $2) | IV |
| 2 | LOW | bit 0 (ie $1) | III |

## Special Controller Peripherals

### Mouse Protocol

### Pachinko Controller Protocol

### Memory Base 128 Protocol

### Develo Box Protocol

