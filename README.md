# LoRa Message Dropbox

A **store-and-retrieve messaging system for LoRa networks**. Devices deposit encrypted messages into a **LoRa dropbox node** when the recipient is offline. The recipient later connects and retrieves their messages at any time.

The dropbox **does not forward messages automatically** and **cannot read message contents**, ensuring true end-to-end encryption.

---

## Overview

LoRa networks operate under significant constraints:

- Very low bandwidth
- Duty cycle limits
- Intermittent connectivity
- Limited downlink capacity

Traditional push messaging fails under these conditions. This project implements a **dead-drop mailbox architecture** instead:

1. Sender encrypts a message for the recipient
2. Sender uploads the message to the dropbox
3. Dropbox stores the encrypted payload — untouched
4. Recipient checks the dropbox and retrieves their messages when ready

The result is an **asynchronous messaging system optimized for LoRa constraints**.

---

## System Architecture

```
Device A
   │
   │ LoRa
   ▼
Dropbox Node
(LoRa + Storage)
   │
   ▼
Encrypted Message Storage
   │
   ▼
Device B retrieves later
```

| Component     | Description                                 |
|---------------|---------------------------------------------|
| LoRa Node     | Embedded device sending/receiving messages  |
| Dropbox Node  | LoRa node with persistent storage           |
| Storage       | SD card, flash memory, or small database    |

---

## Message Flow

### Store a Message

Device A sends an encrypted message to the dropbox:

```
STORE
TO:     deviceX
FROM:   deviceA
MSG_ID: 4382
DATA:   <encrypted_payload>
```

The dropbox stores it at:

```
/mailboxes/deviceX/4382.msg
```

### Check for New Messages

```
CHECK deviceX
```

Response:

```
MSG_COUNT 2
```

### List Messages

```
LIST deviceX
```

Response:

```
4382 size=83
4383 size=44
```

### Retrieve a Message

```
GET deviceX 4382
```

The dropbox returns the encrypted payload. The recipient decrypts it locally — the dropbox never sees plaintext.

---

## Encryption Model

Messages are **encrypted by the sender using the recipient's public key** before transmission. The dropbox stores and returns ciphertext only.

```c
ciphertext = encrypt(message, recipient_public_key)
```

**Recommended libraries:** [libsodium](https://libsodium.org) / [NaCl](https://nacl.cr.yp.to)

| Function      | Algorithm          |
|---------------|--------------------|
| Key exchange  | Curve25519         |
| Encryption    | XSalsa20-Poly1305  |

Device identity is derived from the public key:

```c
device_id = hash(public_key)
```

---

## Packet Protocol

Packets are designed to be compact and suitable for constrained LoRa payloads.

| Type     | Purpose              |
|----------|----------------------|
| `STORE`  | Upload a message     |
| `CHECK`  | Check message count  |
| `LIST`   | List message IDs     |
| `GET`    | Retrieve a message   |
| `DELETE` | Remove a message     |

Packet format:

```
| TYPE | SRC | DST | MSG_ID | DATA |
```

Example:

```
STORE deviceX 4382 <ciphertext>
```

---

## Handling LoRa Payload Limits

Typical LoRa payloads range from **50–220 bytes** depending on spreading factor. Large messages are chunked automatically:

```
MSG 4382 PART 1/3
MSG 4382 PART 2/3
MSG 4382 PART 3/3
```

The recipient reassembles parts in order before decryption.

---

## Storage Design

**Filesystem layout:**

```
/mailboxes
    /deviceA
    /deviceB
    /deviceX
        4382.msg
        4383.msg
```

**Database layout (alternative):**

| device_id | msg_id | payload | timestamp |
|-----------|--------|---------|-----------|

**Supported backends:**

- SD card filesystem
- SQLite
- Flash storage

---

## Hardware Options

### Minimal Embedded Node

| Component       | Details              |
|-----------------|----------------------|
| Microcontroller | ESP32                |
| Radio           | SX1276 LoRa module   |
| Storage         | microSD card         |

Best for low-cost, low-power, fully self-contained deployments.

### Full Dropbox Server

| Component | Details       |
|-----------|---------------|
| Host      | Raspberry Pi  |
| Radio     | LoRa HAT      |
| Storage   | SQLite        |

Best for larger storage capacity, better indexing, and easier software updates.

---

## Optional Features

### Message Expiry (TTL)

Messages can carry a time-to-live value. The dropbox periodically purges expired messages.

```
TTL: 7 days
```

### Multi-Dropbox Redundancy

Devices can register with multiple dropboxes to improve delivery reliability:

```
dropbox1, dropbox2, dropbox3
```

### Rate Limiting

To prevent abuse or mailbox flooding, the dropbox can enforce per-device limits on message count and total storage.

---

## Development Roadmap

| Phase | Focus                   | Key Deliverables                                                    |
|-------|-------------------------|---------------------------------------------------------------------|
| 1     | Prototype               | Packet protocol, basic LoRa communication, manual storage/retrieval |
| 2     | Encryption              | Public key identities, end-to-end encryption, secure storage        |
| 3     | Storage System          | Message indexing, listing, and expiry management                    |
| 4     | Network Improvements    | Message chunking, retransmission handling, compression              |
| 5     | Multi-Dropbox Network   | Redundant dropboxes, discovery protocol, optional sync              |

---

## Security Considerations

- All messages must be encrypted end-to-end before transmission
- The dropbox must never have access to plaintext
- Device identity must be cryptographically tied to public keys
- Message IDs should be used to detect and reject replay attacks

---

## Use Cases

- Off-grid and remote area messaging
- Disaster relief communications
- Remote sensor command queues
- Low-power IoT device messaging
- Experimental mesh and delay-tolerant networks

---

## Future Ideas

- LoRa mesh routing between multiple dropboxes
- Anonymous messaging mode
- Broadcast group mailboxes
- Forward error correction (FEC)
- Payload compression for longer messages

---

## License

This project is licensed under the **[GNU General Public License v3.0](https://www.gnu.org/licenses/gpl-3.0.en.html)**.

You are free to use, study, modify, and redistribute this software. Any distributed modifications or derivative works **must also be released under the GPL**, ensuring this project and its improvements remain free and open.
