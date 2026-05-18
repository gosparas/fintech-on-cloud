# Glossary

> Reference companion to the Fintech on Cloud - Payment Series

---

## A

**AML (Anti-Money Laundering)**
A set of regulatory checks that screen payment instructions against risk rules and behavioural patterns before processing. In SCT Inst, AML must be pre-cached to meet the 10-second deadline — a cold AML lookup is too slow for a synchronous real-time flow.

**Apigee X**
Google Cloud's API gateway. Handles authentication, rate limiting, and load balancing for all inbound payment requests. The single entry point before any business logic runs. In SCT Inst, policies are kept minimal to avoid adding latency inside the 10-second window.

**AISP (Account Information Service Provider)** 
A PSD2-licensed third party that reads account data (balances, transactions) from a customer's bank via the bank's XS2A API. Read-only, consent-driven, requires token refresh cycles. Consent valid for 90 days under PSD2, proposed 180 days under PSD3.

**ASPSP (Account Servicing Payment Service Provider)** 
The customer's bank. Under PSD2, ASPSPs must expose dedicated APIs for licensed TPPs. The party that executes SCA and processes the actual payment or account data request.

---

## B

**Bank Adapter Layer**
A per-bank translation layer that converts internal payment objects into the ISO 20022 message format required by each specific bank or clearing network. Abstracts connectivity differences (SFTP, EBICS, REST, SWIFT FileAct) behind a uniform internal interface.

**Berlin Group NextGenPSD2** 
The dominant API specification for PSD2 access-to-account interfaces across continental Europe. Implemented by 75%+ of European banks. Defines endpoints for payment initiation, consent management, and account information. Not a single implementation — each bank interprets the spec differently.

**BigQuery**
Google Cloud's serverless data warehouse. Stores PII-stripped, enriched transaction events for analytics, SLA monitoring, and reconciliation reporting. Fed by Dataflow streaming pipelines.

---

## C

**camt.054**
An ISO 20022 Credit Notification message sent by the bank to confirm that funds have been credited to the beneficiary account. In SCT, this is the definitive settlement confirmation — not the pain.002 acknowledgement.

**camt.056**
An ISO 20022 message used to request cancellation of a credit transfer that has already been submitted to clearing. Used in the SCT recall / compensation flow alongside pacs.007.

**CANC (Cancelled)**
ISO 20022 status indicating the payment was cancelled before execution, typically because the PSU cancelled during SCA. Terminal.

**Circuit Breaker**
A resilience pattern that stops retrying a failing downstream call after a threshold of consecutive failures, preventing cascade failures. Sits inside the Retry Mechanism alongside exponential backoff. When tripped, traffic is routed to the Manual Review Queue rather than continuing to hammer a degraded service.

**Clearing**
The process of routing and validating a payment instruction between the sending bank and the receiving bank via a central infrastructure (CSM). Distinct from settlement — clearing confirms the instruction is valid and routable; settlement moves the actual funds.

**Cloud Monitoring**
GCP's observability service. Tracks latency, error rates, DLQ depth, and SLA compliance. Fires PagerDuty alerts on threshold breaches — including the p99 latency alert critical for SCT Inst and DLQ non-zero depth alerts.

**Cloud Pub/Sub**
GCP's managed messaging service. Used to decouple payment phases asynchronously — state changes, ledger updates, and webhook triggers are all published as events rather than called directly.

**Cloud Run**
GCP's serverless container runtime. Suitable for SCT workloads but requires minimum instance configuration (≥1) for SCT Inst — cold starts of ~800ms are incompatible with the 10-second EPC deadline.

**Cloud Spanner**
GCP's globally distributed, strongly consistent relational database. Used as the primary state store for payment state machine transitions and idempotency keys. Strong reads prevent stale state from being served under concurrent requests.

**Cloud Tasks**
GCP's managed task queue. Used in SCT to schedule and retry pain.002 and camt.054 status polls against the bank — offloads polling from the main processing path.

**Compensation Flow**
The process triggered when a payment needs to be unwound after it has been submitted or settled. In SCT this is a recall (pacs.007 / camt.056). In SCT Inst this is a recon-gated ledger reversal. Compensation always requires reconciliation confirmation before the ledger is touched — never triggered directly by an incoming failure message.

**CSM (Clearing and Settlement Mechanism)**
The interbank infrastructure that processes and routes payment instructions between banks. EBA STEP2 is the primary CSM for SCT. RT1 and TIPS are the CSMs for SCT Inst.

---

## D

**Dataflow**
GCP's managed Apache Beam service. Runs streaming pipelines that strip PII from transaction events before writing to BigQuery, and applies ISO enrichment and windowed aggregates for analytics.

**Dead Letter Queue (DLQ)**
A queue that captures messages or tasks that have failed processing after the maximum number of retries. In SCT it captures failed Pub/Sub messages and Cloud Tasks. In SCT Inst it captures timeout events. DLQ depth is a first-class operational SLI — a non-zero DLQ always triggers an alert.

**Double-Entry Bookkeeping**
An accounting principle where every transaction creates two ledger entries — a debit on one account and a credit on another. The Ledger Service enforces this to ensure the ledger always balances and every reversal is fully traceable.

---

## E

**EBA STEP2**
The European Banking Authority's pan-European ACH (Automated Clearing House) system. Processes SCT instructions in up to 4 batch cycles per business day, with D+1 settlement via TARGET2. Also operates RT1, a separate real-time clearing service for SCT Inst.

**eIDAS Certificate** 
Electronic identification certificate issued by a QTSP, required for all TPP-to-bank communication. Two types: QWAC (for mTLS authentication) and QSealC (for request signing). The TPP's licence number is embedded in the certificate. Expiry without rotation causes total TPP outage.

**EPC (European Payments Council)**
The organisation that defines and maintains the SEPA payment scheme rulebooks — including SCT, SCT Inst, and SDD. Sets the 10-second settlement deadline for SCT Inst and mandates automatic rejection and full refund if the deadline is missed.

**Exponential Backoff**
A retry strategy where each successive retry waits exponentially longer than the previous one (e.g. 1s, 2s, 4s, 8s…). Prevents retry storms from overwhelming a recovering downstream service. Combined with the Circuit Breaker pattern in the Recovery Layer.

---

## I

**IBAN (International Bank Account Number)**
A standardised format for identifying bank accounts across SEPA countries. Used as the primary account identifier in all SCT and SCT Inst instructions. Validated via MOD-97 checksum before any instruction reaches the bank adapter.

**Idempotency**
The property that processing the same request multiple times produces the same result as processing it once. Implemented by checking a deduplication key (hash of payment parameters) before any state transition — never after. Critical in payment systems to prevent double debits on retried requests. The idempotency key is written atomically with the initial state transition in a single Spanner transaction.

**ISO 20022**
The international standard for electronic financial messaging. SEPA payments use ISO 20022 XML message formats. Key messages: pain.001 and pain.002 for SCT initiation and status; pacs.008 and pacs.002 for SCT Inst clearing.

---

## L

**Ledger Reversal**
The accounting entry that unwinds a previously confirmed debit or credit in the double-entry ledger. Always recon-gated — never triggered directly by an incoming failure message. Idempotent by design to prevent double-reversals.

**Ledger Service**
The internal service responsible for writing and managing double-entry ledger records. Transitions entries from pending (on initiation) to confirmed (on ACCP) and processes reversals only after reconciliation confirmation.

---

## P

**p99 Latency**
The 99th percentile latency — the response time below which 99% of requests fall. The relevant metric for SCT Inst SLA sizing. Designing for average (p50) latency is insufficient — the system must meet the 10-second EPC deadline at p99 under peak load, not average load.

**pacs.002**
An ISO 20022 Payment Status Report message. In SCT Inst, this is the synchronous response from the creditor bank confirming ACCP (accepted) or RJCT (rejected) — must arrive within 10 seconds. In SCT, it arrives asynchronously as part of the STEP2 clearing cycle.

**pacs.007**
An ISO 20022 Payment Return message. Used by the creditor in SCT to initiate a recall of a previously submitted payment — the creditor-initiated compensation flow. Always paired with camt.056.

**pacs.008**
An ISO 20022 Financial Institution Credit Transfer message. The core payment instruction used in SCT Inst, sent directly to RT1 or TIPS for real-time clearing. Must be constructed and dispatched within the 10-second window.

**pain.001**
An ISO 20022 Customer Credit Transfer Initiation message. The payment instruction submitted by the merchant or PSP to their bank in the SCT batch flow.

**pain.002**
An ISO 20022 Customer Payment Status Report message. The bank's response to a pain.001, confirming acceptance (ACCP) or rejection (RJCT) of the payment instruction. In SCT, polled via Cloud Tasks. In SCT Inst, returned synchronously as pacs.002.

**PagerDuty**
An incident management platform integrated with Cloud Monitoring. Receives alerts on critical payment system events — including SCT Inst p99 SLA breaches, DLQ non-zero depth, and elevated timeout rates.

**PII (Personally Identifiable Information)**
Data that can identify an individual — names, IBANs, account numbers. Stripped from transaction events by the Dataflow pipeline before writing to BigQuery, in compliance with GDPR.

**PSP (Payment Service Provider)**
A company that provides payment processing services to merchants. The architecture in this blog is described from the PSP perspective — sitting between the merchant app and the underlying banking infrastructure.

**PISP (Payment Initiation Service Provider)** 
A PSD2-licensed third party that initiates payments from a customer's bank account. Higher-risk than AISP — each payment triggers SCA. Uses `POST /v1/payments/sepa-credit-transfers` under NextGenPSD2.

**PSD2 (Revised Payment Services Directive)** 
EU directive (2015/2366) mandating that banks expose APIs for licensed third parties. Effective January 2018, with RTS on SCA following September 2019. Being replaced by PSD3 and PSR.

**PSD3 / PSR (Payment Services Regulation)** 
Successor to PSD2. Political agreement reached November 2025, application expected late 2027. Key changes: 180-day consent validity, mandatory Verification of Payee, APP fraud liability on PSPs, fallback interfaces eliminated, permission dashboards required.

**PSU (Payment Service User)** 
The end customer whose bank account is being accessed. Authenticates via SCA at the bank during the redirect flow. May abandon the SCA redirect chain at any point.

---
## Q

**QTSP (Qualified Trust Service Provider)** The certificate authority that issues eIDAS certificates (e.g. D-Trust, Digicert, Entrust). Renewal can take days — emergency reissue is possible but not guaranteed within 24 hours.

**QWAC (Qualified Website Authentication Certificate)** eIDAS certificate used for mTLS between the TPP and the bank API. Authenticates the TPP's server identity. The bank validates the certificate against the public TPP register.

**QSealC (Qualified Certificate for Electronic Seals)** eIDAS certificate used to digitally sign API requests. Proves the request originated from the licensed TPP and was not tampered with in transit. Some banks require it on every request; others only on payment initiation.

---

## R

**Reconciliation**
The process of matching the internal ledger state against the bank's records (statements, camt.054, pain.002 files) to confirm all transactions are accounted for. Serves as the mandatory gate before any ledger reversal — a payment must be confirmed reconciled before a reversal entry is written.

**Reconciliation Engine**
The internal service that performs ledger-vs-bank-statement matching. Resolves ambiguous states — particularly SCT Inst TIMEOUT, where the payment outcome is unknown — and gates all ledger reversals. Feeds unresolved mismatches to Vertex AI for classification.

**RT1**
EBA's real-time gross settlement system for SCT Inst. Processes instant payment instructions 24/7/365, routing pacs.008 messages between participating banks and returning pacs.002 confirmations within the 10-second EPC window.

---

## S

**Sanctions Screening**
A compliance check that validates payment parties (sender, beneficiary) against sanctioned entity lists (OFAC, EU, UN). Applied before any payment instruction reaches the bank adapter. In SCT Inst, results must be pre-cached — a live sanctions lookup cannot complete within the latency budget.

**SCA (Strong Customer Authentication)** 
PSD2 mandate requiring two of three factors (knowledge, possession, inherence) for payment authorisation. In the redirect approach, the customer leaves the TPP's app, authenticates at the bank, and is redirected back with an authorisation code. The redirect callback is the most fragile point in the entire flow.

**SCT (SEPA Credit Transfer)**
The standard SEPA payment scheme for euro credit transfers. Batch-based, D+1 settlement, asynchronous by design. Uses pain.001 / pain.002 message formats and clears via EBA STEP2. Suitable for payroll, supplier payments, and high-volume batch payouts.

**SCT Inst (SEPA Instant Credit Transfer)**
The real-time SEPA payment scheme. Synchronous, 10-second hard EPC deadline, 24/7/365. Uses pacs.008 / pacs.002 message formats and clears via RT1 or TIPS. Maximum transaction value €100,000 (as of 2024). Requires always-on infrastructure with strict latency SLAs at every hop.

**SEPA (Single Euro Payments Area)**
A European Union initiative that harmonised euro-denominated payments across 36 countries, including all EU member states plus Iceland, Norway, Liechtenstein, Switzerland, and the UK. Standardised formats (IBAN, ISO 20022), processing rules, and timelines — making cross-border euro payments equivalent to domestic transfers.

**Settlement**
The actual movement of funds between banks, completing a payment. Distinct from clearing. In SCT, settlement occurs on D+1 via TARGET2. In SCT Inst, settlement is immediate — within the 10-second window.

**State Machine**
A model that tracks a payment through every stage of its lifecycle with defined states and explicit, idempotent transitions between them. The authoritative source of truth for payment status. All transitions are persisted to Spanner before acknowledgement to ensure no state is lost on crash or restart.

---

## T

**TARGET2**
The European Central Bank's real-time gross settlement system for interbank euro payments. Processes final cash settlement for both SCT (D+1 batches) and SCT Inst (immediate, 24/7).

**TIMEOUT**
In SCT Inst, the state reached when no pacs.002 response is received within the 10-second EPC window. A timeout is **not** the same as a failure — the bank may or may not have processed the payment. The outcome is unknown until reconciliation resolves it. The ledger reversal is provisional and must never be finalised until reconciliation confirms FAILED. If reconciliation confirms ACCP, the provisional reversal is voided.

**TIPS (TARGET Instant Payment Settlement)**
The European Central Bank's infrastructure for SCT Inst settlement, operated by the Eurosystem. An alternative to RT1. Provides 24/7/365 real-time settlement in central bank money across SEPA-participating banks.

**TPP (Third-Party Provider)** 
Collective term for AISPs and PISPs — licensed non-bank entities that access customer bank accounts via PSD2 APIs. Must hold a licence from a national competent authority and possess valid eIDAS certificates.

**TPP-Redirect-URI** 
The callback URL registered with each bank where the customer is redirected after completing SCA. If this endpoint is unavailable during a redirect, the payment is lost. Must survive deployments with zero downtime.

---

## V–W

**Vertex AI (Gemini)**
GCP's AI platform. Used in the reconciliation layer to classify ambiguous RJCT reason codes, explain ledger mismatches in plain language for the ops team, and reduce manual investigation time on unresolved payment failures.

**Webhook**
An HTTP callback sent to a merchant or PSP's registered endpoint when a payment event occurs (completed, failed, reversed). In SCT Inst, webhooks fire asynchronously after the synchronous pacs.002 result has already been returned to the caller — serving disconnected or polling clients that cannot hold a synchronous connection open.

---
## X

**XS2A (Access to Account)** 
The technical term for the PSD2-mandated interface that banks must expose to TPPs. The Berlin Group NextGenPSD2 framework is the most widely adopted XS2A specification.

## Quick reference — message formats

| Message | Standard | Used in | Purpose |
|---|---|---|---|
| pain.001 | ISO 20022 | SCT | Payment initiation (merchant → bank) |
| pain.002 | ISO 20022 | SCT | Payment status report (bank → merchant) |
| pacs.008 | ISO 20022 | SCT Inst | Credit transfer instruction (bank → RT1/TIPS) |
| pacs.002 | ISO 20022 | SCT Inst | Payment status response (RT1/TIPS → bank) |
| pacs.007 | ISO 20022 | SCT | Payment recall / return (creditor-initiated) |
| camt.054 | ISO 20022 | SCT | Credit notification / settlement confirmation |
| camt.056 | ISO 20022 | SCT | Cancellation request |

---

## Quick reference — error codes

| Code | Scheme | Meaning |
|---|---|---|
| AC01 | SCT | Incorrect account number — IBAN failed checksum |
| AM04 | SCT | Insufficient funds |
| AM18 | SCT Inst | Amount exceeds €100,000 limit |
| AG01 | SCT Inst | Account not eligible for instant payments |
| DS04 | SCT Inst | Rejected by creditor bank |
| FF01 | SCT | Invalid file format — pain.001 failed XSD validation |
| NARR | SCT Inst | Narrative — free-text, typically AML/compliance |
| RR01 | SCT | Missing account or identification details |

---
