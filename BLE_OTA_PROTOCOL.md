# BLE OTA Protocol for Firmware Update

This document describes the OTA firmware update protocol used by the  device and matched by the browser-based updater in this repository. The implementation follows the BLE GATT command/data flow observed in the test tool and is intended for engineering reference and interoperability testing.

## 1. Overview

The OTA update uses Bluetooth Low Energy GATT over a custom service with two characteristics:

- Command characteristic: `00008022-0000-1000-8000-00805f9b34fb`
- Firmware receive characteristic: `00008020-0000-1000-8000-00805f9b34fb`

The device is upgraded in the following phases:

1. Connect to the BLE service.
2. Set MTU to 517
3. Send OTA START command.
4. Transfer firmware data sector by sector.
5. Send OTA STOP command.
6. Wait for firmware validation and reboot.

The protocol is intentionally simple: command frames are sent over the command characteristic, while firmware data is streamed over the data receive characteristic.

## 2. GATT Service and Characteristics

The OTA service UUID used by the app is:

- `00008018-0000-1000-8000-00805f9b34fb`

The relevant characteristics are:

- `00008020-0000-1000-8000-00805f9b34fb` — firmware data receive channel
- `00008022-0000-1000-8000-00805f9b34fb` — command channel

The browser tool uses `writeValueWithoutResponse()` for all OTA writes and listens for notifications on the firmware receive characteristic and acknowledgements on the command channel.

## 3. Command Frame Format

All command packets are 20 bytes long and use a small custom frame format.

### 3.1 Structure

| Offset | Size | Field |
|---|---:|---|
| 0 | 2 | Command ID (little-endian uint16) |
| 2 | 16 | Payload (max 16 bytes) |
| 18 | 2 | CRC16-CCITT over bytes 0..17 |

### 3.2 CRC

The command CRC is calculated over the first 18 bytes of the frame using CRC16-CCITT. The CRC is stored in little-endian format at bytes 18 and 19.

The implementation uses a CRC16 routine equivalent to the CCITT algorithm with a configurable initial value.

### 3.3 Commands

The app defines the following command IDs:

- `CMD_FLASH = 0x0001`
- `CMD_STOP = 0x0002`
- `CMD_ACK = 0x0003`
- `CMD_SPIFFS = 0x0004`

The OTA sequence typically uses:

- START: `CMD_FLASH` or `CMD_SPIFFS`
- STOP: `CMD_STOP`

### 3.4 START command payload

The START command payload contains the total firmware size as a 32-bit little-endian value:

- bytes 2..5: `total_size` (uint32, little-endian)
- remaining payload bytes are zero-filled if unused

Example:

- `total_size = 123456` -> payload bytes `[0x40, 0xE2, 0x01, 0x00]`

## 4. Sector-Based Firmware Transfer

Firmware data is sent in fixed-size sectors of 4096 bytes.

- `SECTOR_SZ = 4096`
- The final sector may be smaller than 4096 bytes.

For each sector, the sender:

1. Resets the running CRC.
2. Divides the sector into chunks.
3. Sends each chunk as a BLE packet.
4. Sends the final chunk with its CRC.
5. Waits for a sector acknowledgment.
6. Retries the sector if needed.

## 5. Data Packet Format

Each firmware packet written to the receive characteristic begins with a 3-byte OTA header:

| Offset | Size | Field |
|---|---:|---|
| 0 | 2 | `sector_index` (little-endian uint16) |
| 2 | 1 | Sequence number or final-marker byte |

After the header, the payload follows.

### 5.1 Normal chunk

For non-final chunk packets:

- `header[0..1] = sector_index`
- `header[2] = sequence number (0,1,2,...)`
- `payload = chunk bytes`

### 5.2 Final chunk

For the last packet of a sector:

- `header[0..1] = sector_index`
- `header[2] = 0xFF`
- `payload = chunk bytes`
- `CRC16 = running CRC for the sector` appended after the payload

The implementation appends the CRC immediately after the final payload bytes in little-endian format.

## 6. CRC Behavior

The protocol computes a CRC over the actual chunk payload bytes, and the final packet includes the accumulated sector CRC at the end.

The implementation does the following:

- Starts with `crc = 0`
- Updates `crc` with each chunk payload as it is generated
- On the last chunk, appends the final 16-bit CRC after the chunk data

If the device reports a CRC error, the sender retries the sector from sequence 0.

## 7. Acknowledgement Semantics

### 7.1 Command ACK

The command characteristic is used for both START and STOP commands. The device responds with an ACK payload whose format is the command response code.

The app expects the device to send a response where:

- byte 0 = command ID or the command acknowledge frame
- payload value `ans` identifies the result

In the implementation, `ans` values are interpreted as:

- `0x0000` = success
- `0x0001` = size mismatch / flash write error
- `0x0003` = signature rejected

For START, success is a command ACK with `ans = 0x0000`.

### 7.2 Sector ACK

After a sector is transmitted, the sender waits for a notification on the firmware receive characteristic.

The notification contains:

- `bytes 0..1`: `sector_index`
- `bytes 2..3`: `ack` value

ACK values observed by the app:

- `0x0000` = success
- `0x0001` = CRC error
- `0x0002` = index error
- `0x0003` = payload length error
- `0x0005` = cannot start OTA

If any non-zero error code is received, the sender retries the sector.

## 8. OTA Sequence

The sequence is as follows:

### 8.1 Start

1. Connect to the BLE device.
2. Enable notifications on both OTA characteristics.
3. Build a START command frame using the command ID (`CMD_FLASH` or `CMD_SPIFFS`).
4. Set the payload to the total firmware size in little-endian 32-bit form.
5. Compute and append the CRC16.
6. Write the 20-byte command frame to the command characteristic.
7. Wait for command ACK.

If ACK is not successful, abort the process.

### 8.2 Data upload

1. Split firmware into 4096-byte sectors.
2. For each sector:
   - build all packets for that sector
   - send them as quick bursts
   - wait for sector ACK notification
   - if ACK indicates CRC/index error or timeout, retry up to 3 times
3. Update the progress and continue to the next sector.

### 8.3 Stop

1. Send a STOP command with command ID `0x0002`.
2. Wait for command ACK.
3. If success is reported, the device reboots after approximately 500 ms.
4. If the signature is rejected, report secure OTA verification failure.

## 9. Retry and Fault Handling

The implementation includes robust retry logic.

- Maximum sector retries: `3`
- If a sector fails due to CRC/index error or timeout, it is retried from sequence 0.
- If a START or STOP command fails, OTA aborts.
- If the process is aborted by the user, the implementation sends a STOP command and disconnects the device.

## 10. Example Flow

A simplified sequence is:

```text
Client                   Device
  |                        |
  |-- START(CMD_FLASH) -->|
  |<-- CMD_ACK(0x0000) --|
  |                        |
  |-- sector 0 packets -->|
  |<-- FW_ACK(sector=0, 0) |
  |                        |
  |-- sector 1 packets -->|
  |<-- FW_ACK(sector=1, 0) |
  |                        |
  |-- STOP(CMD_STOP) -->  |
  |<-- CMD_ACK(0x0000) --|
  |                        |
  |<--- reboot / done --- |
```

## 11. Notes on Secure OTA

The code includes logic that treats STOP as the point where the firmware signature is validated. If secure OTA is enabled, `CMD_STOP` may be rejected with a signature error code.

This aligns with the project’s implementation notes:

- success: device reboots after OTA completes
- signature rejected: `0x0003`
- mismatch or flash write error: `0x0001`

## 12. Implementation References

This document is based on the OTA logic implemented in:

- `index.html`

Additional references in the repository:

- `README.md`
- GitHub BLE OTA reference: `https://github.com/gb88/BLEOTA`

## 13. Summary

The  OTA protocol is a compact GATT-based file transfer mechanism using a command channel for lifecycle control and a firmware data channel for sector-based payload transfer. The protocol is resilient to transient BLE issues through sector retries and CRC validation, and it terminates with an explicit STOP command and verification step.
