# Performance Optimization

<cite>
**Referenced Files in This Document**
- [app/db/engine.py](file://app/db/engine.py)
- [app/db/session.py](file://app/db/session.py)
- [app/db/async_session.py](file://app/db/async_session.py)
- [app/db/models/payment.py](file://app/db/models/payment.py)
- [app/db/models/chain.py](file://app/db/models/chain.py)
- [app/blockchain/base.py](file://app/blockchain/base.py)
- [app/blockchain/w3.py](file://app/blockchain/w3.py)
- [app/blockchain/manager.py](file://app/blockchain/manager.py)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py)
- [app/workers/listener.py](file://app/workers/listener.py)
- [app/workers/webhook.py](file://app/workers/webhook.py)
- [app/services/webhook.py](file://app/services/webhook.py)
- [app/api/v1/payments.py](file://app/api/v1/payments.py)
- [app/core/config.py](file://app/core/config.py)
- [requirements.txt](file://requirements.txt)
- [alembic/versions/2026_01_27_1807-5ec6405addd0_initial_database_schema.py](file://alembic/versions/2026_01_27_1807-5ec6405addd0_initial_database_schema.py)
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
This document provides a comprehensive performance optimization guide for the cTrip Payment Gateway. It focuses on system-wide tuning strategies across database connection pooling, asynchronous blockchain interactions, memory management, query optimization, caching for gas price and blockchain data, async/await best practices, monitoring and bottleneck identification, and scaling for high-throughput payment processing. Specific optimization examples are drawn from the codebase, including gas price caching, connection reuse patterns, and efficient blockchain polling strategies.

## Project Structure
The application follows a layered architecture:
- API layer: FastAPI endpoints for payment creation and health checks
- Worker layer: Dramatiq actors orchestrating scanning and sweeping tasks
- Service layer: Business logic for scanning, confirming, and webhook dispatch
- Blockchain layer: Web3 providers abstraction with per-chain configuration
- Database layer: SQLAlchemy ORM models and async engines with connection pooling

```mermaid
graph TB
subgraph "API Layer"
API["FastAPI Payments Endpoint"]
end
subgraph "Workers"
Listener["Dramatiq Listener Actor"]
Sweeper["Dramatiq Sweeper Actor"]
WebhookActor["Dramatiq Webhook Actor"]
end
subgraph "Services"
Scanner["ScannerService"]
SweeperSvc["SweeperService"]
WebhookSvc["WebhookService"]
end
subgraph "Blockchain"
Manager["Blockchain Manager"]
W3["get_w3()"]
Base["BlockchainBase<br/>Gas Price Caching"]
end
subgraph "Database"
Engine["SQLAlchemy Engines<br/>Connection Pooling"]
AsyncEngine["Async Engines"]
Models["ORM Models"]
end
API --> Listener
API --> Sweeper
Listener --> Scanner
Sweeper --> SweeperSvc
Scanner --> WebhookActor
WebhookActor --> WebhookSvc
Scanner --> Engine
Sweeper --> AsyncEngine
Scanner --> Models
Sweeper --> Models
Manager --> W3
W3 --> Base
Base --> Engine
Base --> AsyncEngine
```

**Diagram sources**
- [app/api/v1/payments.py](file://app/api/v1/payments.py#L1-L62)
- [app/workers/listener.py](file://app/workers/listener.py#L1-L46)
- [app/workers/sweeper.py](file://app/workers/sweeper.py#L1-L40)
- [app/workers/webhook.py](file://app/workers/webhook.py#L1-L37)
- [app/services/webhook.py](file://app/services/webhook.py#L1-L45)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L1-L134)
- [app/blockchain/manager.py](file://app/blockchain/manager.py#L1-L33)
- [app/blockchain/w3.py](file://app/blockchain/w3.py#L1-L9)
- [app/blockchain/base.py](file://app/blockchain/base.py#L1-L133)
- [app/db/engine.py](file://app/db/engine.py#L1-L32)
- [app/db/async_session.py](file://app/db/async_session.py#L1-L15)
- [app/db/models/payment.py](file://app/db/models/payment.py#L1-L74)
- [app/db/models/chain.py](file://app/db/models/chain.py#L1-L17)

**Section sources**
- [app/api/v1/payments.py](file://app/api/v1/payments.py#L1-L62)
- [app/workers/listener.py](file://app/workers/listener.py#L1-L46)
- [app/workers/sweeper.py](file://app/workers/sweeper.py#L1-L40)
- [app/workers/webhook.py](file://app/workers/webhook.py#L1-L37)
- [app/services/webhook.py](file://app/services/webhook.py#L1-L45)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L1-L134)
- [app/blockchain/manager.py](file://app/blockchain/manager.py#L1-L33)
- [app/blockchain/w3.py](file://app/blockchain/w3.py#L1-L9)
- [app/blockchain/base.py](file://app/blockchain/base.py#L1-L133)
- [app/db/engine.py](file://app/db/engine.py#L1-L32)
- [app/db/async_session.py](file://app/db/async_session.py#L1-L15)
- [app/db/models/payment.py](file://app/db/models/payment.py#L1-L74)
- [app/db/models/chain.py](file://app/db/models/chain.py#L1-L17)

## Core Components
- Database connection pooling: synchronous and asynchronous engines configured with pre-ping and tuned pool sizes
- Asynchronous session management: scoped async sessions for worker tasks
- Blockchain abstraction: per-chain providers with gas price caching and fee history support
- Worker orchestration: Dramatiq actors for scanning, sweeping, and webhook dispatch
- Query optimization: targeted queries with explicit joins and minimal scans
- Caching: gas price caching with short TTL to reduce RPC overhead
- Async/await best practices: structured concurrency with proper event loops and timeouts

**Section sources**
- [app/db/engine.py](file://app/db/engine.py#L1-L32)
- [app/db/session.py](file://app/db/session.py#L1-L17)
- [app/db/async_session.py](file://app/db/async_session.py#L1-L15)
- [app/blockchain/base.py](file://app/blockchain/base.py#L65-L80)
- [app/blockchain/w3.py](file://app/blockchain/w3.py#L1-L9)
- [app/blockchain/manager.py](file://app/blockchain/manager.py#L1-L33)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L1-L134)
- [app/workers/listener.py](file://app/workers/listener.py#L1-L46)
- [app/workers/webhook.py](file://app/workers/webhook.py#L1-L37)
- [app/services/webhook.py](file://app/services/webhook.py#L1-L45)

## Architecture Overview
The system integrates FastAPI endpoints, Dramatiq workers, and blockchain providers with a robust database layer. The key performance levers are:
- Connection pooling for PostgreSQL and SQLite
- Async session reuse in worker tasks
- Gas price caching in blockchain providers
- Efficient blockchain polling with batched blocks and targeted logs
- Structured async workflows with timeouts and retries

```mermaid
sequenceDiagram
participant Client as "Client"
participant API as "FastAPI Payments"
participant DB as "SQLAlchemy Sessions"
participant HDW as "HDWallet Manager"
participant Chain as "Blockchain Provider"
Client->>API : "POST /api/v1/payments/"
API->>DB : "Create Payment record"
API->>HDW : "Generate address"
HDW-->>API : "Address"
API->>Chain : "Optional token validation via provider"
Chain-->>API : "Validation result"
API-->>Client : "PaymentCreated"
```

**Diagram sources**
- [app/api/v1/payments.py](file://app/api/v1/payments.py#L18-L54)
- [app/db/session.py](file://app/db/session.py#L11-L16)
- [app/blockchain/w3.py](file://app/blockchain/w3.py#L6-L9)

## Detailed Component Analysis

### Database Connection Pooling and Session Management
- Synchronous engine configured with pre-ping and a fixed pool size suitable for typical workloads
- Asynchronous engine configured with pre-ping for connection health checks
- Session factories ensure proper lifecycle management and closure
- Async session factory yields sessions within worker tasks to avoid leaks

```mermaid
flowchart TD
Start(["Worker Task Starts"]) --> Acquire["Acquire Async Session"]
Acquire --> UseDB["Execute Queries / Commits"]
UseDB --> Commit["Commit or Rollback"]
Commit --> Release["Release Session"]
Release --> End(["Task Ends"])
```

**Diagram sources**
- [app/db/async_session.py](file://app/db/async_session.py#L12-L15)
- [app/db/session.py](file://app/db/session.py#L11-L16)

**Section sources**
- [app/db/engine.py](file://app/db/engine.py#L22-L31)
- [app/db/session.py](file://app/db/session.py#L5-L16)
- [app/db/async_session.py](file://app/db/async_session.py#L6-L15)

### Async Operation Optimization for Blockchain Interactions
- BlockchainBase caches gas price with a short TTL to minimize RPC calls
- Fee history is used to construct EIP-1559 transactions when available, falling back to legacy pricing
- get_w3 returns cached AsyncWeb3 instances per chain
- ScannerService batches block scanning and uses targeted logs for ERC20 detection

```mermaid
classDiagram
class BlockchainBase {
+provider_url
+chain_id
+use_poa
+w3
-_gas_price_cache
-_gas_price_timestamp
-_gas_cache_duration
+get_gas_price(use_cache) int
+get_fee_history(block_count, newest_block) dict
+build_transaction(...)
}
class ScannerService {
+session
+confirmations_required
+block_batch_size
+scan_chain(chain_name) void
+confirm_payments(chain_name) void
}
ScannerService --> BlockchainBase : "uses via get_w3()"
```

**Diagram sources**
- [app/blockchain/base.py](file://app/blockchain/base.py#L41-L80)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L14-L96)

**Section sources**
- [app/blockchain/base.py](file://app/blockchain/base.py#L65-L80)
- [app/blockchain/base.py](file://app/blockchain/base.py#L116-L133)
- [app/blockchain/w3.py](file://app/blockchain/w3.py#L6-L9)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L20-L96)

### Memory Management Techniques
- Avoid building large intermediate lists unnecessarily; process logs and transactions iteratively
- Use scalar queries and minimal projections to reduce memory footprint
- Reuse AsyncWeb3 instances per chain to avoid repeated provider initialization
- Close sessions promptly after use to free resources

**Section sources**
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L41-L50)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L72-L92)
- [app/db/async_session.py](file://app/db/async_session.py#L12-L15)

### Query Optimization Strategies for SQLAlchemy Models
- Use targeted selects with filters to avoid full-table scans
- Leverage foreign keys and indexes for joins (e.g., payments.token_id -> tokens.id)
- Use with_for_update() to coordinate scanning state safely
- Keep queries minimal; fetch only required columns

```mermaid
flowchart TD
A["Select Payments by Status and Chain"] --> B["Iterate Results"]
B --> C{"Has Token?"}
C --> |Yes| D["Join Token Details"]
C --> |No| E["Native Transfer Path"]
D --> F["Compare Amount vs Log Value"]
E --> G["Compare Amount vs Tx Value"]
```

**Diagram sources**
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L41-L92)
- [app/db/models/payment.py](file://app/db/models/payment.py#L41-L57)
- [app/db/models/token.py](file://app/db/models/token.py#L1-L74)

**Section sources**
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L24-L32)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L41-L50)
- [app/db/models/payment.py](file://app/db/models/payment.py#L41-L57)

### Caching Mechanisms for Gas Price and Blockchain Data
- Gas price cache with a 10-second TTL reduces frequent RPC calls
- Fee history enables dynamic fee calculation for EIP-1559 transactions
- Per-chain AsyncWeb3 instances are reused across requests

```mermaid
sequenceDiagram
participant Svc as "Transaction Builder"
participant Base as "BlockchainBase"
participant Cache as "Gas Cache"
participant RPC as "RPC Provider"
Svc->>Base : "get_gas_price(use_cache=True)"
Base->>Cache : "Check TTL"
alt "Cache Fresh"
Cache-->>Base : "Cached Gas Price"
else "Cache Stale"
Base->>RPC : "eth_gasPrice"
RPC-->>Base : "Gas Price"
Base->>Cache : "Update Cache"
end
Base-->>Svc : "Gas Price"
```

**Diagram sources**
- [app/blockchain/base.py](file://app/blockchain/base.py#L65-L80)

**Section sources**
- [app/blockchain/base.py](file://app/blockchain/base.py#L41-L80)

### Async/Await Best Practices Throughout the Application
- Workers use asyncio.run within actors to execute async tasks
- Proper event loop management ensures compatibility with Dramatiq
- Webhook actor wraps async execution and raises exceptions for retries
- Timeout configuration for HTTP clients prevents hanging requests

```mermaid
sequenceDiagram
participant Actor as "Dramatiq Actor"
participant Loop as "Event Loop"
participant Runner as "Async Runner"
participant Service as "Service Layer"
Actor->>Loop : "Set loop"
Actor->>Runner : "async def run()"
Runner->>Service : "Call async methods"
Service-->>Runner : "Results"
Runner-->>Actor : "Complete"
```

**Diagram sources**
- [app/workers/listener.py](file://app/workers/listener.py#L29-L40)
- [app/workers/webhook.py](file://app/workers/webhook.py#L24-L36)
- [app/services/webhook.py](file://app/services/webhook.py#L33-L36)

**Section sources**
- [app/workers/listener.py](file://app/workers/listener.py#L18-L46)
- [app/workers/webhook.py](file://app/workers/webhook.py#L9-L37)
- [app/services/webhook.py](file://app/services/webhook.py#L33-L36)

### Efficient Blockchain Polling Strategies
- Batch scanning by configurable block_batch_size to limit per-cycle RPC calls
- Use targeted logs for ERC20 transfers to avoid scanning all blocks
- Confirm payments after sufficient confirmations to reduce re-scans
- Use chain-specific providers with POA middleware when needed

```mermaid
flowchart TD
Start(["Start Scan Cycle"]) --> LoadState["Load ChainState with lock"]
LoadState --> ComputeRange["Compute Block Range"]
ComputeRange --> HasPending{"Any Pending?"}
HasPending --> |No| SaveState["Save Last Scanned Block"] --> End(["Exit"])
HasPending --> |Yes| Iterate["Iterate Blocks in Batch"]
Iterate --> FetchLogs["Fetch ERC20 Logs for Block"]
FetchLogs --> Match["Match Logs to Payments"]
Match --> Native["Check Native Transfers"]
Native --> Update["Update Payment Status"]
Update --> Iterate
SaveState --> End
```

**Diagram sources**
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L20-L96)

**Section sources**
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L14-L18)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L34-L39)
- [app/services/blockchain/scanner.py](file://app/services/blockchain/scanner.py#L72-L92)

## Dependency Analysis
External dependencies relevant to performance:
- SQLAlchemy 2.x for ORM and async engines
- asyncpg for async PostgreSQL driver
- aiosqlite for async SQLite driver
- web3 7.x for blockchain interactions
- uvloop for improved asyncio performance
- httpx for async HTTP requests in webhooks

```mermaid
graph TB
SQLAlchemy["SQLAlchemy 2.x"]
AsyncPG["asyncpg"]
AioSQLite["aiosqlite"]
Web3["web3 7.x"]
UvLoop["uvloop"]
HTTPX["httpx"]
SQLAlchemy --> AsyncPG
SQLAlchemy --> AioSQLite
Web3 --> HTTPX
UvLoop --> Web3
```

**Diagram sources**
- [requirements.txt](file://requirements.txt#L89-L106)

**Section sources**
- [requirements.txt](file://requirements.txt#L89-L106)

## Performance Considerations
- Connection pooling
  - Tune pool_size and pool_pre_ping according to database capacity and concurrent workload
  - Prefer async engines for I/O-bound tasks; keep sync engines for CPU-bound migrations
- Async operations
  - Use structured concurrency; avoid blocking calls inside async contexts
  - Set timeouts for external HTTP calls and RPC requests
- Memory usage
  - Minimize intermediate collections; stream results where possible
  - Reuse provider instances; avoid recreating AsyncWeb3 per request
- Query performance
  - Add indexes on frequently filtered columns (e.g., payments.status, payments.chain)
  - Use targeted selects and joins; avoid N+1 queries
- Caching
  - Adjust gas cache duration based on network volatility
  - Cache chain metadata and provider configurations
- Monitoring and observability
  - Instrument async tasks and database queries
  - Track latency distributions and error rates
- Scaling
  - Horizontal scaling via multiple worker processes
  - Use separate queues for high-priority tasks (e.g., confirmations)
  - Consider read replicas for heavy reads

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
- Database connectivity issues
  - Verify pool_pre_ping settings and connection URLs
  - Monitor pool utilization and timeouts
- Blockchain provider errors
  - Validate RPC URLs and chain IDs
  - Check gas price cache staleness and fallback behavior
- Worker failures
  - Inspect Dramatiq actor logs and retry policies
  - Ensure proper event loop setup in actors
- Webhook delivery problems
  - Confirm timeout settings and signatures
  - Review webhook service error handling

**Section sources**
- [app/db/engine.py](file://app/db/engine.py#L22-L31)
- [app/blockchain/base.py](file://app/blockchain/base.py#L45-L50)
- [app/workers/webhook.py](file://app/workers/webhook.py#L32-L36)
- [app/services/webhook.py](file://app/services/webhook.py#L39-L44)

## Conclusion
By combining efficient database pooling, structured async workflows, targeted query patterns, and strategic caching, the cTrip Payment Gateway can achieve high throughput while maintaining reliability. The provided patterns—connection reuse, gas price caching, and batched blockchain polling—are grounded in the existing codebase and can be tuned further based on production metrics and scaling needs.

[No sources needed since this section summarizes without analyzing specific files]

## Appendices

### Database Schema and Indexes
- Payments table includes numeric amount and indexed fields for chain and status
- Tokens table includes address and chain indexes for efficient joins
- ChainStates table tracks scanned blocks with a unique constraint on chain

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
datetime expires_at
datetime created_at
}
TOKENS {
uuid id PK
string chain
string address
string symbol
integer decimals
boolean enabled
datetime created_at
}
CHAIN_STATES {
integer id PK
string chain UK
integer last_scanned_block
}
PAYMENTS }o--|| TOKENS : "token_id"
PAYMENTS }o--o| CHAIN_STATES : "chain"
```

**Diagram sources**
- [alembic/versions/2026_01_27_1807-5ec6405addd0_initial_database_schema.py](file://alembic/versions/2026_01_27_1807-5ec6405addd0_initial_database_schema.py#L58-L82)
- [app/db/models/payment.py](file://app/db/models/payment.py#L41-L57)
- [app/db/models/token.py](file://app/db/models/token.py#L1-L74)
- [app/db/models/chain.py](file://app/db/models/chain.py#L9-L17)