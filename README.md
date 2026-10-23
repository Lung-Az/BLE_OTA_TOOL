# Firmware OTA Tool

This repository contains firmware upgrade instructions and a browser-based testing tool for updating devices using USB or BLE OTA.

## Upgrade Firmware by USB

### Prepare

1. Install Python 3.x.
2. Install `esptool`:

**Windows:**

```bash
pip install esptool
```

**macOS (iMac):**

```bash
python3 -m pip install esptool
```

### Upgrade procedure

1. Find the serial port connected to the device.
2. Run the command for your operating system to flash the firmware:

**Windows:**

```bash
python -m esptool --port <COM port number> write_flash --flash-mode dio --flash-freq 80m --flash-size 4MB 0x0 <Project Name>.ino.bootloader.bin 0x8000 <Project Name>.ino.partitions.bin 0xe000 boot_app0.bin 0x10000 <Project Name>.ino.bin
```

**macOS (iMac):** Replace `/dev/cu.usbserial-XXXX` with the serial port for your device. Find available ports with `ls /dev/cu.*`.

```bash
python3 -m esptool --port /dev/cu.usbserial-XXXX write_flash --flash-mode dio --flash-freq 80m --flash-size 4MB 0x0 <Project Name>.ino.bootloader.bin 0x8000 <Project Name>.ino.partitions.bin 0xe000 boot_app0.bin 0x10000 <Project Name>.ino.bin
```

## Upgrade Firmware by BLE OTA

### 1) Connect and enable notifications

Connect to the device and enable notifications on:

- `00008022-...` (COMMAND_CHAR)
- `00008020-...` (RECV_FW_CHAR)

### 2) Send START command

Build a 20-byte command frame with:

- `Command_ID = 0x0001` (FLASH)
- `Payload[0..3] = total_size` (little-endian `uint32`; include signature bytes if secure mode is enabled)
- `Bytes 18..19 = CRC16 over bytes 0..17`

Write the frame to `COMMAND_CHAR` and wait for an ACK (`0x0003`) with `ans = 0x0000`.

### 3) Send firmware data by 4 KB sectors

Process each 4 KB sector (the last sector may be smaller):

1. Reset the running CRC.
2. Split the sector into MTU-sized chunks.
3. For each chunk:
   - `header[0..1] = sector_index` (little-endian)
   - `header[2] = seq` (`0, 1, 2, ...`), or `0xFF` for the final chunk
   - Append the chunk bytes to the payload
   - Update the running CRC over the chunk data
   - On the final chunk, append the 2-byte CRC after the payload
   - Write the data with `writeWithoutResponse` to `RECV_FW_CHAR`
4. Wait for a notification on `RECV_FW_CHAR` where:
   - `bytes 0..1` match `sector_index`
   - `bytes 2..3 == 0x0000` (ACK)

If a CRC or index error occurs, restart the sector from `seq = 0` and retry up to 3 times.

### 4) Send STOP command

Send a STOP frame with:

- `Command_ID = 0x0002`
- zero-length payload
- `CRC16` at bytes `18..19`

Then wait for the ACK result:

- `0x0000` → success (device reboots after approximately 500 ms)
- `0x0003` → signature rejected (secure mode)
- `0x0001` → size mismatch or flash write error

## Test Tool

The web-based OTA test page is available here:

- https://lung-az.github.io/BLE_OTA_TOOL/

## Protocol Reference

Detailed OTA wire protocol documentation is available here:

- [BLE_OTA_PROTOCOL.md](./BLE_OTA_PROTOCOL.md)

## Reference

- BLE OTA reference: https://github.com/gb88/BLEOTA
- Project source page: https://github.com/Lung-Az/BLE_OTA_TOOL/blob/main/index.html
