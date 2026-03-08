# Verity: CRE & AI Workflow Documentation

**Contracts:** `packages/contracts/`  
**Workflows:** `packages/contracts/workflows/`

---

## Quick Summary

Verity uses **Chainlink CRE** to autonomously evaluate teaching sessions and settle payments based on AI-verified quality metrics.

**The Flow:**

1. Learner books session → USDC locked in escrow
2. CRE workflow creates Recall bot → records meeting → generates transcript
3. Teacher requests evaluation → CRE workflow calls Gemini AI
4. AI scores session (0-10000 basepoints) → workflow settles payment on-chain
5. Teachers and learners claim funds based on merit

**Key Innovation:** AI evaluation happens via Chainlink DON, not a centralized server — making it trustless and verifiable.

---

## Hackathon Requirements Met


| Requirement                    | Implementation                                                      |
| ------------------------------ | ------------------------------------------------------------------- |
| **Blockchain + External API**  | EVM (Sepolia) + Recall.ai (recording) + Gemini (evaluation)         |
| **LLM/AI Agent Integration**   | Google Gemini evaluates transcripts against weighted learning goals |
| **CRE Workflow Orchestration** | Two workflows handle initiation → evaluation → settlement           |
| **Simulation/Deployment**      | `cre workflow simulate` for testing; deployable to CRE network      |


**Value Proposition:**

- **Trustless:** AI evaluation via Chainlink DON, verifiable by anyone
- **Fair:** Quadratic scoring rewards high-quality teaching
- **Autonomous:** No human in the loop — session completes → AI evaluates → payment flows

---

## Deployment (Sepolia Testnet)


| Contract              | Address                                      |
| --------------------- | -------------------------------------------- |
| **KXManager**         | `0x6ee50eeafa56f269a8875bb09e2a3ab9608bdf4b` |
| **KXSessionRegistry** | `0x42c105b36825778ca323bf850df6e007b0407dca` |


**Workflow Registry Names:**

- Initiation: `verity-initiation-workflow`
- Settlement: `verity-settlement-workflow`

---

## Workflow Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     CHAINLINK CRE NETWORK                        │
│  ┌─────────────────────┐        ┌─────────────────────────────┐  │
│  │  Initiation WF      │        │  Settlement WF              │  │
│  │  (verity-initiation)│        │  (verity-settlement)          │  │
│  └─────────────────────┘        └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
           ▲                                  ▲
           │                                  │
    EVM Log Trigger                   EVM Log Trigger
           │                                  │
┌──────────┴──────────────────────────────────┴──────────────────┐
│                        EVM (Sepolia)                           │
│  ┌──────────────────┐                 ┌──────────────────────┐  │
│  │  KXManager       │                 │  KXSessionRegistry     │  │
│  │  - Listings      │                 │  - Escrow              │  │
│  │  - Initiation    │                 │  - AI Evaluation       │  │
│  └──────────────────┘                 └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Workflow 1: Initiation

**Trigger:** `SessionRegistrationRequested` event from `KXManager`

**Purpose:** Bridge on-chain request → off-chain session setup → on-chain registration

**File:** `packages/contracts/workflows/initiation-workflow/`

**6-Step Flow:**

```
1. EVM Log Trigger
   └── Event: SessionRegistrationRequested(teacher, learner, amount, meetingLink, listingIndex)

2. Fetch Listing Data
   └── Call KXManager.listings(listingIndex) → get metadata CID

3. Read Session Metadata
   └── IPFS: Fetch unpublishedSessionData via dataCID

4. Create Recall Bot
   └── POST /bot to Recall API → Returns transcriptId

5. Store Session Data
   └── IPFS: Write sessionData (with transcriptId) → Returns CID

6. On-Chain Registration
   └── CRE Report → KXManager._processReport() → Registers in KXSessionRegistry
```

**Key Code:** `main.ts`

```typescript
const onLogTrigger = (runtime, log): string => {
  const { teacher, learner, amount, meetingLink, listingIndex } = decodedLog.args;

  const { dataCID } = getListingByIndex(runtime, { listingIndex });
  const unpublishedSession = readFromStore(runtime, "unpublishedSessionData", dataCID);
  const { transcriptId } = createRecallBot(runtime, meetingLink);
  
  const session = writeToStore(runtime, "sessionData", transcriptId, {
    ...unpublishedSession,
    transcriptId,
  });

  const txHash = initiateSession(runtime, {
    teacher, learner, amount, dataCid: session.data.cid, listingIndex
  });

  return "Initiation Request Processed";
};
```

---

## Workflow 2: Settlement

**Trigger:** `EvaluationRequested` event from `KXSessionRegistry`

**Purpose:** Evaluate session via AI → calculate payment split → settle on-chain

**File:** `packages/contracts/workflows/settlement-workflow/`

**6-Step Flow:**

```
1. EVM Log Trigger
   └── Event: EvaluationRequested(sessionId, dataCID)

2. Fetch Session Data
   └── IPFS: Read sessionData via dataCID

3. Fetch Transcript
   └── Recall API: GET /transcript/{transcriptId}

4. AI Evaluation
   └── Gemini API: Structured scoring (0-10000) on 6 criteria:
       • Topic relevance  • Depth  • Engagement
       • Substance        • Time   • Clarity

5. Store Evaluation Report
   └── IPFS: Write evaluationReport → Returns evidenceCID

6. On-Chain Settlement
   └── CRE Report → KXSessionRegistry._processReport()
       └── Updates status, calculates payout split
```

**AI Evaluation System Prompt:**

The AI acts as a neutral expert. Scoring (0-10000 basepoints):

- 0-2000: Poor/failed
- 2000-5000: Moderate
- 5000-8000: Good
- 8000-10000: Excellent

**Key Code:** `src/ai.ts`

```typescript
export const askAi = (runtime, session, transcript): ApiResponse => {
  const apiKey = runtime.getSecret({ id: "OPENAI_COMPAT_API_KEY" }).result();

  const result = httpClient.sendRequest(
    runtime,
    PostAiData(session, transcript, apiKey.value),
    consensusIdenticalAggregation<ApiResponse>()
  )(runtime.config).result();

  return result;
};
```

**Score Reduction (Weighted + Sigmoid):**

```typescript
function reduceEvaluationToScore(evaluation, session) {
  // Weighted aggregate across goals
  const { sumWS, sumW } = result.reduce((acc, { goal, score }) => {
    const w = weightByKey.get(toKey(goal)) ?? 0;
    return { sumWS: acc.sumWS + w * (score / 1000), sumW: acc.sumW + w };
  }, { sumWS: 0, sumW: 0 });
  
  const aggregate = sumW ? sumWS / sumW : 0;
  
  // Sigmoid multiplier (rewards high scores)
  const tau = 0.3;
  const multiplier = 0.25 + 0.75 / (1 + Math.exp(-(aggregate - 2.5) / tau));
  
  return { aggregate, multiplier };
}
```

---

## Smart Contracts

### `KXManager.sol` — Listings & Initiation

**Key Functions:**


| Function                                                | Purpose                          |
| ------------------------------------------------------- | -------------------------------- |
| `createListing(dataCID, price)`                         | Teacher creates listing          |
| `requestSessionRegistration(listingIndex, meetingLink)` | Learner books, pays USDC         |
| `_processReport(bytes)`                                 | CRE callback — registers session |


**Key Events:**

```solidity
event SessionRegistrationRequested(
  address indexed teacher,
  address indexed learner,
  uint256 amount,
  string meetingLink,
  uint256 listingIndex
);
```

**Flow:**

1. Learner calls `requestSessionRegistration` → pays USDC to escrow
2. Emits `SessionRegistrationRequested` → triggers Initiation Workflow
3. Workflow calls back `_processReport` → registers in `KXSessionRegistry`

---

### `KXSessionRegistry.sol` — Escrow & Settlement

**Session States:**

```solidity
enum Status {
  None,      // 0
  Funded,    // 1 — USDC in escrow, waiting for eval
  Evaluated, // 2 — AI scores received, payout calculated
  Disputed,  // 3 — Under dispute
  Finalized  // 4 — Funds distributed
}
```

**Key Functions:**


| Function                       | Purpose                        |
| ------------------------------ | ------------------------------ |
| `requestEvaluation(sessionId)` | Teacher triggers AI evaluation |
| `_processReport(bytes)`        | CRE callback with scores       |
| `claimTeacher(sessionId)`      | Teacher claims earned portion  |
| `claimLearner(sessionId)`      | Learner claims refund          |


**Settlement Math (Quadratic Scoring):**

```solidity
uint16 effective = (confidenceBps + learningBps) / 2;
uint256 teacherShareBps = (uint256(effective) * uint256(effective)) / 10000;
uint256 teacherAmount = (session.amount * teacherShareBps) / 10000;
```

Teachers earn more for high-confidence, high-learning scores.

---

## Shared Utilities

### Recall.ai Integration (`shared/src/recall.ts`)

```typescript
// Creates bot to join meeting
const { transcriptId } = createRecallBot(runtime, meetingUrl);

// Fetches transcript text
const { raw } = getRecallTranscript(runtime, transcriptId);
```

### IPFS Storage (`shared/src/store.ts`)

```typescript
// Upload to IPFS via Pinata
const { cid } = writeToStore(runtime, "sessionData", id, data);

// Fetch from IPFS
const data = readFromStore(runtime, "sessionData", cid);
```

### Type Schemas (`shared/src/zod.ts`)

```typescript
// AI evaluation response
zAiEvaluationResponse = {
  result: [{ goal: string, score: 0-10000, reasoning: string, improvements: string[] }],
  confidence: 0-10000
}
```

---

## External API Integrations


| Service   | Endpoint                               | Auth                    | Called By           |
| --------- | -------------------------------------- | ----------------------- | ------------------- |
| Recall.ai | `POST /bot`                            | `RECALL_API_KEY`        | Initiation Workflow |
| Recall.ai | `GET /transcript/{id}`                 | `RECALL_API_KEY`        | Settlement Workflow |
| Gemini    | `POST /v1beta/openai/chat/completions` | `OPENAI_COMPAT_API_KEY` | Settlement Workflow |
| Pinata    | `POST /v3/files`                       | `PINATA_API_JWT`        | Both Workflows      |


---

## File Map for Judges

```
packages/contracts/
├── src/
│   ├── KXManager.sol              # Listing & initiation
│   ├── KXSessionRegistry.sol      # Escrow & AI-evaluated settlement
│   └── interfaces/
│       └── ReceiverTemplate.sol   # CRE receiver base
├── workflows/
│   ├── initiation-workflow/
│   │   ├── main.ts                # Workflow entry
│   │   ├── src/evm.ts             # EVM interactions
│   │   └── workflow.yaml          # CRE config
│   ├── settlement-workflow/
│   │   ├── main.ts                # Workflow entry
│   │   ├── src/ai.ts              # Gemini evaluation
│   │   ├── src/evm.ts             # EVM interactions
│   │   └── workflow.yaml          # CRE config
│   └── shared/
│       └── src/
│           ├── recall.ts          # Recall.ai API
│           ├── store.ts           # IPFS storage
│           └── zod.ts             # Type schemas
```

---

## Simulation Commands

```bash
# Simulate initiation workflow
cre workflow simulate ./packages/contracts/workflows/initiation-workflow --target=test

# Simulate settlement workflow
cre workflow simulate ./packages/contracts/workflows/settlement-workflow --target=test
```

**Config:** `packages/contracts/workflows/project.yaml`

```yaml
test:
  rpcs:
    - chain-name: ethereum-testnet-sepolia
      url: https://0xrpc.io/sep
```

---

*Last updated: March 8, 2026*