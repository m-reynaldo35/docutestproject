# AuraSign Pilot — Project Roadmap

**Goal:** Build a working MVP pilot to demonstrate blockchain-enhanced e-signatures to DocuSign.
The core pitch: DocuSign handles identity, KYC/AML, and the signing UX (unchanged).
AuraSign adds a cryptographic attestation layer on top — immutable notarization, automated escrow
settlement, and a quantum-resistant downloadable receipt.

**Target:** A live, testnet demo that shows one complete deal flow end-to-end.

---

## Phase 0 — Project Setup
**Status:** Complete

- [x] Create project directory `docutestproject/`
- [x] Initialize git + push to GitHub (`m-reynaldo35/docutestproject`)
- [ ] Scaffold directory structure (contracts, src, scripts)
- [ ] Install dependencies (algosdk, tealscript, typescript, dotenv)
- [ ] Configure `.env` with testnet AlgoNode endpoints
- [ ] Generate two testnet Algorand accounts and fund via faucet
- [ ] Acquire testnet USDC (ASA `10458941`) for escrow testing

---

## Phase 1 — Module A: The Auto-Notary
**Goal:** A utility function that hashes a PDF and anchors the hash on Algorand testnet.

### Tasks
- [ ] Create `src/moduleA-notary.ts`
  - Accept a SHA-256 hash string + DocuSign envelope ID
  - Build a JSON note payload (`protocol`, `hash`, `envelope`, `timestamp`)
  - Submit a 0-ALGO transaction to testnet with the note
  - Return `{ txId, round, notePayload }`
- [ ] Create `src/documentHasher.ts`
  - Read a PDF file from disk
  - Generate SHA-256 hash using Node crypto (server-side for pilot; WebCrypto for browser production)
- [ ] Write a test script `scripts/test-notary.ts`
  - Hash a sample PDF
  - Submit to testnet
  - Log the confirmed txId and AlgoExplorer link
- [ ] Verify on https://testnet.explorer.perawallet.app

**Acceptance:** A confirmed testnet transaction whose note field contains the document hash,
visible and permanently readable on-chain.

---

## Phase 2 — Module B: The Settler (TEALScript Escrow)
**Goal:** A smart contract that holds USDC and releases it when both parties sign.

### Tasks
- [ ] Write `contracts/SettleEscrow.algo.ts` (TEALScript)
  - `createApplication(signerA, signerB, asset, amount, envelopeId)`
  - `bootstrap(mbrPay)` — fund MBR + opt-in to USDC ASA
  - `executeAgreement()` — each signer calls once; releases USDC on second call
- [ ] Compile with `npx tealscript` → output to `contracts/artifacts/`
- [ ] Write `src/moduleB-deploy.ts`
  - Deploy the contract to testnet
  - Call `bootstrap()` with 0.2 ALGO MBR payment
  - Deposit USDC into the contract address
  - Signer A calls `executeAgreement()`
  - Signer B calls `executeAgreement()`
  - Verify USDC lands in Signer A's account
- [ ] Write `scripts/test-escrow.ts` end-to-end test

**Acceptance:** Two separate accounts each call `executeAgreement()` and USDC auto-transfers
to Signer A without any manual step. Visible on testnet explorer.

---

## Phase 3 — Module C: The Quantum Receipt
**Goal:** Query Algorand for the State Proof covering the notary transaction and package
everything into a downloadable JSON receipt.

### Tasks
- [ ] Write `src/moduleC-receipt.ts`
  - Accept a confirmed `txId`
  - Fetch transaction details from Indexer
  - Fetch block header from Algod
  - Calculate the State Proof round (`ceil(confirmedRound / 256) * 256`)
  - Fetch State Proof from Algod `/v2/stateproofs/{round}`
  - Fetch transaction Merkle proof from `/v2/blocks/{round}/transactions/{txid}/proof`
  - Assemble into a single `QuantumReceipt` JSON object
- [ ] Write `src/saveReceipt.ts` — save receipt as `receipt-{txId}.json`
- [ ] Document the verification steps in `docs/receipt-verification.md`
  - What each field proves
  - How an auditor verifies the receipt offline
  - State Proof timing note (available ~17 min after confirmation)

**Acceptance:** A saved JSON file that independently proves:
1. This exact document hash was submitted at this timestamp
2. It was included in Algorand block N
3. That block is covered by a post-quantum State Proof

---

## Phase 4 — Demo Integration Script
**Goal:** One command runs the full pilot flow end-to-end.

### Tasks
- [ ] Write `scripts/demo.ts`
  1. Load accounts from `.env`
  2. Hash `sample-contract.pdf`
  3. Notarize (Module A) → log `txId`
  4. Deploy escrow with 1 USDC (Module B deploy)
  5. Signer A calls `executeAgreement()`
  6. Signer B calls `executeAgreement()` → USDC releases
  7. Build Quantum Receipt (Module C) → save `receipt.json`
  8. Print summary table with all txIds and explorer links
- [ ] Add a `sample-contract.pdf` to `assets/` for demo use
- [ ] Add npm script: `npm run demo`

**Acceptance:** `npm run demo` produces a terminal summary and a `receipt.json` with no manual steps.

---

## Phase 5 — DocuSign Webhook Bridge (Pilot Extension)
**Goal:** Connect DocuSign's real API so the notarization triggers automatically on envelope completion.

### Tasks
- [ ] Register a DocuSign developer account at https://developers.docusign.com
- [ ] Create a DocuSign app and obtain OAuth 2.0 credentials
- [ ] Write `src/docusignClient.ts`
  - OAuth 2.0 token exchange
  - Download completed envelope PDF (`/v2.1/accounts/{id}/envelopes/{id}/documents/combined`)
- [ ] Write `src/webhookHandler.ts`
  - Express endpoint to receive DocuSign Connect events
  - Validate HMAC-SHA256 signature
  - On `status: "completed"` → auto-trigger Module A notarization
- [ ] Wire webhook → Module A → Module B settlement trigger
- [ ] Test with a real DocuSign sandbox envelope

**Acceptance:** Send a DocuSign envelope in sandbox, both parties sign, webhook fires, document
is notarized on-chain within 30 seconds, receipt generated automatically.

---

## Phase 6 — Pilot Demo UI (Optional, for presentation quality)
**Goal:** A minimal Next.js page to make the demo visually compelling for DocuSign stakeholders.

### Tasks
- [ ] Scaffold `apps/web` with Next.js 15 + TypeScript + Tailwind
- [ ] Single-page demo flow:
  - Upload PDF → show SHA-256 hash
  - "Notarize" button → submit to Algorand → show txId + explorer link
  - "View Quantum Receipt" → render the JSON in a readable card
- [ ] Escrow status panel showing approval state for both signers
- [ ] Liquid Auth QR integration (additive signing step on top of DocuSign)

---

## Key Reference Information

| Item | Value |
|---|---|
| Testnet Algod | `https://testnet-api.algonode.cloud` |
| Testnet Indexer | `https://testnet-idx.algonode.cloud` |
| Testnet Explorer | `https://testnet.explorer.perawallet.app` |
| Testnet ALGO Faucet | `https://bank.testnet.algorand.network` |
| Testnet USDC ASA ID | `10458941` |
| State Proof Interval | `256 rounds (~17 minutes)` |
| Note field limit | `1024 bytes` |
| Escrow MBR funding | `0.2 ALGO (0.1 base + 0.1 ASA opt-in)` |

---

## Architecture Decisions (Locked)

- **Hashing is client/script-side.** SHA-256 is computed from raw PDF bytes before upload. The server re-validates but is never in the signing trust path.
- **DocuSign identity is not replaced.** Liquid Auth is additive. DocuSign's KYC/AML remains the legal identity anchor.
- **Testnet throughout.** No real funds until DocuSign pilot agreement is signed.
- **2-of-2 escrow for MVP.** M-of-N threshold is Phase 2 of a production contract.
- **Note field schema is versioned.** `"protocol": "aurasign/1"` so future indexer queries can filter by version.
