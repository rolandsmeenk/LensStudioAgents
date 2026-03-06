# BLE — Common GATT UUIDs and Arduino Reference

## Standard Bluetooth GATT service UUIDs (for well-known devices)

| Service | UUID |
|---|---|
| Generic Access | `00001800-0000-1000-8000-00805f9b34fb` |
| Generic Attribute | `00001801-0000-1000-8000-00805f9b34fb` |
| Battery Service | `0000180f-0000-1000-8000-00805f9b34fb` |
| Heart Rate | `0000180d-0000-1000-8000-00805f9b34fb` |
| Device Information | `0000180a-0000-1000-8000-00805f9b34fb` |
| HID (Human Interface) | `00001812-0000-1000-8000-00805f9b34fb` |

> For custom Arduino/IoT devices, use 128-bit UUIDs you define yourself.  
> UUIDs are **case-insensitive** in BLE; Lens Studio accepts any casing.

## Custom UUID pattern (Arduino BLE example)

```cpp
// Arduino sketch — BLE Arduino sample
#include <ArduinoBLE.h>
#include <Arduino_LSM9DS1.h>

// Custom service + characteristic UUIDs
#define SERVICE_UUID  "12345678-1234-1234-1234-1234567890ab"
#define IMU_CHAR_UUID "abcdef01-1234-1234-1234-1234567890ab"

BLEService imuService(SERVICE_UUID);
// 3×float = 12 bytes | BLERead | BLENotify
BLECharacteristic imuChar(IMU_CHAR_UUID, BLERead | BLENotify, 12);

void setup() {
  BLE.begin();
  imuService.addCharacteristic(imuChar);
  BLE.addService(imuService);
  BLE.setAdvertisedService(imuService);
  BLE.advertise();
  IMU.begin();
}

void loop() {
  BLE.poll();
  float roll, pitch, yaw;
  IMU.readGyroscope(roll, pitch, yaw);
  byte buf[12];
  // Little-endian float packing
  memcpy(buf,     &roll,  4);
  memcpy(buf + 4, &pitch, 4);
  memcpy(buf + 8, &yaw,   4);
  imuChar.writeValue(buf, 12);
  delay(20);  // 50 Hz
}
```

## BLE MTU and chunking

The default BLE ATT_MTU on Spectacles is **23 bytes** (20 usable data bytes) without negotiation.

For larger payloads, either:
1. **Negotiate higher MTU** — not always possible with every peripheral
2. **Chunk the data** on the sender and reassemble on the receiver:

```typescript
// Sender-side chunking (peripheral, e.g. Arduino)
const CHUNK_SIZE = 19  // 19 data bytes + 1 sequence byte = 20 total
function sendChunked(data: Uint8Array): void {
  const total = Math.ceil(data.length / CHUNK_SIZE)
  for (let i = 0; i < total; i++) {
    const chunk = new Uint8Array(CHUNK_SIZE + 1)
    chunk[0] = i          // sequence number
    chunk.set(data.slice(i * CHUNK_SIZE, (i + 1) * CHUNK_SIZE), 1)
    activeChar.write(chunk, false, () => {})
  }
}

// Receiver-side reassembly (Spectacles)
const chunks: Uint8Array[] = []
function handleChunk(data: Uint8Array): void {
  const seq = data[0]
  chunks[seq] = data.slice(1)
  if (seq === expectedTotal - 1) {
    const full = reassemble(chunks)
    processComplete(full)
  }
}
```

## Spectacles Mobile Kit — message framing

```typescript
// Lens side
const mobileKit = require('LensStudio:SpectaclesMobileKit')
mobileKit.onMessage.add((msg: string) => {
  const data = JSON.parse(msg) as { type: string; payload: any }
  dispatch(data.type, data.payload)
})
mobileKit.sendMessage(JSON.stringify({ type: 'SCORE', payload: { value: 99 } }))
```

```typescript
// Mobile side (npm: @snap/spectacles-mobile-kit)
import { SpectaclesConnection } from '@snap/spectacles-mobile-kit'
const conn = new SpectaclesConnection()
await conn.connect()
conn.onMessage((msg) => { /* same JSON protocol */ })
conn.sendMessage(JSON.stringify({ type: 'ACK', payload: {} }))
```

