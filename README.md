# UPI Offline Mesh Demo

This project is a Spring Boot demonstration of offline UPI-style payments routed through a Bluetooth-like mesh network. It models a situation where a sender has no internet connection, creates an encrypted payment packet, and relies on nearby devices to relay that packet until a bridge device with internet access uploads it to the backend.

The application includes both the server-side payment ingestion pipeline and a software mesh simulator, so the full flow can be demonstrated on a single machine without Bluetooth hardware.

## Project Overview

The system demonstrates three core ideas:

1. A payment packet can travel through untrusted intermediate devices without exposing or allowing modification of the payment details.
2. A payment packet can be delivered multiple times by different bridge nodes while still settling exactly once.
3. Tampered, stale, or duplicate packets are rejected before they affect the ledger.

The demo uses hybrid encryption, idempotency checks, freshness validation, transactional settlement, and an H2 in-memory database.

## Features

- Spring Boot REST backend for mesh-routed payment ingestion
- Thymeleaf dashboard for running the demo flow
- Simulated mobile mesh with offline devices and an internet-enabled bridge node
- Hybrid cryptography using RSA-OAEP and AES-256-GCM
- Idempotency protection using a ciphertext hash
- Replay protection using packet timestamps and nonces
- Transactional debit, credit, and ledger writing with JPA
- H2 in-memory database for zero-setup execution
- Concurrency test proving that duplicate bridge uploads settle only once

## Technology Stack

| Area | Technology |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 3.3.5 |
| Web | Spring MVC, embedded Tomcat |
| UI | Thymeleaf |
| Persistence | Spring Data JPA |
| Database | H2 in-memory database |
| Security model | RSA-OAEP, AES-256-GCM, SHA-256 |
| Testing | JUnit 5, Spring Boot Test |
| Build | Maven Wrapper |

## How It Works

The project simulates the following flow:

1. A sender creates a payment instruction containing sender VPA, receiver VPA, amount, PIN hash, nonce, and timestamp.
2. The instruction is encrypted with a hybrid encryption scheme.
3. The encrypted payload is wrapped in a `MeshPacket`.
4. The packet is injected into a simulated offline phone.
5. Simulated devices gossip packets to nearby devices while decrementing packet TTL.
6. A bridge device with internet access uploads held packets to `/api/bridge/ingest`.
7. The backend hashes the ciphertext and claims the hash in the idempotency cache.
8. If the packet is new, the backend decrypts it, checks freshness, and settles the transaction.
9. Duplicate deliveries are dropped without running settlement again.

## Architecture

```text
Sender device
  |
  | creates PaymentInstruction
  | encrypts with server public key
  v
MeshPacket
  |
  | offline gossip through simulated devices
  v
Bridge device with internet
  |
  | POST /api/bridge/ingest
  v
Spring Boot backend
  |
  | SHA-256 ciphertext hash
  | idempotency claim
  | RSA-OAEP + AES-GCM decrypt
  | freshness validation
  | transactional settlement
  v
H2 ledger and account balances
```

## Main Modules

| File | Purpose |
|---|---|
| `UpiMeshApplication.java` | Spring Boot entry point |
| `ApiController.java` | REST endpoints for demo, mesh, bridge ingestion, accounts, and transactions |
| `DashboardController.java` | Serves the dashboard at `/` |
| `DemoService.java` | Creates encrypted demo packets and seeds demo behavior |
| `MeshSimulatorService.java` | Simulates gossip between virtual mobile devices |
| `VirtualDevice.java` | Represents one simulated phone in the mesh |
| `BridgeIngestionService.java` | Runs the backend ingestion pipeline |
| `SettlementService.java` | Performs debit, credit, and ledger writes in a transaction |
| `IdempotencyService.java` | Tracks already-seen packet hashes |
| `HybridCryptoService.java` | Handles encryption, decryption, and ciphertext hashing |
| `ServerKeyHolder.java` | Generates and stores the server RSA key pair |
| `Account.java` | JPA account entity |
| `Transaction.java` | JPA transaction ledger entity |
| `MeshPacket.java` | Packet format passed through the mesh |
| `PaymentInstruction.java` | Decrypted payment instruction payload |

## Prerequisites

- JDK 17 or newer
- No external database is required
- Maven installation is not required because the project includes the Maven Wrapper

Java can be checked with:

```bash
java -version
```

## Running the Application

On Windows:

```cmd
.\mvnw.cmd spring-boot:run
```

On macOS or Linux:

```bash
./mvnw spring-boot:run
```

After startup, the dashboard is available at:

```text
http://localhost:8080
```

The H2 console is available at:

```text
http://localhost:8080/h2-console
```

H2 connection details:

| Field | Value |
|---|---|
| JDBC URL | `jdbc:h2:mem:upimesh` |
| Username | `sa` |
| Password | empty |

## Demo Flow

The dashboard provides controls for the complete payment journey:

1. A payment is composed and injected into the mesh.
2. Gossip rounds spread the encrypted packet between simulated devices.
3. The bridge node uploads packets to the backend.
4. Account balances and the transaction ledger update after successful settlement.
5. Duplicate packet delivery can be tested through the concurrency test.

The default mesh contains four offline devices and one bridge device:

- `phone-alice`
- `phone-stranger1`
- `phone-stranger2`
- `phone-stranger3`
- `phone-bridge`

## API Reference

| Method | Path | Description |
|---|---|---|
| `GET` | `/` | Serves the dashboard |
| `GET` | `/api/server-key` | Returns the server public key |
| `GET` | `/api/accounts` | Returns account balances |
| `GET` | `/api/transactions` | Returns recent transactions |
| `GET` | `/api/mesh/state` | Returns simulated mesh state |
| `POST` | `/api/demo/send` | Creates and injects an encrypted payment packet |
| `POST` | `/api/mesh/gossip` | Runs one mesh gossip round |
| `POST` | `/api/mesh/flush` | Uploads packets from bridge devices |
| `POST` | `/api/mesh/reset` | Clears mesh state and idempotency cache |
| `POST` | `/api/bridge/ingest` | Ingests a packet from a bridge node |
| `GET` | `/h2-console` | Opens the H2 database console |

Example bridge ingestion request:

```http
POST /api/bridge/ingest
Content-Type: application/json
X-Bridge-Node-Id: phone-bridge
X-Hop-Count: 3

{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 2,
  "createdAt": 1730000000000,
  "ciphertext": "base64-encoded-rsa-and-aes-blob"
}
```

Example response:

```json
{
  "outcome": "SETTLED",
  "packetHash": "a3f8c9...",
  "reason": null,
  "transactionId": 42
}
```

Possible outcomes:

| Outcome | Meaning |
|---|---|
| `SETTLED` | The packet was accepted and settled |
| `DUPLICATE_DROPPED` | The same ciphertext was already processed |
| `INVALID` | The packet failed decryption, freshness checks, or another validation step |

## Security Design

### Hybrid Encryption

The payment instruction is encrypted using a hybrid cryptography pattern:

1. A fresh AES-256 key is generated for the packet.
2. The payment JSON is encrypted with AES-256-GCM.
3. The AES key is encrypted with the server RSA public key using RSA-OAEP.
4. The final ciphertext contains the encrypted AES key, IV, encrypted payload, and GCM authentication tag.

Only the backend holds the private key, so intermediate devices can forward the packet but cannot read or modify its payment details. AES-GCM also detects ciphertext tampering.

### Idempotency

The backend computes `SHA-256(ciphertext)` as the packet hash and claims that hash before decryption or settlement. The demo uses a `ConcurrentHashMap` with atomic `putIfAbsent` semantics.

This prevents duplicate bridge uploads from settling the same payment more than once. In production, this role would typically be handled by Redis using `SET NX EX`.

### Replay Protection

Each encrypted payment instruction contains a timestamp and nonce. The backend rejects packets outside the configured freshness window:

```properties
upi.mesh.packet-max-age-seconds=86400
```

The idempotency cache uses the same default time window:

```properties
upi.mesh.idempotency-ttl-seconds=86400
```

## Running Tests

All tests can be run with:

```cmd
.\mvnw.cmd test
```

The key test class is:

```text
src/test/java/com/demo/upimesh/IdempotencyConcurrencyTest.java
```

It verifies:

- encryption and decryption round trip
- rejection of tampered ciphertext
- exactly-once settlement when three bridge nodes deliver the same packet concurrently

The concurrency test can be run directly with:

```cmd
.\mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

## Production Considerations

This project is a demo and is not a production UPI implementation. A production version would require several major changes:

| Demo Component | Production Equivalent |
|---|---|
| H2 in-memory database | PostgreSQL, MySQL, or another durable database |
| JVM-local idempotency cache | Redis or another distributed idempotency store |
| Startup-generated RSA key pair | HSM, KMS, Vault, or another managed key system |
| Simulated mesh | Real BLE, Wi-Fi Direct, or platform-specific mobile transport |
| Server-side packet creation | Android/iOS client-side packet creation |
| Demo account balances | Bank/NPCI-backed accounts and real authorization |
| Open demo endpoints | Authenticated and rate-limited APIs |
| H2 console enabled | Disabled in deployed environments |

## Limitations

The project demonstrates mesh-routed deferred settlement, not real-time guaranteed offline UPI. Important limitations remain:

- The receiver cannot confirm final settlement while the entire path is offline.
- A sender may attempt multiple offline payments before packets reach the backend.
- Real Bluetooth background discovery and forwarding are platform-constrained.
- Intermediate devices cannot read the ciphertext, but they still observe packet metadata.
- Real financial deployment would require compliance, fraud controls, user authentication, and integration with banking infrastructure.

## Troubleshooting

| Problem | Fix |
|---|---|
| `java: command not found` | Install JDK 17 or newer and set `JAVA_HOME` if required |
| Port `8080` is already in use | Change `server.port` in `src/main/resources/application.properties` |
| Maven Wrapper is not recognized in PowerShell | Run `.\mvnw.cmd spring-boot:run` from the project directory |
| First run takes time | Maven and dependencies are downloaded during the first run |
| H2 console cannot connect | Use `jdbc:h2:mem:upimesh`, username `sa`, and an empty password |

## License

This project is provided as demo code for learning and presentation purposes.
