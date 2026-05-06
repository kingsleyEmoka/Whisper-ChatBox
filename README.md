# Whisper-ChatBox# WhisperBox 🔐

### End-to-End Encrypted Messaging — Frontend Wizards Stage 4B

A secure messaging application where the server **never sees plaintext**. All encryption and decryption happens exclusively on the client.

-----

## Live Demo

Open `index.html` in any modern browser (Chrome 80+, Firefox 75+, Safari 14+, Edge 80+).  
Or serve it locally:

```bash
npx serve .
# then visit http://localhost:3000
```

-----

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT (Browser)                         │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  Auth / UI   │    │  Crypto      │    │  Key Storage     │  │
│  │              │    │  Engine      │    │  (IndexedDB)     │  │
│  │  - Login     │───▶│              │    │                  │  │
│  │  - Register  │    │  Web Crypto  │    │  Private key:    │  │
│  │  - Messages  │    │  API         │◀──▶│  PBKDF2-wrapped  │  │
│  │              │    │              │    │  AES-GCM blob    │  │
│  └──────┬───────┘    └──────┬───────┘    └──────────────────┘  │
│         │                  │                                    │
│         │   encrypt before │send / decrypt after receive        │
│         │                  │                                    │
└─────────┼──────────────────┼────────────────────────────────────┘
          │                  │ Only ciphertext crosses network
          ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    HTTPS / TLS Layer                            │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                  WhisperBox Backend (Koyeb)                     │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  Auth API    │    │  Users API   │    │  Messages API    │  │
│  │              │    │              │    │                  │  │
│  │  /auth/login │    │  /users/:id  │    │  /messages/:to   │  │
│  │  /auth/reg   │    │  (pub key)   │    │  (ciphertext     │  │
│  │              │    │              │    │   blobs only)    │  │
│  └──────────────┘    └──────────────┘    └──────────────────┘  │
│                                                                 │
│   ⚠️  Server stores: public keys, ciphertext, user identities  │
│   ✅  Server NEVER sees: plaintext, private keys, AES keys     │
└─────────────────────────────────────────────────────────────────┘
```

-----

## Encryption Flow

### Registration

```
1. Browser generates RSA-OAEP 2048-bit key pair (Web Crypto API)
2. Public key → exported as SPKI base64 → sent to backend /auth/register
3. Private key → wrapped with PBKDF2(password, salt, 100k iterations) → AES-GCM encrypted
4. Wrapped private key blob → stored in IndexedDB (never localStorage raw)
5. Raw private key → held in memory for session duration only
```

### Sending a Message

```
1. User types plaintext
2. Random 256-bit AES-GCM key generated (one per message)
3. Plaintext encrypted with AES-GCM key + random IV → ciphertext
4. AES key exported and encrypted with recipient's RSA public key → encKeyForRecipient
5. AES key also encrypted with sender's RSA public key → encKeyForSender (for sender's own history)
6. Payload { ciphertext, iv, encKeyForRecipient, encKeyForSender } → JSON → sent to server
7. Server stores JSON blob; sees no plaintext at any step
```

### Receiving a Message

```
1. Fetch messages from backend → receive JSON blobs
2. If message.sender === me → decrypt encKeyForSender with my private key
   Else → decrypt encKeyForRecipient with my private key
3. Decrypted raw AES key → imported as CryptoKey
4. ciphertext decrypted with AES-GCM key + stored IV → plaintext
5. Display in UI; on failure → show "[Decryption failed]" gracefully
```

-----

## Key Management

|Key                     |Location               |Protection                |Lifetime        |
|------------------------|-----------------------|--------------------------|----------------|
|RSA Public Key          |Backend server + memory|None (public)             |Account lifetime|
|RSA Private Key (raw)   |Memory only            |Never persisted           |Session only    |
|RSA Private Key (stored)|IndexedDB              |AES-GCM wrapped via PBKDF2|Device lifetime |
|AES Message Key         |Memory only            |RSA-encrypted in transit  |One message     |

### Private Key Wrapping

```
password ──▶ PBKDF2 ──▶ AES-256-GCM wrapping key
                  (salt: random 16 bytes, 100,000 iterations)
                         │
private key (pkcs8) ─────▶ wrapKey → encrypted blob
                                        │
                               stored in IndexedDB
                               { wrapped, salt, iv }
```

On login: PBKDF2 re-derives wrapping key from password → unwrapKey → private key in memory.

-----

## Security Trade-offs

### Decisions Made

|Decision                     |Rationale                                        |Trade-off                                                      |
|-----------------------------|-------------------------------------------------|---------------------------------------------------------------|
|RSA-OAEP 2048                |Well-supported by all browsers via Web Crypto    |Larger key size vs ECDH; simpler UX (no DH negotiation)        |
|Hybrid encryption (RSA + AES)|RSA can’t encrypt large messages                 |AES key per message is fast and secure                         |
|PBKDF2 key wrapping          |Standard, Web Crypto native, no deps             |Argon2 would be stronger; PBKDF2 with 100k rounds is acceptable|
|IndexedDB for private key    |Never in localStorage, origin-isolated, encrypted|Key lost if user clears browser storage                        |
|Sender also encrypts for self|Sender can view their own sent messages          |Slightly larger payload per message                            |
|Polling (4s interval)        |No WebSocket dependency, simpler                 |Less real-time than WebSocket; battery on mobile               |

### What the Server CAN See

- Usernames and who is messaging whom (metadata)
- Message timestamps
- Approximate message sizes
- Public keys

### What the Server CANNOT See

- Message content (always AES-GCM ciphertext)
- Private keys (never transmitted)
- AES message keys (always RSA-encrypted before transmission)

-----

## Known Limitations

1. **No perfect forward secrecy** — RSA keys are long-lived. If a private key is ever compromised, historical messages encrypted to that key could be decrypted. ECDH ephemeral key exchange (Signal-style Double Ratchet) would solve this.
1. **Metadata leakage** — Server knows who talks to whom and when, even if not what.
1. **Device-bound keys** — Private key is stored in the browser’s IndexedDB. Logging in on a new device requires re-registration (new key pair), meaning old messages sent to the old key are unreadable on the new device.
1. **No key verification / Trust On First Use (TOFU)** — Users cannot verify they have the correct public key for a recipient; a compromised server could serve a malicious public key (MITM). A key fingerprint verification UI would mitigate this.
1. **Polling vs WebSocket** — Messages update every 4 seconds rather than instantly.
1. **Single device** — No multi-device key sync. Each device has its own key pair.
1. **Message ordering** — Relies on server timestamps; clock skew could affect ordering.

-----

## Bonus Security Considerations

### Replay Attack Mitigation

Each message uses a freshly generated random IV for AES-GCM. AES-GCM with a unique IV is nonce-misuse resistant — a replayed ciphertext with the same IV would require the same AES key, which is re-generated per message. The backend should enforce message ID uniqueness.

### Forward Secrecy (Bonus)

Not fully implemented (see limitation #1). The architecture supports an upgrade path:

- Replace RSA with ECDH (X25519) for key agreement
- Implement ephemeral key pairs per session
- Add Double Ratchet for per-message key derivation

-----

## Tech Stack

|Layer              |Technology                                |
|-------------------|------------------------------------------|
|Encryption         |Web Crypto API (built-in browser standard)|
|Symmetric          |AES-GCM 256-bit                           |
|Asymmetric         |RSA-OAEP 2048-bit, SHA-256                |
|Key Derivation     |PBKDF2, 100,000 iterations, SHA-256       |
|Private Key Storage|IndexedDB (wrapped/encrypted)             |
|Frontend           |Vanilla HTML/CSS/JS (zero dependencies)   |
|Fonts              |IBM Plex Mono + Syne (Google Fonts)       |
|Backend            |WhisperBox API (whisperbox.koyeb.app)     |
|Auth               |JWT Bearer tokens                         |

-----

## File Structure

```
whisperbox/
├── index.html      # Complete app (single file, no build step)
└── README.md       # This file
```

The entire application is a single HTML file with no build tooling, no npm, no bundler — just the browser’s native Web Crypto API.

-----

## How to Test

1. Open `index.html` in two browser windows (or two different browsers)
1. Register User A in window 1, Register User B in window 2
1. In window 1: type User B’s username in the sidebar → start conversation → send a message
1. In window 2: the message appears decrypted using User B’s private key
1. Open DevTools Network tab → confirm the payload is a ciphertext blob, not plaintext
1. Open DevTools Application → IndexedDB → confirm private key is stored encrypted

-----