# ArduPilot on WeAct Mini STM32H743VIT6

This project documents configuring ArduPilot on the WeAct Studio Mini STM32H743VIT6 development board as a custom quadcopter flight controller, using a GY-91 sensor module (MPU9250 + BMP280) connected via SPI.

<img width="1440" height="1920" alt="ab2a0122-4928-4453-b431-a8e554d921f4" src="https://github.com/user-attachments/assets/4a2c7f74-c651-4bee-86f4-d2dba4b1fa38" />
<img width="1440" height="1920" alt="91e1c19b-4896-4ac7-87d8-c3fc123be029" src="https://github.com/user-attachments/assets/71a49211-18c6-4bb0-b609-b492d3b6570b" />

---

## Hardware

| Component | Details |
|-----------|---------|
| MCU | STM32H743VIT6 — 480MHz, 2MB Flash, 1MB RAM |
| Crystal | 25MHz HSE |
| LED | PE3 (active HIGH) |
| USB | PA11/PA12 OTG Full Speed |
| User button | PC13 |
| Onboard flash | 8MB W25Q64 (QSPI) |
| SD card | SDMMC1 |

---

## Sensor Configuration

The GY-91 is a single breakout board combining:
- **MPU9250** — 3-axis accelerometer, 3-axis gyroscope, and AK8963 3-axis compass (auxiliary I2C internal to MPU9250)
- **BMP280** — barometric pressure and temperature sensor

Both chips share the SPI1 bus but use separate CS pins.

| Device | Interface | CS Pin | Notes |
|--------|-----------|--------|-------|
| MPU9250 (IMU + compass) | SPI1 | PB12 | MODE3, max 8MHz |
| BMP280 (barometer) | SPI1 | PB11 | MODE3, max 8MHz |

### GY-91 Wiring

| GY-91 Pin | FC Pin | Function |
|-----------|--------|----------|
| VCC | 3.3V | Power |
| GND | GND | Ground |
| SCL/SCK | PA5 | SPI1 SCK |
| SDA/MOSI | PA7 | SPI1 MOSI |
| SDO/MISO | PA6 | SPI1 MISO |
| NCS | PB12 | MPU9250 CS |
| CSB | PB11 | BMP280 CS |
| INT | PB0 | MPU9250 interrupt (optional) |

> **Voltage:** The GY-91 has an onboard 3.3V regulator — it can accept 3.3V or 5V on VCC. Logic levels are 3.3V.

> **CS pins:** Both CS lines need 4.7kΩ pullups to 3.3V if not already present on the GY-91 board. Most GY-91 modules include these.

---

## Pinout

### Serial Ports

| Port | Function | TX Pin | RX Pin | Baud | Protocol |
|------|----------|--------|--------|------|----------|
| Serial0 | USB / GCS | PA11 | PA12 | — | MAVLink2 |
| Serial1 | GPS | PA9 | PA10 | 38400 | GPS auto |
| Serial2 | RC Receiver (iBUS) | PA2 | PA3 | 115200 | iBUS (RX only) |
| Serial3 | Jetson Orin Nano | PC6 | PC7 | 115200 | MAVLink2 |

### Motor Outputs

| Motor | Pin | Timer | Position |
|-------|-----|-------|----------|
| M1 | PE9 | TIM1_CH1 | Front-Right |
| M2 | PE11 | TIM1_CH2 | Rear-Left |
| M3 | PE13 | TIM1_CH3 | Front-Left |
| M4 | PE14 | TIM1_CH4 | Rear-Right |

### Other Connections

| Function | Pin | Notes |
|----------|-----|-------|
| Arming button | PE4 | Pull to GND, internal pullup enabled |
| Buzzer | PD14 | TIM4_CH3, active buzzer |
| SWDIO | PA13 | SWD debug |
| SWCLK | PA14 | SWD debug |

---

## Wiring Notes

### iBUS RC Receiver
iBUS is single-wire — only RX pin (PA3) is used. TX (PA2) can be left unconnected. Connect the iBUS signal wire to PA3, VCC to 3.3V or 5V, GND to GND.

### Jetson Orin Nano Companion Computer
Connect via the 40-pin header UART (`/dev/ttyTHS0`):

```
Jetson pin 8  (TX) → FC PC7 (USART6 RX)
Jetson pin 10 (RX) → FC PC6 (USART6 TX)
Jetson pin 6  (GND) → FC GND
```

> **Important:** Share GND only. Do NOT connect 3.3V or 5V between Jetson and flight controller.

### Motors / ESCs
ESCs run from a separate LiPo battery. Only PWM signal wires (PE9/11/13/14) and a shared GND connect to the flight controller.

---

## Prerequisites

- Ubuntu 22.04 or later (or WSL2)
- `arm-none-eabi-gcc` 13.2.1
- `dfu-util`
- Python 3 with `pymavlink` installed
- STM32CubeProgrammer (for initial DFU flash)

```bash
sudo apt install gcc-arm-none-eabi dfu-util python3-pip
pip3 install pymavlink
```

---

## Build Instructions

### 1. Clone ArduPilot

```bash
git clone --recurse-submodules https://github.com/ArduPilot/ardupilot.git
cd ardupilot
Tools/environment_install/install-prereqs-ubuntu.sh -y
. ~/.profile
```

### 2. Apply Required Patches

These patches are mandatory — without them the bootloader will crash and USB will not enumerate.

#### 2a. Fix GCC 13.2 strlen miscompilation (critical — causes bootloader hard fault)

```bash
sed -i 's/^size_t strlen(const char \*s1)$/__attribute__((optimize("O0"))) size_t strlen(const char *s1)/' \
    Tools/AP_Bootloader/support.cpp
```

#### 2b. USB OTG reset + CRS init in board.c (critical — required for USB enumeration)

Find `void boardInit(void)` in `libraries/AP_HAL_ChibiOS/hwdef/common/board.c` and add inside the function body:

```c
void boardInit(void) {
  HAL_BOARD_INIT_HOOK_CALL

#if defined(STM32H723xx) || defined(STM32H7xx)
  // Reset USB OTG_HS peripheral to clear bootloader state
  RCC->AHB1RSTR |= RCC_AHB1RSTR_USB1OTGHSRST;
  volatile uint32_t dummy = RCC->AHB1RSTR;
  (void)dummy;
  RCC->AHB1RSTR &= ~RCC_AHB1RSTR_USB1OTGHSRST;

  // Enable CRS: sync HSI48 to USB SOF for stable enumeration
  RCC->APB1HENR |= RCC_APB1HENR_CRSEN;
  CRS->CFGR = (2U << 28);  // SYNCSRC = USB SOF
  CRS->CR |= CRS_CR_AUTOTRIMEN | CRS_CR_CEN;
#endif
}
```

#### 2c. USB turnaround time fix

```bash
sed -i 's/#define TRDT_VALUE_FS           5/#define TRDT_VALUE_FS           9/' \
    modules/ChibiOS/os/hal/ports/STM32/LLD/OTGv1/hal_usb_lld.c
```

#### 2d. Add #ifndef guards to stm32h7_mcuconf.h

Wrap each bare `#define` listed below with `#ifndef`/`#endif` in `libraries/AP_HAL_ChibiOS/hwdef/common/stm32h7_mcuconf.h`:

- `STM32_VOS` (line ~109)
- `STM32_PLL1_DIVM_VALUE`, `STM32_PLL1_DIVN_VALUE`, `STM32_PLL1_DIVP_VALUE`, `STM32_PLL1_DIVR_VALUE` (25MHz block)
- `STM32_PLL3_DIVN_VALUE`, `STM32_PLL3_DIVQ_VALUE`, `STM32_PLL3_DIVR_VALUE` (25MHz block)
- `STM32_USBSEL`
- `STM32_ADC_ADC12_DMA_STREAM`

```c
#ifndef FOO
#define FOO bar
#endif
```

#### 2e. Guard ADCD3 references in AnalogIn.cpp

Apply `patches/AnalogIn_cpp.patch` or wrap all occurrences of `ADCD3` in `libraries/AP_HAL_ChibiOS/AnalogIn.cpp` with `#ifdef ADCD3` / `#endif` guards.

### 3. Create Board Directory

```bash
mkdir -p libraries/AP_HAL_ChibiOS/hwdef/WeActH743
```

Copy `hwdef.dat` and `hwdef-bl.dat` from this repository into that directory.

### 4. Build Bootloader

```bash
./waf configure --board WeActH743 --bootloader
./waf bootloader
```

### 5. Flash Bootloader via DFU

Put the board into DFU mode: hold **BOOT0**, press **RESET**, release RESET, release BOOT0.

```bash
dfu-util -a 0 -s 0x08000000:leave -D build/WeActH743/bin/AP_Bootloader.bin
```

Verify success — the LED on PE3 should blink after reset.

### 6. Build ArduCopter Firmware

```bash
./waf configure --board WeActH743
./waf copter
```

### 7. Flash Firmware via Bootloader

```bash
python3 Tools/scripts/uploader.py --port /dev/ttyACM0 build/WeActH743/bin/arducopter.apj
```

Or via QGroundControl: **Vehicle Setup → Firmware → Custom firmware** and select `arducopter.apj`.

---

## Flashing Workflow (After Initial Setup)

> **Important:** Do NOT use `arducopter_with_bl.hex` — always flash bootloader and firmware separately.

```
Bootloader (once):  DFU → 0x08000000 → AP_Bootloader.bin
Firmware (updates): USB → ttyACM0   → arducopter.apj
```

---

## QGroundControl Parameters

### RC Receiver (iBUS)

| Parameter | Value | Description |
|-----------|-------|-------------|
| `SERIAL2_PROTOCOL` | 23 | RCInput |
| `SERIAL2_BAUD` | 115 | 115200 baud |
| `RC_PROTOCOLS` | 32 | iBUS |

### Companion Computer (Jetson Orin Nano)

| Parameter | Value | Description |
|-----------|-------|-------------|
| `SERIAL3_PROTOCOL` | 2 | MAVLink2 |
| `SERIAL3_BAUD` | 115 | 115200 baud |

### Buzzer

| Parameter | Value | Description |
|-----------|-------|-------------|
| `NOTIFY_BUZZ_ENABLE` | 1 | Enable buzzer |

### Arming Button

| Parameter | Value | Description |
|-----------|-------|-------------|
| `BTN_ENABLE` | 1 | Enable button |
| `BTN_FUNC1` | 41 | Arm/disarm toggle |

---

## Jetson Orin Nano Setup

### MAVProxy

```bash
pip3 install MAVProxy
mavproxy.py --master=/dev/ttyTHS0 --baudrate=115200 --console
```

### MAVSDK (Python)

```python
import asyncio
from mavsdk import System

async def main():
    drone = System()
    await drone.connect(system_address="serial:///dev/ttyTHS0:115200")
    async for state in drone.core.connection_state():
        if state.is_connected:
            print("Connected to flight controller")
            break

asyncio.run(main())
```

### ROS2 with MAVROS

```bash
sudo apt install ros-humble-mavros ros-humble-mavros-extras
ros2 launch mavros apm.launch fcu_url:=/dev/ttyTHS0:115200
```

---

## Connecting to QGroundControl

1. Power the board via USB
2. Bootloader blinks PE3 LED for ~5 seconds
3. ArduCopter starts — LED behaviour changes
4. QGroundControl auto-detects on `/dev/ttyACM0`
5. "Vehicle not ready" is normal until sensors are calibrated

---

## Troubleshooting

### LED stays solid on, no USB enumeration (bootloader)

**Cause:** GCC 13.2.1 miscompiles `strlen()` with `-O2`, causing a hard fault before the blink loop.

**Fix:**
```cpp
__attribute__((optimize("O0"))) size_t strlen(const char *s1)
```

---

### LED never comes on at all

**Cause:** Binary flashed to wrong address.

**Fix:** Address must be exactly `0x08000000` — 8 hex digits after `0x`, starting with `08`.

---

### USB does not enumerate after bootloader blinks

1. **Missing board.c USB reset** — apply the `boardInit()` patch.
2. **Wrong USB clock** — ensure `OSCILLATOR_HZ 25000000` and mcuconf `#ifndef` guards are in place.
3. **VBUS sensing** — use `define BOARD_OTG_NOVBUSSENS 1` in hwdef.
4. **USB turnaround** — apply the `TRDT_VALUE_FS 9` patch.

---

### Build error: "STM32_USBSEL redefined"

**Fix:** Wrap `STM32_USBSEL` in `stm32h7_mcuconf.h` with `#ifndef`/`#endif`.

---

### Build error: "STM32_VOS_SCALE0 is not defined"

**Fix:** Do not set `STM32_VOS` in hwdef — VOS1 at 400MHz is sufficient.

---

### Build error: "STM32_ADC_ADC12_DMA_STREAM not defined"

**Fix:** Add to `hwdef.dat`:
```
define HAL_USE_ADC FALSE
```

---

### Build error: "SERIAL driver activated but no USART/UART peripheral assigned"

**Fix:** Always include at least one physical UART in `hwdef.dat` alongside `OTG1` in `SERIAL_ORDER`.

---

### Build error: mavlink version.h not found

**Fix:**
```bash
git submodule sync --recursive
git submodule update --init --recursive --force
python3 -m pymavlink.tools.mavgen \
    --lang C --wire-protocol 2.0 \
    --output libraries/GCS_MAVLink/include/mavlink/v2.0 \
    modules/mavlink/message_definitions/v1.0/all.xml
```

---

### Firmware flashed but no USB / QGroundControl connection

**Fix:** Never use `_with_bl.hex`. Use `.apj` only:
```bash
python3 Tools/scripts/uploader.py --port /dev/ttyACM0 build/WeActH743/bin/arducopter.apj
```

---

### GY-91 IMU not detected

1. **CS pins floating** — PB12 and PB11 each need 4.7kΩ pullups to 3.3V.
2. **Wiring** — PA5=SCK, PA6=MISO, PA7=MOSI, PB12=MPU9250 CS, PB11=BMP280 CS.
3. **Voltage** — logic must be 3.3V.
4. **SPI speed** — try reducing to `1*MHZ 4*MHZ` in the SPIDEV lines.

---

### GY-91 Barometer not detected

**Cause:** BMP280 CS not wired or pulled up correctly.

**Fix:** Verify PB11 is connected to the CSB pin on the GY-91 with a 4.7kΩ pullup to 3.3V.

---

### GY-91 Compass not detected

**Cause:** The AK8963 compass is internal to the MPU9250 and accessed via its auxiliary I2C bus. ArduPilot enables this automatically when the MPU9250 driver initialises — no extra wiring is needed.

**Fix:** If the compass is not detected, check that `COMPASS AK8963 SPI:mpu9250 false ROTATION_NONE` is in `hwdef.dat` and the MPU9250 itself is initialising correctly.

---

### QGroundControl shows "Vehicle not ready"

Normal until sensors are calibrated. Under **Vehicle Setup** run:
- Accelerometer calibration
- Compass calibration
- Radio calibration

To bypass temporarily: `Parameters → ARMING_CHECK → 0`

---

### Jetson not receiving MAVLink data

1. **TX/RX swapped** — Jetson TX → FC PC7 (RX), Jetson RX → FC PC6 (TX).
2. **No shared GND** — ensure GND is connected between Jetson and flight controller.
3. **Wrong serial port** — try `/dev/ttyTHS1` if `/dev/ttyTHS0` does not respond.
4. **Parameters** — confirm `SERIAL3_PROTOCOL=2` and `SERIAL3_BAUD=115` in QGC.

---

## Files in This Repository

```
WeActH743/
├── hwdef.dat          # Main firmware hardware definition
├── hwdef-bl.dat       # Bootloader hardware definition
└── README.md          # This file

patches/
├── support_cpp.patch        # strlen GCC 13.2 fix
├── board_c.patch            # USB OTG reset + CRS init
├── hal_usb_lld_c.patch      # USB turnaround time
└── AnalogIn_cpp.patch       # ADCD3 guard (from WeAct H723 port)
```

---

## Credits

- WeAct H723 ArduPilot port by [Er-utpal](https://github.com/Er-utpal/WeAct723-Ardupilot) — patches directly applicable to the H743
- ArduPilot community porting documentation at [ardupilot.org/dev](https://ardupilot.org/dev/docs/porting.html)

---

## Licence

This board definition is released under the GNU General Public License v3.0, consistent with the ArduPilot project licence.
