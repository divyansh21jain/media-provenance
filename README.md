# Media Provenance & Integrity Verification System

> Assign an independent, cryptographic identity to every image and video captured via a smartphone camera — ensuring integrity, authenticity, and verifiability.

## The Problem

In an era of AI-generated deep fakes and digital manipulation, there is no reliable way to prove that a photo or video:

- Was captured by an actual camera (not generated)
- Has not been modified since capture
- Was taken at the claimed time and location
- Came from a specific device

## The Solution

This system assigns a **tamper-evident cryptographic identity** to every media capture at the moment it is taken. Anyone can later verify the authenticity of the media.

```
┌──────────────────────────────────────────────────────────────┐
│                    CAPTURE (On Device)                        │
│                                                              │
│  Camera → Raw Bytes → SHA-256 Hash ──┐                       │
│                                      ├─→ Fingerprint → Sign  │
│  Metadata (GPS, time, device) ───────┘       │               │
│                                           Ed25519            │
│  Result: MediaIdentity {                  Private Key        │
│    mediaId, contentHash, fingerprint,                        │
│    signature, publicKey, metadata                            │
│  }                                                           │
└──────────────────────┬───────────────────────────────────────┘
                       │ Register
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                    REGISTRY (Server)                          │
│                                                              │
│  SQLite Database storing:                                    │
│  - Content hash (SHA-256 of media bytes)                     │
│  - Metadata hash (SHA-256 of capture context)                │
│  - Combined fingerprint                                      │
│  - Ed25519 digital signature                                 │
│  - Device public key                                         │
│  - Identity code (MPROV-XXXX-XXXX-XXXX)                     │
└──────────────────────┬───────────────────────────────────────┘
                       │ Verify
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                  VERIFICATION (Anyone)                        │
│                                                              │
│  Upload media OR provide hash/identity code                  │
│                                                              │
│  Checks performed:                                           │
│  ✓ Content hash match (has it been modified?)                │
│  ✓ Metadata integrity (has context been tampered?)           │
│  ✓ Fingerprint binding (are content+metadata linked?)        │
│  ✓ Digital signature (was it signed by the claimed device?)  │
│                                                              │
│  Result: AUTHENTIC ✅ or TAMPERED ❌                          │
└──────────────────────────────────────────────────────────────┘
```

## Quick Start

### 1. Install Dependencies

```bash
cd media-provenance
npm install
```

### 2. Run the Demo

The demo walks through the entire lifecycle — capture, identity generation, registration, verification of authentic media, and detection of tampered media:

```bash
npm run demo
```

### 3. Start the Registry Server

```bash
npm run start:server
# or for development with auto-reload:
npm run start:dev
```

Server runs on `http://localhost:3210`

## API Endpoints

| Method | Endpoint                     | Description                            |
| ------ | ---------------------------- | -------------------------------------- |
| `POST` | `/api/v1/register`           | Register a pre-computed media identity |
| `POST` | `/api/v1/register/upload`    | Register by uploading media + identity |
| `POST` | `/api/v1/verify/upload`      | Verify by uploading media file         |
| `GET`  | `/api/v1/verify/:hash`       | Look up identity by content hash       |
| `GET`  | `/api/v1/lookup/id/:mediaId` | Look up by UUID                        |
| `GET`  | `/api/v1/lookup/code/:code`  | Look up by identity code               |
| `GET`  | `/api/v1/lookup/device/:key` | List all media by a device             |
| `GET`  | `/api/v1/stats`              | Registry statistics                    |
| `GET`  | `/health`                    | Health check                           |

## Usage Examples

### Register a Media Identity (from mobile device)

```bash
curl -X POST http://localhost:3210/api/v1/register \
  -H "Content-Type: application/json" \
  -d '{
    "contentHash": "a1b2c3...",
    "metadataHash": "d4e5f6...",
    "fingerprint": "789abc...",
    "signature": "def012...",
    "publicKey": "345678...",
    "metadata": {
      "capturedAt": "2026-03-02T10:30:00Z",
      "device": {
        "manufacturer": "Apple",
        "model": "iPhone 15 Pro",
        "osVersion": "iOS 17.2",
        "deviceIdHash": "abc123..."
      },
      "mediaType": "IMAGE",
      "originalFilename": "IMG_0001.jpg",
      "fileSizeBytes": 4500000,
      "mimeType": "image/jpeg"
    }
  }'
```

### Verify Media by Upload

```bash
curl -X POST http://localhost:3210/api/v1/verify/upload \
  -F "media=@photo.jpg"
```

### Look up by Identity Code

```bash
curl http://localhost:3210/api/v1/lookup/code/MPROV-A1B2-C3D4-E5F6
```

## Client SDK (for App Integration)

```typescript
import { MediaProvenanceClient } from "media-provenance";

// Initialize (generates key pair automatically)
const client = new MediaProvenanceClient({
  serverUrl: "http://localhost:3210",
});

// Capture and register in one call
const { identity, registration } = await client.captureAndRegister(
  mediaBuffer,
  captureMetadata,
);

console.log(`Registered: ${identity.identityCode}`);

// Later — anyone can verify
const result = await client.verifyByUpload(mediaBuffer);
console.log(result.data?.summary); // "✅ VERIFIED — This media is authentic..."
```

## Cryptographic Details

| Component     | Algorithm                           | Purpose                                |
| ------------- | ----------------------------------- | -------------------------------------- |
| Content Hash  | SHA-256                             | Detect any modification to media bytes |
| Metadata Hash | SHA-256                             | Detect metadata tampering              |
| Fingerprint   | SHA-256(contentHash + metadataHash) | Bind content to its capture context    |
| Signature     | Ed25519                             | Prove device origin (authenticity)     |
| Media ID      | UUID v4                             | Unique identifier                      |
| Identity Code | First 12 hex chars of fingerprint   | Human-readable reference               |

## Security Properties

- **Tamper-Evident**: Changing even 1 bit of the media changes the hash, breaking verification
- **Non-Repudiation**: Ed25519 signature proves which device created the identity
- **Context Binding**: Metadata (time, GPS, device info) is cryptographically bound to content
- **No Secrets on Server**: Only public keys and hashes are stored; private keys never leave the device
- **Input Validation**: All API inputs validated with Zod schemas
- **SQL Injection Prevention**: Parameterized queries throughout

## Project Structure

```
media-provenance/
├── src/
│   ├── types/index.ts          # Type definitions
│   ├── core/
│   │   ├── crypto.ts           # SHA-256 hashing, Ed25519 signing
│   │   ├── identity.ts         # Identity generation engine
│   │   └── verification.ts     # Verification engine
│   ├── storage/
│   │   └── registry.ts         # SQLite-based registry
│   ├── server/
│   │   ├── index.ts            # Express server
│   │   ├── routes.ts           # API route handlers
│   │   └── validation.ts       # Zod input schemas
│   ├── client/
│   │   └── sdk.ts              # Client SDK for app integration
│   ├── capture/
│   │   └── mobile.ts           # Mobile capture simulator + RN guide
│   ├── demo.ts                 # Full lifecycle demo
│   └── index.ts                # Main entry point
├── package.json
├── tsconfig.json
└── README.md
```

## React Native Integration

The capture module includes a complete React Native integration guide. See `src/capture/mobile.ts` for:

- Expo Camera capture
- GPS location extraction
- Device info collection
- Secure key storage (expo-secure-store)
- Registration with the remote registry

## License

MIT
