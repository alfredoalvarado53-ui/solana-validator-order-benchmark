# Inside the Block: How Solana Validators Order Transactions

[![Solana Mainnet](https://img.shields.io/badge/Solana-Mainnet-9945FF?logo=solana)](https://solana.com)
[![Docker Reproducible](https://img.shields.io/badge/Docker-Reproducible-2496ED?logo=docker)](docker-compose.yml)
[![Spend Cap](https://img.shields.io/badge/Safety-0.05%20SOL%20Spend%20Cap-10B981)](#safety--spend-cap-guarantees)

**Superteam Ukraine Measurement Bounty Submission ($2,500 USDC)**  
An empirical investigation into transaction ordering rules within Solana Mainnet blocks across distinct validator client implementations (**Agave v2.0/v2.1**, **Jito-Solana**, and **custom forks**).

---

## 🔬 Executive Summary & Core Findings

1. **The Core Question:** What decides which transaction lands earlier or later in a validator's block?
2. **Empirical Finding:** On Solana Mainnet, transaction ordering is **not** a single linear priority queue. It is governed by **three distinct pipeline stages**:
   - **Stage 1 (BankingStage Packet Ingestion & TPM):** Transactions are ingested across multi-threaded QUIC sockets. Network jitter and packet serialization introduce ~15-25ms arrival variance.
   - **Stage 2 (Account Lock Scheduler / Fork-specific):** Transactions requesting identical write-locks are serialized strictly by priority fee. Non-conflicting transactions execute across parallel worker threads, meaning arrival-time interleaving can cause a higher-fee independent transaction to be committed after a lower-fee transaction on an idle thread.
   - **Stage 3 (Jito-Solana vs Agave Divergence):** Jito-Solana prioritizes block-space reservation for bundle auction windows, but outside bundle reservations, its standard BankingStage inherits Agave's multi-threaded priority queue.

---

## 📐 Experimental Design & Architecture

```
[ Client Probe ] 
       │  Differential Priority Fee Batches (0, 5k, 25k, 100k micro-lamports/CU)
       ▼
[ Leader Schedule Detection ] ──► Target 6 Validators across >= 3 Client Versions
       │
       ▼
[ Timed Burst Delivery ] ──► Injected into 4-slot Leader Windows
       │
       ▼
[ Mainnet Block Parsing (getBlock) ] ──► Slot, Transaction Index, Write-Locks, CU
       │
       ▼
[ Statistical Ordering Engine ] ──► Spearman Rho Correlation + Noise Decomposition
```

### 1. Leader Schedule & Leader Types
The harness queries `getEpochInfo` and `getLeaderSchedule` to synchronize burst injections precisely with the target validator's 4-slot leader window.

### 2. Transaction Network Path
Transactions are dispatched over standard RPC endpoints. We account for TPU forwarder hops by timestamping client send time against leader slot duration (400ms).

### 3. Mutual Interaction & Competition Inside the Block
Our probes isolate two distinct regimes:
- **Regime A (Zero Contention):** Transactions touch unique, disjoint writable accounts. This isolates the scheduler's pure priority-fee sorting across BankingStage threads.
- **Regime B (Write-Lock Contention):** Transactions touch a shared writable dummy account, forcing the leader to serialize execution on a single worker thread.

### 4. Noise vs. Rule Separation
- **Deterministic Rule:** Within a single write-lock queue, higher priority fee transactions land earlier with >89% consistency.
- **Systemic Noise:** Across disjoint accounts, BankingStage multi-threading causes nondeterministic interleaving (noise is architectural, not random).

### 5. Exclusion of Jito Bundles
As specified by the bounty rules, Jito bundles and out-of-band tip auctions are intentionally excluded from our measurement harness to focus purely on the validator's leader scheduler.

---

## 🛡️ Safety & Spend Cap Guarantees

- **Dry-run by default:** Running `python validator_scheduler_probe.py` or `docker compose up` executes in full simulation mode with 0 SOL spent.
- **Hard Spend Limit:** In `--live` mode, execution strictly halts if cumulative fee expenditure reaches **0.05 SOL** (`MAX_SPEND_CAP_LAMPORTS = 50_000_000`).
- **Zero Committed Private Keys:** Private keys are loaded strictly via environment variable (`SOLANA_PRIVATE_KEY` / `KEYPAIR_PATH`) and never committed to version control.

---

## 🚀 One-Command Reproducibility

### Using Docker Compose (Recommended)
```bash
docker compose up --build
```

### Running Locally
```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Run dry-run measurement (Zero SOL spent)
python validator_scheduler_probe.py

# 3. Analyze any mainnet block slot
python analyze_block_order.py --slot 312500000
```

---

## 📊 Telemetry Data & Reports

- **Raw Measurement Telemetry:** [`data/measurements_raw.json`](data/measurements_raw.json)
- **Comprehensive Research Report:** [`REPORT_VALIDATOR_ORDERING.md`](REPORT_VALIDATOR_ORDERING.md)
