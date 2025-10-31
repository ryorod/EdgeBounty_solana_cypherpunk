# Edge Bounty — PRD (36‑Hour MVP)

## 1) Goal: Definition of Done (within 36 hours)
**Objective:** Ship a working MVP that demonstrates smartphone‑native micro‑inference with redundant verification and on‑chain settlement on Solana devnet, captured in a 60–90s demo video and a 10‑slide pitch.

**Done when all are true:**
- Android app (React Native) runs **CLIP ViT‑B/32** image embedding (224×224) **on‑device** and submits a signed execution proof.
- Router/Verifier service assigns jobs to **r=3** workers, reaches consensus with **cosine ≥ 0.98 for 2/3 pairs**, and triggers payout.
- Settlement script pays **USDC‑DEV (SPL demo token)** to worker pubkeys on **Solana devnet** and writes a **Result meta URI** to chain (Memo or PDA).
- Dashboard (Next.js) can **create a job**, display **job progress**, **consensus outcome**, **TX links**, and **Result meta URI**.
- **E2E cold‑start** from zero data to paid result succeeds at least once on camera.
- Reproducible **README** (runbook) and submission materials (video + deck) are complete.

---

## 2) Personas
### Worker (Android phone owner)
- **Context:** Phone is often charging and idle at home/office.
- **Goals:** Earn passive micro‑rewards without privacy risk or battery wear.
- **Constraints:** Limited compute, thermal throttling, data caps.
- **Triggers:** “Charging‑only” mode; minimal configuration; clear earnings feedback.

### Requester (Job submitter/operator)
- **Context:** Needs cheap, verifiable embeddings at scale for prototyping.
- **Goals:** Post small jobs, see status/metrics, pay only for agreed results.
- **Constraints:** Strict latency/price targets; wants transparent verification.
- **Triggers:** Simple dashboard to create jobs and review outcomes quickly.

---

## 3) Use Cases (Top 3)
1. **Single Image Embedding Job**
   - *Flow:* Requester creates a job with `model_hash`, `input_hash`, `reward`, `r=3` → workers claim → local CLIP embedding → submit → consensus → payout.
   - *Success:* TX shows USDC‑DEV transfer; dashboard shows consensus + IPFS link.

2. **Canary Validation**
   - *Flow:* Server injects a known I/O pair as a canary (~5% of jobs). Worker submits result → Verifier checks closeness to known output → adjusts worker score or flags.
   - *Success:* Canary mismatch lowers score; repeated failure stops assignment.

3. **Timeout & Reassignment**
   - *Flow:* One worker stalls; server times out and reassigns to a new worker to preserve r=3 redundancy.
   - *Success:* Job still reaches consensus within deadline; dashboard marks reassignment.

---

## 4) Requirements (Must / Should / Won’t)
### 4.1 Functional
| Priority | Requirement | Notes |
|---|---|---|
| **Must** | Android‑only RN app generates CLIP embeddings on device | `onnxruntime-react-native`; 224×224; L2‑normalized |
| **Must** | Device gates: **charging only**, battery ≥ 40%, temp ≤ 38 °C | NativeModule to read status |
| **Must** | Job assignment & submission API | `/jobs`, `/claim`, `/submit`, `/results` |
| **Must** | Redundant consensus **r=3**, **2/3 pairs cosine ≥ 0.98** | Threshold configurable via env |
| **Must** | Canary jobs ~5% | Server‑side insertion |
| **Must** | Payout in **USDC‑DEV** on devnet; write **Memo/PDA** with Result meta URI | web3.js + SPL token |
| **Must** | Dashboard to create job & monitor progress/TX | Next.js App Router |
| **Should** | Simple worker score (0–100) & basic throttling | Penalize canary failures |
| **Should** | Timeout & reassignment | Avoid stuck jobs |
| **Won’t** | iOS support, voice tasks, state compression NFTs, Token‑22 hooks | Out of scope for 36h |

### 4.2 Non‑Functional
| Priority | Requirement | Target |
|---|---|---|
| **Must** | Privacy: no raw inputs off device | Only embedding + signed proof |
| **Must** | Median embedding latency | ≤ 3s on mid‑tier Android |
| **Must** | E2E time (claim→payout) | ≤ 10s on devnet demo path |
| **Must** | Reliability (consensus success) | ≥ 95% on demo set |
| **Should** | Server resilience | Restart‑safe with SQLite WAL |
| **Won’t** | Full KYC/AML, production security hardening | Post‑MVP |

---

## 5) Design Trade‑offs (Why no compressed NFTs / Token‑22)
- **Compressed NFTs (state compression)**  
  - *Pros:* Extremely cheap for logging many results.  
  - *Cons (MVP):* Tooling & Anchor plumbing add integration time; debugging overhead.  
  - **Decision:** **Skip** for 36h. Record a single **IPFS Result meta URI** via Memo/PDA; upgrade later to compressed NFTs.

- **Token‑22 (transfer hooks/KYC/region rules)**  
  - *Pros:* Production‑grade compliance knobs.  
  - *Cons (MVP):* Hook program, account wiring, and policy testing increase scope.  
  - **Decision:** **Skip** for 36h. Use a simple **SPL demo mint (USDC‑DEV)**, document Token‑22 migration path.

- **On‑device preprocessing simplification**  
  - *Pros:* Faster delivery; reduces native surface area.  
  - *Cons:* Requires controlled input (224×224).  
  - **Decision:** Accept controlled input for demo; note real preprocessing as next step.

---

## 6) Acceptance Criteria (E2E Observable Checklist)
**Job Creation**
- [ ] Dashboard POST `/jobs` returns `{id}` and displays in “Jobs” list.
- [ ] `input_hash`, `model_hash`, `redundancy=3` visible in DB or API.

**Worker Execution**
- [ ] RN app shows **charging‑only toggle**, current battery %, temp °C.
- [ ] When gates satisfied, **Claim → Run → Submit** completes without crash.
- [ ] App produces **embedding (float32[N])** and **signed exec_proof** (proof CBOR or fields).

**Verification & Consensus**
- [ ] Server stores **3 submissions** for the job.
- [ ] Pairwise cosine checks computed; **2/3 ≥ 0.98** triggers **consensus** status.
- [ ] Canary job path penalizes wrong outputs (score decreases).

**Settlement & Recording**
- [ ] **USDC‑DEV** transfer TX signature is returned and visible on Explorer (devnet).
- [ ] **Result meta** JSON is reachable (IPFS or HTTP fallback) and referenced in chain Memo/PDA.
- [ ] Dashboard shows consensus outcome, **ipfs_uri**, and **tx_sig**.

**Cold‑start Demo**
- [ ] New environment from blank DB can run E2E once, on camera, without manual DB edits.

---

## 7) Risks & Mitigations (Top 5)
1. **ONNX model/runtime quirks on certain Android devices**  
   - *Impact:* App crash or incorrect embeddings.  
   - *Mitigation:* Force CPU EP; pin model/versions; add basic smoke test (dimension & norm checks); fallback to smaller model if needed.

2. **Preprocessing pipeline mismatch (224×224 RGB normalization)**  
   - *Impact:* Embeddings drift; consensus fails.  
   - *Mitigation:* Use controlled 224×224 demo inputs; include unit check on mean/std; document real preprocessing as post‑MVP.

3. **Consensus threshold too strict for device variability**  
   - *Impact:* Jobs fail to reach consensus.  
   - *Mitigation:* Make threshold env‑driven (e.g., 0.975–0.99); log cosine trio; set auto‑retry/reassign on first failure.

4. **Devnet congestion or token account issues**  
   - *Impact:* Payout delays or TX failures.  
   - *Mitigation:* Pre‑fund and pre‑create ATAs; implement retry/backoff; show graceful “payout pending” state and re‑submit TX on server resume.

5. **IPFS availability (or API keys) during demo**  
   - *Impact:* Result meta not accessible; acceptance blocked.  
   - *Mitigation:* Implement **HTTP fallback**: save JSON under `/public` and reference that URI in Memo; add health check before demo.

---

## Appendix: Interfaces (summary)
- **API:** `/jobs (POST,GET)`, `/claim (POST)`, `/submit (POST)`, `/results (GET)`  
- **Consensus:** r=3, cosine ≥ 0.98 for 2/3 pairs, canary ≈ 5%  
- **Payout:** SPL demo mint (**USDC‑DEV**), devnet, Memo with `ipfs_uri`

> Scope is intentionally minimal to de‑risk 36‑hour delivery. All items marked “Future” in trade‑offs are queued as the first post‑submission tasks.
