1. Overview

This guide explains how to implement the OTA update flow on iOS using CoreBluetooth.

The device side follows this pattern:

- Connect to the peripheral
- Discover OTA service and characteristics
- Send START command
- Transfer firmware data sector by sector
- Wait for sector ACK
- Send STOP command
- Verify completion and reboot

The browser implementation uses:
- command characteristic: `00008022-0000-1000-8000-00805f9b34fb`
- firmware receive characteristic: `00008020-0000-1000-8000-00805f9b34fb`
- OTA service: `00008018-0000-1000-8000-00805f9b34fb`

Your iOS app should mirror this protocol exactly.

2. Architecture

A good iOS implementation usually separates responsibilities into:

- BLEManager
  - central scanning / connection / service discovery
  - characteristic read/write / notify setup
- OTAManager
  - command encoding
  - sector packet creation
  - firmware transfer logic
  - retry handling
- OTAViewModel or Controller
  - UI state, progress, errors, user actions

Recommended flow:
- app selects device
- app connects to BLE peripheral
- app discovers OTA service and chars
- app verifies file is valid firmware binary
- app starts OTA session
- app updates progress bar and logs
- app disconnects or keeps device connected after update

3. BLE Service and Characteristics

The OTA service and chars used by the device are:

```swift
let OTA_SERVICE_UUID = CBUUID(string: "00008018-0000-1000-8000-00805f9b34fb")
let OTA_RECV_UUID   = CBUUID(string: "00008020-0000-1000-8000-00805f9b34fb")
let OTA_CMD_UUID    = CBUUID(string: "00008022-0000-1000-8000-00805f9b34fb")
```

Use:
- `OTA_CMD_UUID` to send command frames
- `OTA_RECV_UUID` to send firmware packets and receive ACK notifications

4. OTA Protocol Summary

The protocol is:

- START command
  - command ID = `0x0001` (FLASH)
  - payload = total file size as little-endian `UInt32`
  - CRC16 appended at bytes 18..19

- Send firmware in 4096-byte sectors
  - each sector is split into BLE chunks
  - each packet includes:
    - 2-byte sector index
    - 1-byte sequence number or `0xFF` for final packet
    - payload bytes
    - final packet also includes 2-byte CRC

- Wait for sector ACK
  - ACK format:
    - bytes 0..1 = sector_index
    - bytes 2..3 = ack code
  - ack `0x0000` means success
  - error values trigger retry logic

- STOP command
  - command ID = `0x0002`
  - waits for final command ACK

This implementation is important:
- the app sends data using `writeValue(_:for:type:)` with `.withoutResponse`
- the device sends ACKs as notifications
- the app uses a retry loop for sectors, usually up to 3 attempts

5. iOS Implementation Design

A robust iOS OTA implementation should maintain a finite state machine:

- idle
- connecting
- connected
- discoveringServices
- discoveringCharacteristics
- ready
- startingOTA
- sendingSector
- waitingForAck
- verifyingStop
- completed
- failed
- aborted

This makes error handling much cleaner than using ad hoc flags.

6. CoreBluetooth Setup

Minimal BLE manager pattern:

```swift
import CoreBluetooth
import Foundation

final class BLEManager: NSObject, CBCentralManagerDelegate, CBPeripheralDelegate {
    private let centralManager: CBCentralManager
    private var peripheral: CBPeripheral?

    override init() {
        self.centralManager = CBCentralManager(delegate: nil, queue: nil)
        super.init()
        self.centralManager.delegate = self
    }

    func connect(to peripheral: CBPeripheral) {
        self.peripheral = peripheral
        self.peripheral?.delegate = self
        centralManager.connect(peripheral)
    }

    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        switch central.state {
        case .poweredOn:
            print("Bluetooth ready")
        default:
            print("Bluetooth not ready: \(central.state)")
        }
    }

    func centralManager(_ central: CBCentralManager,
                        didConnect peripheral: CBPeripheral) {
        peripheral.discoverServices([OTA_SERVICE_UUID])
    }

    func peripheral(_ peripheral: CBPeripheral,
                    didDiscoverServices error: Error?) {
        guard error == nil else { return }
        guard let services = peripheral.services else { return }

        for service in services where service.uuid == OTA_SERVICE_UUID {
            peripheral.discoverCharacteristics([OTA_RECV_UUID, OTA_CMD_UUID], for: service)
        }
    }

    func peripheral(_ peripheral: CBPeripheral,
                    didDiscoverCharacteristicsFor service: CBService,
                    error: Error?) {
        guard error == nil else { return }
        guard let characteristics = service.characteristics else { return }

        for characteristic in characteristics {
            if characteristic.uuid == OTA_RECV_UUID {
                peripheral.setNotifyValue(true, for: characteristic)
            }
            if characteristic.uuid == OTA_CMD_UUID {
                peripheral.setNotifyValue(true, for: characteristic)
            }
        }
    }
}
```

7. Command Packet Builder

The command packet format is 20 bytes:

- 2 bytes: command ID
- 16 bytes: payload
- 2 bytes: CRC16

Example Swift implementation:

```swift
import Foundation

func crc16CCITT(data: Data) -> UInt16 {
    var crc: UInt16 = 0xFFFF
    for byte in data {
        crc ^= UInt16(byte)
        for _ in 0..<8 {
            if crc & 0x0001 != 0 {
                crc = (crc >> 1) ^ 0x8408
            } else {
                crc = crc >> 1
            }
        }
    }
    return crc ^ 0xFFFF
}

func buildCommand(id: UInt16, payload: [UInt8]) -> Data {
    var bytes = Data(count: 20)

    bytes[0] = UInt8(id & 0xFF)
    bytes[1] = UInt8((id >> 8) & 0xFF)

    for (i, value) in payload.prefix(16).enumerated() {
        bytes[2 + i] = value
    }

    let crc = crc16CCITT(data: bytes.prefix(18))
    bytes[18] = UInt8(crc & 0xFF)
    bytes[19] = UInt8((crc >> 8) & 0xFF)

    return bytes
}
```

For START:
```swift
let totalSize = firmwareData.count
let startCmd = buildCommand(id: 0x0001, payload: [
    UInt8(totalSize & 0xFF),
    UInt8((totalSize >> 8) & 0xFF),
    UInt8((totalSize >> 16) & 0xFF),
    UInt8((totalSize >> 24) & 0xFF)
])
```

8. Build Firmware Sector Packets

The browser implementation uses a 4096-byte sector size and packet headers:

- header[0..1] = sector_index
- header[2] = sequence or `0xFF`
- payload = chunk bytes
- final packet includes CRC16 of chunk accumulation

Swift example:

```swift
let SECTOR_SIZE = 4096

func buildSectorPackets(sectorData: Data, secIdx: UInt16, mtuPayload: Int) -> [Data] {
    var packets: [Data] = []
    var offset = 0
    var seq: UInt8 = 0
    var crc: UInt16 = 0

    while offset < sectorData.count {
        let remaining = sectorData.count - offset
        let toRead = min(mtuPayload, remaining)
        let chunk = sectorData.subdata(in: offset..<(offset + toRead))
        crc = crc16Update(prev: crc, chunk: chunk)

        let isLast = (offset + toRead >= sectorData.count)

        var packet = Data()
        packet.append(UInt8(secIdx & 0xFF))
        packet.append(UInt8((secIdx >> 8) & 0xFF))

        if isLast {
            packet.append(0xFF)
        } else {
            packet.append(seq)
            seq += 1
        }

        packet.append(chunk)

        if isLast {
            packet.append(UInt8(crc & 0xFF))
            packet.append(UInt8((crc >> 8) & 0xFF))
        }

        packets.append(packet)
        offset += toRead
    }

    return packets
}

func crc16Update(prev: UInt16, chunk: Data) -> UInt16 {
    var crc = prev
    for byte in chunk {
        crc ^= UInt16(byte)
        for _ in 0..<8 {
            if crc & 0x0001 != 0 {
                crc = (crc >> 1) ^ 0x8408
            } else {
                crc = crc >> 1
            }
        }
    }
    return crc
}
```

9. MTU Handling

The browser tool probes workable BLE payload sizes and subtracts the 3-byte OTA header:

```swift
func mtuPayload(for peripheral: CBPeripheral) -> Int {
    // safest default: actual MTU minus 3-byte header
    let mtu = peripheral.maximumWriteValueLength(for: .withoutResponse)
    return max(20, mtu - 3)
}
```

In practice:
- use `maximumWriteValueLength(for: .withoutResponse)` when available
- keep the packet size under the allowed BLE write length
- use a conservative payload size if needed

10. Notification Handling

The app receives:
- command ACK from `OTA_CMD_UUID`
- sector ACK from `OTA_RECV_UUID`

The device notification format is similar to:

- sector ACK:
  - bytes 0..1 = sector index
  - bytes 2..3 = ack status
- command ACK:
  - 2 bytes representing result code (for example `0x0000` success)

Example:

```swift
func peripheral(_ peripheral: CBPeripheral,
                didUpdateValueFor characteristic: CBCharacteristic,
                error: Error?) {
    guard error == nil, let value = characteristic.value else { return }

    if characteristic.uuid == OTA_CMD_UUID {
        if value.count >= 2 {
            let ack = UInt16(value[0]) | (UInt16(value[1]) << 8)
            print("Command ACK = 0x\(String(ack, radix: 16))")
        }
    }

    if characteristic.uuid == OTA_RECV_UUID {
        if value.count >= 4 {
            let sector = UInt16(value[0]) | (UInt16(value[1]) << 8)
            let status = UInt16(value[2]) | (UInt16(value[3]) << 8)
            print("Sector \(sector) ack = 0x\(String(status, radix: 16))")
        }
    }
}
```

11. OTA Transfer Flow in iOS

A typical OTA method looks like this:

```swift
func startOTA(firmwareData: Data) async throws {
    guard let peripheral = peripheral,
          let cmdChar = cmdChar,
          let recvChar = recvChar else {
        throw OTAError.notReady
    }

    let totalSize = firmwareData.count

    // 1) START
    let startCmd = buildCommand(id: 0x0001, payload: [
        UInt8(totalSize & 0xFF),
        UInt8((totalSize >> 8) & 0xFF),
        UInt8((totalSize >> 16) & 0xFF),
        UInt8((totalSize >> 24) & 0xFF)
    ])

    peripheral.writeValue(startCmd, for: cmdChar, type: .withoutResponse)

    let startAck = try await waitForCommandAck()
    guard startAck == 0 else {
        throw OTAError.commandRejected("START")
    }

    // 2) Send data
    var offset = 0
    let maxRetries = 3

    while offset < totalSize {
        let sectorData = firmwareData.subdata(in: offset..<(min(offset + SECTOR_SIZE, totalSize)))
        let sectorIndex = UInt16(offset / SECTOR_SIZE)
        let packets = buildSectorPackets(sectorData: sectorData,
                                         secIdx: sectorIndex,
                                         mtuPayload: mtuPayload(for: peripheral))

        var success = false

        for attempt in 0..<maxRetries {
            // send packets
            for packet in packets {
                peripheral.writeValue(packet, for: recvChar, type: .withoutResponse)
            }

            do {
                let ack = try await waitForSectorAck(expectedSector: sectorIndex)
                if ack == 0 {
                    success = true
                    break
                }
            } catch {
                if attempt == maxRetries - 1 {
                    throw OTAError.sectorRetryExceeded
                }
            }
        }

        guard success else {
            throw OTAError.sectorFailed
        }

        offset += sectorData.count
    }

    // 3) STOP
    let stopCmd = buildCommand(id: 0x0002, payload: [])
    peripheral.writeValue(stopCmd, for: cmdChar, type: .withoutResponse)

    let stopAck = try await waitForCommandAck()
    guard stopAck == 0 else {
        throw OTAError.signatureRejected
    }
}
```

12. Error Handling

Your app should interpret common OTA errors:

- `0x0000`: success
- `0x0001`: size mismatch or flash write error
- `0x0003`: signature rejected
- `0x0005`: cannot start OTA

On the sector side:
- CRC error
- index mismatch
- payload length error
- timeout

A clean error enum:

```swift
enum OTAError: Error {
    case notReady
    case commandRejected(String)
    case signatureRejected
    case sectorFailed
    case sectorRetryExceeded
    case timeout
}
```

13. Retry Strategy

The browser app performs:
- 3 retries per sector
- retry from sequence 0
- wait for sector ACK after burst sending

This pattern is recommended in iOS apps as well.

Good retry logic:
- if ack is `0x0001` or timeout → retry
- if ack is unrecoverable → abort the OTA
- all retries should be bounded
- stop the transfer if user cancels

14. Progress Reporting

Use a `Progress` object or custom state:

- firmware size
- bytes transferred
- percentage
- current sector
- transfer speed
- status text

Example:

```swift
@MainActor
final class OTAViewModel: ObservableObject {
    @Published var progress: Double = 0
    @Published var statusText: String = "Idle"
    @Published var currentSector: Int = 0
    @Published var bytesSent: Int = 0
    @Published var totalBytes: Int = 0
}
```

This can be updated during each sector transfer.

15. Example SwiftUI Flow

```swift
import SwiftUI

struct OTAView: View {
    @StateObject var viewModel = OTAViewModel()

    var body: some View {
        VStack {
            Text(viewModel.statusText)
            ProgressView(value: viewModel.progress)
                .progressViewStyle(.linear)

            Text("Sector: \(viewModel.currentSector)")
            Text("Bytes: \(viewModel.bytesSent) / \(viewModel.totalBytes)")

            Button("Start OTA") {
                Task {
                    do {
                        try await viewModel.startOTA()
                    } catch {
                        print("OTA failed: \(error)")
                    }
                }
            }
            .disabled(viewModel.isUploading)
        }
        .padding()
    }
}
```