# Verity

**Proof-of-Value infrastructure for the global consultation market.**

> Don't just consult. Verify with Verity.

Verity replaces blind trust with **Verifiable Outcomes**. It ensures experts are paid for their impact and attendees only pay for what they actually gain.

## Overview

Verity is a decentralized knowledge exchange platform that connects hosts and attendees for sessions with fair, AI-powered evaluation and on-chain settlement:

1. **Host creates listing** — Offer to teach a topic for a set price in USDC
2. **Attendee books session** — Pays upfront into escrow
3. **Session occurs off-chain** — Video call with Recall.ai recording
4. **AI evaluates quality** — Gemini scores transcript against weighted goals
5. **Smart contracts settle** — Quadratic payout formula distributes funds
6. **Host and attendee claim** — Based on merit

**Key innovation:** AI evaluation runs via Chainlink DON, not a centralized server — trustless and verifiable.

## Core Features

- **Smart Escrow** — USDC locked before sessions; no payment risk
- **Autonomous Witness** — Recall.ai bots join and record sessions
- **Merit-Based Settlement** — Quadratic scoring; transparent payouts
- **Chainlink CRE** — Two workflows: initiation and settlement

## Tech Stack

| Layer | Stack |
|-------|-------|
| **Contracts** | Solidity, KXManager, KXSessionRegistry |
| **Workflows** | Chainlink CRE, TypeScript, Recall.ai, Gemini, IPFS |
| **Web** | React 19, TanStack Router, Hono, Bun, Drizzle, Privy |

## Quick Start

```bash
cd packages/web
bun install
bun run dev
```

Visit the demo flow: `/demo/onboarding` → `/demo/host/dashboard` → `/demo/host/create/details` → `/demo/learner/join` → `/demo/session/live`

## Sepolia Deployment

| Contract | Address |
|----------|---------|
| USDC | `0xb243a36d2cb3937b40043050bf4f7d36795322db` |
| KXManager | `0x42c105b36825778ca323bf850df6e007b0407dca` |
| KXSessionRegistry | `0xB9f475C996A61c8BC9b2E72B7Df3de3017Dd3C76` |

## Documentation

Full documentation is available at **[docs.hetairoi.xyz](https://docs.hetairoi.xyz)**:

| Topic | Link |
|-------|------|
| **Overview** | [docs.hetairoi.xyz/overview](https://docs.hetairoi.xyz/overview) |
| **Contracts and CRE** | [docs.hetairoi.xyz/contracts-and-cre](https://docs.hetairoi.xyz/contracts-and-cre) |
| **Server and Client** | [docs.hetairoi.xyz/server-and-client](https://docs.hetairoi.xyz/server-and-client) |
| **Product and User Flow** | [docs.hetairoi.xyz/product-and-user-flow](https://docs.hetairoi.xyz/product-and-user-flow) |
| **Testing CRE Workflows** | [docs.hetairoi.xyz/cre-testing](https://docs.hetairoi.xyz/cre-testing) |
| **Quickstart** | [docs.hetairoi.xyz/quickstart](https://docs.hetairoi.xyz/quickstart) |
| **Deployment** | [docs.hetairoi.xyz/deployment](https://docs.hetairoi.xyz/deployment) |
| **API Reference** | [docs.hetairoi.xyz/api-reference/introduction](https://docs.hetairoi.xyz/api-reference/introduction) |

## License

See [LICENSE](LICENSE) for details.
