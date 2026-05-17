# Core Services

<cite>
**Referenced Files in This Document**
- [app/services/blockchain/scanner.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/scanner.py)
- [app/services/blockchain/sweeper.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/sweeper.py)
- [app/services/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/webhook.py)
- [app/utils/crypto.py](https://github.com/rakibhossain72/ctrip/blob/main/app/utils/crypto.py)
- [app/blockchain/base.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/base.py)
- [app/blockchain/w3.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/w3.py)
- [app/blockchain/manager.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/manager.py)
- [app/db/models/payment.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/payment.py)
- [app/db/models/chain.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/chain.py)
- [app/core/config.py](https://github.com/rakibhossain72/ctrip/blob/main/app/core/config.py)
- [app/workers/listener.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/listener.py)
- [app/workers/sweeper.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/sweeper.py)
- [app/workers/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/webhook.py)
- [app/api/dependencies.py](https://github.com/rakibhossain72/ctrip/blob/main/app/api/dependencies.py)
- [app/api/v1/payments.py](https://github.com/rakibhossain72/ctrip/blob/main/app/api/v1/payments.py)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Project Structure](#project-structure)
3. [Core Components](#core-components)
4. [Architecture Overview](#architecture-overview)
5. [Detailed Component Analysis](#detailed-component-analysis)
6. [Dependency Analysis](#dependency-analysis)
7. [Performance Considerations](#performance-considerations)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Conclusion](#conclusion)
10. [Appendices](#appendices)

## Introduction
This document describes the core services powering the cTrip Payment Gateway business logic. It focuses on:
- Blockchain scanner service for real-time payment detection via ChainSniper WebSocket listeners and periodic confirmation monitoring across multiple chains
- Sweeper service for automated fund transfer to admin wallets after successful payments (currently a placeholder — marks payments as settled without broadcasting transactions)
- Webhook service for notifying external systems about payment status changes
- Crypto utilities for HD wallet management, address generation, and cryptographic operations

It also covers initialization patterns, dependency injection, error handling strategies, service interfaces, integration patterns, performance considerations, retry logic, and monitoring approaches.

> **Note on Worker System**: The background processing layer uses **ARQ** (not Dramatiq). Workers are started with `python run_worker.py`. The `WorkerSettings` class in `app/workers/worker.py` defines all cron jobs and task functions.

## Project Structure
The core services are organized around domain-driven boundaries:
- Services: business logic for scanning, sweeping, and webhook delivery
- Blockchain: chain abstraction and RPC integration via AsyncWeb3
- Utils: cryptographic helpers and HD wallet management
- Workers: scheduled actors orchestrated by ARQ
- API: payment creation and dependency injection points
- DB Models: persistence schema for payments and chain state
- Config: centralized settings and validation

```mermaid
graph TB
subgraph "API Layer"
API["FastAPI Router<br/>/api/v1/payments"]
Admin["FastAPI Router<br/>/admin/*"]
end
subgraph "Workers (ARQ)"
Listener["ARQ Cron: listen_for_payments<br/>(every second)"]
SweeperActor["ARQ Cron: sweep_funds<br/>(every 30s, disabled)"]
end
subgraph "ChainSniper (WebSocket)"
Sniper["ScannerService.start_listeners()<br/>(started on worker startup)"]
end
subgraph "Services"
Scanner["ScannerService"]
Sweeper["SweeperService"]
WebhookSvc["WebhookService"]
end
subgraph "Blockchain"
W3["get_w3(chain_name)"]
Manager["get_blockchains()"]
Base["BlockchainBase (AsyncWeb3)"]
end
subgraph "Crypto"
HD["HDWalletManager"]
end
subgraph "Persistence"
PaymentModel["Payment Model"]
ChainStateModel["ChainState Model"]
end
API --> HD
API --> PaymentModel
Admin --> Listener
Admin --> SweeperActor
Sniper --> Scanner
Listener --> Scanner
SweeperActor --> Sweeper
Scanner --> W3
Sweeper --> W3
W3 --> Manager
Manager --> Base
Scanner --> PaymentModel
Scanner --> ChainStateModel
Sweeper --> PaymentModel
Scanner --> WebhookSvc
```

**Diagram sources**
- [app/api/v1/payments.py](https://github.com/rakibhossain72/ctrip/blob/main/app/api/v1/payments.py#L1-L62)
- [app/workers/listener.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/listener.py#L1-L46)
- [app/workers/sweeper.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/sweeper.py#L1-L40)
- [app/workers/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/webhook.py#L1-L37)
- [app/services/blockchain/scanner.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/scanner.py#L1-L134)
- [app/services/blockchain/sweeper.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/sweeper.py#L1-L54)
- [app/services/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/webhook.py#L1-L45)
- [app/blockchain/w3.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/w3.py#L1-L9)
- [app/blockchain/manager.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/manager.py#L1-L33)
- [app/blockchain/base.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/base.py#L1-L146)
- [app/utils/crypto.py](https://github.com/rakibhossain72/ctrip/blob/main/app/utils/crypto.py#L1-L90)
- [app/db/models/payment.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/payment.py#L1-L74)
- [app/db/models/chain.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/chain.py#L1-L17)

**Section sources**
- [app/api/v1/payments.py](https://github.com/rakibhossain72/ctrip/blob/main/app/api/v1/payments.py#L1-L62)
- [app/workers/listener.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/listener.py#L1-L46)
- [app/workers/sweeper.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/sweeper.py#L1-L40)
- [app/workers/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/webhook.py#L1-L37)
- [app/services/blockchain/scanner.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/scanner.py#L1-L134)
- [app/services/blockchain/sweeper.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/sweeper.py#L1-L54)
- [app/services/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/webhook.py#L1-L45)
- [app/blockchain/w3.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/w3.py#L1-L9)
- [app/blockchain/manager.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/manager.py#L1-L33)
- [app/blockchain/base.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/base.py#L1-L146)
- [app/utils/crypto.py](https://github.com/rakibhossain72/ctrip/blob/main/app/utils/crypto.py#L1-L90)
- [app/db/models/payment.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/payment.py#L1-L74)
- [app/db/models/chain.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/chain.py#L1-L17)

## Core Components
- ScannerService: Provides real-time detection via ChainSniper WebSocket listeners (`start_listeners()`), handles native and ERC20 transfer callbacks, and runs periodic confirmation/expiry checks.
- SweeperService: Transfers funds from payment addresses to the admin wallet after confirmation and marks payments as settled. **Currently a placeholder** — logs sweep attempts and marks as settled without broadcasting actual transactions.
- WebhookService: Sends signed webhook notifications to external systems with HMAC-SHA256 when a secret is provided.
- HDWalletManager: Generates deterministic payment addresses using BIP-44 derivation from a mnemonic.
- BlockchainBase: Provides AsyncWeb3 integration, gas estimation, transaction building, and receipt polling.
- Worker orchestration: ARQ cron jobs schedule periodic confirmation checks, sweeping, and webhook delivery.

Key interfaces and responsibilities:
- ScannerService.start_listeners(): Starts one ChainSniper WebSocket listener per chain with a ws:// URL.
- ScannerService.confirm_payments(chain_name): Confirms detected payments based on required confirmations and emits webhooks.
- ScannerService.check_expired_payments(): Marks PENDING/DETECTED payments as EXPIRED past their deadline.
- SweeperService.sweep_confirmed_payments(chain_name): Sweeps confirmed payments to the admin wallet (placeholder).
- WebhookService.send_webhook(url, payload, secret): Asynchronously posts signed webhook payloads.
- HDWalletManager.get_address(index): Derives a checksummed address for a payment index.
- BlockchainBase.build_transaction(...) and send_transaction(...): Constructs and submits transactions with dynamic gas pricing.

**Section sources**
- [app/services/blockchain/scanner.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/scanner.py#L14-L134)
- [app/services/blockchain/sweeper.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/sweeper.py#L11-L54)
- [app/services/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/webhook.py#L10-L45)
- [app/utils/crypto.py](https://github.com/rakibhossain72/ctrip/blob/main/app/utils/crypto.py#L5-L67)
- [app/blockchain/base.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/base.py#L22-L146)

## Architecture Overview
The system integrates FastAPI for payment creation, ARQ for background jobs, SQLAlchemy for persistence, and AsyncWeb3 for blockchain interactions. Configuration is centralized via Settings with validation and environment-aware defaults.

```mermaid
sequenceDiagram
participant Client as "Client"
participant API as "FastAPI Payments API"
participant DB as "SQLAlchemy ORM"
participant HD as "HDWalletManager"
participant Sniper as "ChainSniper (WebSocket)"
participant Scanner as "ScannerService"
participant W3 as "AsyncWeb3 via get_w3"
participant Chain as "BlockchainBase"
participant WebhookSvc as "WebhookService"
Client->>API : "POST /api/v1/payments"
API->>HD : "get_address(index)"
HD-->>API : "checksummed address"
API->>DB : "create Payment record"
DB-->>API : "Payment persisted"
API-->>Client : "PaymentRead"
Note over Sniper : "Always-on WebSocket listener"
Sniper->>Scanner : "_on_block(block, chain)"
Scanner->>DB : "match tx.to → update Payment.status to detected"
Sniper->>Scanner : "_on_log(log, chain)"
Scanner->>DB : "match ERC20 Transfer → update Payment.status to detected"
Note over Scanner : "ARQ cron (every second)"
Scanner->>W3 : "get latest block_number"
Scanner->>DB : "select DETECTED payments"
Scanner->>DB : "update Payment.status to confirmed"
Scanner->>WebhookSvc : "send_webhook(url, payload, secret)"
WebhookSvc-->>Scanner : "success/failure"
```

**Diagram sources**
- [app/api/v1/payments.py](https://github.com/rakibhossain72/ctrip/blob/main/app/api/v1/payments.py#L18-L54)
- [app/workers/listener.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/listener.py#L21-L46)
- [app/services/blockchain/scanner.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/scanner.py#L20-L134)
- [app/blockchain/w3.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/w3.py#L6-L9)
- [app/blockchain/base.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/base.py#L34-L146)
- [app/workers/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/webhook.py#L13-L37)
- [app/services/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/webhook.py#L12-L45)

## Detailed Component Analysis

### ScannerService
Responsibilities:
- Scan a chain within a bounded block range
- Detect native ETH transfers and ERC20 token transfers
- Update payment records to detected and later confirmed
- Emit webhooks on confirmation when configured

Key behaviors:
- Uses ChainState to track last scanned block per chain
- Filters pending payments by chain and address
- Compares received amounts against required amounts
- Applies configurable confirmations threshold before marking confirmed
- Emits webhooks asynchronously via ARQ

```mermaid
flowchart TD
WorkerStart(["Worker Startup"]) --> StartListeners["ScannerService.start_listeners()"]
StartListeners --> CreateSniper["Create ChainSniper task per chain (ws:// URL)"]
CreateSniper --> OnBlock["_on_block(block, chain_name)"]
CreateSniper --> OnLog["_on_log(log, chain_name)"]
OnBlock --> MatchNative["Match tx.to to PENDING native payments"]
OnLog --> MatchERC20["Match ERC20 Transfer log to PENDING token payments"]
MatchNative --> SetDetected["status = DETECTED, detected_in_block = block_number"]
MatchERC20 --> SetDetected
SetDetected --> DispatchWebhook["_dispatch_webhook(payment)"]

Cron(["ARQ Cron (every second)"]) --> ConfirmStep["confirm_payments(chain_name)"]
ConfirmStep --> GetLatest["get latest block_number via Web3"]
GetLatest --> CheckConf{"confirmations >= required?"}
CheckConf --> |Yes| MarkConfirmed["status = CONFIRMED, dispatch webhook"]
CheckConf --> |No| NextPayment["Next payment"]
MarkConfirmed --> Commit["Commit session"]

Cron --> ExpireStep["check_expired_payments()"]
ExpireStep --> CheckExpiry{"expires_at <= now?"}
CheckExpiry --> |Yes| MarkExpired["status = EXPIRED, dispatch webhook"]
CheckExpiry --> |No| Skip["Skip"]
```

**Diagram sources**
- [app/services/blockchain/scanner.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/scanner.py#L20-L134)
- [app/db/models/chain.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/chain.py#L9-L17)
- [app/db/models/payment.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/payment.py#L41-L57)

**Section sources**
- [app/services/blockchain/scanner.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/scanner.py#L14-L134)
- [app/db/models/chain.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/chain.py#L9-L17)
- [app/db/models/payment.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/payment.py#L41-L57)

### SweeperService
Responsibilities:
- Sweep confirmed payments to an admin wallet
- Marks payments as settled after sweeping

**Current implementation status: Placeholder**
The sweeper currently logs sweep attempts and marks payments as `settled` without broadcasting actual blockchain transactions. The code includes comments describing the intended full implementation:
1. Get private key for `payment.address` from HD wallet
2. Check balance (native or token)
3. If native: send all minus gas to admin address
4. If token: send native for gas if needed, then send all tokens to admin address
5. Mark as "settled"

```mermaid
flowchart TD
SStart(["sweep_confirmed_payments(chain_name)"]) --> LoadAdmin["Load admin account from private key"]
LoadAdmin --> FetchConfirmed["Select confirmed payments for chain"]
FetchConfirmed --> Any{"Any confirmed payments?"}
Any --> |No| SEnd["Log and exit"]
Any --> |Yes| ForEach["For each payment"]
ForEach --> LogSweep["Log sweep attempt"]
LogSweep --> MarkSettled["Mark status=settled (placeholder)"]
MarkSettled --> Next{"More payments?"}
Next --> |Yes| ForEach
Next --> |No| Commit["Commit session"]
Commit --> SEnd
```

**Diagram sources**
- [app/services/blockchain/sweeper.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/sweeper.py#L16-L54)
- [app/db/models/payment.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/payment.py#L41-L57)

**Section sources**
- [app/services/blockchain/sweeper.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/sweeper.py#L11-L54)
- [app/db/models/payment.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/payment.py#L41-L57)

### WebhookService
Responsibilities:
- Send signed webhook payloads to external systems
- Sign payloads using HMAC-SHA256 when a secret is provided
- Asynchronous HTTP posting with structured logging

Integration pattern:
- Called by ScannerService on confirmation
- Wrapped in a ARQ task with retry policy

```mermaid
sequenceDiagram
participant Scanner as "ScannerService"
participant Settings as "Settings"
participant Actor as "send_webhook_task"
participant Service as "WebhookService"
participant Remote as "External System"
Scanner->>Settings : "Read webhook_url and webhook_secret"
Scanner->>Actor : "send_webhook_task(url, payload, secret)"
Actor->>Service : "send_webhook(url, payload, secret)"
Service->>Service : "Sign payload with HMAC-SHA256"
Service->>Remote : "POST JSON with X-Webhook-Signature"
Remote-->>Service : "HTTP 2xx/4xx/5xx"
Service-->>Actor : "Success/Failure"
Actor-->>Scanner : "Retry on failure"
```

**Diagram sources**
- [app/services/blockchain/scanner.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/scanner.py#L117-L131)
- [app/workers/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/webhook.py#L13-L37)
- [app/services/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/webhook.py#L12-L45)
- [app/core/config.py](https://github.com/rakibhossain72/ctrip/blob/main/app/core/config.py#L63-L71)

**Section sources**
- [app/services/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/webhook.py#L10-L45)
- [app/workers/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/webhook.py#L13-L37)
- [app/core/config.py](https://github.com/rakibhossain72/ctrip/blob/main/app/core/config.py#L63-L71)

### HDWalletManager
Responsibilities:
- Generate mnemonic phrases or accept provided mnemonics
- Derive checksummed addresses using BIP-44 path m/44'/60'/0'/0/{index}
- Provide multiple addresses for batch generation

Usage:
- Used by API to generate payment addresses during creation
- Injected via dependency to ensure consistent state across requests

```mermaid
classDiagram
class HDWalletManager {
+mnemonic : str
+seed : bytes
+get_address(index) dict
+get_multiple_addresses(count, start_index) list
+get_mnemonic() str
}
```

**Diagram sources**
- [app/utils/crypto.py](https://github.com/rakibhossain72/ctrip/blob/main/app/utils/crypto.py#L5-L67)

**Section sources**
- [app/utils/crypto.py](https://github.com/rakibhossain72/ctrip/blob/main/app/utils/crypto.py#L5-L67)
- [app/api/v1/payments.py](https://github.com/rakibhossain72/ctrip/blob/main/app/api/v1/payments.py#L36-L37)
- [app/api/dependencies.py](https://github.com/rakibhossain72/ctrip/blob/main/app/api/dependencies.py#L11-L14)

### BlockchainBase and RPC Integration
Responsibilities:
- Manage AsyncWeb3 connections and middleware
- Provide balance queries, gas estimation, and transaction building
- Support EIP-1559 fee calculation with fallback to legacy gas pricing
- Send raw transactions and wait for receipts

Integration:
- get_w3(chain_name) resolves AsyncWeb3 instances via BlockchainBase
- get_blockchains() constructs chain-specific providers from configuration

```mermaid
classDiagram
class BlockchainBase {
+provider_url : str
+chain_id : int
+use_poa : bool
+is_connected() bool
+get_balance(address) int
+get_token_balance(token_address, user_address) int
+get_gas_price(use_cache) int
+get_fee_history(block_count, newest_block) dict
+estimate_gas(tx) int
+build_transaction(from_address, to_address, value_wei, data, gas_limit, nonce) dict
+send_transaction(tx, private_key) str
+get_receipt(tx_hash, timeout) Any
+get_latest_block() int
}
```

**Diagram sources**
- [app/blockchain/base.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/base.py#L22-L146)
- [app/blockchain/w3.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/w3.py#L6-L9)
- [app/blockchain/manager.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/manager.py#L8-L33)

**Section sources**
- [app/blockchain/base.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/base.py#L22-L146)
- [app/blockchain/w3.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/w3.py#L6-L9)
- [app/blockchain/manager.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/manager.py#L8-L33)

### Worker Orchestration
- `listen_for_payments`: ARQ cron task (every second) that confirms detected payments and expires stale ones. Block detection is handled by always-on ChainSniper WebSocket listeners started at worker startup.
- `sweep_funds`: ARQ cron task (every 30 seconds, currently commented out in `WorkerSettings`) that sweeps confirmed payments to admin wallet.
- `send_webhook_notification`: ARQ task that sends signed webhooks for payment events.
- `process_single_payment`: ARQ task for manually triggering confirmation check for a specific payment.

Initialization and scheduling:
- Worker started via `python run_worker.py` which calls `arq.run_worker(WorkerSettings)`
- `WorkerSettings.on_startup` calls `ScannerService.start_listeners()` to launch ChainSniper WebSocket listeners
- Cron jobs defined in `WorkerSettings.cron_jobs` using ARQ's `cron()` helper
- Sessions are scoped per task run

**Section sources**
- [app/workers/listener.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/listener.py#L21-L46)
- [app/workers/sweeper.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/sweeper.py#L19-L40)
- [app/workers/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/webhook.py#L13-L37)
- [app/core/config.py](https://github.com/rakibhossain72/ctrip/blob/main/app/core/config.py#L44-L56)

## Dependency Analysis
- ScannerService depends on AsyncWeb3 via get_w3, ChainState, Payment, and Token models, and the webhook actor
- SweeperService depends on AsyncWeb3 via get_w3, Payment model, and HDWalletManager
- WebhookService depends on httpx and HMAC signing
- API depends on HDWalletManager and Payment model
- BlockchainBase encapsulates AsyncWeb3 and gas/fee logic
- Workers depend on services and settings

```mermaid
graph LR
Scanner["ScannerService"] --> W3["get_w3"]
Sweeper["SweeperService"] --> W3
WebhookActor["send_webhook_task"] --> WebhookSvc["WebhookService"]
API["create_payment"] --> HD["HDWalletManager"]
API --> PaymentModel["Payment Model"]
Scanner --> PaymentModel
Scanner --> ChainStateModel["ChainState Model"]
Sweeper --> PaymentModel
W3 --> Manager["get_blockchains"]
Manager --> Base["BlockchainBase"]
```

**Diagram sources**
- [app/services/blockchain/scanner.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/scanner.py#L1-L10)
- [app/services/blockchain/sweeper.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/sweeper.py#L1-L7)
- [app/services/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/webhook.py#L1-L8)
- [app/api/v1/payments.py](https://github.com/rakibhossain72/ctrip/blob/main/app/api/v1/payments.py#L18-L54)
- [app/blockchain/w3.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/w3.py#L1-L9)
- [app/blockchain/manager.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/manager.py#L1-L33)
- [app/blockchain/base.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/base.py#L1-L146)
- [app/db/models/payment.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/payment.py#L41-L57)
- [app/db/models/chain.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/chain.py#L9-L17)

**Section sources**
- [app/services/blockchain/scanner.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/scanner.py#L1-L10)
- [app/services/blockchain/sweeper.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/sweeper.py#L1-L7)
- [app/services/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/webhook.py#L1-L8)
- [app/api/v1/payments.py](https://github.com/rakibhossain72/ctrip/blob/main/app/api/v1/payments.py#L18-L54)
- [app/blockchain/w3.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/w3.py#L1-L9)
- [app/blockchain/manager.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/manager.py#L1-L33)
- [app/blockchain/base.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/base.py#L1-L146)
- [app/db/models/payment.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/payment.py#L41-L57)
- [app/db/models/chain.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/chain.py#L9-L17)

## Performance Considerations
- Block scanning batching: Limit block ranges per cycle to avoid long-running operations and excessive RPC calls
- Gas caching: BlockchainBase caches gas price for a short duration to reduce repeated network calls
- Confirmation thresholds: Tune confirmations_required to balance speed vs. reorg safety
- Retry and backoff: Webhook actor has retries; consider exponential backoff for resilience
- Concurrency: Use separate actors per chain to parallelize work; ensure database sessions are not shared across chains
- Monitoring: Log scan windows, detected counts, confirmation rates, and webhook delivery outcomes

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
Common issues and resolutions:
- Chain not configured: get_w3 raises an error if chain_name is not present in registered blockchains
- Invalid private key: Settings validates private key format; ensure a proper Ethereum private key is set
- Webhook failures: WebhookService logs HTTP errors; verify URL, secret, and network connectivity
- No payments detected: Verify pending payments exist for the target chain and addresses match
- Sweeping placeholder: Actual transaction sending is not implemented; implement transaction building and submission before enabling production sweeping

Operational checks:
- Verify settings.chains and chains.yaml are correctly loaded
- Confirm AsyncWeb3 connectivity via BlockchainBase.is_connected
- Inspect ChainState.last_scanned_block progression
- Review webhook actor logs for retryable failures

**Section sources**
- [app/blockchain/w3.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/w3.py#L7-L9)
- [app/core/config.py](https://github.com/rakibhossain72/ctrip/blob/main/app/core/config.py#L94-L102)
- [app/services/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/webhook.py#L39-L44)
- [app/blockchain/base.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/base.py#L45-L50)
- [app/db/models/chain.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/chain.py#L9-L17)

## Conclusion
The cTrip Payment Gateway implements a modular, asynchronous architecture for payment lifecycle management. ScannerService provides robust detection and confirmation logic, SweeperService prepares the foundation for automated fund settlement, and WebhookService enables reliable external notifications. HDWalletManager ensures deterministic address generation, while BlockchainBase abstracts RPC complexity. Workers orchestrate periodic tasks with clear retry semantics. Together, these components form a scalable and maintainable payment infrastructure.

[No sources needed since this section summarizes without analyzing specific files]

## Appendices

### Service Interfaces and Method Signatures
- ScannerService
  - start_listeners() -> list[asyncio.Task]
  - confirm_payments(chain_name: str) -> None
  - check_expired_payments() -> None
  - _on_block(block: dict, chain_name: str) -> None (WebSocket callback)
  - _on_log(log: dict, chain_name: str) -> None (WebSocket callback)
- SweeperService
  - sweep_confirmed_payments(chain_name: str) -> None
- WebhookService
  - send_webhook(url: str, payload: Dict[str, Any], secret: Optional[str] = None) -> bool
- HDWalletManager
  - get_address(index: int) -> Dict[str, Any]
  - get_multiple_addresses(count: int, start_index: int = 0) -> List[Dict[str, Any]]
  - get_mnemonic() -> str

**Section sources**
- [app/services/blockchain/scanner.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/scanner.py#L20-L134)
- [app/services/blockchain/sweeper.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/blockchain/sweeper.py#L16-L54)
- [app/services/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/webhook.py#L12-L16)
- [app/utils/crypto.py](https://github.com/rakibhossain72/ctrip/blob/main/app/utils/crypto.py#L27-L66)

### Initialization Patterns and Dependency Injection
- Settings: Centralized configuration with environment-aware defaults and validators
- API dependencies: get_hdwallet and get_blockchains inject runtime-managed instances
- Worker initialization: Actors construct services with database sessions and settings
- Blockchain registry: get_blockchains builds provider instances from chains configuration

**Section sources**
- [app/core/config.py](https://github.com/rakibhossain72/ctrip/blob/main/app/core/config.py#L10-L126)
- [app/api/dependencies.py](https://github.com/rakibhossain72/ctrip/blob/main/app/api/dependencies.py#L5-L14)
- [app/workers/listener.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/listener.py#L30-L35)
- [app/workers/sweeper.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/sweeper.py#L28-L30)
- [app/blockchain/manager.py](https://github.com/rakibhossain72/ctrip/blob/main/app/blockchain/manager.py#L8-L33)

### Integration Patterns
- Payment creation: API creates Payment records and derives addresses via HDWalletManager
- Background processing: Workers schedule scanning, sweeping, and webhook delivery
- External notifications: WebhookService signs and posts payloads to remote systems
- Persistence: SQLAlchemy models define payment lifecycle and chain state

**Section sources**
- [app/api/v1/payments.py](https://github.com/rakibhossain72/ctrip/blob/main/app/api/v1/payments.py#L18-L54)
- [app/workers/listener.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/listener.py#L21-L46)
- [app/workers/sweeper.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/sweeper.py#L19-L40)
- [app/workers/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/workers/webhook.py#L13-L37)
- [app/services/webhook.py](https://github.com/rakibhossain72/ctrip/blob/main/app/services/webhook.py#L12-L45)
- [app/db/models/payment.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/payment.py#L41-L57)
- [app/db/models/chain.py](https://github.com/rakibhossain72/ctrip/blob/main/app/db/models/chain.py#L9-L17)