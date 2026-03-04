---
name: spectacles-ble
description: Reference guide for Bluetooth Low Energy (BLE) in Spectacles lenses — covering enabling the experimental BleModule API, scanning for peripherals by service UUID (with stopScan for battery), connecting, discoverServices/discoverCharacteristics, read-once and notify-stream characteristic patterns, DataView byte parsing and little-endian endianness, writing command bytes and floats, the disconnection/reconnect loop, BLE MTU limits (20 bytes default, chunking for larger payloads), and the Spectacles Mobile Kit for phone↔lens JSON messaging. Use this skill for any lens that connects to Arduino hardware, a game controller, a BLE sensor, or a companion mobile app — covering BLE Arduino, BLE Playground, BLE Game Controller, and Spectacles Mobile Kit samples.
---

# Spectacles BLE — Reference Guide

Spectacles can act as a **BLE Central** (scanner + connector) to communicate bidirectionally with any BLE Peripheral — an Arduino, a game controller, a phone app, or any custom BLE device.

> **Experimental API**: BLE is currently an Experimental API on Spectacles. Enable it in Lens Studio: *Project Settings → Experimental APIs → Bluetooth Low Energy*.

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
const bleModule = require('LensStudio:BleModule');
```

### Scanning for peripherals

```typescript
// Start scanning for devices advertising a specific service UUID
const TARGET_SERVICE_UUID = '12345678-1234-1234-1234-1234567890ab';

bleModule.startScan([TARGET_SERVICE_UUID], (peripheral) => {
  print('Found device: ' + peripheral.name + ' (' + peripheral.id + ')');
  bleModule.stopScan();
  connectToPeripheral(peripheral);
});

// Stop scanning (important for battery)
bleModule.stopScan();
```

### Connecting

```typescript
function connectToPeripheral(peripheral: BlePeripheral) {
  peripheral.connect((success, error) => {
    if (success) {
      print('Connected to ' + peripheral.name);
      discoverServices(peripheral);
    } else {
      print('Connection failed: ' + error);
    }
  });
}
```

### Discovering services and characteristics

```typescript
const SERVICE_UUID = '12345678-1234-1234-1234-1234567890ab';
const CHAR_UUID    = 'abcdef01-1234-1234-1234-1234567890ab';

function discoverServices(peripheral: BlePeripheral) {
  peripheral.discoverServices([SERVICE_UUID], (services) => {
    const service = services.find(s => s.uuid === SERVICE_UUID);
    if (!service) return;

    service.discoverCharacteristics([CHAR_UUID], (characteristics) => {
      const char = characteristics.find(c => c.uuid === CHAR_UUID);
      if (char) {
        activeCharacteristic = char;
        subscribeToNotifications(char);
      }
    });
  });
}
```

---

## Reading & Writing Characteristics

### Read once
```typescript
activeCharacteristic.read((data: Uint8Array, error) => {
  if (error) { print('Read error: ' + error); return; }
  const value = new DataView(data.buffer).getFloat32(0, true); // little-endian float
  print('Sensor value: ' + value);
});
```

### Subscribe to notifications (streaming data)
```typescript
function subscribeToNotifications(char: BleCharacteristic) {
  char.setNotifyValue(true, (data: Uint8Array, error) => {
    if (error) { print('Notify error: ' + error); return; }
    parseIncomingData(data);
  });
}

function parseIncomingData(data: Uint8Array) {
  const view = new DataView(data.buffer);
  const roll  = view.getFloat32(0, true);
  const pitch = view.getFloat32(4, true);
  const yaw   = view.getFloat32(8, true);
  print(`Roll: ${roll}, Pitch: ${pitch}, Yaw: ${yaw}`);
}
```

### Write to a characteristic
```typescript
// Send a single byte command
function sendCommand(command: number) {
  const buffer = new Uint8Array([command]);
  activeCharacteristic.write(buffer, true, (success, error) => {
    if (!success) print('Write error: ' + error);
  });
}

// Send a float value
function sendFloat(value: number) {
  const buffer = new ArrayBuffer(4);
  new DataView(buffer).setFloat32(0, value, true);
  activeCharacteristic.write(new Uint8Array(buffer), false, () => {});
}
```

---

## Arduino Pattern (BLE Arduino Sample)

The Arduino side:
```cpp
#include <ArduinoBLE.h>
#include <Arduino_LSM9DS1.h>  // IMU library

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

Lens Studio script reads the three Euler angles and applies them to a scene object's rotation to mirror the physical board's orientation.

---

## Game Controller Pattern (BLE Game Controller Sample)

Game controllers (HID-over-BLE or custom protocol) typically send button / joystick data as packed bytes:

```typescript
function parseControllerInput(data: Uint8Array) {
  const buttons = data[0];          // bitmask: bit 0 = A, bit 1 = B, etc.
  const joyX = (data[1] - 128) / 128; // joystick X: signed normalised [-1, 1]
  const joyY = (data[2] - 128) / 128; // joystick Y: signed normalised [-1, 1]

  const buttonA = (buttons & 0x01) !== 0;
  if (buttonA) onJump();
  moveCharacter(joyX, joyY);
}
```

Haptic feedback (write to a haptic characteristic):
```typescript
function rumble(intensity: number, durationMs: number) {
  sendCommand(0x01); // start rumble
  const timeout = script.createEvent('DelayedCallbackEvent');
  timeout.bind(() => sendCommand(0x00)); // stop
  timeout.reset(durationMs / 1000);
}
```

---

## Spectacles Mobile Kit

The **Spectacles Mobile Kit** is an SDK for bidirectional BLE communication between a mobile app and a Spectacles lens. It handles pairing, reconnection, and message framing automatically.

### Lens side (Lens Studio)
```typescript
const mobileKit = require('LensStudio:SpectaclesMobileKit');

mobileKit.onMessage.add((message: string) => {
  const data = JSON.parse(message);
  print('From phone: ' + JSON.stringify(data));
});

mobileKit.sendMessage(JSON.stringify({ event: 'scoreUpdate', score: 42 }));
```

### Mobile side (iOS / Android)
Install the Spectacles Mobile Kit SDK from npm:
```bash
npm install @snap/spectacles-mobile-kit
```
Then use the SDK's `SpectaclesConnection` class to scan, connect, and exchange JSON messages.

---

## Connection Lifecycle

Handle disconnections gracefully — BLE is lossy and devices move around:

```typescript
peripheral.onDisconnected.add(() => {
  print('Device disconnected — retrying in 3s');
  hideConnectedUI();
  const retry = script.createEvent('DelayedCallbackEvent');
  retry.bind(() => connectToPeripheral(peripheral));
  retry.reset(3);
});
```

---

## Common Gotchas

- **Experimental API** — BLE may not be available on all Spectacles OS versions; check the compatibility list.
- **Scan drains battery** — always call `stopScan()` once you find your target device.
- **Service UUID must match exactly** between lens and peripheral (case-insensitive, dashes required).
- **Data endianness**: Arduino's `memcpy` and `DataView.setFloat32` must agree on byte order (little-endian by default on most MCUs).
- **BLE MTU** (Maximum Transmission Unit) on Spectacles is typically 20 bytes for writes without negotiation. For larger payloads, split into chunks or negotiate a higher MTU.
- **iOS background mode**: if building a companion mobile app, enable the CoreBluetooth background mode in the iOS app's entitlements.
