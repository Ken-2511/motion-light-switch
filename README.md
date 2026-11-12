# Corridor Motion-Sensing Light Switch

A motion-activated light controller for school corridors in mainland China. A battery-powered **slave** sits on each existing wall switch and presses it with a small servo, so the device never touches 220 V mains. One USB-powered **master** commands all slaves over LoRa and serves a web page over WiFi. Top design priority: slave battery life.

**How to read this file:** Section 1 holds only decisions the author has confirmed. Section 2 lists open questions. Section 3 lists suggestions that are **not** confirmed. Section 4 is an estimate, not a measurement.

## 1. Confirmed Decisions

### 1.1 Site

| Item | Decision |
| --- | --- |
| Light | Existing 220 V AC corridor lights |
| Switch | Single-gang rocker; lower half = on, upper half = off |
| Switch panel | Believed to be the Chinese 86-type; not yet measured |
| Layout | Several switches along the corridor; each switch's lights cover a few meters |
| Foot traffic | Heavy during class breaks, arrival and dismissal; sparse otherwise |

### 1.2 System

| Item | Decision |
| --- | --- |
| Units | One master; one slave per switch |
| Radio protocol | Plain LoRa between master and slaves (not LoRaWAN) |
| Radio chip | Semtech SX1268 in both master and slave |
| Band | 433 MHz (433.05–434.79 MHz), transmit power ≤ 10 mW per MIIT micro-power rules. 470–510 MHz rejected: MIIT designates it mainly for civil metering devices. |
| Custom PCB | Slave only, as small as possible. The master uses off-the-shelf boards. |
| EDA | KiCad |

### 1.3 Control Logic

1. When a slave detects a person, it presses the switch to turn the light on and reports to the master.
2. Only the master decides when to turn a light off.
3. The slaves do not sense whether the light is actually on.
4. Daytime handling combines the slave's ambient-light reading with the master's time-of-day rules.
5. The master can command any slave at any time, for example turning on the lights at slaves next to one that detected motion. About 1 s from master command to slave action is acceptable.
6. Each slave listens using the SX1268's built-in duty-cycled receive (`SetRxDutyCycle`) with a period of about 1 s while its MCU sleeps. The master sends every packet with a preamble longer than that period.

### 1.4 Master

| Item | Decision |
| --- | --- |
| Hardware | ESP32-C3 SuperMini board hand-wired on perfboard to an Ebyte E22-400M22S (same module as the slave) on an adapter board with 2.54 mm pin headers |
| Power | USB, always on, placed near a power outlet |
| Network | Existing password-protected WiFi |
| Functions | Shows slave status and accepts commands from a phone; decides when lights turn off; applies time-of-day rules |
| Phone access | Same WiFi network only; no custom app |

### 1.5 Slave Electronics

| Item | Decision |
| --- | --- |
| MCU | STM32L051K8T6 (Cortex-M0+, 64 KB flash, LQFP32). Chosen over the 32 KB STM32L031 for firmware headroom. |
| Radio | Ebyte E22-400M22S (SX1268, 14×20 mm, stamp holes, TCXO) on SPI |
| Antenna | 433 MHz FPC antenna (e.g. TX433-FPC-3208) plugged into the module's own IPEX socket; the module's ANT pad is left unconnected and the slave PCB has no antenna connector |
| Battery | One 18650 cell, removed and charged in an external charger |
| Battery holder | Generic 1-cell through-hole 18650 holder from Taobao, a clone of the MPD BH-18650-PC (77.7 mm long; solder pins 72.90 mm apart; two Ø3.2 mm mounting holes 55.61 mm apart). Soldered onto the slave PCB with KiCad footprint `Battery:BatteryHolder_MPD_BH-18650-PC`, whose pad 1 is +. The genuine MPD part (LCSC C5339083) was out of stock. |
| Reverse-polarity protection | AO3401 P-MOSFET: drain to battery +, source to the circuit, gate to ground through a resistor. Chosen over a diode to avoid a ~0.3 V drop. |
| 3.3 V regulator | TPS7A0233 (fixed 3.3 V, ~25 nA quiescent current, SOT-23-5). Powers the MCU, radio and sensors only (~30 mA peak). |
| Servo power switch | High-side P-MOSFET from the battery, off whenever the servo is idle. A ~100 kΩ resistor pulls its gate up to the battery; a 2N7002 driven by the MCU pulls it low. |
| Servo supply voltage | PCB has a footprint for a boost converter plus a 0 Ω bypass resistor, so a bench test can choose direct battery or 5 V |
| Motion sensor | Bare AM312 PIR (LCSC C114881, TO-5 can; pins 1 VDD, 2 OUT, 3 GND per LCSC) soldered on the PCB, behind a separate Fresnel lens in the enclosure. mmWave rejected for its current draw. Current: ~13–15 µA measured by others at 3.3 V; datasheet maximum 0.1 mA. |
| PIR power switching | PIR powered from PA8 through a 100 Ω / 1 µF filter, so firmware can switch it off when lights are not needed (daytime, weekends, holidays). Firmware ignores PIR_OUT for ~30 s after power-up and keeps PA0's pull-up off while the PIR is off. |
| Ambient light | Everlight ALS-PT19-315C/L177/TR8 visible-light phototransistor (LCSC C146233, 0603), read by the ADC with a 100 kΩ load and powered from a GPIO only while measuring. Chosen over the infrared PT15-21C, which barely sees LED lighting. |
| Battery monitoring | No extra parts. Firmware measures VDD with the internal voltage reference; VDD follows the battery once it falls below ~3.3 V. This gives a low-battery warning only, not a full-range gauge. |
| Status LED | Shows pairing, low battery and motion trigger |
| External button | User function, e.g. pause this slave for 2 hours |
| Internal button | Pairing and debug; not reachable from outside |
| Clock | 32.768 kHz crystal on the LSE pins for accurate timers (under 1 µA). The internal LSI RC oscillator (26–56 kHz) was rejected as too inaccurate. |
| Crystal | HCI FC31M2-32.768-N07LLDTL (LCSC C5120558): 3215 package, 7 pF load, ±20 ppm, ESR < 70 kΩ, with two 10 pF capacitors. Its ESR needs the STM32's medium-high LSE drive (LSEDRV = 10); firmware must set this before enabling the LSE. Chosen because the lower-ESR Epson FC-135R was out of stock. Drop-in alternative: ECS-.327-7-34R-TR (LCSC C6547871, ESR 50 kΩ, medium-low drive), only 5 in stock on 2026-11-11. |
| Firmware updates | SWD programming pads; no OTA |

### 1.6 Slave Pin Assignment (STM32L051K8T6)

Checked against ST datasheet DS10184 Rev 10. PA0–PA3 each use their own EXTI line, so all four can wake the MCU from Stop mode.

| Pin | Use | Notes |
| --- | --- | --- |
| PA0 | PIR output | EXTI0 |
| PA1 | SX1268 DIO1 | EXTI1 |
| PA2 | External button | EXTI2 |
| PA3 | Internal button | EXTI3 |
| PA4 | SX1268 NSS | Chip select |
| PA5 / PA6 / PA7 | SX1268 SCK / MISO / MOSI | SPI1 (AF0) |
| PB0 | SX1268 BUSY | Input |
| PB7 | SX1268 NRESET | Output |
| PA11 / PA12 | SX1268 TXEN / RXEN | Connect only if the chosen module needs them |
| PA15 | Servo PWM | TIM2_CH1 (AF5) |
| PB3 | Spare PWM | TIM2_CH2 (AF2), for a future double-gang version |
| PB6 | Servo power enable | Gate of the 2N7002 |
| PB1 | Phototransistor signal | ADC_IN9 |
| PB5 | Phototransistor power | High only while measuring |
| PB4 | Status LED | |
| PA9 / PA10 | Debug UART TX / RX | USART1 (AF4) |
| PA13 / PA14 | SWDIO / SWCLK | Programming pads |
| PC14 / PC15 | 32.768 kHz crystal (LSE) | HCI FC31M2, 7 pF load, with 10 pF caps |
| PA8 | PIR power (PIR_PWR) | Through a 100 Ω / 1 µF filter |
| BOOT0 (pin 31) | Boot mode | 10 kΩ pull-down to ground, so the chip always runs user firmware |
| NRST (pin 4) | Reset | To the reset button and the SWD pads |

Firmware must configure PB9, PB12–PB15, PH0, PH1 and PC13 as outputs driven low. These exist inside the chip but have no pins on LQFP32, and the datasheet requires this.

### 1.7 Slave Enclosure

| Item | Decision |
| --- | --- |
| Construction | Two pieces. A backplate sticks to the wall with removable double-sided adhesive; the main body clips onto it. |
| Tamper resistance | Hidden hex screw on the bottom, against children |
| Maintenance | Remove the main body to swap the battery |
| Fabrication | 3D printed (author has a printer) |

## 2. Open Questions

- [ ] Servo model
- [ ] Servo supply: direct from battery or boosted to 5 V (bench test)
- [ ] Force needed to flip the switch, and servo torque (bench test with a real switch and servo)
- [ ] Mechanism that presses the rocker (after the bench tests)
- [ ] Boost converter IC, if the bench test shows 5 V is needed
- [ ] Adapter board with pin headers for the master's E22-400M22S
- [ ] Motion-to-light-on latency target
- [ ] Light-off delay and report interval
- [ ] Phone side: master-hosted web page, Mi Home, or another platform
- [ ] Switch panel dimensions (measure)
- [ ] Number of slaves

## 3. Unconfirmed Suggestions

| Area | Suggestion | Reason |
| --- | --- | --- |
| Parts | MG90S servo | Metal gears, durable |
| Parts | Boost IC with an enable pin | Can be shut off between servo actions |
| Mechanical | Servo arm rests at the middle of the rocker, swings one way for on and the other for off, then returns to the middle | The switch still works by hand |
| Mechanical | Short servo arm pressing near the outer edge of the rocker; add a cam or lever if more force is needed | Less servo torque needed |
| Mechanical | Servo pushes toward the wall, so the reaction force goes through the enclosure into the wall | The press cannot pry the unit off its adhesive |
| Mechanical | Locating pins on the backplate | Servo returns to the same position after reassembly |
| Firmware | Turn the light off locally if the master is unreachable for a long time | Lights do not stay on when the master fails |
| Firmware | Configurable servo direction | Some switches may be installed upside down |
| Firmware | While the light is on, report at most every 1/3 of the off-delay | Fewer wake-ups |
| Firmware | Off-delay slightly longer than a class break | One break causes one on/off cycle |
| Firmware | Measure ambient light only while the light is off | Otherwise the sensor reads the lamp |
| Firmware | Below ~3.0 V, stop driving the servo and report low battery | Prevents over-discharge |
| Firmware | LoRa ACK with random-backoff retransmission | Reports from several slaves can collide |
| Firmware | Watchdog timer | A slave stuck awake drains the battery quickly |
| Phone | Master-hosted web page, mDNS name, and a fixed IP via router DHCP reservation | No app needed; some Android phones do not support mDNS |
| Tools | easyeda2kicad to import LCSC footprints; a PPK2-class meter to measure current | |

## 4. Slave Battery Life Estimate

One 3000 mAh 18650 (~2400 mAh usable) at ~4.5 mAh/day lasts about 500 days in theory. Plan on one battery swap per semester until current is measured.

| Item | Estimate |
| --- | --- |
| Sleep current (PIR ~15 µA, radio duty-cycled receive ~10–20 µA, MCU, regulator and crystal ~2 µA) | ~25–40 µA (~0.6–1.0 mAh/day) |
| One on + off cycle (two servo presses) | ~0.13 mAh |
| One report to the master | ~0.01 mAh |
| School day (~25 on/off cycles, at most one report per minute while a light is on) | ~6 mAh/day |
| Weekend or holiday | ~1 mAh/day |
| Average | ~4.5 mAh/day |

Largest consumers, in order: servo presses, reports while a light is on, sleep current.
