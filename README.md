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
| Communication | LoRa between master and slaves |
| Actuation | Device does not touch 220 V; slave presses the existing switch with a small servo |
| PCB | Separate boards for master and slave; slave as small as possible |
| Component level | SMD modules (e.g. ESP32-C3 module, LoRa stamp-hole module) |

### 2.3 Master

| Item | Description |
| --- | --- |
| Power | USB, always-on; placed near a power outlet |
| Network | Standard password-protected WiFi |
| Functions | Networked; view status / control remotely from a phone; decides when to turn lights off; time-of-day control |
| Phone access | Only needed on the same WiFi network; no custom app to be developed |

### 2.4 Slave

| Item | Description |
| --- | --- |
| Power | One 18650 cell; removed and charged with an external charger |
| Motion sensor | On the slave; PIR or mmWave (either acceptable) |
| Ambient light | Has an ambient light sensor |
| Battery monitoring | Desired, if not too complex |
| Status LED | Pairing, low battery, trigger indication |
| External button | Function button, e.g. pause this slave for 2 hours |
| Internal button | Pairing / debug button, not exposed |
| Enclosure | Two-piece: backplate attached with double-sided / removable adhesive; main body clips onto backplate; hidden hex screw on the bottom to deter tampering by children; main body is removed for maintenance / battery swap |
| Enclosure fabrication | Author has access to a 3D printer |
| Firmware updates | OTA preferred; otherwise on-board USB-C or programming pads |

### 2.5 Control Logic

1. Slave detects a person → presses the switch locally to turn the light on + reports to master.
2. Master alone decides when to turn the light off.
3. Press the lower half of the rocker to turn on, the upper half to turn off; actual light state is not detected.
4. Daytime handling: slave ambient-light check combined with master time-of-day control.
5. Response latency does not need to be very fast; trade off against battery life.

### 2.6 Development Setup & Priorities

| Item | Description |
| --- | --- |
| EDA | KiCad |
| Top priority | Power consumption and battery life |

## 3. Open Questions

- [ ] Final motion sensor type: PIR or mmWave
- [ ] Servo model
- [ ] Servo power: direct from battery or boosted to 5 V
- [ ] MCU and LoRa module part numbers
- [ ] LoRa band: 433 MHz or 470–510 MHz
- [ ] Target response latency
- [ ] Light-off delay and report interval values
- [ ] Phone-side implementation: master-hosted web page, Mi Home, or another platform
- [ ] Actual switch panel dimensions (to be measured)
- [ ] Number of slaves

## 4. Suggestions from Discussion (Unconfirmed)

| Category | Suggestion | Rationale |
| --- | --- | --- |
| Parts | ESP32-C3 module as MCU | Built-in WiFi (OTA, networking), native USB, low deep-sleep current |
| Parts | SX1268-series LoRa module, same model on master and slaves | Supports Chinese bands; supports autonomous duty-cycled RX; avoid 868/915 MHz |
| Parts | PIR on slaves (e.g. AM312) | mmWave draws roughly two orders of magnitude more current; unsuitable for battery |
| Parts | MG90S servo | Metal gears, durable |
| Parts | Low-Iq LDO rated ≥ 500 mA for 3.3 V (e.g. RT9080) | Handles WiFi current peaks; avoid parts like AMS1117 with mA-level quiescent current |
| Parts | Phototransistor into ADC for ambient light | Uses fewer pins than an I²C sensor |
| Circuit | MOSFET to cut servo power | Servo draws current even when idle |
| Circuit | If using a high-side P-MOSFET, add a gate driver transistor | 3.3 V GPIO cannot fully turn it off at 4.2 V battery |
| Circuit | Footprint for a boost converter + 0 Ω bypass resistor | One board revision can test direct vs. boosted supply; boost IC must have an enable pin |
| Circuit | Battery divider and phototransistor share one power-enable pin | Saves power and GPIO |
| Circuit | LoRa interrupt, PIR, and external button on GPIO0–5 | These can wake from deep sleep |
| Circuit | Internal button reuses BOOT pin (GPIO9); add RESET | One button serves both flashing and pairing |
| Circuit | Break out one spare PWM pin | Allows for double-gang switches |
| Circuit | Create a pin assignment table before schematic | ESP32-C3 GPIO is tight; includes strapping pins GPIO2/8/9 |
| Firmware | LoRa command triggers slave to briefly enable WiFi for OTA | No extra hardware needed |
| Firmware | SX1268 handles duty-cycled listening (default ~1 s, adjustable) and wakes ESP32 only on packet | Keeps average sleep current in the µA range |
| Firmware | Fall back to local off-timer when master is unreachable for a long time | Lights don't stay on if master fails |
| Firmware | Configurable servo direction | Switch may be installed upside down |
| Firmware | Throttle reports while light is on (~1/3 of off-delay) | Reduces wake-up energy |
| Firmware | Off-delay slightly longer than a class break | One break = one on/off cycle |
| Firmware | Measure ambient light only while the light is off | Sensor would otherwise read the lamp itself |
| Firmware | Stop driving servo and report below ~3.0 V | Prevents over-discharge |
| Firmware | LoRa ACK + random-backoff retransmission | Simultaneous reports from multiple slaves collide |
| Firmware | Watchdog and WiFi connection timeout | A slave stuck awake drains the battery in ~4 days |
| Firmware | Master pre-emptively turns on adjacent slaves' lights | Compensates for listening latency |
| Phone | Master-hosted web page + mDNS + fixed IP via router DHCP reservation | No app development; some Android devices don't support mDNS |
| Mechanical | Servo presses toward the wall; reaction force goes through enclosure into the wall | Prevents prying the unit off its adhesive |
| Mechanical | Locating pins on backplate | Consistent servo position after reinstalling |
| Mechanical | USB-C port on the back of the main body | Hidden once installed |
| Tools | easyeda2kicad to import LCSC footprints; PPK2-class tool for power measurement | |
| Compliance | Keep transmit power within MIIT micro-power device rules | Verify current limits independently |

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
