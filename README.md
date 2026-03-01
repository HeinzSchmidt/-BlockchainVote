# Trustless Voting Management System
## Project Summary

---

## What We Are Building

A digital voting system that makes it **mathematically impossible** to vote twice, while ensuring no single organisation can manipulate the outcome, and no one can link a ballot to the person who cast it.

The system is inspired by the same principle that prevents Bitcoin fraud: instead of trusting a central authority to enforce the rules, the rules are enforced by cryptographic mathematics and a public, tamper-proof ledger.

---

## The Core Problem Being Solved

Every voting system must answer three questions:

1. **Is this a real, eligible voter?**
2. **Have they already voted?**
3. **Who are they voting for?**

Traditional systems answer all three using a single trusted authority — a government body or election commission. This creates a single point of failure, political capture, and public distrust.

This system answers each question differently:

| Question | How it is answered | Who knows |
|---|---|---|
| Is this an eligible voter? | Two competing firms independently verify identity | Identity firms only |
| Have they already voted? | Cryptographic nullifier on a public ledger | Nobody — it's mathematical |
| Who are they voting for? | Encrypted ballot — decrypted only at tally time | Nobody until polls close |

---

## How It Works — In Plain English

1. **Registration**: A voter registers with two independent firms. Both must confirm eligibility before anything proceeds.

2. **Token issuance**: The voter receives a one-time cryptographic vote token — like a digital ballot paper — with no link to their identity.

3. **Casting a vote**: The voter uses their token to cast an encrypted ballot on a public ledger. The token is consumed in the process — it can never be used again.

4. **Verification**: Anyone can inspect the public ledger to confirm every vote was cast by a valid token, and no token was used twice.

5. **Tallying**: After polls close, a group of independent parties jointly unlock the tally using a shared key — no single party can do it alone.

---

## The Key Insight

The double-vote problem is identical in structure to the double-spend problem in Bitcoin — and Bitcoin solved it without any central authority, in 2008.

Once that parallel is recognised, the entire trust problem reduces to a single bounded question:

> *How do we ensure each real human gets exactly one token?*

That question — and only that question — requires human institutions. Everything after token issuance is trustless by design.

---

## Governance Structure

The system is governed by three structurally separated bodies:

'''
┌─────────────────────┐    cross-audits    ┌─────────────────────┐
│  Issuance Firm A    │ ◄────────────────► │  Issuance Firm B    │
│  Voter registration │                    │  Voter registration │
│  Token issuance     │                    │  Token issuance     │
└────────┬────────────┘                    └──────────┬──────────┘
│                                            │
│         ZK cryptographic bridge            │
│         (identity never crosses)           │
└───────────────────┬────────────────────────┘
▼
┌────────────────────────────────┐
│  Independent Ledger Authority  │
│  Vote recording & tally        │
│  Cannot access identity data   │
└────────────────────────────────┘
'''


- **Two competing commercial firms** handle voter registration and audit each other
- **A structurally independent Ledger Authority** manages the vote record
- **A cryptographic bridge** ensures identity data can never reach the ledger — by mathematical design, not policy

---

## Technical Architecture — AWS

The system is hosted on AWS and uses separate AWS accounts for each functional layer, enforced by Service Control Policies at the Organisation level. No account can access another's data — this is an infrastructure guarantee, not an administrative one.

| Layer | AWS Services | Purpose |
|---|---|---|
| **Identity & Registration** | API Gateway, ECS Fargate, RDS Aurora, CloudHSM, Cognito | Voter registry, eligibility verification, credential issuance |
| **ZK Issuance Bridge** | SQS, ECS Fargate, DynamoDB, CloudHSM, EventBridge | ZK proof verification, nullifier registry, token minting |
| **Vote Ledger** | Amazon Managed Blockchain (Hyperledger Fabric), Lambda, QLDB | Vote recording, nullifier enforcement, public tally |
| **Security & Audit** | CloudTrail, GuardDuty, Security Hub, AWS Config, OpenSearch | Centralised audit, threat detection, compliance monitoring |

The system is deployed in **eu-west-2 (London)** as primary with warm standby in **eu-west-1 (Ireland)**, maintaining UK data residency throughout.

---

## Security & Compliance

The architecture is aligned to:

- **NCSC Cloud Security Principles** (all 14 principles addressed)
- **NCSC Cyber Essentials Plus**
- **NCSC Zero Trust Architecture guidance**
- **UK GDPR and Data Protection Act 2018**

Key security properties:

- All data encrypted at rest (AES-256) and in transit (TLS 1.3 / mTLS)
- Cryptographic keys stored in FIPS 140-2 Level 3 Hardware Security Modules (CloudHSM)
- No single person or organisation can decrypt ballots — requires a 3-of-5 key ceremony
- All administrative actions are permanently recorded in tamper-evident logs (7-year retention)
- Public verification portal allows any citizen to independently confirm tally integrity

---

## What Makes This Different

| Traditional E-Voting | This System |
|---|---|
| Central authority enforces rules | Mathematics enforces rules |
| Double-vote prevented by policy | Double-vote is structurally impossible |
| Ballot privacy depends on authority | Ballot privacy is cryptographic |
| Single point of failure | No single point of failure |
| Limited public auditability | Full public verifiability |
| Trust is unconditional | Trust is minimal and distributed |

---

## Document Index

This project is documented across the following papers, each building on the last:

1. **Conceptual Paper** — *A Trustless Voting Token System with Decentralized Issuance* — the cryptographic and governance model
2. **AWS Architecture Document** — *AWS Technical Architecture: Trustless Voting Management System* — the full technical design on AWS
3. **This Document** — Project summary

---

## Status

> Concept and architecture design phase complete. Next step: formal threat modelling, NCSC assurance review, and ZK proof library selection.

---

*This is a design and planning project. No production systems have been built. Independent security review is required before any deployment.*
