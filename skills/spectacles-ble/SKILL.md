---
name: spectacles-ble
description: Reference guide for Bluetooth Low Energy (BLE) in Spectacles lenses — covering enabling the experimental BleModule API (Project Settings → Spectacles → Experimental APIs → Bluetooth), scanning for peripherals by service UUID (with stopScan for battery), connecting, discoverServices/discoverCharacteristics, read-once and notify-stream characteristic patterns, DataView byte parsing and little-endian endianness, writing command bytes and floats, MTU negotiation with requestMtu, the disconnection/reconnect loop, BLE MTU limits (20 bytes default, chunking for larger payloads), and the Spectacles Mobile Kit for phone↔lens JSON messaging. Use this skill for any lens that connects to Arduino hardware, a game controller, a BLE sensor, or a companion mobile app — covering BLE Arduino, BLE Playground, BLE Game Controller, and Spectacles Mobile Kit samples.
---

# Spectacles BLE — Reference Guide

Spectacles can act as a **BLE Central** (scanner + connector) to communicate bidirectionally with any BLE Peripheral — an Arduino, a game controller, a phone app, or any custom BLE device.

**Official docs:** [Spectacles Home](https://developers.snap.com/spectacles/home) · [Experimental APIs](https://developers.snap.com/spectacles/permission-privacy/experimental-apis) (BLE enablement)

> **Enable BLE**: *Project Settings → Spectacles → Experimental APIs → ✅ Bluetooth Low Energy*

---

## Core Concepts

| Term | Meaning |
|---|---|
| **Central** | The device that initiates connections (Spectacles) |
| **Peripheral** | The device that advertises and accepts connections (Arduino, phone, etc.) |
| **Service** | A logical grouping of related functionality, identified by a UUID |
| **Characteristic** | A specific data value within a service; readable, writable, and/or notifiable |

---

## BLE Module API

```typescript
const bleModule = require('LensStudio:BleModule')
```

### Scanning for peripherals

```typescript
const TARGET_SERVICE_UUID = '12345678-1234-1234-1234-1234567890ab'

bleModule.startScan([TARGET_SERVICE_UUID], (peripheral: BlePeripheral) => {
  print('Found device: ' + peripheral.name + ' (' + peripheral.id + ')')
  bleModule.stopScan()  // stop immediately — scanning drains battery
  connectToPeripheral(peripheral)
})
```

### Connecting

```typescript
function connectToPeripheral(peripheral: BlePeripheral): void {
  peripheral.connect((success: boolean, error: string) => {
    if (success) {
      print('Connected to ' + peripheral.name)
      discoverServices(peripheral)
    } else {
      print('Connection failed: ' + error)
    }
  })
}
```

### Discovering services and characteristics

```typescript
const SERVICE_UUID = '12345678-1234-1234-1234-1234567890ab'
const CHAR_UUID    = 'abcdef01-1234-1234-1234-1234567890ab'

function discoverServices(peripheral: BlePeripheral): void {
  peripheral.discoverServices([SERVICE_UUID], (services: BleService[]) => {
    const service = services.find(s => s.uuid === SERVICE_UUID)
    if (!service) return

    service.discoverCharacteristics([CHAR_UUID], (characteristics: BleCharacteristic[]) => {
      const char = characteristics.find(c => c.uuid === CHAR_UUID)
      if (char) {
        activeCharacteristic = char
        subscribeToNotifications(char)
      }
    })
  })
}
```

---

## Reading & Writing Characteristics

### Read once
```typescript
activeCharacteristic.read((data: Uint8Array, error: string) => {
  if (error) { print('Read error: ' + error); return }
  const value = new DataView(data.buffer).getFloat32(0, true) // little-endian float
  print('Sensor value: ' + value)
})
```

### Subscribe to notifications (streaming data)
```typescript
function subscribeToNotifications(char: BleCharacteristic): void {
  char.setNotifyValue(true, (data: Uint8Array, error: string) => {
    if (error) { print('Notify error: ' + error); return }
    parseIncomingData(data)
  })
}

function parseIncomingData(data: Uint8Array): void {
  if (data.byteLength < 12) {
    print('[BLE] Unexpected packet length: ' + data.byteLength)
    return
  }
  const view = new DataView(data.buffer)
  const roll  = view.getFloat32(0, true)  // bytes 0–3, little-endian
  const pitch = view.getFloat32(4, true)  // bytes 4–7
  const yaw   = view.getFloat32(8, true)  // bytes 8–11
  print(`Roll: ${roll}, Pitch: ${pitch}, Yaw: ${yaw}`)
}
```

### Write to a characteristic
```typescript
// Send a single byte command
function sendCommand(command: number): void {
  const buffer = new Uint8Array([command])
  activeCharacteristic.write(buffer, true, (success: boolean, error: string) => {
    if (!success) print('Write error: ' + error)
  })
}

// Send a float value
function sendFloat(value: number): void {
  const buffer = new ArrayBuffer(4)
  new DataView(buffer).setFloat32(0, value, true)  // little-endian
  activeCharacteristic.write(new Uint8Array(buffer), false, () => {})
}
```

---

## MTU Negotiation

The default BLE MTU is **20 bytes** per write. For larger payloads, negotiate a higher MTU first:

```typescript
peripheral.requestMtu(512, (negotiatedMtu: number, error: string) => {
  if (error) {
    print('MTU negotiation failed: ' + error)
    return
  }
  print('Negotiated MTU: ' + negotiatedMtu + ' bytes')
  // Now you can write payloads up to (negotiatedMtu - 3) bytes at once
  // (3 bytes reserved for ATT overhead)
})
```

If MTU negotiation isn't possible, split large payloads into 20-byte chunks and send sequentially.

---

## Arduino Pattern

Arduino side (C++):
```cpp
#include <ArduinoBLE.h>
#include <Arduino_LSM9DS1.h>  // IMU

BLEService imuService("12345678-...");
BLECharacteristic imuChar("abcdef01-...", BLERead | BLENotify, 12); // 3 floats = 12 bytes

void loop() {
  float roll, pitch, yaw;
  IMU.readGyroscope(roll, pitch, yaw);
  byte data[12];
  memcpy(data,     &roll,  4);
  memcpy(data + 4, &pitch, 4);
  memcpy(data + 8, &yaw,   4);
  imuChar.writeValue(data, 12);
  delay(20); // 50 Hz
}
```

Lens Studio TypeScript reads the three Euler angles and applies them to a scene object's rotation to mirror the physical board's orientation.

---

## Game Controller Pattern

```typescript
function parseControllerInput(data: Uint8Array): void {
  const buttons = data[0]                       // bitmask: bit 0 = A, bit 1 = B, etc.
  const joyX = (data[1] - 128) / 128           // signed normalised [-1, 1]
  const joyY = (data[2] - 128) / 128

  const buttonA = (buttons & 0x01) !== 0
  if (buttonA) onJump()
  moveCharacter(joyX, joyY)
}
```

Haptic feedback:
```typescript
function rumble(durationMs: number): void {
  sendCommand(0x01)  // start rumble
  const timeout = this.createEvent('DelayedCallbackEvent')
  timeout.bind(() => sendCommand(0x00))  // stop
  timeout.reset(durationMs / 1000)
}
```

---

## Spectacles Mobile Kit

```typescript
const mobileKit = require('LensStudio:SpectaclesMobileKit')

mobileKit.onMessage.add((message: string) => {
  const data = JSON.parse(message)
  print('From phone: ' + JSON.stringify(data))
})

mobileKit.sendMessage(JSON.stringify({ event: 'scoreUpdate', score: 42 }))
```

---

## Connection Lifecycle

```typescript
let retryCount = 0
const MAX_RETRIES = 5

peripheral.onDisconnected.add(() => {
  print('Device disconnected — retrying in 3s')
  hideConnectedUI()
  if (retryCount >= MAX_RETRIES) {
    print('[BLE] Max retries reached. Stopping reconnect.')
    return
  }
  retryCount++
  const retry = this.createEvent('DelayedCallbackEvent')
  retry.bind(() => connectToPeripheral(peripheral))
  retry.reset(3)
})
```

---

## Common Gotchas

- **Enable via**: *Project Settings → Spectacles → Experimental APIs → Bluetooth Low Energy*. **Lenses using Experimental APIs cannot be published** to a wider audience — BLE is development and sideloading only.
- **Scan drains battery** — always call `stopScan()` once you find your target device.
- **Service UUID must match exactly** between lens and peripheral (case-insensitive, dashes required).
- **Data endianness**: Arduino's `memcpy` and `DataView.setFloat32` must agree on byte order (`true` = little-endian on most MCUs).
- **Always check packet length** before reading with `DataView` — a peripheral sending fewer bytes than expected causes silent out-of-bounds reads.
- **Default MTU is 20 bytes** — negotiate with `requestMtu()` for larger payloads; expect up to 3 bytes of ATT overhead.
- **Cap the reconnect loop** to avoid infinite retries against a device that keeps dropping (e.g., a spoofed peripheral).
- **iOS background mode**: if building a companion mobile app, enable CoreBluetooth background mode in the iOS app's entitlements.

---

## Reference Examples
*   [BLEArduino.ts](references/BLEArduino.md) - Shows scanning, connecting, and reading/writing GATT UUIDs from a generic embedded device.

