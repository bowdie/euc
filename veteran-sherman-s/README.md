# Screen Module
The screen module is the only known way to input all settings that are made available to the user. Only a very small subset are made available over bluetooth.

The screen module consists of a custom segment screen, 4 tactile switches and microprocessor (MCU) on a PCB.

It uses a 5-pin male JST XH connector for power and mainboard communication. From bottom to top the pins are:

`5V` `RX` `TX` `GND` `PWR`

The presence of `TX` and `RX` pins would normally indicate the use of a full-duplex serial bus using two data lines, but Leaperkim have utilised these pins differently.

## Screen
The Sherman S uses a custom 51x60mm segment LCD to display information and configure settings. It is driven by the controller.

## Controller
The screen module uses an [STM32F030C8](https://www.st.com/resource/en/datasheet/stm32f030c8.pdf) microcontroller (MCU) with an Arm Cortex-M0 MCU with 64 Kbytes of Flash memory and operating at 48 MHz. Its purpose is to:
* Drive the segment LCD
* Accept input from buttons
* Communicate with the mainboard controller

## Serial Bus System
The Sherman S uses a proprietary half-duplex single-wire serial bus system to communicate between the screen module and mainboard. We'll call it *V-Wire* here.

The bus operates at 38400 baud.

> It's possible the bus system is based on an open standard and I'm not aware of it. [LIN](forums.ni.com/attachments/ni/30/3619/1/LIN.pdf) has *some* similarities, but it has other differences that make it incompatible.

The V-Wire bus contains two types of devices - controller and peripheral. On the Sherman S the controller is the screen module MCU and the peripheral is the mainboard MCU.

### Single Wire
Both the controller and peripheral send and receive data on a single data line, so it's important that they take turns at transmitting to prevent collisions from occuring.

### Frames
A frame consists of a header (provided by the controller) and a response (provided by the peripheral.)

### Header
The header is a 17-byte transmission by the screen module MCU.

| Bytes | Value | Description |
| :---: | ----- | ------- |
| 1-4  | `0x6443544D` | Preamble. Destination and source address |
| 5     | `0x11` | Count of bytes in this header including preamble and CRC |
| 6     | `0x00` No Button Pressed<br>`0x01` OK (bottom-left)<br>`0x02` Headlights (top-right)<br>`0x04` Next (top-left) | Command: Button Pressed |
| 7     | 8-bit number starts at `0x00` and increments by 1 on every header transmission. Restarts when `0xFF` is reached | Counter |
| 8-9    | 16-bit number starts at `0x0000` and increments by 1 on every *second* header transmission. Restarts when `0xFFFF` is reached | Counter |
| 10-11  | 16-bit number starts at `0x0000` and increments by 1 on every *second* header transmission. Restarts when `0xFFFF` is reached | Counter |
| 12     | `0x00` | End of counter
| 13     | `0xFF` | End of frame data
| 14-17  | CRC of bytes 1-13. See below for implementation details. | CRC

### Response
The header is a 56-byte transmission by the motherboard MCU.

| Bytes | Value | Description |
| :---: | ----- | ------- |
| 1-4 | `0x644D5443` | Preamble. Destination and source address |
| 5     | `0x38` | Count of bytes in this header including preamble and CRC |
| 6     | `0x28` | Command: Unknown
| 7-22 | Screen symbols | See [screen symbol bits](#response-screen-symbols) |
| 23-24 | `0x0000` or `0xFFFF` |
| 25-26 | 16-bit number to be displayed. Use 2's complement for negative numbers. Last digit will become decimal if decimal point activated. | Info area number |
| 27 | `0x0` Display numbers in info area<br>`0x6` Display flashing ASCII text in info area
| 28-30 | `0x6E6E6E` or something else...
| 31-33 | `0x6E6E6E` for blank, or<br>ASCII of text to be displayed. | Info area  text |
| 34-35 | `0x6E6E` when no text displayed on 31-33<br>`0x0000` when alphanumeric text set on 31-33|
| 36 | `0xFF` |
| 37 | `0xFF` Solid info area characters<br>`0xAA` Flashing info area characters|
| 38-39 | `0x0000` |
| 40-41 | 16-bit number to be displayed. Use 2's complement for negative numbers. Last digit will become decimal if decimal point activated. | Setting area number | Setting area number |
| 42 | `0x0` Display numbers in settings area<br>`0x6` Display ASCII text in settings area
| 43-50 | ASCII to be displayed in capitals. Set to `0x6E6E6E6E6E6E6E6E` when no ASCII displayed. | Setting area text |
| 51 | `0xFF` | End of frame data |
| 52 | `0xFF` Solid settings area characters<br>`0xAA` Flashing settings area characters<br>Bit mask over settings area characters - `0xFE` means last character flashing, all others solid | |
| 53-56  | CRC of bytes 1-13. See below for implementation details. | CRC

### Response Screen Symbols
Each symbol on the segment LCD is represented by 2 bits. `0b00` turns the symbol off and `0b11` turns the symbol on.

Byte | Bits | Symbol | Area | Description |
| :---: | :---: | :-----: | ---- | ----------- |
| 7 | 0-1 | `.` | Info | Number decimal point |
| 7 | 2-3 | `BL` | Info | Left Battery Charge Low / High |
| 7 | 4-5 | `OC` | Info | Over Current |
| 7 | 6-7 | `EM` | Info | Motor Error |
| 8 | 0-1 | `NR` | Info | Motor Stall |
| 8 | 2-3 | `BR` | Info | Right Battery Charge Low / High |
| 8 | 4-5 | `mph` | Info | Miles per hour |
| 8 | 6-7 | `kph` | Info | Kilometers per hour |
| 9 | 0-1 | `◼︎` | Info | Battery 1 |
| 9 | 2-3 | `◼︎` | Info | Battery 2 |
| 9 | 4-5 | `◼︎` | Info | Battery 3 |
| 9 | 6-7 | `◼︎` | Info | Battery 4 |
| 10 | 0-1 | `◼︎` | Info | Battery 5 |
| 10 | 2-3 | `◼︎` | Info | Battery 6 |
| 10 | 4-5 | `◼︎` | Info | Battery 7 |
| 10 | 6-7 | `◼︎` | Info | Battery 8 |
| 11 | 0-1 | Left `⏤` | Info | Left Battery Communication |
| 11 | 2-3 | Right `⏤` | Info | Right Battery Communication |
| 11 | 4-5 | `Mode` | Setting | Pedal Hardness Mode |
| 11 | 6-7 | `Soft` | Setting | Soft Pedal Mode |
| 12 | 0-1 | `Medium` | Setting | Medium Pedal Mode |
| 12 | 2-3 | `Strong` | Setting | Strong Pedal Mode |
| 12 | 4-5 | `PAA` | Setting | Pedal Angle Adjustment |
| 12 | 6-7 | `OS tilt back` | Setting | Over Speed Tilt Back |
| 13 | 0-1 | `OS alarm` | Setting | Over Speed Alarm |
| 13 | 2-3 | `Brightness` | Setting | Screen Brightness |
| 13 | 4-5 | `Total mileage` | Setting | Odometer |
| 13 | 6-7 | `%` | Setting | Percent |
| 14 | 0-1 | `°C` | Setting | Temperature in Celcius |
| 14 | 2-3 | `mph` | Setting | Miles per hour |
| 14 | 4-5 | `kph` | Setting | Kilometers per hour |
| 14 | 6-7 | `.` | Setting | Number decimal point |
| 15 | 0-1 | `Distance` | Setting | Trip meter |
| 15 | 2-3 | `V` | Setting | Voltage |
| 15 | 4-5 | `A` | Setting | Amps |
| 15 | 6-7 | `mi` | Setting | Miles |
| 16 | 1-2 | `km` | Setting | Kilometers |
| 16 | 3-4 | `Calibration` | Setting | Balance Calibration |
| 16 | 5-6 | `Sleep Mode` | Setting | Sleep Time |
| 16 | 7-8 | `mph / kph` | Setting | Distance Unit Adjustment |
| 17 | 1-2 | `VA` | Setting | Voltage Adjustment / Watts |
| 17 | 3-8 | | | Not Connected. Set all bits to `0x0` |
| 18-22 | 1-8 | | | Not Connected. Set all bits to `0x0` |

### CRC
The CRC-32 algorithm is applied to all data preceding the CRC including preamble, and appended in little endian format to the end of every header and response.
