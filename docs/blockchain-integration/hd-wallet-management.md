# HD Wallet Management

<cite>
**Referenced Files in This Document**
- [crypto.py](file://app/utils/crypto.py)
- [dependencies.py](file://app/api/dependencies.py)
- [payments.py](file://app/api/v1/payments.py)
- [server.py](file://server.py)
- [config.py](file://app/core/config.py)
- [manager.py](file://app/blockchain/manager.py)
- [base.py](file://app/blockchain/base.py)
- [ethereum.py](file://app/blockchain/ethereum.py)
- [bsc.py](file://app/blockchain/bsc.py)
- [anvil.py](file://app/blockchain/anvil.py)
- [payment.py](file://app/db/models/payment.py)
- [transaction.py](file://app/db/models/transaction.py)
- [chains.yaml](file://chains.yaml)
</cite>

## Update Summary
**Changes Made**
- Enhanced HD wallet functionality documentation with automatic index calculation capabilities
- Added documentation for sequential address generation with persistent tracking
- Updated payment creation process to reflect automatic index assignment
- Added comprehensive coverage of HDWalletAddress model and its role in sequential derivation
- Enhanced BIP-44 derivation documentation with automatic index management

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

## Introduction
This document explains the hierarchical deterministic (HD) wallet system and BIP-44 derivation used to generate unique payment addresses per transaction. The system has been enhanced with sequential address generation capabilities, automatic index calculation, and persistent address tracking. It covers how the HD wallet integrates with blockchain interactions, how addresses are derived, and how payment records are persisted. It also documents security considerations, private key management, and seed phrase handling, along with practical examples and troubleshooting guidance.

## Project Structure
The HD wallet and BIP-44 implementation is centered around a dedicated utility module that derives Ethereum-compatible addresses from a mnemonic seed. These addresses are used by the payment API to create unique payment addresses for incoming transactions with automatic sequential indexing. Blockchain connectivity is abstracted behind a shared base class and chain-specific implementations, while persistence is handled by SQLAlchemy models including dedicated tracking for generated addresses.

```mermaid
graph TB
subgraph "API Layer"
PaymentsAPI["Payments API<br/>create_payment()"]
Deps["Dependencies<br/>get_hdwallet(), get_blockchains()"]
end
subgraph "Wallet Layer"
HDWM["HDWalletManager<br/>BIP-44 derivation<br/>Sequential Indexing"]
HDAddr["HDWalletAddress<br/>Address Tracking"]
end
subgraph "Blockchain Layer"
BCBase["BlockchainBase<br/>common ops"]
Eth["EthereumBlockchain"]
Bsc["BSCBlockchain"]
Anvil["AnvilBlockchain"]
Manager["Blockchain Manager<br/>get_blockchains()"]
end
subgraph "Persistence"
PaymentModel["Payment Model"]
TxModel["Transaction Model"]
end
PaymentsAPI --> Deps
Deps --> HDWM
PaymentsAPI --> Manager
Manager --> Eth
Manager --> Bsc
Manager --> Anvil
Eth --> BCBase
Bsc --> BCBase
Anvil --> BCBase
PaymentsAPI --> PaymentModel
PaymentsAPI --> TxModel
HDWM --> HDAddr
```

**Diagram sources**
- [payments.py](file://app/api/v1/payments.py#L18-L54)
- [dependencies.py](file://app/api/dependencies.py#L5-L15)
- [crypto.py](file://app/utils/crypto.py#L5-L66)
- [manager.py](file://app/blockchain/manager.py#L8-L32)
- [base.py](file://app/blockchain/base.py#L22-L146)
- [ethereum.py](file://app/blockchain/ethereum.py#L3-L6)
- [bsc.py](file://app/blockchain/bsc.py#L3-L6)
- [anvil.py](file://app/blockchain/anvil.py#L8-L56)
- [payment.py](file://app/db/models/payment.py#L41-L74)
- [transaction.py](file://app/db/models/transaction.py#L29-L39)

**Section sources**
- [payments.py](file://app/api/v1/payments.py#L1-L62)
- [dependencies.py](file://app/api/dependencies.py#L1-L15)
- [crypto.py](file://app/utils/crypto.py#L1-L90)
- [manager.py](file://app/blockchain/manager.py#L1-L33)
- [base.py](file://app/blockchain/base.py#L1-L146)
- [ethereum.py](file://app/blockchain/ethereum.py#L1-L7)
- [bsc.py](file://app/blockchain/bsc.py#L1-L7)
- [anvil.py](file://app/blockchain/anvil.py#L1-L57)
- [payment.py](file://app/db/models/payment.py#L1-L74)
- [transaction.py](file://app/db/models/transaction.py#L1-L40)

## Core Components
- **HDWalletManager**: Generates a mnemonic (if not provided), derives a seed, and produces Ethereum addresses using BIP-44 path m/44'/60'/0'/0/{index}. It supports deriving a single address or multiple sequential addresses and maintains automatic index calculation for sequential payment generation.
- **HDWalletAddress Model**: Tracks generated addresses and their sequential indices to ensure proper BIP-44 derivation order and prevent index reuse.
- **Payments API**: Validates chain support, optionally validates token, automatically calculates the next sequential index, derives a new address from the HD wallet, persists both payment and address tracking records, and returns the payment.
- **BlockchainBase and chain implementations**: Provide asynchronous connectivity to various chains, gas estimation, transaction building, signing, and sending.
- **Persistence models**: Payment and Transaction models persist payment requests and transaction outcomes, while HDWalletAddress tracks generated addresses.

Key implementation references:
- HD derivation and address generation: [crypto.py](file://app/utils/crypto.py#L27-L46)
- Multiple address generation: [crypto.py](file://app/utils/crypto.py#L48-L62)
- Automatic index calculation: [payments.py](file://app/api/v1/payments.py#L36-L38)
- Address tracking persistence: [payments.py](file://app/api/v1/payments.py#L44-L49)
- HDWalletAddress model definition: [payment.py](file://app/db/models/payment.py#L65-L74)
- Payment creation and address derivation: [payments.py](file://app/api/v1/payments.py#L36-L52)
- Blockchain abstraction and transaction building: [base.py](file://app/blockchain/base.py#L93-L133)
- Chain selection and instantiation: [manager.py](file://app/blockchain/manager.py#L8-L32)

**Section sources**
- [crypto.py](file://app/utils/crypto.py#L5-L66)
- [payments.py](file://app/api/v1/payments.py#L18-L54)
- [payment.py](file://app/db/models/payment.py#L65-L74)
- [base.py](file://app/blockchain/base.py#L22-L146)
- [manager.py](file://app/blockchain/manager.py#L8-L32)

## Architecture Overview
The system initializes an HD wallet during application startup and exposes it via dependency injection. When a payment is requested, the API automatically calculates the next sequential index, derives a unique address from the HD wallet, persists both payment and address tracking records, and relies on blockchain scanning and webhook services to detect and confirm payments.

```mermaid
sequenceDiagram
participant Client as "Client"
participant API as "Payments API"
participant Deps as "Dependencies"
participant HD as "HDWalletManager"
participant AddrTracker as "HDWalletAddress"
participant DB as "Payment Model"
participant BC as "Blockchain Manager"
Client->>API : "POST /api/v1/payments/"
API->>Deps : "get_hdwallet()"
Deps-->>API : "HDWalletManager instance"
API->>AddrTracker : "Query last address index"
AddrTracker-->>API : "Next index value"
API->>HD : "get_address(index=next_index)"
HD-->>API : "{address, path, index}"
API->>AddrTracker : "Persist HDWalletAddress"
API->>DB : "Persist Payment"
DB-->>API : "Saved Payment"
API-->>Client : "201 Created Payment"
API->>BC : "get_blockchains()"
BC-->>API : "Chain clients"
```

**Diagram sources**
- [server.py](file://server.py#L21-L28)
- [dependencies.py](file://app/api/dependencies.py#L11-L15)
- [crypto.py](file://app/utils/crypto.py#L27-L46)
- [payments.py](file://app/api/v1/payments.py#L18-L54)
- [manager.py](file://app/blockchain/manager.py#L8-L32)

## Detailed Component Analysis

### HDWalletManager: BIP-44 Derivation and Sequential Address Generation
The HD wallet manager encapsulates:
- Mnemonic generation (when none is provided) and seed derivation.
- BIP-44 path construction for Ethereum: m/44'/60'/0'/0/{index}.
- Private key derivation from seed and path, followed by account creation to produce an address.
- Sequential address generation for multiple indices with configurable start positions.
- Automatic index calculation support for payment systems requiring sequential derivation.

```mermaid
classDiagram
class HDWalletManager {
+mnemonic
+seed
+__init__(mnemonic_phrase)
+get_address(index) dict
+get_multiple_addresses(count, start_index) list
+get_mnemonic() str
}
```

**Diagram sources**
- [crypto.py](file://app/utils/crypto.py#L5-L66)

Implementation highlights:
- Initialization and seed derivation: [crypto.py](file://app/utils/crypto.py#L11-L25)
- Single address derivation: [crypto.py](file://app/utils/crypto.py#L27-L46)
- Multiple sequential addresses: [crypto.py](file://app/utils/crypto.py#L48-L62)
- Mnemonic retrieval: [crypto.py](file://app/utils/crypto.py#L64-L66)

Security note:
- The mnemonic must be kept secret and protected at rest and in transit.
- Private keys are derived on-demand for signing; avoid storing raw private keys.

**Section sources**
- [crypto.py](file://app/utils/crypto.py#L5-L66)

### HDWalletAddress Model: Sequential Address Tracking
The HDWalletAddress model provides persistent tracking of generated addresses and their sequential indices:
- Stores generated addresses with their corresponding BIP-44 indices.
- Prevents index reuse and ensures proper sequential derivation order.
- Supports address verification and audit trails for payment processing.
- Includes timestamp tracking for address generation history.

```mermaid
classDiagram
class HDWalletAddress {
+id UUID
+address String
+index BigInteger
+is_swapped String
+created_at DateTime
}
```

**Diagram sources**
- [payment.py](file://app/db/models/payment.py#L65-L74)

Implementation highlights:
- Address storage with unique constraints: [payment.py](file://app/db/models/payment.py#L68-L70)
- Sequential index tracking: [payment.py](file://app/db/models/payment.py#L70)
- Timestamp management: [payment.py](file://app/db/models/payment.py#L72-L74)

**Section sources**
- [payment.py](file://app/db/models/payment.py#L65-L74)

### Payments API: Enhanced Wallet Integration and Automatic Index Management
The payment creation endpoint has been enhanced with automatic sequential index management:
- Validates the requested chain against configured chains.
- Optionally validates token availability for the chain.
- Automatically calculates the next sequential index by querying the HDWalletAddress table.
- Derives a new address from the HD wallet using the calculated sequential index.
- Persists both the payment record and address tracking record.
- Returns the created payment with all relevant metadata.

```mermaid
sequenceDiagram
participant Client as "Client"
participant API as "create_payment()"
participant AddrTracker as "HDWalletAddress"
participant HD as "HDWalletManager"
participant DB as "Payment Model"
Client->>API : "PaymentCreate"
API->>API : "Validate chain and token"
API->>AddrTracker : "Query last address index"
AddrTracker-->>API : "Last index + 1"
API->>HD : "get_address(index=next_index)"
HD-->>API : "address"
API->>AddrTracker : "Insert HDWalletAddress"
API->>DB : "Insert Payment"
DB-->>API : "Saved Payment"
API-->>Client : "201 PaymentRead"
```

**Diagram sources**
- [payments.py](file://app/api/v1/payments.py#L18-L54)
- [crypto.py](file://app/utils/crypto.py#L27-L46)
- [payment.py](file://app/db/models/payment.py#L41-L58)

Operational details:
- Chain validation and token lookup: [payments.py](file://app/api/v1/payments.py#L26-L34)
- Automatic index calculation: [payments.py](file://app/api/v1/payments.py#L36-L38)
- Address derivation and dual persistence: [payments.py](file://app/api/v1/payments.py#L40-L50)

**Section sources**
- [payments.py](file://app/api/v1/payments.py#L18-L54)
- [payment.py](file://app/db/models/payment.py#L41-L58)

### Blockchain Layer: Transaction Building, Signing, and Sending
The base blockchain class provides:
- Asynchronous connectivity and POA middleware support.
- Gas price and fee history retrieval, with fallbacks.
- Transaction building with EIP-1559 or legacy pricing.
- Transaction signing and submission using a private key.
- Balance and token balance queries.

```mermaid
classDiagram
class BlockchainBase {
+provider_url
+chain_id
+w3
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
class EthereumBlockchain
class BSCBlockchain
class AnvilBlockchain
EthereumBlockchain --|> BlockchainBase
BSCBlockchain --|> BlockchainBase
AnvilBlockchain --|> BlockchainBase
```

**Diagram sources**
- [base.py](file://app/blockchain/base.py#L22-L146)
- [ethereum.py](file://app/blockchain/ethereum.py#L3-L6)
- [bsc.py](file://app/blockchain/bsc.py#L3-L6)
- [anvil.py](file://app/blockchain/anvil.py#L8-L56)

Usage references:
- Transaction building and signing: [base.py](file://app/blockchain/base.py#L93-L139)
- Gas estimation and caching: [base.py](file://app/blockchain/base.py#L65-L91)

**Section sources**
- [base.py](file://app/blockchain/base.py#L22-L146)
- [ethereum.py](file://app/blockchain/ethereum.py#L1-L7)
- [bsc.py](file://app/blockchain/bsc.py#L1-L7)
- [anvil.py](file://app/blockchain/anvil.py#L1-L57)

### Chain Configuration and Selection
Chains are loaded from a YAML configuration file and instantiated at startup. The manager maps chain names to concrete blockchain implementations.

```mermaid
flowchart TD
LoadYAML["Load chains.yaml"] --> Iterate["Iterate entries"]
Iterate --> Name{"Name present?"}
Name --> |ethereum| Eth["EthereumBlockchain"]
Name --> |bsc| Bsc["BSCBlockchain"]
Name --> |anvil| Anvil["AnvilBlockchain"]
Name --> |other| Base["BlockchainBase"]
Eth --> Map["Add to blockchains map"]
Bsc --> Map
Anvil --> Map
Base --> Map
```

**Diagram sources**
- [chains.yaml](file://chains.yaml#L1-L24)
- [manager.py](file://app/blockchain/manager.py#L8-L32)

**Section sources**
- [chains.yaml](file://chains.yaml#L1-L24)
- [manager.py](file://app/blockchain/manager.py#L8-L32)

### Persistence Models: Payments, Transactions, and Address Tracking
Payment records capture chain, address, amount, status, confirmations, and expiration. Transaction records track hash, block number, confirmations, and status. HDWalletAddress records track generated addresses and their sequential indices for proper BIP-44 derivation management.

```mermaid
erDiagram
PAYMENTS {
uuid id PK
uuid token_id FK
string chain
string address
numeric amount
enum status
integer confirmations
integer detected_in_block
timestamp expires_at
timestamp created_at
}
TOKENS {
uuid id PK
string symbol
string address
integer decimals
string chain
}
TRANSACTIONS {
uuid id PK
uuid payment_id FK
string tx_hash UK
integer block_number
integer confirmations
enum status
}
HDWALLET_ADDRESSES {
uuid id PK
string address
bigint index
string is_swapped
timestamp created_at
}
TOKENS ||--o{ PAYMENTS : "token_id"
PAYMENTS ||--o{ TRANSACTIONS : "id"
PAYMENTS ||--o{ HDWALLET_ADDRESSES : "address"
```

**Diagram sources**
- [payment.py](file://app/db/models/payment.py#L41-L74)
- [transaction.py](file://app/db/models/transaction.py#L29-L39)

**Section sources**
- [payment.py](file://app/db/models/payment.py#L1-L74)
- [transaction.py](file://app/db/models/transaction.py#L1-L40)

## Dependency Analysis
The system exhibits clean separation of concerns with enhanced sequential address management:
- API depends on HD wallet and blockchain manager for payment creation.
- HD wallet is injected via dependency injection and initialized at startup.
- Automatic index calculation ensures sequential BIP-44 derivation order.
- HDWalletAddress model provides persistent tracking of generated addresses.
- Blockchain manager selects chain implementations based on configuration.
- Persistence models decouple payment lifecycle from blockchain operations.

```mermaid
graph LR
API["payments.py"] --> HD["crypto.py: HDWalletManager"]
API --> DB["payment.py, transaction.py"]
API --> BCMgr["manager.py:get_blockchains()"]
HD --> AddrTrack["payment.py: HDWalletAddress"]
Server["server.py"] --> HD
Server --> BCMgr
```

**Diagram sources**
- [payments.py](file://app/api/v1/payments.py#L1-L62)
- [crypto.py](file://app/utils/crypto.py#L1-L90)
- [manager.py](file://app/blockchain/manager.py#L1-L33)
- [base.py](file://app/blockchain/base.py#L1-L146)
- [ethereum.py](file://app/blockchain/ethereum.py#L1-L7)
- [bsc.py](file://app/blockchain/bsc.py#L1-L7)
- [anvil.py](file://app/blockchain/anvil.py#L1-L57)
- [payment.py](file://app/db/models/payment.py#L1-L74)
- [transaction.py](file://app/db/models/transaction.py#L1-L40)
- [server.py](file://server.py#L21-L28)

**Section sources**
- [payments.py](file://app/api/v1/payments.py#L1-L62)
- [crypto.py](file://app/utils/crypto.py#L1-L90)
- [manager.py](file://app/blockchain/manager.py#L1-L33)
- [base.py](file://app/blockchain/base.py#L1-L146)
- [ethereum.py](file://app/blockchain/ethereum.py#L1-L7)
- [bsc.py](file://app/blockchain/bsc.py#L1-L7)
- [anvil.py](file://app/blockchain/anvil.py#L1-L57)
- [payment.py](file://app/db/models/payment.py#L1-L74)
- [transaction.py](file://app/db/models/transaction.py#L1-L40)
- [server.py](file://server.py#L21-L28)

## Performance Considerations
- Gas estimation and caching: The base blockchain class caches gas price for a short duration to reduce RPC calls.
- Fee calculation: EIP-1559 fee history is preferred with a legacy fallback to ensure transaction propagation.
- Transaction building buffer: A small gas buffer is applied to estimated gas limits to improve success rates.
- Sequential index optimization: HDWalletAddress table uses efficient indexing on the index column for fast sequential lookup.
- Batch address generation: The get_multiple_addresses method enables batch generation for high-volume scenarios.

Recommendations:
- Monitor fee spikes and adjust buffers accordingly.
- Prefer batched operations for high-volume scenarios using get_multiple_addresses.
- Use chain-specific optimizations (e.g., POA middleware for BSC).
- Implement proper database indexing on HDWalletAddress.index for optimal performance.

**Section sources**
- [base.py](file://app/blockchain/base.py#L65-L91)
- [base.py](file://app/blockchain/base.py#L116-L131)
- [crypto.py](file://app/utils/crypto.py#L48-L62)

## Troubleshooting Guide
Common issues and resolutions:
- Unsupported chain in payment creation:
  - Cause: Chain not present in configuration or not recognized.
  - Resolution: Verify chains.yaml and ensure the chain name matches supported names.
  - Reference: [payments.py](file://app/api/v1/payments.py#L27-L28), [manager.py](file://app/blockchain/manager.py#L13-L26)
- Token not found for chain:
  - Cause: Token does not exist or mismatched chain.
  - Resolution: Confirm token exists and belongs to the requested chain.
  - Reference: [payments.py](file://app/api/v1/payments.py#L32-L34)
- HD wallet not initialized:
  - Cause: Dependency injection expects app state to contain the HD wallet.
  - Resolution: Ensure server lifespan initializes the HD wallet.
  - Reference: [server.py](file://server.py#L21-L28), [dependencies.py](file://app/api/dependencies.py#L11-L15)
- Sequential index calculation failures:
  - Cause: HDWalletAddress table empty or corrupted index values.
  - Resolution: Verify HDWalletAddress table integrity and ensure proper sequential indexing.
  - Reference: [payments.py](file://app/api/v1/payments.py#L36-L38)
- Address tracking inconsistencies:
  - Cause: Missing or duplicate address entries in HDWalletAddress table.
  - Resolution: Implement proper transaction handling and address uniqueness constraints.
  - Reference: [payments.py](file://app/api/v1/payments.py#L44-L49)
- Blockchain connectivity failures:
  - Cause: RPC endpoint unreachable or invalid.
  - Resolution: Validate provider URL and network accessibility.
  - Reference: [base.py](file://app/blockchain/base.py#L45-L50)
- Transaction signing errors:
  - Cause: Invalid private key or nonce issues.
  - Resolution: Validate private key and check pending nonce.
  - Reference: [base.py](file://app/blockchain/base.py#L135-L139)

**Section sources**
- [payments.py](file://app/api/v1/payments.py#L26-L34)
- [server.py](file://server.py#L21-L28)
- [dependencies.py](file://app/api/dependencies.py#L11-L15)
- [base.py](file://app/blockchain/base.py#L45-L50)
- [base.py](file://app/blockchain/base.py#L135-L139)

## Conclusion
The system implements a robust HD wallet and BIP-44 derivation pipeline integrated with blockchain connectivity and persistent payment records. The enhanced functionality now includes automatic sequential address generation with proper index management, ensuring unique addresses per payment while maintaining BIP-44 compliance. By deriving unique addresses per payment with automatic index calculation and leveraging chain-specific implementations, it supports scalable and secure payment processing. The addition of HDWalletAddress tracking provides reliable sequential derivation management and audit capabilities. Adhering to the security and operational recommendations herein ensures reliable and maintainable deployments with proper sequential address management.