# MeshPay – Offline Mesh Payment & Deferred Settlement System

**A Spring Boot-based offline payment simulation that routes encrypted payment instructions through a multi-hop device mesh network and settles them when an internet-connected bridge reaches the backend.**

<p align="center">
  <img src="https://img.shields.io/badge/Java-17+-orange?logo=openjdk" />
  <img src="https://img.shields.io/badge/Spring%20Boot-3.3-brightgreen?logo=springboot" />
  <img src="https://img.shields.io/badge/Security-RSA%20%2B%20AES--GCM-blue" />
  <img src="https://img.shields.io/badge/Database-H2-lightgrey" />
  <img src="https://img.shields.io/badge/Build-Maven-red?logo=apachemaven" />
  <img src="https://img.shields.io/badge/Status-Educational%20Demo-blue" />
</p>

---

## Overview

Imagine being in a basement with no internet connectivity. You want to send ₹500 to a friend, but neither phone can connect to a payment server.

**What if your payment could travel through nearby smartphones until one of them gets internet connectivity?**

MeshPay demonstrates this concept through a software-simulated Bluetooth-style mesh network.

The sender creates an encrypted payment instruction, which is forwarded across virtual devices using a gossip-based store-and-forward mechanism. When an internet-connected bridge device receives the packet, it uploads it to the Spring Boot backend.

The backend decrypts, validates, deduplicates, and settles the transaction.

This repository implements the backend and a software simulator of the complete mesh network. No physical Bluetooth hardware or mobile application is required.

> **Important:** MeshPay is an educational demonstration of mesh-routed deferred settlement. It is not an actual UPI implementation and does not integrate with NPCI or real banking infrastructure.

## Key Features

- **Offline Payment Routing:** Simulates payment delivery without requiring direct internet connectivity from the sender.
- **Hybrid Encryption:** Uses RSA-OAEP and AES-256-GCM to protect payment payloads.
- **Multi-Hop Gossip Simulation:** Distributes encrypted packets across virtual devices.
- **Bridge-Based Synchronization:** Internet-connected devices upload stored packets to the backend.
- **Atomic Idempotency:** Prevents duplicate payment processing using SHA-256 ciphertext fingerprints.
- **Replay Protection:** Rejects packets outside the configured freshness window.
- **Tamper Detection:** AES-GCM authentication detects modifications to encrypted payloads.
- **Transactional Settlement:** Performs sender debit, receiver credit, and ledger insertion using Spring transactions.
- **Optimistic Locking:** Uses JPA `@Version` to handle concurrent account updates.
- **Concurrency Testing:** Verifies that simultaneous duplicate deliveries result in exactly one settlement.
- **Interactive Dashboard:** Visualizes packet propagation, bridge uploads, account balances, and transaction history.

## Architecture

### High-Level Architecture

```text
                   SENDER DEVICE
                  (Offline Phone)
                         |
                         v
              +----------------------+
              | Payment Instruction  |
              +----------------------+
                         |
                         v
              +----------------------+
              | Hybrid Encryption    |
              | RSA-OAEP + AES-GCM   |
              +----------------------+
                         |
                         v
              +----------------------+
              |     Mesh Packet      |
              |  TTL + Ciphertext    |
              +----------------------+
                         |
                         v
                MESH NETWORK
              (Gossip Simulation)
                         |
              +----------+----------+
              |          |          |
              v          v          v
           Phone 1    Phone 2    Phone 3
              |          |          |
              +----------+----------+
                         |
                         v
                 BRIDGE DEVICE
                (Internet Available)
                         |
                         v
                   HTTPS POST
                         |
                         v
             +-----------------------+
             |   Spring Boot API     |
             | /api/bridge/ingest    |
             +-----------------------+
                         |
                         v
             +-----------------------+
             | SHA-256 Fingerprint   |
             +-----------------------+
                         |
                         v
             +-----------------------+
             | Idempotency Check     |
             +-----------------------+
                         |
                         v
             +-----------------------+
             | Decrypt & Validate    |
             +-----------------------+
                         |
                         v
             +-----------------------+
             | Settlement Service    |
             +-----------------------+
                         |
                         v
                  H2 DATABASE
                   /       \
                  v         v
              Accounts   Transactions
```

### Payment Processing Pipeline

Every incoming bridge packet passes through the following stages:

1. **Hash Generation:** Calculates SHA-256 of the encrypted packet.
2. **Idempotency Check:** Atomically claims the hash to prevent duplicate processing.
3. **Decryption:** Uses the RSA private key to unwrap the AES key and AES-GCM to decrypt the payload.
4. **Freshness Validation:** Checks whether the payment timestamp falls within the allowed 24-hour window.
5. **Settlement:** Executes debit, credit, and ledger insertion inside a database transaction.

Invalid, tampered, stale, and duplicate packets are rejected without executing the normal settlement flow.

---

## How It Works

### Step 1: Compose a Payment

The sender initiates a payment through the interactive dashboard.

Example:

```json
{
  "sender": "alice",
  "receiver": "bob",
  "amount": 500,
  "pin": "1234"
}
```

The backend simulates the sender's device and creates a `PaymentInstruction` containing the payment details, nonce, and timestamp.

### Step 2: Encrypt the Payment

MeshPay uses hybrid encryption:

- Generates a fresh AES-256 key.
- Encrypts the payment JSON using AES-256-GCM.
- Generates a unique 12-byte IV.
- Encrypts the AES key using RSA-2048 with OAEP padding.
- Combines the encrypted AES key, IV, and encrypted payload.

The resulting ciphertext is unreadable to intermediate devices.

AES-GCM also authenticates the encrypted payload, so unauthorized modifications are detected during decryption.

### Step 3: Create a Mesh Packet

The encrypted payload is wrapped inside a `MeshPacket`.

```json
{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 5,
  "createdAt": 1793600000000,
  "ciphertext": "BASE64_ENCRYPTED_DATA"
}
```

TTL (Time To Live) controls the maximum number of forwarding hops.

### Step 4: Gossip-Based Packet Propagation

The `MeshSimulatorService` simulates device-to-device communication.

Virtual devices exchange packets with other devices during gossip rounds.

Each device temporarily stores the packets it receives and forwards them according to the TTL rules.

Intermediate devices only carry encrypted packets. They do not need to decrypt or understand payment information.

### Step 5: Bridge Upload

An internet-connected device acts as a bridge.

When it obtains connectivity, it uploads its stored packets to:

```http
POST /api/bridge/ingest
```

The backend receives and processes the packet.

### Step 6: Validate and Settle

After successful validation, the settlement service:

- Debits the sender's account.
- Credits the receiver's account.
- Inserts a transaction ledger entry.

All operations are executed transactionally.

---

## Core Engineering Challenges

### 1. Secure Communication Through Untrusted Devices

**Problem:** Intermediate devices should not be able to read or modify payment information.

**Solution:** Hybrid RSA-OAEP + AES-256-GCM encryption.

RSA protects the AES key, while AES encrypts the actual payment payload. GCM provides authenticated encryption and detects ciphertext tampering.

### 2. Concurrent Duplicate Processing

**Problem:** Multiple bridge devices may upload the same packet simultaneously, potentially resulting in multiple debits.

**Solution:** Atomic idempotency using `ConcurrentHashMap.putIfAbsent()`.

```java
Instant previous = seen.putIfAbsent(packetHash, now);

boolean firstRequest = previous == null;
```

Only the first request claiming the ciphertext hash proceeds.

The database also enforces a unique constraint on `packetHash` as a defense-in-depth mechanism.

**Production extension:** Replace the JVM-local map with Redis `SET NX EX` for distributed idempotency across multiple backend instances.

### 3. Replay Attacks

**Problem:** Previously captured payment packets could be submitted again.

**Solution:**

- Each payment includes a unique nonce.
- Each payment includes an authenticated timestamp.
- The backend rejects stale packets.
- Ciphertext-hash idempotency prevents repeated delivery of the same packet.

### 4. Concurrent Account Updates

**Problem:** Simultaneous account modifications can cause conflicting balance updates.

**Solution:**

- Spring `@Transactional` for atomic settlement.
- JPA `@Version` for optimistic locking.
- Unique database constraints for duplicate transaction protection.

---

## Technology Stack

| Category | Technology |
|---|---|
| Language | Java 17+ |
| Backend Framework | Spring Boot 3.3 |
| API Development | Spring MVC / REST |
| Persistence | Spring Data JPA |
| ORM | Hibernate |
| Database | H2 In-Memory Database |
| Symmetric Encryption | AES-256-GCM |
| Asymmetric Encryption | RSA-2048 / OAEP |
| Hashing | SHA-256 |
| Concurrency | ConcurrentHashMap |
| Build Tool | Maven |
| Testing | JUnit |
| Frontend | HTML, CSS, JavaScript |

## Project Structure

```text
upi-offline-mesh/
│
├── pom.xml
├── mvnw
├── mvnw.cmd
├── README.md
│
└── src/
    ├── main/
    │   ├── java/com/demo/upimesh/
    │   │   │
    │   │   ├── UpiMeshApplication.java
    │   │   │
    │   │   ├── model/
    │   │   │   ├── Account.java
    │   │   │   ├── AccountRepository.java
    │   │   │   ├── Transaction.java
    │   │   │   ├── TransactionRepository.java
    │   │   │   ├── MeshPacket.java
    │   │   │   └── PaymentInstruction.java
    │   │   │
    │   │   ├── crypto/
    │   │   │   ├── ServerKeyHolder.java
    │   │   │   └── HybridCryptoService.java
    │   │   │
    │   │   ├── service/
    │   │   │   ├── DemoService.java
    │   │   │   ├── VirtualDevice.java
    │   │   │   ├── MeshSimulatorService.java
    │   │   │   ├── IdempotencyService.java
    │   │   │   ├── SettlementService.java
    │   │   │   └── BridgeIngestionService.java
    │   │   │
    │   │   ├── controller/
    │   │   │   ├── ApiController.java
    │   │   │   └── DashboardController.java
    │   │   │
    │   │   └── config/
    │   │       └── AppConfig.java
    │   │
    │   └── resources/
    │       ├── application.properties
    │       └── templates/
    │           └── dashboard.html
    │
    └── test/
        └── java/com/demo/upimesh/
            └── IdempotencyConcurrencyTest.java
```

## Getting Started

### Prerequisites

- Java JDK 17 or newer
- Git
- Internet connection for the initial Maven dependency download
- No separate Maven installation required (Maven Wrapper is included).

Verify Java:

```bash
java -version
```

### Clone the Repository

```bash
git clone <YOUR_REPOSITORY_URL>
cd upi-offline-mesh
```

### Run on Windows

PowerShell:

```powershell
.\mvnw.cmd spring-boot:run
```

Command Prompt:

```cmd
mvnw.cmd spring-boot:run
```

### Run on macOS / Linux

```bash
chmod +x mvnw
./mvnw spring-boot:run
```

### Open the Dashboard

Once Spring Boot starts successfully, visit:

```text
http://localhost:8080
```

The interactive dashboard allows you to simulate the complete payment journey.

### Stop the Application

Press:

```text
CTRL + C
```

---

## Demo Walkthrough

Follow these steps to demonstrate the complete workflow.

| Step | Dashboard Action | Expected Result |
|---|---|---|
| 1 | Inject into Mesh | Payment encrypted and injected into sender device |
| 2 | Run Gossip Round | Packet distributed among virtual devices |
| 3 | Run Gossip Round Again | Devices retain and propagate packet |
| 4 | Bridges Upload | Internet-connected bridge uploads packet |
| 5 | Observe Dashboard | Account balances and ledger update |
| 6 | Reset Mesh | Simulator state and idempotency cache reset |

### Demonstrating Duplicate Protection

Run the dedicated concurrency test:

```bash
./mvnw test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

On Windows:

```powershell
.\mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

The test submits the same payment from three concurrent threads.

Expected result:

```text
Bridge 1 -> SETTLED
Bridge 2 -> DUPLICATE_DROPPED
Bridge 3 -> DUPLICATE_DROPPED

Total successful settlements: 1
Total duplicate rejections:    2
```

---

## API Reference

| Method | Endpoint | Description |
|---|---|---|
| GET | `/` | Interactive dashboard |
| GET | `/api/server-key` | Retrieve server RSA public key |
| GET | `/api/accounts` | Fetch account balances |
| GET | `/api/transactions` | Fetch recent transactions |
| GET | `/api/mesh/state` | Fetch virtual mesh state |
| POST | `/api/demo/send` | Simulate payment creation |
| POST | `/api/mesh/gossip` | Execute gossip round |
| POST | `/api/mesh/flush` | Upload packets from bridges |
| POST | `/api/mesh/reset` | Reset mesh simulation |
| POST | `/api/bridge/ingest` | Receive and process bridge packets |
| GET | `/h2-console` | H2 database console |

### Bridge Ingestion API

**Request**

```http
POST /api/bridge/ingest
Content-Type: application/json
X-Bridge-Node-Id: phone-bridge-42
X-Hop-Count: 3
```

```json
{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 2,
  "createdAt": 1793600000000,
  "ciphertext": "BASE64_ENCRYPTED_DATA"
}
```

**Successful Response**

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
| SETTLED | Payment successfully processed |
| DUPLICATE_DROPPED | Packet was already claimed |
| INVALID | Packet failed validation |

The exact error response fields and HTTP status codes depend on the controller implementation.

---

## Running Tests

Run all automated tests:

```bash
./mvnw test
```

Important test cases:

| Test | Purpose |
|---|---|
| `encryptDecryptRoundTrip` | Validates encryption and decryption |
| `tamperedCiphertextIsRejected` | Verifies tamper detection |
| `singlePacketDeliveredByThreeBridgesSettlesExactlyOnce` | Verifies concurrent idempotency |

---

## Production Considerations

MeshPay is a proof-of-concept. A real deployment would require significant additional engineering.

| Demo Implementation | Production Direction |
|---|---|
| H2 In-Memory DB | PostgreSQL / MySQL |
| ConcurrentHashMap | Redis distributed idempotency |
| Startup-generated RSA keys | AWS KMS / HSM / Vault |
| Simulated virtual devices | Android BLE / Wi-Fi Direct |
| Demo sender accounts | Bank-integrated account infrastructure |
| Simulated settlement | Authorized banking/payment network |
| No bridge authentication | Mutual TLS and device certificates |
| Demo PIN handling | Secure issuer-side authorization |
| Basic logging | Structured logging, monitoring and alerting |
| No distributed recovery | Durable transaction state and recovery workflows |

### Important Conceptual Limitations

**1. Offline Double Spending**

Without a pre-funded offline wallet or another securely enforced spending limit, a sender may create multiple offline payment instructions against the same balance.

Only one may successfully settle when they reach the backend.

**2. Receiver Cannot Confirm Funds Offline**

A payment instruction is not equivalent to confirmed settlement. The receiver cannot independently establish that the sender has sufficient funds while completely disconnected.

**3. Bluetooth Is Not Implemented**

The current project simulates mesh communication in software. Real BLE networking introduces device discovery, connection management, background execution, battery, and security challenges.

**4. Sender Authentication**

Encryption with the server's public key does not prove the sender's identity. A production implementation requires cryptographic sender authorization and secure device credentials.

**5. Distributed Idempotency**

The current idempotency map is JVM-local. Production deployments need durable, distributed coordination and recovery for interrupted processing.

---

## Future Enhancements

- Replace H2 with PostgreSQL.
- Introduce Redis for distributed idempotency.
- Add Kafka for asynchronous transaction events.
- Implement bridge-node authentication using mutual TLS.
- Introduce sender digital signatures and device identity.
- Build an Android application with real BLE communication.
- Implement persistent transaction states and retry recovery.
- Add a React-based real-time monitoring dashboard.
- Deploy the backend using Docker and AWS.
- Introduce fraud detection and transaction risk scoring.

---

## Key Engineering Concepts Demonstrated

- Hybrid public-key and symmetric cryptography
- Authenticated encryption
- Multi-hop store-and-forward communication
- Gossip-based distributed networking
- Atomic idempotency and concurrency control
- Replay attack mitigation
- Database ACID transactions
- Optimistic locking
- Unique constraints and data integrity
- RESTful backend architecture
- Automated security and concurrency testing

---

## License

This project is an educational demonstration. No formal open-source license has been specified.

---

## Author

**Tanmay Bhadauria**

Java | Spring Boot | Backend Engineering | Distributed Systems | Cryptography

---

**MeshPay — Payments that travel, even when connectivity doesn't.**
