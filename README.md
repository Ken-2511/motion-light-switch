# Corridor Motion-Sensing Light Switch

A motion-activated light controller for school corridors. Each slave unit mounts on an existing wall switch and presses it with a small servo, so the device never touches 220 V mains. A master unit talks to the slaves over LoRa and connects to WiFi.

**Conventions:** Section 2 lists only requirements the author has explicitly confirmed. Section 3 lists open questions. Section 4 lists suggestions raised during design discussions that have **not** been confirmed. All figures in Section 5 are estimates pending measurement.

## 1. Overview

| Item | Description |
| --- | --- |
| Goal | Automatically turn corridor lights on when people are present |
| Method | Servo physically presses the existing rocker switch |
| Mains contact | None |
| Communication | LoRa between master and slaves |

## 2. Confirmed Requirements

### 2.1 Controlled Load & Environment

| Item | Description |
| --- | --- |
| Light | Existing 220 V AC corridor light |
| Switch | Rocker switch, single gang |
| Panel size | Believed to be Chinese standard 86-type; not yet measured |
| Switch layout | Multiple switches along the corridor; each covers a radius of a few meters |
| Foot traffic | Concentrated during class breaks and arrival/dismissal; sparse otherwise |
| Region | Mainland China |

### 2.2 System Architecture

| Item | Description |
| --- | --- |
| Composition | One master + multiple slaves (one slave per switch) |
| Communication | Plain LoRa (not LoRaWAN) between master and slaves |
| LoRa band | 433 MHz (433.05–434.79 MHz, ≤ 10 mW, MIIT micro-power rules); 470–510 MHz rejected because it is designated mainly for civil metering devices |
| Radio chip | Semtech SX1268 on both master and slave, same 433 MHz tuning |
| Actuation | Device does not touch 220 V; slave presses the existing switch with a small servo |
| PCB | Custom PCB for the slave only, as small as possible; master is built from off-the-shelf boards with no custom PCB |
| Component level | Slave: SMD parts and a stamp-hole LoRa module |

### 2.3 Master

| Item | Description |
| --- | --- |
| Hardware | ESP32-C3 SuperMini board hand-wired (perfboard) to an SX1268 433 MHz module with 2.54 mm pin headers or a breakout board |
| Power | USB, always-on; placed near a power outlet |
| Network | Standard password-protected WiFi |
| Functions | Networked; view status / control remotely from a phone; decides when to turn lights off; time-of-day control |
| Phone access | Only needed on the same WiFi network; no custom app to be developed |

### 2.4 Slave

| Item | Description |
| --- | --- |
| MCU | STM32L051 (Cortex-M0+, 64 KB flash); chosen over the STM32L031 (32 KB) for firmware headroom |
| Radio | SX1268 433 MHz module over SPI |
| Power | One 18650 cell; removed and charged with an external charger |
| Motion sensor | On the slave; PIR or mmWave (either acceptable) |
| Ambient light | Has an ambient light sensor |
| Battery monitoring | Desired, if not too complex |
| Status LED | Pairing, low battery, trigger indication |
| External button | Function button, e.g. pause this slave for 2 hours |
| Internal button | Pairing / debug button, not exposed |
| Enclosure | Two-piece: backplate attached with double-sided / removable adhesive; main body clips onto backplate; hidden hex screw on the bottom to deter tampering by children; main body is removed for maintenance / battery swap |
| Enclosure fabrication | Author has access to a 3D printer |
| Firmware updates | Programming pads (SWD); OTA is not required |

### 2.5 Control Logic

1. Slave detects a person → presses the switch locally to turn the light on + reports to master.
2. Master alone decides when to turn the light off.
3. Press the lower half of the rocker to turn on, the upper half to turn off; actual light state is not detected.
4. Daytime handling: slave ambient-light check combined with master time-of-day control.
5. Response latency does not need to be very fast; trade off against battery life.
6. The master can command any slave at any time, including turning on the lights of slaves next to one that detected motion. About 1 s latency is acceptable.
7. Slave listening uses the SX1268's own duty-cycled receive (`SetRxDutyCycle`), about 1 s period, while the MCU sleeps; the master sends each packet with a preamble longer than the sleep period.

### 2.6 Development Setup & Priorities

| Item | Description |
| --- | --- |
| EDA | KiCad |
| Top priority | Power consumption and battery life |

## 3. Open Questions

- [ ] Final motion sensor type: PIR or mmWave
- [ ] Servo model
- [ ] Servo power: direct from battery or boosted to 5 V
- [x] MCU: STM32L051 (slave), ESP32-C3 SuperMini (master)
- [ ] Exact SX1268 module part numbers (slave: stamp-hole; master: pin headers or breakout)
- [ ] Exact STM32L051 package
- [x] LoRa band: 433 MHz
- [ ] Target response latency
- [ ] Light-off delay and report interval values
- [ ] Phone-side implementation: master-hosted web page, Mi Home, or another platform
- [ ] Actual switch panel dimensions (to be measured)
- [ ] Number of slaves

## 4. Suggestions from Discussion (Unconfirmed)

| Category | Suggestion | Rationale |
| --- | --- | --- |
| Parts | PIR on slaves (e.g. AM312) | mmWave draws roughly two orders of magnitude more current; unsuitable for battery |
| Parts | MG90S servo | Metal gears, durable |
| Parts | Low-Iq LDO for 3.3 V (e.g. RT9080) | Avoid parts like AMS1117 with mA-level quiescent current |
| Parts | Phototransistor into ADC for ambient light | Uses fewer pins than an I²C sensor |
| Circuit | MOSFET to cut servo power | Servo draws current even when idle |
| Circuit | If using a high-side P-MOSFET, add a gate driver transistor | 3.3 V GPIO cannot fully turn it off at 4.2 V battery |
| Circuit | Footprint for a boost converter + 0 Ω bypass resistor | One board revision can test direct vs. boosted supply; boost IC must have an enable pin |
| Circuit | Battery divider and phototransistor share one power-enable pin | Saves power and GPIO |
| Circuit | LoRa DIO1, PIR, and external button on EXTI-capable pins | These can wake the STM32L0 from Stop mode |
| Circuit | Add RESET button and SWD programming pads | Needed for flashing and debugging without OTA |
| Circuit | Break out one spare PWM pin | Allows for double-gang switches |
| Circuit | Create a pin assignment table before schematic | Avoids pin conflicts between SPI, timers, ADC and wake pins |
| Firmware | Fall back to local off-timer when master is unreachable for a long time | Lights don't stay on if master fails |
| Firmware | Configurable servo direction | Switch may be installed upside down |
| Firmware | Throttle reports while light is on (~1/3 of off-delay) | Reduces wake-up energy |
| Firmware | Off-delay slightly longer than a class break | One break = one on/off cycle |
| Firmware | Measure ambient light only while the light is off | Sensor would otherwise read the lamp itself |
| Firmware | Stop driving servo and report below ~3.0 V | Prevents over-discharge |
| Firmware | LoRa ACK + random-backoff retransmission | Simultaneous reports from multiple slaves collide |
| Firmware | Watchdog | A slave stuck awake drains the battery quickly |
| Phone | Master-hosted web page + mDNS + fixed IP via router DHCP reservation | No app development; some Android devices don't support mDNS |
| Mechanical | Servo presses toward the wall; reaction force goes through enclosure into the wall | Prevents prying the unit off its adhesive |
| Mechanical | Locating pins on backplate | Consistent servo position after reinstalling |
| Tools | easyeda2kicad to import LCSC footprints; PPK2-class tool for power measurement | |
| Compliance | Keep transmit power within MIIT micro-power device rules (≤ 10 mW at 433 MHz) | Verify current limits independently |

## 5. Slave Battery Life Estimate (Reference)

Based on the suggestions in Section 4, one 3000 mAh 18650 (~2400 mAh usable) should theoretically last over a year; a conservative expectation is one battery swap per semester. All figures pending measurement.

| Item | Estimate |
| --- | --- |
| Sleep current | ~40–60 µA (~1.2 mAh/day) |
| One on + off cycle (two servo actions) | ~0.13 mAh |
| One wake-up report | ~0.009 mAh |
| School day (~25 on/off cycles, at most 1 report/min) | ~6–7 mAh/day |
| Weekends / holidays | ~1.5 mAh/day |
| Average | ~5 mAh/day |

Largest consumers, in order: servo actuation, reports while the light is on, sleep current.
