**# LabVIEW-LINX-I2C-Devices-RaspberryPi**
LabVIEW LINX drivers and examples for interfacing SHT20, PCF8563 RTC, and ADS1115 ADC with Raspberry Pi 4 over I²C.

This repository contains LabVIEW VIs for communicating with three commonly used I²C devices through a Raspberry Pi 4 using the MakerHub LINX toolkit:

| Device  | Function                        | I²C Address |
| ------- | ------------------------------- | ----------: |
| SHT20   | Temperature and humidity sensor |      `0x40` |
| PCF8563 | Real-Time Clock / Calendar      |      `0x51` |
| ADS1115 | 16-bit 4-channel ADC            | `0x48–0x4B` |

The purpose of this project is to demonstrate how standard I²C devices can be controlled directly from LabVIEW through LINX without requiring a dedicated LabVIEW driver for each device.
The implementation covers low-level I²C communication, register access, byte manipulation, sensor conversion formulas, BCD conversion, ADC configuration, and multi-channel acquisition.

**# System Architecture**

The communication architecture is:

                  LabVIEW Application
                         │
                         │ LINX
                         ▼
                  Raspberry Pi 4 or 5
                         │
                         │ I²C
             ┌───────────┼───────────┐
             │           │           │
             ▼           ▼           ▼
           SHT20      PCF8563      ADS1115
        Temperature      RTC       16-bit ADC
         Humidity

The Raspberry Pi acts as the I²C master, while LabVIEW performs device configuration, data acquisition, conversion, and processing.

**# Hardware Platform**

The project was developed for:

| Component               | Description            |
| ----------------------- | ---------------------- |
| Controller              | Raspberry Pi 4 or 5    |
| Development Environment | LabVIEW 2024           |
| Communication Framework | MakerHub LINX          |
| Bus                     | I²C                    |
| I²C Bus                 | `/dev/i2c-1`           |
| Raspberry Pi SDA        | GPIO2 / Physical Pin 3 |
| Raspberry Pi SCL        | GPIO3 / Physical Pin 5 |
| Logic Voltage           | 3.3 V                  |

A common ground must be used between the Raspberry Pi and all connected modules.

# Raspberry Pi I²C Wiring

Raspberry Pi 4 or 5               I²C Device

3.3 V  -------------------------- VCC
GND    -------------------------- GND
GPIO2  / Pin 3 ------------------ SDA
GPIO3  / Pin 5 ------------------ SCL

Most commercial breakout modules already contain pull-up resistors on SDA and SCL.
If bare devices are used, external pull-ups such as approximately `4.7 kΩ` to `3.3 V` may be required.
Do not pull Raspberry Pi I²C lines to 5 V.

**# Raspberry Pi Configuration**

Enable I²C:
>> sudo raspi-config

Navigate to:
Interface Options
    ↓
I2C
    ↓
Enable

Then reboot:
>> sudo reboot

Install I²C utilities:

>> sudo apt update
>> sudo apt install i2c-tools

Check the I²C interface:

>> ls /dev/i2c*

Normally:

/dev/i2c-1

Scan the bus:

>> sudo i2cdetect -y 1

Example with all three devices connected:

0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f

40: 40 -- -- -- -- -- -- -- 48 -- -- -- -- -- -- --
50: -- 51 -- -- -- -- -- -- -- -- -- -- -- -- -- --

Here:

0x40 → SHT20
0x48 → ADS1115
0x51 → PCF8563

**# 1. SHT20 Temperature and Humidity Sensor**

The SHT20 communicates through I²C at address:
0x40

The VI demonstrates direct command-based communication with the sensor.

**## Main functions**

The implementation includes:

I²C Open
   ↓
Send measurement command
   ↓
Wait for measurement
   ↓
Read sensor bytes
   ↓
Combine raw bytes
   ↓
Apply conversion formula
   ↓
Temperature / Relative Humidity

### Common commands

| Function                             | Command |
| ------------------------------------ | ------: |
| Temperature measurement, hold master |  `0xE3` |
| Humidity measurement, hold master    |  `0xE5` |
| Temperature measurement, no hold     |  `0xF3` |
| Humidity measurement, no hold        |  `0xF5` |
| Read user register                   |  `0xE7` |
| Write user register                  |  `0xE6` |
| Soft reset                           |  `0xFE` |

The raw sensor value must be converted according to the SHT20 conversion equations.

Temperature:

Temperature [°C] =
-46.85 + 175.72 × RawTemperature / 65536

Relative humidity:

RH [%] =
-6 + 125 × RawHumidity / 65536

The two status bits in the received measurement value should be removed before performing the conversion.


**# 2. PCF8563 Real-Time Clock**

The PCF8563 RTC uses:

I²C address = 0x51

The VI supports both reading and setting the date and time.

The device stores time information in BCD format rather than ordinary decimal numbers.

## BCD conversion

For example:

Decimal 25

is represented as:

0x25

BCD to decimal:

Decimal =
((BCD >> 4) × 10)
+
(BCD AND 0x0F)

Decimal to BCD:

BCD =
((Decimal / 10) << 4)
OR
(Decimal MOD 10)

The LabVIEW implementation includes both conversion directions.

## RTC data flow

LabVIEW
   │
   ▼
Select RTC register
   │
   ▼
I²C Read
   │
   ▼
Seconds
Minutes
Hours
Days
Weekdays
Months
Years
   │
   ▼
BCD → Decimal
   │
   ▼
LabVIEW Date/Time

The VI can also obtain the current PC time, convert the values to BCD, build the required byte array, and write the time to the PCF8563.
A particularly important implementation detail is that the correct LINX **I²C channel reference must be passed to the read operation**.

**# 3. ADS1115 16-bit ADC**

The ADS1115 is a 16-bit delta-sigma ADC with four analog inputs.
Its I²C address depends on the ADDR pin:

| ADDR Connection | Address |
| --------------- | ------: |
| GND             |  `0x48` |
| VDD             |  `0x49` |
| SDA             |  `0x4A` |
| SCL             |  `0x4B` |

The driver does not require a custom LINX device implementation. Communication is performed using standard LINX I²C Read and Write VIs.

# ADS1115 Configuration Register

The main configuration register is:

Register 0x01

Bit organization:

15        14 13 12     11 10 9      8      7 6 5      4 3 2 1 0

OS           MUX          PGA       MODE      DR          COMP

Useful masks used by the LabVIEW VI:

| Field      |     Mask | Decimal |
| ---------- | -------: | ------: |
| OS         | `0x8000` |   32768 |
| MUX        | `0x7000` |   28672 |
| PGA        | `0x0E00` |    3584 |
| MODE       | `0x0100` |     256 |
| DR         | `0x00E0` |     224 |
| Comparator | `0x001F` |      31 |

Individual fields are changed using:

New_Config =
(Current_Config AND NOT Field_Mask)
OR
New_Field_Value

This allows a setting such as MUX, PGA, or data rate to be changed without modifying the other configuration bits.

# ADS1115 Input Selection

For single-ended measurements:

| Module Channel | ADS1115 Input | MUX Bits |
| -------------- | ------------- | -------- |
| Channel 1      | AIN0-GND      | `100`    |
| Channel 2      | AIN1-GND      | `101`    |
| Channel 3      | AIN2-GND      | `110`    |
| Channel 4      | AIN3-GND      | `111`    |

The VI supports selecting one or several channels.

# ADS1115 PGA Configuration

| PGA   |    Range |      Resolution |
| ----- | -------: | --------------: |
| `000` | ±6.144 V |  187.5 µV/count |
| `001` | ±4.096 V |    125 µV/count |
| `010` | ±2.048 V |   62.5 µV/count |
| `011` | ±1.024 V |  31.25 µV/count |
| `100` | ±0.512 V | 15.625 µV/count |
| `101` | ±0.256 V | 7.8125 µV/count |

Codes `110` and `111` also select the ±0.256 V range.

For most Raspberry Pi 3.3 V measurements, the project uses:

PGA = ±4.096 V

therefore:

Voltage = RawValue × 0.000125

# ADS1115 Data Rate

Supported rates are:

8
16
32
64
128
250
475
860 samples/second

The maximum rate used in the project is:
860 SPS


with a nominal conversion period of:
1 / 860 ≈ 1.163 ms

# ADS1115 Signed Conversion

The ADS1115 conversion register contains a signed 16-bit two's-complement value.

For example:
Received bytes:
FF FD

Combined:
0xFFFD

Unsigned interpretation would incorrectly produce:

65533

which would lead to an incorrect voltage close to:

8.19 V

The correct signed interpretation is:

0xFFFD = -3

Therefore:

-3 × 0.000125
=
-0.000375 V

which correctly represents approximately 0 V.
For this reason, the LabVIEW driver converts the assembled U16 value into an **I16 signed value before voltage scaling**.

# ADS1115 Single-Shot Acquisition

Single-shot mode proved particularly useful when scanning multiple analog inputs.
Example configuration for Channel 1 / AIN0:

01 C3 FF

Communication sequence:

Write configuration
      │
      ▼
Start conversion
      │
      ▼
Poll OS bit
      │
      ▼
Conversion ready
      │
      ▼
Write conversion register pointer 0x00
      │
      ▼
Read two bytes
      │
      ▼
Convert to signed I16
      │
      ▼
Calculate voltage

# Using the OS Bit

The OS bit is bit 15 of the Configuration Register.

In single-shot mode:

OS = 0 → Conversion in progress

OS = 1 → Conversion complete

The LabVIEW implementation reads the Configuration Register and performs:

Configuration AND 0x8000

Alternatively, because the configuration MSB is returned first:

MSB AND 0x80

can be used directly.
This removes the need for an unnecessarily long fixed delay and allows the program to continue as soon as conversion is finished.

# Multi-Channel ADS1115 Acquisition

Because the ADS1115 contains one ADC and an analog multiplexer, the four inputs are not converted simultaneously.
The scanning process is:

Select CH1
   ↓
Start conversion
   ↓
Wait for OS ready
   ↓
Read CH1
   ↓
Select CH2
   ↓
Start conversion
   ↓
Wait for OS ready
   ↓
Read CH2
   ↓
Select CH3
   ↓

This architecture prevents previous-channel data from being mistaken for the newly selected channel.
Unused ADC inputs should not be relied upon while floating. For testing, connect unused inputs to GND.

# LINX Implementation

No dedicated SHT20, PCF8563, or ADS1115 LINX firmware extension is required.

The project uses the standard LINX functionality:

LINX Open
    │
I²C Open
    │
I²C Write
    │
I²C Read
    │
Device-specific conversion
    │
LINX Close

The device protocol is implemented entirely in LabVIEW.

# LabVIEW VI Design

A modular design is recommended:

├── SHT20.vi
│   ├── RawToRH
│   ├── RawToTemp
│   ├── ReadTemperature
│   └── ReadRelHumidity
│
├── PCF8563.vi
│   ├── BCD To Number
│   ├── Number to BCD
│   ├── ReadClock
│   └── WriteClock
│
└── ADS1115 Read.vi
│   ├── Config
│   └── Convert to voltage

This allows the device logic to be reused in larger monitoring applications.

**# Important Notes**

Current Raspberry Pi OS releases may require additional work to install or configure the older MakerHub LINX runtime. The actual LINX installation procedure can depend on Raspberry Pi OS version, architecture, and LabVIEW version.
This repository focuses primarily on the **device-level I²C implementation after a working LINX connection has been established**.
Users should verify:

Raspberry Pi I²C enabled
LINX communication working
Correct I²C bus selected
Device visible with i2cdetect
Correct electrical voltage levels
Common ground present

before troubleshooting the individual device VIs.

And good repository topics would be:

`labview` · `raspberry-pi` · `raspberry-pi-4` · `linx` · `i2c` · `ads1115` · `sht20` · `pcf8563` · `adc` · `rtc` · `embedded-systems` · `iot`

** This is not just three finished examples: it demonstrates the general pattern for implementing **register-based I²C device drivers directly in LabVIEW LINX**, which makes the repository useful as a reference for adding other sensors and peripherals later.**

