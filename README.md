# UPI — Offline Mesh Payment Router

**A Spring Boot backend that routes UPI payments through a Bluetooth mesh network when the internet is down.**

Send ₹500 to your friend in a basement with zero connectivity. Your phone encrypts the payment, broadcasts it through nearby devices, and it hops device-by-device until someone walks outside and uploads it to the backend. The system decrypts, deduplicates, and settles—even if multiple copies arrive simultaneously.

> This is the **server-side implementation + mesh simulator**. Run the entire system on one laptop without physical Bluetooth hardware.

---

## Why This Matters

  **End-to-end encrypted** — No intermediary reads your transaction  
  **Truly idempotent** — Same packet arriving 5 times? Settles exactly once  
  **Tamper-proof** — Invalid packets rejected before touching the ledger  

---

## Quick Start

### Requirements
- **JDK 17+** (check: `java -version`)
- That's it. No database, no external dependencies.

### Run
```bash
# Windows
mvnw.cmd spring-boot:run

# Mac/Linux
./mvnw spring-boot:run
```

Open **http://localhost:8080** → Interactive dashboard appears (dark theme, 4-step demo).

### Tests
```bash
mvnw.cmd test
```
Highlights: `IdempotencyConcurrencyTest` fires 3 threads at one packet simultaneously and asserts exactly one settles.

---

## The Flow

| Step | Action | What Happens |
|------|--------|---|
| 1 | **Compose** | Choose sender/receiver/amount. Server encrypts with RSA, wraps in mesh packet. |
| 2 | **Gossip** | Packet hops through virtual devices. TTL decrements each hop. |
| 3 | **Bridge** | Device with internet uploads to `/api/bridge/ingest`. Backend decrypts + settles. |
| 4 | **Verify** | Same packet from 5 devices? All dropped except first. Balances updated once. |

---

## Architecture at a Glance

```
Sender Phone (offline)
  ↓ encrypt with server RSA public key
MeshPacket { packetId, ttl, createdAt, ciphertext }
  ↓ Bluetooth gossip (hops: +1 ttl/device)
[stranger1] → [stranger2] → [bridge with 4G]
                                    ↓ HTTPS POST
                            Backend Pipeline
                            ├─ hash ciphertext (SHA-256)
                            ├─ claim in idempotency cache
                            ├─ decrypt with RSA private key
                            ├─ verify freshness (< 24h)
                            └─ atomic debit/credit transaction
                            ↓ Success
                    Account Balances Updated
                    Transaction Ledger Entry
```

---

## How Idempotency Works

**The killer feature:** One payment instruction encrypted → same ciphertext → same hash. If the same hash arrives 100 times (via different bridges, retries, network duplicates), **only the first settles**. The rest are dropped as `DUPLICATE_DROPPED` without touching the account.

- **Unique nonce inside** the encrypted payload ensures legitimate repeated sends have different ciphertexts
- **Atomic compare-and-set** on ciphertext hash prevents race conditions
- **Concurrent idempotency test** proves: 3 threads, 1 packet, simultaneous delivery = exactly 1 settled, 2 dropped, sender debited once

---

## API Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/bridge/ingest` | **Production endpoint.** Bridge nodes POST mesh packets here. |
| POST | `/api/demo/send` | Simulate a sender phone (dev/demo only). |
| POST | `/api/mesh/gossip` | Run one gossip round (dev/demo only). |
| POST | `/api/mesh/flush` | Bridges upload to backend (dev/demo only). |
| GET | `/api/accounts` | View account balances. |
| GET | `/api/transactions` | View last 20 settled transactions. |
| GET | `/api/server-key` | Get server's RSA public key (for client apps). |
| GET | `/` | Dashboard UI. |

### Request: `/api/bridge/ingest`
```json
POST /api/bridge/ingest
Content-Type: application/json
X-Bridge-Node-Id: phone-bridge-42

{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 2,
  "createdAt": 1730000000000,
  "ciphertext": "base64-rsa-aes-blob"
}
```

Response:
```json
{
  "outcome": "SETTLED",
  "packetHash": "a3f8c9...",
  "transactionId": 42
}
```

---

## Project Structure

```
src/main/java/com/demo/upimesh/
├── UpiMeshApplication.java      Spring Boot main
├── controller/                  HTTP endpoints
├── service/                     Business logic: gossip, settlement, idempotency
├── crypto/                      RSA-2048 + AES-256-GCM hybrid encryption
├── model/                       JPA entities: Account, Transaction, MeshPacket
└── config/                      Spring configuration
```

---

## What's Real vs. Demo

| Demo | Production |
|------|-----------|
| H2 in-memory DB | PostgreSQL + replicas |
| ConcurrentHashMap idempotency | Redis `SET NX EX` |
| RSA key regenerated on startup | HSM-backed private key |
| Software mesh simulator | Real BLE/Wi-Fi Direct |
| No bridge auth | Mutual TLS + signed certificates |
| Seeded accounts | Real KYC + banking core integration |

**The cryptography and idempotency are production-ready.** The infrastructure layers are intentionally simplified for teaching.

---

## Known Limitations

1. **Receiver can't verify funds offline** — It's an IOU until backend confirms. (Real UPI Lite uses pre-funded hardware wallets.)
2. **Offline double-spend possible** — Sender could send ₹500 to two people; backend settles first, rejects second.
3. **Bluetooth is hard in practice** — Real-world BLE background sync on Android/iOS is heavily throttled. This demo sidesteps it.
4. **Metadata privacy** — Encrypted packet exists on intermediary phones; regulatory considerations apply in production.

**For portfolio projects:** Call it "mesh-routed deferred settlement," not "real-time offline UPI," and you'll have a much stronger pitch. The engineering here is solid.

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `java: command not found` | Install JDK 17+ |
| Port 8080 in use | Change `server.port` in `application.properties` |
| First run hangs | Downloading Maven (~10 MB) + deps (~80 MB). Wait 2–3 min. |
| Tests flaky | Concurrency test is timing-sensitive. Re-run a few times. |

---

## License

MIT — Use freely for learning and projects.

---

**Built with:** Spring Boot 3.3 | Java 17 | H2 in-memory DB | Hybrid RSA + AES encryption  
**Tests:** JUnit 5 | Concurrent idempotency proofs  
**Dashboard:** Dark-themed interactive demo UI