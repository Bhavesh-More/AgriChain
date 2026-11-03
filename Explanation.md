# AgriChain Sentinel — How the Blockchain Works, and Where the ML Fits

This document explains the **blockchain ledger** in depth (data structures, hashing, Merkle trees, proof-of-work,
digital signatures, verification and attacks). It then explains **where and how machine learning is used**, and how the
two halves meet. Every claim here maps to code you can open; file references are given as `path:function`.

> One-line summary: every agricultural supply-chain transaction is **scored for fraud by ML before it is allowed onto
> the chain**. What reaches the chain is sealed into SHA-256 / Merkle / proof-of-work blocks and signed by the seller
> (Ed25519), so any later edit to the history is detected and located.

---

## 1. The big picture

```
 seller submits invoice
        │
        ▼
 ┌──────────────── ML gate (backend/app/services.py:assess) ────────────────┐
 │ 21 features → 7 named rules + Random Forest + Isolation Forest           │
 │ + Ed25519 signature check                                                │
 │ risk = 0.60·P(fraud) + 0.25·rule_score + 0.15·anomaly                    │
 └───────┬───────────────────────┬───────────────────────────┬──────────────┘
   risk < 0.40              0.40 – 0.70                  ≥ 0.70 (or forged)
   CLEAR                    FLAGGED                      QUARANTINED
        │                       │                            │  held OFF-chain
        ▼                       ▼                            ▼
   mempool (COMMITTED)     mempool (FLAGGED)          auditor: approve / reject
        │                       │                            │
        └───────────┬───────────┘             APPROVED / REJECTED → mempool
                    ▼                                        │
 ┌──────────── Blockchain (backend/app/chain/ledger.py) ─────▼──────────────┐
 │ 5 records → Merkle root → block header → proof-of-work → append to chain │
 │ verify_chain(): recompute every hash, root, PoW, link and signature      │
 └──────────────────────────────────────────────────────────────────────────┘
```

The ML decides **what is allowed in, and with which status**. The blockchain makes sure that **nobody can change
what was decided** without being caught.

---

## 2. Why a blockchain here?

An agricultural supply chain has many parties (farmer, cooperative, processor, distributor, retailer) who do not fully
trust each other, and an auditor who must be able to prove later what was recorded and decided. An ordinary database
table can be edited silently by anyone with write access. A hash-chained ledger turns that silent edit into a
**detectable** one:

| Property | How it is achieved here |
|---|---|
| **Tamper evidence** | Each record is hashed; hashes go into a Merkle root; the root goes into a block header; each header contains the previous block's hash. Changing anything changes a hash that no longer matches. |
| **Ordering / immutability of history** | Blocks are linked by `prev_hash`; inserting, removing or reordering a block breaks the link. |
| **Cost to rewrite** | Proof-of-work: every block's hash must start with `POW_DIFFICULTY` hex zeros. |
| **Authenticity (who issued it)** | Every invoice is signed by the seller's **Ed25519** private key; verification uses the seller's public key. |
| **Auditability of decisions** | The final ledger status (COMMITTED / FLAGGED / APPROVED / REJECTED) is part of the record hash, so the *verdict* is on-chain, not just the data. |

---

## 3. Data structures (`backend/app/models.py`)

| Table | Role in the chain | Key fields |
|---|---|---|
| `transaction` | The invoice itself (immutable fields) | 21 business fields (see §4.1) + `signature` (hex Ed25519) |
| `participant` | Identity registry | `id`, `role`, `public_key` (hex Ed25519) |
| `ledgerentry` | One **record** per transaction on its way to / inside a block | `tx_id`, `status`, `block_index` (NULL = in the mempool), `position`, `tx_hash` |
| `block` | Block header | `index`, `timestamp`, `prev_hash`, `merkle_root`, `nonce`, `difficulty`, `hash`, `tx_count` |
| `assessment` | The ML verdict (off-chain metadata) | `risk`, `p_fraud`, `rule_score`, `anomaly_score`, `pred_type`, `decision`, `rule_hits`, `shap_top`, `features`, `model_version` |
| `review` | The auditor's verdict (off-chain metadata; its outcome is on-chain via `status`) | `action`, `note`, `reviewed_at` |

**Mempool** = all `ledgerentry` rows whose `block_index` is `NULL`. **Sealing** = assigning them to a new block.

---

## 4. Hashing

### 4.1 Canonical serialization (`backend/app/chain/identity.py`)

Two machines must produce byte-identical input for the same transaction, otherwise hashes would not be reproducible.
The project defines the signed/hashed fields **once**:

```python
SIGNED_FIELDS = ["id", "batch_id", "crop", "stage", "seller_id", "buyer_id", "timestamp",
                 "quantity_kg", "unit_price", "total_value", "ref_price", "quality_grade",
                 "moisture_pct", "organic_claimed", "organic_cert_valid", "transit_km",
                 "transit_hours", "temp_avg_c", "temp_max_c", "cold_chain_ok_claimed", "payment_mode"]
```

`canon()` normalises each value: datetimes → ISO-8601 to the second, floats → rounded to 4 decimals. The payload is then
`json.dumps(..., sort_keys=True, separators=(",", ":"))` — sorted keys, no whitespace.

### 4.2 Record hash (`ledger.py:tx_hash`)

```python
payload = {f: canon(tx.f) for f in SIGNED_FIELDS}
payload["signature"]     = tx.signature        # seller's Ed25519 signature
payload["ledger_status"] = status              # COMMITTED / FLAGGED / APPROVED / REJECTED
tx_hash = SHA-256(canonical_json(payload))
```

Why include the **status**? Because the most valuable insider attack in a fraud system is not editing a quantity — it is
quietly turning a *rejected* fraud into an *accepted* one. Since the status is inside the hash, flipping it changes the
hash, which changes the Merkle root, which no longer matches the sealed block header.

Why include the **signature**? So that swapping a signature (e.g. to replace a forged one with a valid-looking one) also
breaks the record hash.

`sim_label` (the simulator's ground truth) is deliberately **not** hashed and never used for scoring.

### 4.3 Merkle root (`ledger.py:merkle_root`)

The record hashes of a block are combined pairwise up a binary tree:

```
                          root = H(A + B)                      ← stored in the block header
                        /                 \
           A = H(H01 + H23)          B = H(H44 + H44)          ← 3 nodes: odd, so H44 is paired with itself
             /          \                 |
   H01 = H(h0 + h1)  H23 = H(h2 + h3)  H44 = H(h4 + h4)        ← 5 leaves: odd, so h4 is paired with itself
      /     \          /     \            |
     h0     h1        h2     h3           h4                   ← record hashes (BLOCK_SIZE = 5)
```
Inclusion proof for `h2`: `[h3 (right), H01 (left), B (right)]` — three hashes rebuild the root.

* Pairwise `SHA-256(left + right)` on the hex strings; an odd last node is paired with itself.
* An empty block (genesis) has root `SHA-256("")`.
* Order matters: swapping two records changes the root (tested in `tests/test_ledger.py`).

**Inclusion proof** (`ledger.py:merkle_proof`): to prove record *i* is in a block you only need the sibling hash at each
level — ⌈log₂ n⌉ hashes instead of all n records. `GET /api/chain/blocks/{i}` returns the proof for every record, and
the **Ledger** page rebuilds the whole tree in the browser with the Web Crypto API and highlights the proof path when
you click **Prove** — an independent check that does not trust the server's Python code.

### 4.4 Block header and proof-of-work (`ledger.py:header_hash`, `ledger.py:mine`)

```python
header = f"{index}|{timestamp}|{prev_hash}|{merkle_root}|{nonce}|{difficulty}"
block_hash = SHA-256(header)

def mine(...):
    nonce = 0
    while not header_hash(..., nonce, ...).startswith("0" * difficulty):
        nonce += 1
```

* `POW_DIFFICULTY = 3` → the hash must start with `000`. Each hex digit is 1/16, so the expected number of tries is
  16³ = **4,096** (milliseconds in Python). In a live run the demo printed nonces of 9,347 and 230 — exactly the kind of
  spread you expect from a geometric distribution.
* Each extra zero multiplies the work by 16. Difficulty is stored per block and covered by the header hash.
* **Genesis** block: index 0, `prev_hash = "0" × 64`, no records.

### 4.5 Sealing rules (`ledger.py:seal_pending`)

* A block is sealed whenever the mempool holds `BLOCK_SIZE` (5) records.
* An auditor verdict, or `POST /api/chain/seal`, **force-seals** a partial block so the result is visible at once.
* Sealing = compute every record hash with its *final* status → Merkle root → mine → store the block → set
  `block_index`/`position` on each record.

---

## 5. Digital signatures — identity on the chain (`backend/app/chain/identity.py`)

Hashes prove that data did not change **after sealing**. They do not prove **who** created it, and an attacker who
controls the database could edit a record and simply recompute all hashes and re-mine. Signatures close that gap.

| Step | What happens |
|---|---|
| Keys | Each participant has an **Ed25519** key pair. The public key is stored on `participant.public_key`. |
| Signing | The seller signs the canonical JSON of the 21 `SIGNED_FIELDS` (the same fields that are hashed, without the status). |
| At ingest | `services.assess` verifies the signature against the seller's public key. If it fails, rule `SIGNATURE_INVALID` (severity 1.0) fires and the scorer **hard-quarantines** the invoice (`risk ≥ 0.99`, type "Forged signature") — regardless of what the ML model thinks. |
| At verification | `verify_chain` re-checks **every** sealed record's signature. A record whose content changed no longer matches what the seller signed. |
| Rejected forgeries | A forged invoice that an auditor rejects is sealed as `REJECTED` evidence; its invalid signature is expected and accepted *only* for that case (`ledger.py:rejected_forgery`). |

**Demo custody (stated honestly):** private keys are derived on the server from `KEY_SEED + participant id`
(`identity.private_key`) so that the simulator can sign on everyone's behalf. In a real deployment each party would
hold its own private key and only the public key would be registered. The threat model of the demo is therefore
*"an insider can write to the database but does not hold the signing keys."*

---

## 6. Verification (`ledger.py:verify_chain`, `ledger.py:verify_block`)

`GET /api/chain/verify` walks every block in order and, for each one:

| # | Check | Detects | Message shown |
|---|---|---|---|
| 1 | Recompute each record hash from **today's** DB rows (with its stored status) and compare | edited data, flipped verdict, swapped signature, deleted transaction | — (feeds check 2, and lists the tampered tx ids) |
| 2 | Recompute the Merkle root from the recomputed hashes and compare with the header | any record change, deletion or reordering | *Merkle root mismatch — transaction data or verdict changed after sealing (TX-…)* |
| 3 | Recompute the header hash | edited nonce / timestamp / root / difficulty | *Block header was edited — its hash no longer matches its contents.* |
| 4 | Hash meets the difficulty target | a block that was never properly mined | *Proof-of-work invalid* |
| 5 | `prev_hash` equals the previous block's hash | removed / inserted / reordered blocks | *Broken link — previous-hash pointer does not match the previous block.* |
| 6 | Every seller signature verifies | edits hidden by recomputing hashes and re-mining | *Seller signature invalid — … Hashes were recomputed to hide the edit, but signatures cannot be.* |

The result names the **first broken block**, the reason, and the tampered transaction ids; every block also gets its
own ok/fail so the UI can colour the chain.

### 6.1 Attack matrix (verified by running each attack against a fresh database)

| Attack (insider with DB write access) | Caught? | Caught by |
|---|---|---|
| Edit a sealed quantity (demo button *Edit a quantity*) | ✅ | Merkle root (check 2), tx id named |
| Flip a sealed REJECTED verdict to COMMITTED (demo button) | ✅ | Merkle root — the status is hashed |
| Edit a block header / nonce (demo button) | ✅ | Header hash (check 3) |
| Edit a quantity **and** recompute every hash and re-mine all later blocks (demo button *Rewrite history*) | ✅ | Seller signature (check 6) — the only check left |
| Delete a sealed record or transaction row | ✅ | Merkle root |
| Delete a block from the middle of the chain | ✅ | Broken link on the next block |
| Replace a stored signature | ✅ | Merkle root (signature is hashed) |
| Replace a seller's registered public key | ✅ | Signature check fails for that seller's records |
| **Flip a verdict and then re-mine** | ❌ | Not caught. Signatures cover the transaction data but not the ledger status. Fix: a validator signature on each block header. This is recorded as a strict `xfail` test in `tests/test_demo_scenarios.py`. |
| **Delete the newest block(s)** (tail truncation) | ❌ | Not caught. Every remaining block still links correctly. Fix: anchor the latest block hash outside the database (e.g. on Ethereum, listed under "Later"). |
| Edit an assessment's risk score, rule hits or SHAP values, or an auditor's note | ❌ | Not covered: these are off-chain metadata. The **decision outcome** is protected through the ledger status; the numbers behind it are not. |

The four demo buttons on the **Ledger** page (and `POST /api/dev/tamper`, available only when `DEMO_MODE=true`) run
the first four rows live. `tests/test_ledger.py`, `tests/test_identity_and_loop.py` and `tests/test_demo_scenarios.py`
assert each of them.

### 6.2 What this is — and is not

* **It is** a real, single-node, tamper-evident blockchain: SHA-256 record hashes, Merkle roots with inclusion proofs,
  linked block headers, proof-of-work, Ed25519 signatures and full re-verification.
* **It is not** a distributed network: there are no peers, no gossip and no multi-party consensus. With one node,
  proof-of-work is a *cost to rewrite* rather than a consensus mechanism — an attacker who re-mines on the same machine
  pays that cost easily, which is exactly why the signature layer exists. This was a deliberate scope decision
  (`DESIGN.md` §3); anchoring block hashes on a local Ethereum node is the documented next step.

---

## 7. Where machine learning is used

| Place | Code | What ML does there |
|---|---|---|
| **Every incoming transaction** (API, Check page, simulator) | `services.assess` → `ml/features.py`, `ml/rules.py`, `ml/scorer.py`, `ml/explain.py` | Computes features, scores fraud risk, predicts the fraud type, explains the decision. This is the gate in front of the chain. |
| **Dry-run scoring** (Check page "what-if") | `POST /api/transactions/score` | Same pipeline, nothing persisted. |
| **Demo history seeding** | `sim/state.py:seed_demo` | Scores ~300 historic transactions in one vectorised batch (`score_many`). |
| **Training** | `ml/training.py:train` (`scripts/train_model.py`) | Fits the Random Forest, Isolation Forest and the Logistic Regression baseline; writes `artifacts/model.joblib` + `metrics.json`. |
| **Learning loop** | `ml/feedback.py`, `POST /api/model/retrain` | Turns auditor verdicts into labels, retrains, hot-swaps the model. |
| **Model Insights page** | `GET /api/model/report` | Shows the evaluation that training produced. |

Where ML is **not** used: chain verification, hashing, Merkle proofs and signature checks are pure cryptography —
deterministic and independent of any model. The forged-signature gate is also cryptographic, not learned.

### 7.1 Data — trained from scratch on a synthetic simulator (`backend/app/sim/generator.py`)

No public labelled dataset of agricultural-transaction fraud exists, and a pretrained Hugging Face model would be a
text/vision model forced onto tabular data. The project therefore simulates the supply chain:

* 140 participants (80 farmers with acreage, 12 coops, 10 processors, 14 distributors, 24 retailers) in 6 Indian states;
  6 crops (wheat, rice, soybean; perishable onion, tomato, mango with temperature limits).
* A deterministic daily "mandi" reference price per crop and stage, with a seasonal wave.
* Batches move Farmer → Coop → Processor → Distributor → Retailer, losing 1–5 % per hop.
* **Six injected fraud types** — price manipulation, phantom quantity, organic label fraud, concealed cold-chain breach,
  circular trading, duplicate invoice — each with ground-truth labels.
* **Realism guards** so the model can't just memorise the generator: ~4 % legitimate outliers (drought price spike,
  bumper harvest, honestly disclosed temperature breach, fast highway trip, cash payment), 0.4 % label noise, and a
  **shifted test set** where fraud is subtler.
* Dataset: 20,000 transactions (7 % fraud), split 75/25 stratified, plus 5,000 shifted transactions.

### 7.2 Features — one function, used everywhere (`ml/features.py:compute_features`)

21 features: crop, perishability, stage, log quantity, price deviation vs. reference (signed and absolute), quantity
vs. what the seller could have (acreage × yield for farmers, quantity received for later stages), organic claim and
mismatch, temperature over limit, breach hidden behind "cold chain OK", implied transit speed, moisture, grade, seller
transactions in 24 h, same-pair trades in 7 days, buyer already in the batch trail, identical invoice seen, seller
account age, cash payment, suspiciously round total.

Training (in-memory records) and serving (SQL rows) both call the same `history_from_records` + `compute_features`, and
`test_feature_parity_between_db_and_in_memory_history` proves both paths give identical features (no train/serve skew).

### 7.3 The hybrid scorer (`ml/scorer.py`)

| Component | Config | Output |
|---|---|---|
| **Random Forest** | 160 trees, depth 14, balanced class weights, **7 classes** (none + 6 fraud types) | `P(fraud) = 1 − P(none)`; fraud-type guess = most probable fraud class |
| **Isolation Forest** | 200 trees, trained on legitimate rows only | anomaly score, normalised so median legit = 0 and the 99.5th percentile = 1 |
| **Rule engine** | 7 named rules (`PRICE_BAND`, `YIELD_IMPOSSIBLE`, `COLD_CHAIN`, `ORGANIC_UNCERTIFIED`, `CIRCULAR`, `DUPLICATE`, `IMPOSSIBLE_TRANSIT`) + `SIGNATURE_INVALID` | `rule_score` = highest severity; each hit has a human-readable reason |
| **Blend** | `risk = 0.60·P(fraud) + 0.25·rule_score + 0.15·anomaly` | CLEAR < 0.40 ≤ FLAGGED < 0.70 ≤ QUARANTINED (thresholds from `.env`) |

### 7.4 Explanations

* **Rules** give sentences such as *"Quantity 48,510 kg is 3.5× this farm's maximum plausible harvest."*
* **SHAP** (`TreeExplainer` on the Random Forest) gives each feature's contribution toward fraud
  (= −SHAP value of the "none" class); the top 6 are stored with the assessment and drawn as bars.
* The score breakdown (ML / rules / anomaly) and the per-type probabilities are shown on the Quarantine page.

### 7.5 Results (model `rf-20261027-0753`, `backend/artifacts/metrics.json`)

| Metric | Held-out test (5,000) | Shifted test (5,000) |
|---|---|---|
| PR-AUC | 0.958 | 0.939 |
| ROC-AUC | 0.972 | 0.975 |
| Precision / Recall @ 0.40 | 0.980 / 0.949 | 0.886 / 0.929 |
| F1 @ 0.40 | 0.964 | 0.907 |

Ablation, F1 (held-out / shifted): Rules only 0.949 / 0.870 · Isolation Forest only 0.356 / — · Logistic Regression
0.943 / — · Random Forest only 0.968 / 0.928 · **Hybrid 0.964 / 0.907**.

Read honestly: the **Random Forest alone has the best F1** on both sets; the hybrid is within 0.004 on the held-out set
and has the best shifted ROC-AUC. Rules alone degrade most on the shifted set (0.949 → 0.870) — hand-written thresholds
are brittle when fraud gets subtler. The hybrid is kept because it attaches **named, human-readable reasons** to every
alert, enforces hard gates (forged signatures) that a model cannot learn, and adds an anomaly signal for patterns
outside the six known types. Per-type recall on the held-out set is 92–98 %. Through the live pipeline (`live_demo.py --benchmark 20`) the
simulator's injections were caught 139 / 140 times, with 0 / 20 false alarms on clean traffic. All data is synthetic, so the numbers are
optimistic; the Model page says so.

### 7.6 Learning loop (`ml/feedback.py`, `ml/training.py`)

* **Approved** quarantine → label `none` (false alarm). **Rejected** → the fraud type the model suspected. Forged-signature
  rejections are skipped (cryptography caught them, not the model).
* The labels are added to the **training split only**, weight ×5, using the feature vector stored with the assessment.
  Test sets are unchanged, so before/after metrics are comparable.
* The new model is written, caches are cleared, and live scoring switches immediately. Every assessment keeps the
  `model_version` that produced it; `artifacts/history.json` tracks every version.

---

## 8. How the ML and the blockchain meet

1. **ML decides the ledger status.** `decision` → status (`CLEAR → COMMITTED`, `FLAGGED → FLAGGED`); quarantined items
   stay off-chain until a human decides (`APPROVED` / `REJECTED`). That status is **inside the record hash**.
2. **Fraud is stopped before commit.** A quarantined transaction is not in any block until it is reviewed, so the chain
   never contains an unreviewed high-risk record.
3. **The verdict is permanent.** Once sealed, an auditor's REJECTED decision cannot be silently flipped without breaking
   the Merkle root (but see §6.1 for flip-and-re-mine).
4. **Cryptography backs up the model.** A forged invoice can look perfectly normal to the ML (in the live demo the
   forged invoice has `P(fraud) = 0.11`); the signature check quarantines it anyway.
5. **The chain feeds the model.** Sealed auditor verdicts become labels for the next model version.

---

## 9. See it yourself

```bash
# backend/ with the app running (launcher option 1, or uvicorn on :8000)
python scripts/live_demo.py            # narrated walkthrough (press Enter between steps); launcher option D
python scripts/live_demo.py --yes --benchmark 10   # straight through + detection rate per fraud type
pytest -q                              # 38 tests + 1 documented xfail (the flip-and-re-mine gap)
```

* API explorer: http://localhost:8000/docs — try `GET /api/chain/verify`, `GET /api/chain/blocks/1`.
* UI: http://localhost:5173 → **Ledger** (Verify, the four attacks, Merkle proofs), **Quarantine**, **Model**.

### Code map

| Concept | Where |
|---|---|
| Record hash, Merkle root/proof, header hash, PoW, seal, verify, re-mine | `backend/app/chain/ledger.py` |
| Ed25519 keys, signing payload, verify | `backend/app/chain/identity.py` |
| Ingest pipeline (assess → persist → enqueue → seal; auditor review) | `backend/app/services.py` |
| Chain API (`/blocks`, `/blocks/{i}`, `/seal`, `/verify`) | `backend/app/routers/chain.py` |
| Demo attacks and reset (`DEMO_MODE` only) | `backend/app/routers/dev.py` |
| Features / rules / scorer / SHAP / training / feedback | `backend/app/ml/` |
| Simulator and hand-written demo scenarios | `backend/app/sim/generator.py`, `backend/app/sim/scenarios.py` |
| In-browser Merkle tree + inclusion proof | `frontend/src/pages/Ledger.tsx` (`MerkleTree`) |
