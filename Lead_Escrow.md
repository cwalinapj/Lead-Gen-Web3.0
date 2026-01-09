Lead‑Escrow – On‑chain Lead Marketplace with Dynamic‑Multiplier Payouts
Token‑priced, verified local leads – pay only for quality.

A minimal‑viable‑product (MVP) that lets buyers fund lead‑generation campaigns on Solana, workers submit leads, and an off‑chain oracle scores them. Payouts are calculated on‑chain with a 0.00 – 1.00× multiplier (basis‑points) that is fully deterministic and transparent.

Table of Contents
Project Overview
Key Features
Architecture Diagram
On‑chain Program (Anchor)
Off‑chain Oracle / Ingestion Service
Integrations
Getting Started – Local Development
Deploying to Devnet / Mainnet
Testing
Usage Walk‑through (CLI example)
Security & Privacy Considerations
Configuration / Environment Variables
Contributing
License
Project Overview
Component	Responsibility
Solana on‑chain program (lead_escrow)	Holds campaign escrow, records claims, validates Oracle signatures, calculates base × score_bps / 10 000 payouts, guarantees atomicity & transparency.
Oracle / Ingestion Service	Receives leads from Twilio, HubSpot, Fieldd, Zapier, etc., runs fraud/geo checks, computes the score_bps based on CRM outcomes, and submits attest_and_pay_lead.
Integrations	Twilio SMS/voice webhooks, HubSpot workflow webhooks, Zapier “catch‑hook” & “new booking” triggers, Fieldd booking confirmations.
Frontend / CLI (optional)	Simple scripts that let a buyer create a campaign, a worker claim a campaign, and anybody trigger a lead submission.
All PII stays off‑chain – only hashes and content‑addressed URIs are persisted on Solana.

Key Features
Budgeted Campaigns – buyers lock a token budget into a PDA escrow.
Dynamic Multipliers – a basis‑point (score_bps) supplied by the Oracle produces a payout in the range [0, base_payout].
Single‑Claim / Exclusive Workers – a campaign can be claimed by a worker (or opened for open‑submissions later).
One‑Lead‑One‑Pay Guard – a LeadReceipt PDA guarantees each lead_hash is paid at most once.
Transparent, On‑chain Accounting – all budget, spent, and payout values are publicly viewable.
Plug‑and‑Play Integrations – Twilio, HubSpot, Zapier, Fieldd, custom webhook sources.
Anti‑Fraud Checks – datacenter/VPN detection, IP‑geo radius verification, and CRM‑derived outcome scoring.
No PII On‑chain – only lead_hash (e.g., sha256(email+timestamp)) and optional result_uri_hash (IPFS/Arweave pointer) are stored.
Architecture Diagram

graph LR
    subgraph On‑Chain (Solana)
        C[LeadEscrow Program] --> Escrow[Escrow Token Account (PDA)]
        C --> Campaign[Campaign PDA]
        C --> LeadRec[LeadReceipt PDA]
    end

    subgraph Off‑Chain
        O[Oracle Service] -->|attest_and_pay_lead| C
        T[Twilio Webhook] --> O
        H[HubSpot Webhook] --> O
        Z[Zapier / Fieldd] --> O
    end

    Buyer -->|create_campaign| C
    Worker -->|claim_campaign| C
    Worker -->|submit lead (email/form/SMS)| O
    O -->|score_bps + hashes| C
    C -->|payout token| Worker


On‑chain Program (Anchor)
The on‑chain logic lives in programs/lead_escrow/src/lib.rs.
Core instructions

Instruction	Params	Description
initialize_config	oracle: Pubkey	Sets the Oracle signer that may call attest_and_pay_lead.
create_campaign	CreateCampaignArgs	Buyer funds an escrow, defines budget, base payout, radius, deadline, geo‑center, and other requirements.
claim_campaign	ClaimCampaignArgs	Worker (exclusive) claims the campaign, locking the worker field.
attest_and_pay_lead	AttestLeadArgs (lead_hash, score_bps, result_uri_hash)	Oracle validates the lead, computes payout = base_payout * score_bps / 10_000, transfers tokens from escrow → worker, updates spent.

Important constants

pub const BPS_DENOM: u64 = 10_000; // basis‑point denominator

Safety guarantees

score_bps ≤ 10_000 (=> payout ≤ base_payout).
payout ≤ remaining_budget.
lead_hash is unique per campaign (LeadReceipt PDA).
Only the pre‑registered oracle can call attest_and_pay_lead.

Safety guarantees

score_bps ≤ 10_000 (=> payout ≤ base_payout).
payout ≤ remaining_budget.
lead_hash is unique per campaign (LeadReceipt PDA).
Only the pre‑registered oracle can call attest_and_pay_lead.
