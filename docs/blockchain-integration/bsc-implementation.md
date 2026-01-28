# BSC Implementation

<cite>
**Referenced Files in This Document**
- [app/blockchain/bsc.py](file://app/blockchain/bsc.py)
- [app/blockchain/base.py](file://app/blockchain/base.py)
- [app/blockchain/ethereum.py](file://app/blockchain/ethereum.py)
- [app/blockchain/anvil.py](file://app/blockchain/anvil.py)
- [app/blockchain/manager.py](file://app/blockchain/manager.py)
- [app/blockchain/w3.py](file://app/blockchain/w3.py)
- [app/blockchain/ABI/ERC20.json](file://app/blockchain/ABI/ERC20.json)
- [chains.yaml](file://chains.yaml)
- [app/core/config.py](file://app/core/config.py)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py)
- [app/services/blockchain/sweeper.py](file://app/services/blockchain/sweeper.py)
- [app/db/models/chain.py](file://app/db/models/chain.py)
- [app/db/models/token.py](file://app/db/models/token.py)
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
This document explains the Binance Smart Chain (BSC) implementation in the codebase. It focuses on the BSCBlockchain class, network-specific optimizations, RPC configuration, and transaction monitoring via scanning and sweeping services. It also covers BSC mainnet configuration, token standard support (ERC-20), integration patterns, performance considerations, and troubleshooting steps.

## Project Structure
The BSC implementation is organized around a shared base class and a specialized BSC subclass, with configuration loaded from a YAML file and accessed through a central manager and Web3 accessor.

```mermaid
graph TB
subgraph "Blockchain Layer"
Base["BlockchainBase<br/>base.py"]
BSC["BSCBlockchain<br/>bsc.py"]
ETH["EthereumBlockchain<br/>ethereum.py"]
ANVIL["AnvilBlockchain<br/>anvil.py"]
Manager["get_blockchains()<br/>manager.py"]
W3["get_w3()<br/>w3.py"]
end
subgraph "Config"
CFG["Settings<br/>config.py"]
CHAINS["chains.yaml"]
end
subgraph "Services"
Scanner["ScannerService<br/>scanner.py"]
Sweeper["SweeperService<br/>sweeper.py"]
end
subgraph "Models"
ChainState["ChainState<br/>chain.py"]
Token["Token<br/>token.py"]
end
CFG --> CHAINS
Manager --> BSC
Manager --> ETH
Manager --> ANVIL
W3 --> Manager
Scanner --> W3
Sweeper --> W3
Scanner --> ChainState
Scanner --> Token
Sweeper --> Token
```

**Diagram sources**
- [app/blockchain/base.py](file://app/blockchain/base.py#L22-L146)
- [app/blockchain/bsc.py](file://app/blockchain/bsc.py#L1-L7)
- [app/blockchain/ethereum.py](file://app/blockchain/ethereum.py#L1-L7)
- [app/blockchain/anvil.py](file://app/blockchain/anvil.py#L1-L57)
- [app/blockchain/manager.py](file://app/blockchain/manager.py#L8-L33)
- [app/blockchain/w3.py](file://app/blockchain/w3.py#L1-L9)
- [app/core/config.py](file://app/core/config.py#L44-L57)
- [chains.yaml](file://chains.yaml#L12-L18)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L14-L134)
- [app/services/blockchain/sweeper.py](file://app/services/blockchain/sweeper.py#L11-L54)
- [app/db/models/chain.py](file://app/db/models/chain.py#L9-L17)
- [app/db/models/token.py](file://app/db/models/token.py#L6-L15)

**Section sources**
- [app/blockchain/bsc.py](file://app/blockchain/bsc.py#L1-L7)
- [app/blockchain/base.py](file://app/blockchain/base.py#L22-L146)
- [app/blockchain/manager.py](file://app/blockchain/manager.py#L8-L33)
- [app/blockchain/w3.py](file://app/blockchain/w3.py#L1-L9)
- [app/core/config.py](file://app/core/config.py#L44-L57)
- [chains.yaml](file://chains.yaml#L12-L18)

## Core Components
- BSCBlockchain: Specializes the base blockchain client for BSC mainnet with chain ID 56 and Proof-of-Authority middleware enabled.
- BlockchainBase: Provides shared RPC connectivity, gas estimation, transaction building, signing, sending, and receipt polling. Includes EIP-1559 fee handling and a gas cache.
- Manager and W3 Accessor: Centralized factory that loads chain configs and exposes AsyncWeb3 instances per chain.
- ScannerService: Scans blocks for native and ERC-20 payments, updates detection state, and triggers confirmations.
- SweeperService: Collects confirmed payments and prepares settlement logic (placeholder in this codebase).
- Models: ChainState tracks scanned progress; Token defines supported tokens per chain.

Key BSC specifics:
- Chain ID 56 and POA middleware enable compatibility with BSC’s consensus.
- Gas optimization via EIP-1559 fee history fallback to legacy gas price.
- ERC-20 support via shared ABI and log-based detection.

**Section sources**
- [app/blockchain/bsc.py](file://app/blockchain/bsc.py#L3-L6)
- [app/blockchain/base.py](file://app/blockchain/base.py#L22-L146)
- [app/blockchain/manager.py](file://app/blockchain/manager.py#L8-L33)
- [app/blockchain/w3.py](file://app/blockchain/w3.py#L6-L9)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L14-L134)
- [app/services/blockchain/sweeper.py](file://app/services/blockchain/sweeper.py#L11-L54)
- [app/db/models/chain.py](file://app/db/models/chain.py#L9-L17)
- [app/db/models/token.py](file://app/db/models/token.py#L6-L15)

## Architecture Overview
The BSC implementation follows a layered design:
- Configuration layer: chains.yaml and Settings load chain definitions.
- Factory layer: get_blockchains builds chain-specific clients.
- Accessor layer: get_w3 retrieves AsyncWeb3 for a named chain.
- Services layer: ScannerService monitors chain activity; SweeperService handles settlement.

```mermaid
sequenceDiagram
participant Cfg as "Settings<br/>config.py"
participant Yaml as "chains.yaml"
participant Manager as "get_blockchains()<br/>manager.py"
participant BSC as "BSCBlockchain<br/>bsc.py"
participant W3 as "get_w3()<br/>w3.py"
participant Scanner as "ScannerService<br/>scanner.py"
Cfg->>Yaml : Load chains
Manager->>Manager : Iterate chains
Manager->>BSC : Instantiate with rpc_url
W3->>Manager : get_blockchains()
W3-->>W3 : Return BSC.w3
Scanner->>W3 : get_w3("bsc")
Scanner->>Scanner : Scan blocks, detect payments
```

**Diagram sources**
- [app/core/config.py](file://app/core/config.py#L44-L57)
- [chains.yaml](file://chains.yaml#L12-L18)
- [app/blockchain/manager.py](file://app/blockchain/manager.py#L8-L33)
- [app/blockchain/bsc.py](file://app/blockchain/bsc.py#L3-L6)
- [app/blockchain/w3.py](file://app/blockchain/w3.py#L6-L9)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L20-L96)

## Detailed Component Analysis

### BSCBlockchain Class
BSCBlockchain extends the base client and sets:
- chain_id to 56 for BSC mainnet.
- use_poa to True to enable POA middleware for ExtraData compatibility.

```mermaid
classDiagram
class BlockchainBase {
+provider_url : string
+chain_id : int
+use_poa : bool
+w3 : AsyncWeb3
+is_connected() bool
+get_balance(address) int
+get_token_balance(token_address, user_address) int
+get_gas_price(use_cache) int
+get_fee_history(block_count, newest_block) dict
+estimate_gas(tx) int
+build_transaction(...)
+send_transaction(tx, private_key) string
+get_receipt(tx_hash, timeout) any
+get_latest_block() int
}
class BSCBlockchain {
+__init__(provider_url, **kwargs)
}
BlockchainBase <|-- BSCBlockchain
```

**Diagram sources**
- [app/blockchain/base.py](file://app/blockchain/base.py#L22-L146)
- [app/blockchain/bsc.py](file://app/blockchain/bsc.py#L3-L6)

**Section sources**
- [app/blockchain/bsc.py](file://app/blockchain/bsc.py#L3-L6)
- [app/blockchain/base.py](file://app/blockchain/base.py#L22-L146)

### Transaction Building and Gas Optimization
The base class implements EIP-1559 fee calculation using fee history and falls back to legacy gas pricing. It caches gas prices and adds a small buffer to gas estimates.

```mermaid
flowchart TD
Start(["build_transaction Entry"]) --> GetNonce["Resolve nonce"]
GetNonce --> BuildTx["Build tx fields<br/>chainId, from, to, value, data"]
BuildTx --> FeeHistory["Fetch fee_history"]
FeeHistory --> Has1559{"Fee history available?"}
Has1559 --> |Yes| Set1559["Set maxFeePerGas<br/>and maxPriorityFeePerGas"]
Has1559 --> |No| LegacyGas["Set gasPrice via get_gas_price()"]
Set1559 --> EstimateGas["estimate_gas(tx)"]
LegacyGas --> EstimateGas
EstimateGas --> AddBuffer["Multiply gas by 1.1"]
AddBuffer --> ReturnTx["Return tx"]
```

**Diagram sources**
- [app/blockchain/base.py](file://app/blockchain/base.py#L93-L133)
- [app/blockchain/base.py](file://app/blockchain/base.py#L65-L81)

**Section sources**
- [app/blockchain/base.py](file://app/blockchain/base.py#L65-L133)

### Transaction Monitoring and Confirmation
The ScannerService scans blocks for native and ERC-20 transfers, marks detections, and confirms after a configurable number of confirmations. Confirmations trigger optional webhooks.

```mermaid
sequenceDiagram
participant Svc as "ScannerService"
participant W3 as "AsyncWeb3"
participant DB as "Database"
Svc->>DB : Load ChainState and pending payments
loop For each block batch
Svc->>W3 : get_block(full_transactions=True)
Svc->>Svc : Match native transfers
Svc->>W3 : get_logs(ERC20 Transfer topic)
Svc->>Svc : Match ERC20 transfers
Svc->>DB : Update detected payments
end
Svc->>DB : Commit scanned progress
participant Conf as "confirm_payments"
Conf->>W3 : get block_number
Conf->>DB : Select detected payments
Conf->>Conf : Compute confirmations
Conf->>DB : Update confirmed + optional webhook
```

**Diagram sources**
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L20-L96)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L97-L134)
- [app/db/models/chain.py](file://app/db/models/chain.py#L9-L17)
- [app/db/models/token.py](file://app/db/models/token.py#L6-L15)

**Section sources**
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L14-L134)
- [app/db/models/chain.py](file://app/db/models/chain.py#L9-L17)
- [app/db/models/token.py](file://app/db/models/token.py#L6-L15)

### Settlement Workflow
The SweeperService identifies confirmed payments and prepares settlement logic. In this codebase, settlement is a placeholder; production systems would sign and submit transactions here.

```mermaid
flowchart TD
Start(["sweep_confirmed_payments Entry"]) --> LoadPayments["Select confirmed payments"]
LoadPayments --> Loop{"Any payments?"}
Loop --> |No| End(["Exit"])
Loop --> |Yes| ForEach["For each payment"]
ForEach --> Prepare["Prepare settlement<br/>(placeholder)"]
Prepare --> Mark["Mark as settled"]
Mark --> Loop
End
```

**Diagram sources**
- [app/services/blockchain/sweeper.py](file://app/services/blockchain/sweeper.py#L16-L54)

**Section sources**
- [app/services/blockchain/sweeper.py](file://app/services/blockchain/sweeper.py#L11-L54)

### Configuration and Integration Patterns
- Chains are defined in chains.yaml with name, RPC URL, and token metadata.
- Settings.chains property loads and parses the YAML.
- get_blockchains constructs BSCBlockchain instances for configured chains.
- get_w3 resolves an AsyncWeb3 instance by chain name.

```mermaid
graph LR
YAML["chains.yaml"] --> CFG["Settings.chains"]
CFG --> MAN["get_blockchains()"]
MAN --> BSC["BSCBlockchain"]
BSC --> W3["AsyncWeb3"]
W3 --> ACCESS["get_w3(chain_name)"]
```

**Diagram sources**
- [chains.yaml](file://chains.yaml#L12-L18)
- [app/core/config.py](file://app/core/config.py#L44-L57)
- [app/blockchain/manager.py](file://app/blockchain/manager.py#L8-L33)
- [app/blockchain/w3.py](file://app/blockchain/w3.py#L6-L9)

**Section sources**
- [chains.yaml](file://chains.yaml#L12-L18)
- [app/core/config.py](file://app/core/config.py#L44-L57)
- [app/blockchain/manager.py](file://app/blockchain/manager.py#L8-L33)
- [app/blockchain/w3.py](file://app/blockchain/w3.py#L6-L9)

## Dependency Analysis
- BSCBlockchain depends on BlockchainBase for RPC, gas, and transaction primitives.
- Manager selects BSCBlockchain when chains.yaml specifies name "bsc".
- ScannerService and SweeperService depend on get_w3 to access BSC.w3.
- ERC-20 detection relies on shared ABI and event topic filtering.

```mermaid
graph TB
BSC["BSCBlockchain"] --> Base["BlockchainBase"]
Manager["get_blockchains()"] --> BSC
W3["get_w3()"] --> Manager
Scanner["ScannerService"] --> W3
Sweeper["SweeperService"] --> W3
Scanner --> ABI["ERC20 ABI"]
```

**Diagram sources**
- [app/blockchain/bsc.py](file://app/blockchain/bsc.py#L3-L6)
- [app/blockchain/base.py](file://app/blockchain/base.py#L22-L146)
- [app/blockchain/manager.py](file://app/blockchain/manager.py#L8-L33)
- [app/blockchain/w3.py](file://app/blockchain/w3.py#L6-L9)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L12)
- [app/blockchain/ABI/ERC20.json](file://app/blockchain/ABI/ERC20.json#L1-L1)

**Section sources**
- [app/blockchain/bsc.py](file://app/blockchain/bsc.py#L3-L6)
- [app/blockchain/base.py](file://app/blockchain/base.py#L22-L146)
- [app/blockchain/manager.py](file://app/blockchain/manager.py#L8-L33)
- [app/blockchain/w3.py](file://app/blockchain/w3.py#L6-L9)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L12)
- [app/blockchain/ABI/ERC20.json](file://app/blockchain/ABI/ERC20.json#L1-L1)

## Performance Considerations
- Gas optimization:
  - EIP-1559 fee calculation reduces variability and improves throughput predictability.
  - Gas cache avoids frequent RPC calls; gas estimates include a small buffer to prevent failures.
- Block scanning:
  - Batch size controls RPC load; adjust to balance responsiveness and provider rate limits.
  - Confirmations reduce reorg risk; tune based on chain stability and SLAs.
- Network reliability:
  - AsyncWeb3 with timeouts supports resilient connections; monitor connection health and retry strategies.
- Throughput expectations:
  - BSC’s faster block times compared to Ethereum enable quicker confirmations; align confirmation thresholds accordingly.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
Common issues and remedies:
- Provider connectivity:
  - Verify provider URL and network reachability; use is_connected checks and logs.
- POA middleware:
  - Ensure use_poa is enabled for BSC to handle ExtraData; misconfiguration causes signature errors.
- Gas estimation failures:
  - Fallback values are applied when estimation fails; validate transaction parameters and balances.
- Missing chain configuration:
  - Confirm chains.yaml includes the "bsc" entry with a valid RPC URL; otherwise, manager falls back to local Anvil.
- Detection gaps:
  - Adjust block batch size and confirmations; ensure ChainState last_scanned_block is advancing.
- Token detection:
  - Confirm token addresses and decimals match chains.yaml; ERC-20 topics must match the ABI.

**Section sources**
- [app/blockchain/base.py](file://app/blockchain/base.py#L45-L50)
- [app/blockchain/base.py](file://app/blockchain/base.py#L86-L92)
- [app/blockchain/manager.py](file://app/blockchain/manager.py#L28-L32)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L34-L96)
- [chains.yaml](file://chains.yaml#L12-L18)

## Conclusion
The BSC implementation leverages a shared base class for RPC, gas, and transaction primitives while specializing BSC with chain ID 56 and POA middleware. Configuration-driven instantiation and a centralized Web3 accessor enable clean integration. The scanner and sweeper services provide robust monitoring and settlement foundations, with room for production enhancements such as dynamic gas buffers, improved error handling, and token-native sweeping logic.

[No sources needed since this section summarizes without analyzing specific files]

## Appendices

### BSC Mainnet and Testnet Configuration
- Mainnet:
  - Chain name: "bsc"
  - RPC URL: BSC mainnet endpoint
  - Tokens: USDT with chain address and decimals
- Testnet:
  - Not explicitly configured in the provided chains.yaml; add a new entry with a testnet RPC URL and token addresses if needed.

**Section sources**
- [chains.yaml](file://chains.yaml#L12-L18)

### Token Standard Support
- ERC-20:
  - Shared ABI enables balance queries and transfer log detection.
  - ScannerService filters ERC-20 Transfer events by topic and verifies token address and amount.

**Section sources**
- [app/blockchain/ABI/ERC20.json](file://app/blockchain/ABI/ERC20.json#L1-L1)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L71-L92)

### Practical Configuration Examples
- Define BSC chain in chains.yaml with rpc_url and tokens.
- Ensure Settings.chains is loaded and get_blockchains instantiates BSCBlockchain.
- Use get_w3("bsc") to access AsyncWeb3 for BSC operations.

**Section sources**
- [chains.yaml](file://chains.yaml#L12-L18)
- [app/core/config.py](file://app/core/config.py#L44-L57)
- [app/blockchain/manager.py](file://app/blockchain/manager.py#L8-L33)
- [app/blockchain/w3.py](file://app/blockchain/w3.py#L6-L9)