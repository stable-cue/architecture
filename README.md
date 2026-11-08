# Stablecue Technical Architecture

**Stellar Anchor Platform integration track · Tranches 1–3 · Stellar-specific build plan**

---

## 1. Purpose and Scope

This document defines the technical architecture for Stablecue's integration with the Stellar Anchor Platform. It covers what will be built across Tranches 1 to 3 only. Business model, marketing and other non-technical deliverables are addressed elsewhere in the submission.

| | |
|---|---|
| **Integration target** | Stellar Anchor Platform (SEP-10, SEP-12, SEP-24 and SEP-31 servers) |
| **Network** | Stellar Testnet for Tranche 3 deployment, with a defined Mainnet path |
| **Assets** | USDC and USDT0 on day one, EURC as a follow-on. USDT0 is the omnichain form of USDT on Stellar |

---

## 2. Integration Target: Stellar Anchor Platform

Stablecue integrates with the Stellar Anchor Platform as the regulated building block that executes fiat-to-stablecoin and stablecoin-to-fiat transactions on behalf of institutional clients.

The Anchor Platform provides SEP-compliant endpoints, run by regulated partners (anchors), which Stablecue orchestrates through its API layer. In SEP terms Stablecue acts as the client application toward each partner. Stablecue does not hold custody, does not hold Stellar secret keys, does not sign Stellar transactions and does not hold regulatory licences. All fiat acceptance, conversion and on-chain delivery are executed by the partner within its own compliance framework.

### SEPs used

| SEP | Purpose | Stablecue integration point | Scope |
|-----|---------|-----------------------------|-------|
| SEP-1 | `stellar.toml` | Partner discovery: `WEB_AUTH_ENDPOINT`, `KYC_SERVER`, `TRANSFER_SERVER_SEP0024` and the partner `SIGNING_KEY` | Tranches 1–3 |
| SEP-10 | Stellar Web Authentication | Obtain a JWT bound to the client's Stellar account. The challenge is signed in the client's own wallet | Tranches 1–3 |
| SEP-12 | KYC API | Read the partner's KYC status for the client and submit customer data where required | Tranches 1–3 |
| SEP-24 | Hosted Deposit and Withdrawal | Fiat-to-stablecoin and stablecoin-to-fiat ramp orchestration through the partner's interactive flow | Tranches 1–3 |
| SEP-31 | Cross-Border Payments | Multi-jurisdiction payment routing. Adapter interface designed in Tranche 1, not built in Tranches 2–3 | Design only |
| SEP-38 | Anchor RFQ API | Firm quotes for non-equivalent pairs such as EUR to USDC | Follow-on |

Spec versions (SEP-24 is currently at 3.x, SEP-12 at 1.x) are pinned in Tranche 1.

---

## 3. System Architecture Overview

Stablecue acts as an orchestration layer between institutional B2B clients and one or more Anchor Platform partner instances.

![Stablecue architecture overview: clients, orchestration engine, Anchor Platform partners and the Stellar network](diagrams/architecture-overview.svg)

### 3.1 Components

| Component | Responsibility |
|-----------|----------------|
| **API Gateway** | OAuth2 authentication, per-client rate limiting and a key-value store for keys and limits |
| **Routing Engine** | Selects the Anchor Platform partner by jurisdiction, volume and rail availability |
| **Intent State Machine** | Persists every intent transition (see section 7) |
| **Event Log** | Append-only audit trail linking intent ID, partner transaction ID and Stellar transaction hash |
| **Stellar Address Validator** | Validates destination and source addresses before any execution: StrKey format, memo, account exists, trustline present |
| **Anchor Platform Client** | SEP-10, SEP-12 and SEP-24 adapters, with the SEP-31 interface designed |
| **Reconciliation** | Signed partner callbacks as the primary sync, with a heartbeat poller as fallback |
| **Expiry and Compensation** | Intent TTL, retry with exponential backoff and refund tracking |

### 3.2 Authentication and signing model

- **SEP-10.** Stablecue requests the challenge from the partner and relays it to the client. The client signs it in their own wallet and Stablecue submits it to obtain a JWT. Stablecue only ever holds the JWT, never a Stellar secret key. For clients that share one Stellar account across several users, the challenge carries an ID memo, or an `M…` muxed address is used.
- **On-chain payments.** On a deposit the partner submits the payment to the client's bound address. On a withdrawal the client's wallet submits the payment to the partner.
- **Callbacks.** Partner callbacks are accepted only after their signature is verified against the partner `SIGNING_KEY` (section 6.3).
- **Ledger access.** Stablecue reads the Stellar network (Horizon or Stellar RPC) to verify transactions. It never submits transactions.
- **Account types.** Classic `G…` and muxed `M…` accounts are supported. Soroban contract accounts (`C…`) would require SEP-45 and are out of scope.

---

## 4. Architecture by Tranche

### Tranche 1: Architecture and Foundation (Approved)

**Objective**: Define the Stablecue engine architecture, design the Anchor Platform integration contracts and provision the development environment.

**Technical deliverables**:

1. **Engine architecture and API design**
   - Stablecue API gateway specification (OAuth2, rate limiting, request and response schemas)
   - Anchor Platform client interface design (SEP-10, SEP-12, SEP-24 and SEP-31 adapters)
   - Intent state machine definition (section 7)
   - Event log schema (append-only, read-only audit trail)

2. **Partner integration groundwork**
   - Anchor Platform SEP endpoint mapping per partner, driven by each partner's `stellar.toml`
   - Partner credential management (API keys, callback verification keys, JWT handling)
   - Callback endpoint design for Anchor Platform status updates

3. **Dev environment and hosting setup**
   - Isolated dev and staging environments for Anchor Platform sandbox testing
   - CI/CD pipeline for engine builds
   - Logging and error tracking infrastructure

**Stellar-specific decisions made in Tranche 1**:
- Which Anchor Platform partner(s) to integrate with first
- SEP version alignment (SEP-10, SEP-12, SEP-24 and SEP-31 versions in use)
- Stellar address validation rules (StrKey, memo handling, trustline checks)
- Asset identifier configuration for USDC, USDT0 and EURC on Stellar Testnet

---

### Tranche 2: MVP Core Engine Build

**Objective**: Build the core Stablecue orchestration engine with Anchor Platform partner API integration.

**Technical deliverables**:

1. **Core engine build: intent flow, state machine, API**
   - Intent creation endpoint: the client submits a fiat amount, the stablecoin asset (USDC, USDT0 or EURC) and a Stellar address
   - Authentication middleware: validates the client API key and permissions
   - Routing logic: selects the Anchor Platform partner by jurisdiction, volume and rail availability
   - State machine: persists intent transitions to the append-only event log
   - Client API: status queries, intent cancellation and transaction history

2. **Partner API integration**
   - SEP-10 client adapter: obtains the challenge, relays it to the client for signing and exchanges it for a JWT
   - SEP-12 client adapter: queries the partner's KYC status and triggers verification when it is missing or expired
   - SEP-24 client adapter: starts interactive deposits and withdrawals, receives signed callbacks and polls transaction status
   - Error handling: maps SEP-24 statuses and error responses to Stablecue intent states
   - Retry logic: exponential backoff for transient Anchor Platform failures

3. **API documentation for client integrations**
   - OpenAPI specification for the Stablecue client-facing endpoints
   - Integration guide for B2B clients: binding Stellar addresses, submitting intents, handling callbacks and presenting the partner's hosted flow

**Stellar-specific decisions made in Tranche 2**:
- SEP-10 flow: how the challenge is relayed so that the client signs in their own wallet
- SEP-12 flow: how Stablecue checks the partner's KYC status before allowing execution
- SEP-24 flow: how Stablecue starts the interactive flow, hands the hosted URL to the client and monitors execution
- Address validation: pre-binding and checking Stellar addresses before any execution

---

### Tranche 3: Stellar Integration (Testnet)

**Objective**: Complete the Stellar-specific integration (USDC, USDT0 and EURC delivery, Stellar address binding, on-chain verification and reliability logic), deployed and tested on Stellar Testnet.

**Technical deliverables**:

1. **Stellar integration**
   - USDC, USDT0 and EURC delivery orchestration: Stablecue starts the SEP-24 flow and the partner converts fiat and delivers the asset on Stellar
   - Stellar address binding: clients pre-bind Stellar addresses and the engine validates format, account existence and trustline before execution
   - On-chain verification: the partner releases the asset to the client's bound address and the engine verifies the transaction on the ledger before completing the intent
   - Transaction linking: Stablecue intent ID, partner transaction ID and Stellar transaction hash (complete paper trail)
   - Asset configuration: identifiers for USDC, USDT0 and EURC on Stellar Testnet, with EURC enabled after day one

2. **Reliability logic**
   - Reconciliation: callback-driven status sync from the partner plus a background heartbeat poller as fallback
   - Fallback checks: periodic synchronisation of internal state with the partner's actual transaction status
   - Refund tracking: when execution fails after funds were received, the partner's refund is tracked to completion
   - Intent expiry: TTL on all intents, with automatic expiry and client notification for incomplete transactions

3. **API documentation**
   - Updated OpenAPI specification including Stellar-specific endpoints (address binding, on-chain status)
   - Callback documentation for Anchor Platform status updates

4. **Stellar testnet testing and monitoring**
   - End-to-end testnet deployment with the Anchor Platform sandbox
   - USDC, USDT0 and EURC delivery test flows
   - Reconciliation and refund path testing
   - Monitoring dashboards for intent success and failure rates

**Stellar-specific decisions made in Tranche 3**:
- Which Stellar assets (USDC, USDT0 and EURC) and their identifiers: issuer account for classic assets, contract ID for Soroban tokens
- Memo handling for transaction identification
- On-chain confirmation rule before marking an intent COMPLETED (successful transaction in a closed ledger)
- Trustline policy: block intents until the trustline exists, or accept claimable balances where the partner supports them
- Testnet and Mainnet configuration separation

---

## 5. Stellar Transaction Flows

In both flows the partner's SEP-24 interactive flow collects KYC details and bank information in a hosted web app. Stablecue hands the hosted URL to its client, so the client's user completes that step directly with the regulated partner. Stablecue tracks the transaction through signed callbacks.

### 5.1 Deposit (fiat to USDC, USDT0 or EURC on Stellar)

![Deposit sequence: client, Stablecue, Anchor Platform partner and Stellar network](diagrams/deposit-flow.svg)

Notes:
- The destination account must exist and hold a trustline for the asset. Stablecue checks this before creating the intent. Where a trustline is missing and the partner supports it, delivery can use a claimable balance. Otherwise the partner reports `pending_trust`.
- Stablecue marks the intent COMPLETED only after it has verified the Stellar transaction itself, not on the partner's callback alone.

### 5.2 Withdrawal (USDC, USDT0 or EURC on Stellar to fiat)

![Withdrawal sequence: client, Stablecue, Anchor Platform partner and Stellar network](diagrams/withdrawal-flow.svg)

Notes:
- The partner returns its Stellar account and memo once the hosted flow is complete. Stablecue relays them to the client, whose wallet makes the payment. Stablecue never signs it.
- Stablecue passes a `refund_memo` so that any refund of the on-chain payment can be matched to the intent.

---

## 6. API Contracts

### 6.1 Client to Stablecue

| Endpoint | Purpose |
|----------|---------|
| `POST /v1/intents` | Create a deposit or withdrawal intent |
| `GET /v1/intents/{intent_id}` | Get intent status |
| `POST /v1/intents/{intent_id}/cancel` | Cancel an intent that has not started execution |
| `GET /v1/intents` | Transaction history |

**Create Intent**:
```
POST /v1/intents
{
  "direction": "deposit | withdrawal",
  "fiat_amount": "decimal",
  "fiat_currency": "EUR | USD",
  "stellar_asset": "USDC | USDT0 | EURC",
  "destination_stellar_address": "G… (deposit)",
  "source_stellar_address": "G… (withdrawal)",
  "memo": "optional"
}
```
Returns `intent_id`, `state` and `expires_at`. Once the partner returns it, `GET /v1/intents/{intent_id}` also returns the `interactive_url` for the hosted flow.

**Get Intent Status**:
```
GET /v1/intents/{intent_id}
```
Returns the current state, the partner transaction ID, the Stellar transaction hash (when available) and timestamps.

### 6.2 Stablecue to Anchor Platform partner (SEP calls)

All endpoint locations come from the partner's `stellar.toml` (SEP-1).

**SEP-10: obtain a JWT**:
```
GET {WEB_AUTH_ENDPOINT}?account=G…[&memo=<id memo>]
→ { "transaction": "<challenge XDR>", "network_passphrase": "…" }

POST {WEB_AUTH_ENDPOINT}
{ "transaction": "<challenge signed by the client wallet>" }
→ { "token": "<JWT>" }
```
The memo is optional, must be of type `id` and can only be used with a `G…` account.

**SEP-12: check KYC status**:
```
GET {KYC_SERVER}/customer?account=G…&type=<type>
Authorization: Bearer <JWT>
→ { "id": "…", "status": "ACCEPTED | PROCESSING | NEEDS_INFO | REJECTED", "fields": { … } }
```
`NEEDS_INFO` lists the missing fields, which are submitted with `PUT /customer`, after which the status is checked again.

**SEP-24: start a deposit** (a withdrawal uses `/transactions/withdraw/interactive`):
```
POST {TRANSFER_SERVER_SEP0024}/transactions/deposit/interactive
Authorization: Bearer <JWT>
{
  "asset_code": "USDC",
  "account": "G…",
  "amount": "100000.00",
  "claimable_balance_supported": true
}
→ { "type": "interactive_customer_info_needed", "url": "https://…", "id": "…" }
```
For withdrawals, `refund_memo` and `refund_memo_type` are also sent. Before handing the `url` to the client, Stablecue adds an `on_change_callback` parameter pointing at its callback endpoint.

**SEP-24: poll a transaction** (fallback to callbacks):
```
GET {TRANSFER_SERVER_SEP0024}/transaction?id=<partner transaction id>
Authorization: Bearer <JWT>
```

### 6.3 Partner to Stablecue (signed callbacks)

```
POST /webhooks/sep24/{partner_id}
Signature: t=<unix timestamp>, s=<base64 signature>

{
  "transaction": {
    "id": "partner transaction id",
    "kind": "deposit | withdrawal",
    "status": "pending_stellar",
    "amount_in": "…",
    "amount_out": "…",
    "stellar_transaction_id": "<Stellar tx hash, once available>",
    "refunds": { … }
  }
}
```
The body is the SEP-24 transaction object. Stablecue accepts a callback only if all of these hold:
1. The `Signature` header is present and parses.
2. The timestamp is fresh (at most one to two minutes old).
3. The signature over `<timestamp>.<Stablecue host>.<body>` verifies against the partner `SIGNING_KEY` from its `stellar.toml`.

### 6.4 SEP-24 status mapping

| SEP-24 status | Stablecue state | Action |
|---------------|-----------------|--------|
| `incomplete` | EXECUTING | Client is in the hosted flow |
| `pending_user_transfer_start` | EXECUTING | Deposit: waiting for the client's fiat transfer. Withdrawal: relay payment instructions to the client |
| `pending_user_transfer_complete` | EXECUTING | Withdrawal: payment received, fiat ready for pick-up |
| `pending_anchor`, `pending_external`, `pending_stellar` | EXECUTING | Partner is processing |
| `pending_trust`, `pending_user` | EXECUTING | Alert the client: trustline missing or client action required |
| `completed` | ON_CHAIN_RELEASED, then COMPLETED | Verify the Stellar transaction on the ledger, then complete |
| `refunded` | REFUNDED | Record the `refunds` details |
| `expired` | EXPIRED | Notify the client |
| `error`, `no_market`, `too_small`, `too_large` | FAILED | Notify the client |

---

## 7. State Machine and Event Log

![Intent state machine: INTENT_CREATED through COMPLETED, with FAILED, REFUNDED and EXPIRED](diagrams/intent-state-machine.svg)

### Intent states

| State | Description | Trigger |
|-------|-------------|---------|
| INTENT_CREATED | Client request accepted and validated | `POST /v1/intents` |
| AUTHENTICATED | Client account authenticated with the partner | SEP-10 JWT issued |
| VERIFIED | KYC confirmed with the partner | SEP-12 status `ACCEPTED` |
| EXECUTING | Partner is processing the fiat and on-chain legs | SEP-24 transaction submitted |
| ON_CHAIN_RELEASED | On-chain leg settled: for a deposit the asset reached the client address, for a withdrawal the client payment reached the partner | Partner reports `completed` with a Stellar transaction hash |
| COMPLETED | Transaction verified in a closed ledger | Stablecue ledger check |
| FAILED | Execution failed | Partner error status or KYC `REJECTED` |
| REFUNDED | Partner refunded funds it had received | SEP-24 `refunded` status |
| EXPIRED | TTL reached without completion | Internal timer |

Stellar closes a ledger roughly every five seconds and finality is deterministic, so there is no confirmation depth to configure: a successful transaction in a closed ledger is final.

### Event log schema (append-only, immutable)

```json
{
  "event_id": "uuid",
  "intent_id": "uuid",
  "from_state": "string",
  "to_state": "string",
  "trigger": "string",
  "anchor_transaction_id": "optional string",
  "stellar_transaction_hash": "optional string",
  "timestamp": "ISO8601",
  "metadata": "optional object"
}
```

---

## 8. Reliability and Reconciliation

### 8.1 Callback-driven primary sync

Partners send signed `on_change_callback` messages whenever a transaction's status changes. Stablecue verifies each one (section 6.3) and updates the intent state machine accordingly.

### 8.2 Heartbeat poller fallback

If callbacks are missed or delayed, a background poller queries the partner's SEP-24 `GET /transaction` endpoint at configurable intervals to detect state divergence. Transient failures are retried with exponential backoff.

### 8.3 Compensation

SEP-24 has no client-initiated refund call: refunds are issued by the partner and reported through the `refunded` status and the `refunds` object. Stablecue's role is to create the conditions for a clean refund and to track it.

| Failure point | Partner behaviour | Stablecue action |
|---------------|-------------------|------------------|
| Deposit: fiat received, asset not delivered | Partner refunds the fiat off-chain | Track to REFUNDED and notify the client |
| Withdrawal: client payment received, fiat payout fails | Partner refunds the on-chain payment using the supplied `refund_memo` | Track to REFUNDED and notify the client |
| Execution fails before any funds move | None | Mark FAILED and notify the client |
| Asset delivered on-chain, later disputed | Clawback only if the issuer has enabled it | Not relied upon; handled manually with the partner |

### 8.4 Intent TTL

All intents have an expiration timestamp, never later than the partner's `user_action_required_by` when one is provided. Intents not reaching COMPLETED within the window are automatically expired and the client is notified.

---

## 9. Onchain Footprint and Attribution

For SCF metric tracking, every transaction routed through Stablecue and an Anchor Platform partner produces:

1. **Stellar transaction hash**: recorded in the event log and linked to the intent ID
2. **Asset**: USDC, USDT0 or EURC on Stellar (issuer account or contract ID configured per network)
3. **Client address**: the bound destination (deposit) or source (withdrawal) Stellar address, and memo where used
4. **Anchor Platform transaction ID**: links the off-chain fiat leg to the on-chain stablecoin leg

These data points allow attribution of TPV and volume to Stablecue's integration for final-tranche metric reporting.

---

## 10. Security Boundaries

- **No custody**: Stablecue never holds client funds.
- **No keys, no signing**: Stablecue holds no Stellar secret keys and signs no Stellar transactions. SEP-10 challenges are signed by the client's own wallet, and on-chain payments are submitted by the partner (deposit) or the client (withdrawal).
- **Read-only ledger access**: Stablecue only reads the Stellar network to verify transactions.
- **Authenticated callbacks**: partner callbacks are accepted only after signature and freshness checks.
- **Minimal data retention**: KYC and KYB data stays with the Anchor Platform partner. Stablecue stores only verification status references.
- **OAuth2**: client API authentication with scoped permissions.
- **Rate limiting**: per-client API rate limits to prevent abuse.
