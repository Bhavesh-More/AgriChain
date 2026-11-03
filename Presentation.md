# AgriChain Sentinel — Presentation Slides

Slide-by-slide content for the PPT. Each slide has **on-slide text** (keep it short), a **visual** suggestion, and
**speaker notes** (what to say). Screenshots are in `docs/screenshots/`; the colour scheme is near-black with gold
`#E8C15A` (see `docs/ui-references/reference-dark-gold.png`). Suggested length: 18 slides ≈ 12 minutes + live demo.

---

## Slide 1 — Title

**AgriChain Sentinel**
ML-Driven Blockchain Framework for Detecting Fraudulent Agricultural Supply-Chain Transactions

Team names · Course: Blockchain · VIT · Semester 1

- **Visual:** `docs/screenshots/dashboard.png` as a dimmed background.
- **Speaker notes:** "We built a system that checks every agricultural trade for fraud *before* it is written to a
  blockchain, and makes the recorded history tamper-evident."

---

## Slide 2 — The problem

- Produce changes hands 4 times: **Farmer → Cooperative → Processor → Distributor → Retailer**
- Every hand-over is an invoice — and an opportunity for fraud:
  - inflated prices, **phantom quantities** a farm could never grow
  - fake **organic** labels, hidden **cold-chain** breaches
  - **circular trading** rings, **duplicate** invoices, invoices in someone else's name
- Records live in databases that insiders can quietly edit

- **Visual:** a simple 5-node chain of icons with red warning markers on the arrows.
- **Speaker notes:** Two problems: detecting fraud as it happens, and proving later that the record of what happened
  was not changed. ML answers the first, blockchain the second.

---

## Slide 3 — Objectives

1. Score each transaction for fraud **before** it is committed (ML + rules)
2. **Explain** every alert in plain language
3. Hold high-risk transactions for a **human auditor**
4. Record every outcome on a **tamper-evident ledger** (hash chain, Merkle trees, proof-of-work)
5. Authenticate every invoice with **digital signatures**
6. Let the model **learn** from auditor verdicts

- **Speaker notes:** Success criteria from our design doc: per-type recall ≥ 0.80, PR-AUC ≥ 0.85, and chain
  verification that names the exact broken block.

---

## Slide 4 — Solution overview

```
Invoice ─▶ ML gate ─▶ CLEAR ─────────▶ mempool ─▶ block (Merkle + PoW) ─▶ chain
              │      FLAGGED ───────▶ mempool (with flag)
              └────▶ QUARANTINED ─▶ auditor ─▶ APPROVED / REJECTED ─▶ block
                                          └──▶ training labels ─▶ retrain
```

- Risk = **0.60 · ML probability + 0.25 · rule score + 0.15 · anomaly score**
- < 0.40 clear · 0.40–0.70 flagged · ≥ 0.70 quarantined

- **Speaker notes:** The key design idea: fraud is stopped *before* commit, and the auditor's decision itself is
  sealed on-chain.

---

## Slide 5 — Architecture & tech stack

| Layer | Technology |
|---|---|
| Frontend | React 19 + TypeScript, Vite, Tailwind, Recharts, TanStack Query |
| API | FastAPI (Python 3.11+), auto Swagger docs |
| ML | scikit-learn (Random Forest, Isolation Forest, Logistic Regression), SHAP, pandas |
| Ledger | Own module: SHA-256, Merkle tree, proof-of-work, Ed25519 (`cryptography`) |
| Storage | SQLite (WAL mode) via SQLModel |
| Graphs | networkx (cycle detection) |

- **Visual:** the architecture diagram from `Project-Map.md` §3.
- **Speaker notes:** Everything runs locally on a laptop — no cloud, no internet, no wallet. One-click launchers for
  Windows and macOS.

---

## Slide 6 — Data: a simulated agricultural supply chain

- No labelled public dataset exists → we **simulate** one and train from scratch
- 140 participants · 6 states · 6 crops (3 perishable with temperature limits)
- Daily "mandi" reference prices with seasonal variation
- 20,000 transactions, 7 % fraud + 5,000 **shifted** (harder) test transactions
- Realism guards: 4 % legitimate outliers, label noise, overlapping distributions

- **Speaker notes:** Synthetic data gives ground truth for *which* fraud each case is — needed for per-type evaluation.
  We say openly that synthetic scores are optimistic; the shifted set tests robustness.

---

## Slide 7 — Six fraud types

| Fraud | How it looks | Rule |
|---|---|---|
| Price manipulation | price 35–120 % off the market | `PRICE_BAND` |
| Phantom quantity | more than the land can grow | `YIELD_IMPOSSIBLE` |
| Organic label fraud | "organic", no certificate | `ORGANIC_UNCERTIFIED` |
| Concealed cold-chain breach | too warm, declared "cold chain OK" | `COLD_CHAIN` |
| Circular trading | sold back to an earlier holder | `CIRCULAR` |
| Duplicate invoice | same invoice submitted twice | `DUPLICATE` |
| + Forged signature | signed with someone else's key | `SIGNATURE_INVALID` |

- **Speaker notes:** The seventh case is not learned — it is caught by cryptography.

---

## Slide 8 — The ML pipeline

- **21 features**, computed in *one* function for training and serving (no train/serve skew)
  - price deviation, quantity vs capacity, temperature over limit, buyer already in trail, duplicate seen, …
- **Random Forest** (160 trees, 7 classes) → probability of fraud + fraud type
- **Isolation Forest** (trained on legitimate data only) → "this looks unusual"
- **7 named rules** → known patterns with exact reasons
- **Blend** → one risk score → decision

- **Visual:** three boxes (RF / IF / Rules) merging into a gauge.
- **Speaker notes:** Rules are precise for known patterns; the forest generalises across noisy combinations; the
  isolation forest catches the unknown.

---

## Slide 9 — Explainable alerts

- Every alert shows:
  - **Rule reasons**: "Quantity 48,510 kg is 3.5× this farm's maximum plausible harvest."
  - **SHAP bars**: which features pushed the score toward fraud
  - **Score breakdown**: ML · rules · anomaly
  - **Fraud-type probabilities**
- Deterministic — no LLM, same input → same explanation

- **Visual:** `docs/screenshots/check.png`.
- **Speaker notes:** Auditors must justify decisions; a black-box score is not enough.

---

## Slide 10 — The blockchain ledger

- **Record hash** = SHA-256(21 fields + seller signature + **ledger status**)
- **Merkle root** of the 5 records in a block
- **Block hash** = SHA-256(index | timestamp | prev_hash | merkle_root | nonce | difficulty)
- **Proof-of-work**: hash must start with `000` (~4,096 tries)
- **Chain**: every block stores the previous block's hash

```
[#60 hash 00011c…] ◀── prev ── [#61 hash 0007d0…  root ee7c7d…  nonce 9,347]
```

- **Speaker notes:** Because the *status* is inside the hash, "un-rejecting" a fraud is as detectable as editing a
  quantity.

---

## Slide 11 — Merkle trees & inclusion proofs

```
              root = H(A + B)
             /             \
     A = H(H01 + H23)    B = H(H44 + H44)
       /       \            |
     H01       H23         H44 = H(h4 + h4)
     / \       / \          |
    h0  h1    h2  h3        h4
```

- Proof that `h2` is in the block: just **h3, H01, B** — not all 5 records
- The Ledger page **rebuilds the tree in the browser** (Web Crypto) and highlights the proof

- **Speaker notes:** This is how light clients verify a single transaction without downloading the whole block.

---

## Slide 12 — Digital signatures (Ed25519)

- Every participant has a key pair; the public key is registered
- The **seller signs** each invoice
- **At ingest:** wrong signature → hard quarantine, whatever the ML says
  - live demo: forged invoice had ML fraud probability **0.11** — caught anyway
- **At verification:** every sealed record's signature is re-checked
  - an attacker who edits data **and re-mines the whole chain** is still caught

- **Speaker notes:** Hashes prove data didn't change; signatures prove *who* created it. In the demo the server holds
  the keys so the simulator can sign — in production each party holds its own.

---

## Slide 13 — Verification & attack demo

| Attack | Result |
|---|---|
| Edit a sealed quantity | ✗ Merkle root mismatch at block #61 |
| Flip a REJECTED verdict to COMMITTED | ✗ Merkle root mismatch |
| Edit a block header | ✗ header hash mismatch |
| Edit + recompute all hashes + re-mine | ✗ seller signature invalid |
| Delete a block mid-chain | ✗ broken previous-hash link |

- Verification names **the first broken block, the reason and the transaction**

- **Visual:** `docs/screenshots/ledger-tamper.png` and `docs/screenshots/ledger-rewrite-attack.png`.

---

## Slide 14 — Auditor workflow & learning loop

- Quarantine queue sorted by risk, with full explanation
- **Approve / Reject** with a note → verdict sealed into a block → CSV audit trail
- Verdicts become **training labels** (approved = false alarm, rejected = confirmed type)
- **Retrain** button: before/after metrics, version history, live model hot-swapped

- **Visual:** `docs/screenshots/model-learning-loop.png`.

---

## Slide 15 — Results

| Metric | Held-out | Shifted (harder) |
|---|---|---|
| PR-AUC | **0.958** | 0.939 |
| ROC-AUC | 0.972 | 0.975 |
| F1 | **0.964** | 0.907 |
| Per-type recall | 92–98 % | 81–97 % |

- Live benchmark through the running pipeline (20 injections per type): **139 / 140 frauds caught**
  (one phantom quantity slipped through), **0 / 20** false alarms on clean traffic
- Ablation (F1, held-out / shifted): Rules 0.949 / 0.870 · RF 0.968 / 0.928 · **Hybrid 0.964 / 0.907**

- **Speaker notes:** Be honest: RF alone scores slightly higher F1; the hybrid keeps near-equal accuracy *and* adds
  rule reasons, a cryptographic hard gate and an anomaly signal. Rules alone break down on the shifted set. Synthetic
  data → optimistic numbers.

---

## Slide 16 — The application

- **Dashboard** — KPIs, live risk stream, fraud by type, region × crop heatmap, mini-chain
- **Check** — what-if form with example loaders for every fraud type
- **Quarantine** — auditor review
- **Ledger** — chain, verify, attacks, Merkle proofs
- **Trace** — farm-to-retail journey of a batch
- **Network** — trade graph, circular rings, trust scores
- **Model** — evaluation, threshold slider, retrain

- **Visual:** 2×2 grid of screenshots.

---

## Slide 17 — Live demo (script)

1. Dashboard → **Simulator on** → **Inject fraud: Phantom quantity**
2. Quarantine → read the reason → **Reject**
3. Ledger → **Verify** ✓ → **Flip a verdict** ✗ → **Rewrite & re-mine** ✗ → **Reset**
4. Trace / Network → circular trading ring
5. Model → **Retrain with auditor feedback**

Terminal alternative: `python scripts/live_demo.py` (launcher option **D**) narrates the same story step by step.

---

## Slide 18 — Testing & quality

- **38 backend tests** (pytest) + **11 frontend tests** (Vitest)
  - every rule, every ledger attack, signatures, feature parity, learning loop, API flows, demo storyline
- ruff, Prettier, TypeScript strict — all clean
- Reproducible: fixed seeds, committed model, one-click setup and reset

---

## Slide 19 — Limitations

- **Single node** — no peer-to-peer network or multi-party consensus; PoW is a rewrite cost, not consensus
- Signing keys are **server-held** for the demo
- Not caught yet: verdict flip **plus** re-mine; deleting the newest block; edits to off-chain risk scores and notes
- **Synthetic data** — results are optimistic

- **Speaker notes:** Being upfront about these is part of the engineering; each has a known fix (next slide).

---

## Slide 20 — Future work

- **Validator signature on every block** → closes flip-and-re-mine
- **Anchor block hashes on Ethereum** (local Hardhat node) → closes tail truncation
- Multi-validator (2-of-3 auditor) approval
- Benchmark on a real public dataset
- Map view, Hindi/Marathi UI

---

## Slide 21 — Conclusion & Q&A

- ML stops fraud **before** it reaches the ledger, and explains why
- The blockchain makes every decision **permanent and verifiable**
- Signatures make even a re-mined rewrite of the data detectable

**Thank you — questions?**

---

## Appendix — likely viva questions

| Question | Short answer |
|---|---|
| Is this a real blockchain? | It implements every tamper-evidence mechanism (hashes, Merkle roots, links, PoW, signatures, verification) on a single node. There is no P2P consensus. |
| Why proof-of-work on one node? | It makes rewriting costly and demonstrates mining; the signature layer is what actually defeats a determined rewrite. |
| Why put the verdict in the hash? | So that un-rejecting a fraud is detectable, not just editing data. |
| What does the Merkle tree give you? | One root summarising all records, and short inclusion proofs (log₂ n hashes). |
| Why not a pretrained model? | None exists for tabular agri-fraud; training on our simulator gives per-type labels. |
| Why hybrid if RF alone has higher F1? | Near-equal accuracy plus named reasons, a cryptographic hard gate and an anomaly detector for unknown patterns. |
| How do you avoid the model memorising the generator? | Benign outliers, label noise, overlapping distributions, and a shifted test set. |
| What if an attacker controls the keys? | Then they can forge — that's why real deployments give each party its own key and anchor hashes externally. |
