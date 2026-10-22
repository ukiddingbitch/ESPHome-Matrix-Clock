# ESPHome-Matrix-Clock
A digital clock built with an ESP32-C3 and four MAX7219 LED matrix modules. The display shows the current time, date, and full weekday name in a repeating sequence.

## Features

- Digital clock in `HH:MM:SS` format
- Date in `DD.MM.YY` format
- Full English weekday name with horizontal scrolling
- Automatic time synchronization using NTP
- Scheduled Wi-Fi activation four times per day
- Wi-Fi disabled after successful time synchronization
- Animated horizontal transitions between screens
- Custom bitmap font for digits and supported letters
- ESPHome firmware using the ESP-IDF framework

## Hardware

| Component | Description |
|---|---|
| Microcontroller | ESP32-C3 DevKitM-1 |
| LED display | 4 × MAX7219 LED matrix modules |
| Display resolution | 32 × 8 pixels |
| Communication interface | SPI |
| Firmware platform | ESPHome |
| Framework | ESP-IDF |

## GPIO Pinout

| ESP32-C3 GPIO | Function |
|---|---|
| GPIO6 | SPI MOSI / DIN |
| GPIO7 | SPI CLK |
| GPIO10 | MAX7219 chip select (CS) |

## ESPHome Configuration

The project uses the ESP32-C3 DevKitM-1 with the ESP-IDF framework.

```yaml
esphome:
  name: matrixclock
  friendly_name: Matrix Clock

esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: esp-idf
```

### Display Configuration

Four MAX7219 chips are connected in series to create a 32 × 8 pixel display.

```yaml
spi:
  clk_pin: GPIO7
  mosi_pin: GPIO6

display:
  - platform: max7219digit
    id: matrix_display
    cs_pin: GPIO10
    num_chips: 4
    intensity: 10
    rotate_chip: 0
    scroll_enable: false
    update_interval: 50ms
```

The display is rendered using a custom C++ lambda. Each pixel is calculated individually to control the digits, separators, date, weekday text, and transition animations.

### Time Synchronization

The clock uses the `Europe/Bratislava` time zone and synchronizes with public NTP servers.

```yaml
time:
  - platform: sntp
    id: sntp_time
    timezone: Europe/Bratislava
    servers:
      - 1.sk.pool.ntp.org
      - pool.ntp.org
    update_interval: 1min
```

The time is checked every minute. After successful synchronization, Wi-Fi is disabled to reduce power consumption.

### Scheduled Wi-Fi Operation

Wi-Fi is enabled at 00:00, 06:00, 12:00, and 18:00.

```yaml
on_time_sync:
  then:
    - logger.log: "NTP: TIME SYNCHRONIZED, DISABLING WI-FI"
    - wifi.disable:
```

Each scheduled connection uses the following automation pattern:

```yaml
on_time:
  - seconds: 0
    minutes: 0
    hours: 6
    then:
      - logger.log: "NTP: SCHEDULED CONNECTION AT 06:00"
      - wifi.enable:
```

The same pattern is configured for midnight, noon, and 18:00.

**Note:** The schedule enables Wi-Fi; it does not itself guarantee that NTP synchronization will succeed. The device needs a working network connection and an available NTP server.

### Wi-Fi Configuration

```yaml
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  fast_connect: false
  power_save_mode: none
  reboot_timeout: 0s
```

Replace the example access point password with a secure password before deploying the device. The fallback access point is not the same as the normal Wi-Fi connection.

## Display Sequence

| Screen | Content | Duration |
|---|---|---|
| 1 | Current time (`HH:MM:SS`) | 7 seconds |
| 2 | Date (`DD.MM.YY`) | 2 seconds |
| 3 | Full weekday name | 3 seconds |

The three screens repeat in a 12-second cycle. Each screen transition uses an 800 ms horizontal animation.

The time separators blink every second. The weekday name scrolls horizontally across the display.

## Custom Font

The display uses a custom bitmap font defined in the C++ lambda. Digits are stored as three-column bitmap patterns, while punctuation and letters are implemented using individual character definitions.

This approach allows the display layout and animations to be controlled directly without relying on a standard text-rendering font.
