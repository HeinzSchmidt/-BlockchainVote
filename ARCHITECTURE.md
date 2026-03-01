# AWS Technical Architecture: Trustless Voting Management System
## Architecture Design & Planning Document
**Version 1.0 | Confidential Design Draft**

---

## 1. Executive Summary

This document defines the technical architecture for the Trustless Voting Management System (TVMS) hosted on Amazon Web Services (AWS). The architecture realises the three-layer cryptographic model described in the system paper — Federated Identity Anchors, ZK Issuance Bridge, and Vote Ledger — mapped onto AWS-managed services, with security controls aligned to the UK National Cyber Security Centre (NCSC) Cloud Security Principles and Cyber Essentials Plus framework.

The design prioritises: cryptographic separation between identity and vote data, high availability with no single point of failure, end-to-end auditability, and compliance with NCSC guidance for PUBLIC SERVICES classification.

---

## 2. Architectural Principles

The following principles govern all design decisions:

| Principle | Application |
|---|---|
| **Zero Trust** | No implicit trust between services; all inter-service calls authenticated and authorised |
| **Cryptographic Separation** | Identity data and vote data physically and logically isolated across separate AWS accounts |
| **Defence in Depth** | Multiple overlapping security controls at network, application, and data layers |
| **Least Privilege** | All IAM roles scoped to minimum required permissions |
| **Immutability** | Vote records and audit logs are append-only; no update or delete permissions exist |
| **Auditability** | Every action logged, timestamped, and tamper-evident |
| **Resilience** | Multi-AZ deployment; RTO < 1 hour, RPO < 15 minutes |
| **Open Auditability** | Public ledger endpoints available for independent verification |

---

## 3. AWS Account Structure

The system uses a multi-account AWS Organisation structure. This is the most critical architectural decision: it enforces the governance separation model at the infrastructure level — not just at the policy level.

```
┌─────────────────────────────────────────────────────────────┐
│  AWS ORGANISATION ROOT (Management Account)                 │
│  Governed by: Regulatory Oversight Body                     │
│  Controls: SCPs, account provisioning, billing              │
└──────────────┬──────────────────────────────────────────────┘
               │
    ┌──────────┴──────────────────────────────────┐
    │                                             │
┌───▼──────────────────┐           ┌─────────────▼────────────┐
│  IDENTITY OU         │           │  LEDGER OU               │
│                      │           │                          │
│  ┌────────────────┐  │           │  ┌────────────────────┐  │
│  │ Firm A Account │  │           │  │ Ledger Authority   │  │
│  │ (Identity &    │  │           │  │ Account            │  │
│  │  Registration) │  │           │  │ (Vote Ledger &     │  │
│  └────────────────┘  │           │  │  Tally)            │  │
│  ┌────────────────┐  │           │  └────────────────────┘  │
│  │ Firm B Account │  │           │                          │
│  │ (Identity &    │  │           └──────────────────────────┘
│  │  Registration) │  │
│  └────────────────┘  │           ┌──────────────────────────┐
│                      │           │  SHARED SERVICES OU      │
└──────────────────────┘           │                          │
                                   │  ┌────────────────────┐  │
┌──────────────────────┐           │  │ Security & Audit   │  │
│  ZK BRIDGE OU        │           │  │ Account            │  │
│                      │           │  │ (GuardDuty,        │  │
│  ┌────────────────┐  │           │  │  Security Hub,     │  │
│  │ ZK Bridge      │  │           │  │  CloudTrail Org)   │  │
│  │ Account        │  │           │  └────────────────────┘  │
│  │ (Issuance      │  │           │  ┌────────────────────┐  │
│  │  Bridge)       │  │           │  │ Network Account    │  │
│  └────────────────┘  │           │  │ (Transit GW,       │  │
│                      │           │  │  VPCs, WAF)        │  │
└──────────────────────┘           └──────────────────────────┘
```

Service Control Policies (SCPs) at the Organisation level enforce hard boundaries:

- Identity OU accounts cannot create resources in the Ledger OU and vice versa
- ZK Bridge account can receive from Identity OU and send to Ledger OU — but cannot read from either
- All accounts denied ability to disable CloudTrail or GuardDuty
- All accounts restricted to approved AWS regions (UK: eu-west-2 primary, eu-west-1 DR)

---

## 4. Layer-by-Layer Architecture

### 4.1 Layer 1 — Federated Identity & Registration (Firm A and Firm B Accounts)

Each issuance firm operates an identical stack in its own AWS account. The two stacks are independent but cross-auditable via a shared, read-only registry comparison mechanism.

```
                         FIRM A ACCOUNT (mirrored in Firm B)
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  ┌─────────────────┐    ┌──────────────────────────────────┐    │
│  │   Voter Portal  │    │   Admin Portal                   │    │
│  │   (CloudFront + │    │   (CloudFront + WAF +            │    │
│  │    WAF + S3)    │    │    Cognito MFA)                  │    │
│  └────────┬────────┘    └──────────────┬───────────────────┘    │
│           │                            │                        │
│           ▼                            ▼                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              API GATEWAY (Regional)                     │    │
│  │         mTLS enforced | WAF attached | Throttled        │    │
│  └──────────────────────────┬──────────────────────────────┘    │
│                             │                                   │
│           ┌─────────────────┼─────────────────┐                │
│           ▼                 ▼                 ▼                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐      │
│  │  Registration│  │  Eligibility │  │  Credential      │      │
│  │  Service     │  │  Verification│  │  Issuance        │      │
│  │  (Lambda /   │  │  Service     │  │  Service         │      │
│  │   ECS)       │  │  (Lambda /   │  │  (Lambda / ECS)  │      │
│  └──────┬───────┘  │   ECS)       │  └───────┬──────────┘      │
│         │          └──────┬───────┘          │                 │
│         ▼                 ▼                  ▼                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Amazon RDS (Aurora PostgreSQL)             │    │
│  │         Multi-AZ | Encrypted at rest (KMS CMK)         │    │
│  │         Voter Registry | Eligibility Records            │    │
│  │         NO vote data stored here — ever                 │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              AWS KMS (Customer Managed Keys)            │    │
│  │         Credential signing keys | Registry encryption   │    │
│  │         HSM-backed (CloudHSM for production)            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Key AWS services — Identity Layer:**

| Service | Role |
|---|---|
| **Amazon CloudFront + WAF** | DDoS protection, geo-restriction, OWASP rule sets |
| **Amazon Cognito** | Admin portal MFA; voter portal step-up authentication |
| **API Gateway (Regional)** | mTLS-enforced API layer; rate limiting; request validation |
| **AWS Lambda / ECS Fargate** | Stateless microservices for registration, eligibility, issuance |
| **Amazon RDS Aurora (PostgreSQL)** | Voter registry — identity data only, encrypted with CMK |
| **AWS KMS + CloudHSM** | Credential signing keys; FIPS 140-2 Level 3 HSM backing |
| **Amazon SQS** | Async inter-service messaging; decouples credential issuance from ZK bridge |
| **AWS Secrets Manager** | Rotation of service credentials and API keys |

Cross-firm registry audit: Firm A and Firm B expose a dedicated read-only API endpoint (via API Gateway with cross-account IAM role) that allows each firm to query aggregate registry statistics and flag duplicate identity commitments — without exposing raw voter data. Discrepancies trigger an SNS alert to the regulatory oversight function.

---

### 4.2 Layer 2 — ZK Issuance Bridge (ZK Bridge Account)

This account is the cryptographic heart of the system. It receives eligibility credentials from both issuance firms, verifies ZK proofs, manages the nullifier registry, and mints vote tokens.

```
                         ZK BRIDGE ACCOUNT
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  INPUT: Credential requests from Firm A & Firm B                │
│  (via cross-account SQS + KMS envelope encryption)              │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              SQS Credential Intake Queue                │    │
│  │         Encrypted in transit and at rest                │    │
│  │         Dead-letter queue for failed proofs             │    │
│  └───────────────────────────┬─────────────────────────────┘    │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │          ZK Proof Verification Service (ECS Fargate)    │    │
│  │                                                         │    │
│  │   1. Verify M-of-2 firm attestations present           │    │
│  │   2. Verify ZK proof validity (zk-SNARK verification)  │    │
│  │   3. Check nullifier registry — reject if seen before  │    │
│  │   4. Record nullifier commitment                        │    │
│  │   5. Mint vote token (keypair generation)               │    │
│  │                                                         │    │
│  └────────────────────┬────────────────────────────────────┘    │
│                       │                                         │
│          ┌────────────┴──────────────┐                         │
│          ▼                           ▼                          │
│  ┌──────────────────┐    ┌──────────────────────────────────┐   │
│  │ Nullifier        │    │  Vote Token Registry             │   │
│  │ Registry         │    │  (DynamoDB — append only)        │   │
│  │ (DynamoDB Global │    │  Token public keys registered    │   │
│  │  Tables)         │    │  here before ledger              │   │
│  │  Append-only     │    └──────────────┬───────────────────┘   │
│  │  IAM: no delete  │                   │                       │
│  └──────────────────┘                   │                       │
│                                         ▼                       │
│                          OUTPUT: Vote token to Ledger Account   │
│                          (via cross-account EventBridge)        │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │   AWS CloudHSM — ZK verification keys                   │    │
│  │   KMS CMK — nullifier registry encryption               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Key AWS services — ZK Bridge Layer:**

| Service | Role |
|---|---|
| **Amazon SQS (encrypted)** | Credential intake; decoupled from identity accounts |
| **ECS Fargate** | ZK proof verification workload — containerised, no persistent compute |
| **Amazon DynamoDB** | Nullifier registry and token registry — append-only via IAM condition keys |
| **AWS CloudHSM** | ZK verification key storage — FIPS 140-2 Level 3 |
| **Amazon EventBridge** | Cross-account token minting notification to Ledger account |
| **AWS KMS** | Envelope encryption of all inter-account messages |

---

### 4.3 Layer 3 — Vote Ledger (Ledger Authority Account)

The Vote Ledger is built on Amazon Managed Blockchain using the Hyperledger Fabric framework. This provides a permissioned, append-only, distributed ledger with cryptographic tamper-evidence — well-suited to the voting use case where public verifiability is required but full anonymity of the validator network is not.

```
                      LEDGER AUTHORITY ACCOUNT
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │           Public Verification Portal                    │    │
│  │     (CloudFront + S3 — read-only, unauthenticated)      │    │
│  │     Any citizen can verify tally and nullifier list     │    │
│  └──────────────────────────┬──────────────────────────────┘    │
│                             │                                   │
│  ┌──────────────────────────▼──────────────────────────────┐    │
│  │              API Gateway (Regional)                     │    │
│  │    Vote submission (authenticated) | Public read        │    │
│  └──────────────────────────┬──────────────────────────────┘    │
│                             │                                   │
│           ┌─────────────────┼────────────────────┐             │
│           ▼                 ▼                    ▼              │
│  ┌──────────────┐  ┌──────────────────┐  ┌─────────────────┐   │
│  │  Vote        │  │  Token           │  │  Tally          │   │
│  │  Submission  │  │  Validation      │  │  Service        │   │
│  │  Service     │  │  Service         │  │  (Homomorphic   │   │
│  │  (Lambda)    │  │  (Lambda)        │  │   decryption)   │   │
│  └──────┬───────┘  └──────┬───────────┘  └──────┬──────────┘   │
│         │                 │                      │              │
│         └─────────────────┼──────────────────────┘             │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │         AMAZON MANAGED BLOCKCHAIN                       │    │
│  │         (Hyperledger Fabric)                            │    │
│  │                                                         │    │
│  │   Channel: VoteChannel (permissioned)                   │    │
│  │   Chaincode (Smart Contracts):                          │    │
│  │     - submitVote(tokenPubKey, encryptedBallot,          │    │
│  │                  voteNullifier)                         │    │
│  │     - validateToken(tokenPubKey)                        │    │
│  │     - rejectIfNullifierSeen(voteNullifier)              │    │
│  │     - getTally() — public read                          │    │
│  │                                                         │    │
│  │   Ordering Service: 3-node Raft consensus               │    │
│  │   Peers: Ledger Authority + 2 independent observers     │    │
│  │   All peers in separate AZs                             │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │   Amazon QLDB — Immutable audit log of all              │    │
│  │   administrative actions on the ledger                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Why Amazon Managed Blockchain (Hyperledger Fabric) over a public chain:**

| Factor | Hyperledger Fabric | Public Chain (e.g. Ethereum) |
|---|---|---|
| Permissioned validators | Yes — known, accountable nodes | No — anonymous miners |
| Transaction privacy | Channel-based | Fully public by default |
| Throughput | High (thousands TPS) | Limited, variable |
| Finality | Deterministic | Probabilistic |
| NCSC compliance | Achievable — infrastructure in UK region | Difficult — no data residency control |
| Regulatory oversight | Operator identity known | Anonymous |

**Key AWS services — Ledger Layer:**

| Service | Role |
|---|---|
| **Amazon Managed Blockchain** | Hyperledger Fabric ledger — vote record, nullifier check, tally |
| **AWS Lambda** | Vote submission and token validation microservices |
| **Amazon QLDB** | Immutable administrative audit trail |
| **Amazon S3 + CloudFront** | Public verification portal — tally and nullifier list |
| **AWS KMS** | Tally decryption key management (threshold ceremony required) |

---

## 5. Cross-Account Data Flows

```
VOTER                FIRM A / B           ZK BRIDGE           LEDGER
  │                      │                    │                   │
  │─── Registration ────►│                    │                   │
  │                      │                    │                   │
  │◄── Eligibility ──────│                    │                   │
  │    Credential         │                    │                   │
  │                      │                    │                   │
  │─── ZK Proof ─────────┼───────────────────►│                   │
  │    Request            │   (SQS + KMS)      │                   │
  │                      │                    │                   │
  │                      │   Firm A attest ──►│                   │
  │                      │   Firm B attest ──►│                   │
  │                      │                    │                   │
  │                      │                    │─ Nullifier check  │
  │                      │                    │  (DynamoDB)       │
  │                      │                    │                   │
  │◄── Vote Token ────────────────────────────│                   │
  │    (private key)      │                    │                   │
  │                      │                    │─ Token registered►│
  │                      │                    │  (EventBridge)    │
  │                      │                    │                   │
  │─── Cast Vote ─────────────────────────────────────────────── ►│
  │    (sign with token key)                      Chaincode       │
  │                                               validates &     │
  │                                               records         │
  │◄── Vote Receipt ─────────────────────────────────────────── ─│
  │    (blockchain tx ID)                                         │
```

**Data never crosses these boundaries:**

- Voter PII never enters the ZK Bridge or Ledger accounts
- Vote choices never enter the Identity or ZK Bridge accounts
- The ZK Bridge receives only cryptographic proofs — never raw credentials

---

## 6. Security Architecture

### 6.1 Network Architecture

```
                    INTERNET
                       │
              ┌────────▼────────┐
              │   AWS Shield    │
              │   Advanced      │
              │   (DDoS)        │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │  CloudFront     │
              │  + WAF          │
              │  (Edge layer)   │
              └────────┬────────┘
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │  Firm A  │  │  ZK      │  │  Ledger  │
   │  VPC     │  │  Bridge  │  │  VPC     │
   │          │  │  VPC     │  │          │
   │ Public   │  │          │  │ Public   │
   │ subnet:  │  │ No public│  │ subnet:  │
   │ ALB only │  │ subnets  │  │ ALB only │
   │          │  │          │  │          │
   │ Private  │  │ Private  │  │ Private  │
   │ subnets: │  │ subnets: │  │ subnets: │
   │ ECS/RDS  │  │ ECS/     │  │ Lambda/  │
   │          │  │ DynamoDB │  │ Blockchain│
   └──────────┘  └──────────┘  └──────────┘
         │             │             │
         └─────────────┼─────────────┘
                       │
              ┌────────▼────────┐
              │  AWS Transit    │
              │  Gateway        │
              │  (controlled    │
              │   inter-VPC)    │
              └─────────────────┘
```

- ZK Bridge VPC has no public subnets — all inbound via SQS, all outbound via EventBridge
- All VPCs use VPC Flow Logs to S3 (centralised Security account)
- PrivateLink used for all AWS service access — no traffic traverses public internet
- Security Groups follow deny-all-by-default with explicit allow rules only

### 6.2 Identity & Access Management

| Control | Implementation |
|---|---|
| **Human admin access** | AWS IAM Identity Centre + MFA; just-in-time access via AWS SSO |
| **Service-to-service** | IAM roles with condition keys; no long-lived credentials |
| **Cross-account access** | Explicit trust policies; SCP guardrails prevent privilege escalation |
| **Break-glass access** | Separate emergency IAM roles; use triggers SNS alert; reviewed within 1 hour |
| **Key management** | CloudHSM for signing keys; KMS CMK for data encryption; annual rotation |

### 6.3 Encryption Standards

| Layer | Standard |
|---|---|
| Data at rest | AES-256 via KMS CMK |
| Data in transit | TLS 1.3 minimum; mTLS for inter-service |
| Credential signing | ECDSA P-256 |
| ZK proof scheme | Groth16 (zk-SNARK) or PLONK |
| Ballot encryption | ElGamal homomorphic encryption |
| Key storage | FIPS 140-2 Level 3 (CloudHSM) |

---

## 7. Audit & Observability Architecture

All accounts feed into the centralised Security & Audit Account:

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Firm A      │  │  Firm B      │  │  ZK Bridge   │  │  Ledger      │
│  Account     │  │  Account     │  │  Account     │  │  Account     │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │                  │
       └─────────────────┴────────┬────────┴──────────────────┘
                                  │  (Organisation CloudTrail)
                                  ▼
                     ┌────────────────────────┐
                     │  SECURITY & AUDIT      │
                     │  ACCOUNT               │
                     │                        │
                     │  CloudTrail Org Trail  │
                     │  → S3 (Object Lock     │
                     │    WORM, 7yr retain)   │
                     │                        │
                     │  AWS Security Hub      │
                     │  (aggregated findings) │
                     │                        │
                     │  Amazon GuardDuty      │
                     │  (threat detection)    │
                     │                        │
                     │  AWS Config            │
                     │  (compliance rules)    │
                     │                        │
                     │  Amazon OpenSearch     │
                     │  (SIEM / log analysis) │
                     │                        │
                     │  CloudWatch Dashboards │
                     │  (operational health)  │
                     └────────────────────────┘
```

- **CloudTrail** Organisation trail captures every API call across all accounts — stored in S3 with Object Lock (WORM) — immutable for 7 years
- **AWS Config** rules enforce: encryption enabled, no public S3 buckets, MFA on all accounts, approved AMIs only
- **Security Hub** aggregates findings from GuardDuty, Inspector, Macie, and Config into a single compliance dashboard
- **GuardDuty** provides continuous threat detection including unusual API call patterns, credential exfiltration attempts, and cryptomining detection
- **Amazon Macie** scans for PII appearing in unexpected locations (e.g. vote data buckets)

---

## 8. Resilience & Disaster Recovery

### 8.1 Multi-AZ and Multi-Region Design

```
PRIMARY REGION: eu-west-2 (London)
DR REGION: eu-west-1 (Ireland)

┌───────────────────────────────────┐
│  eu-west-2 (PRIMARY)              │
│                                   │
│  AZ-a    AZ-b    AZ-c             │
│   │       │       │               │
│   └───────┴───────┘               │
│         Active                    │
└───────────────┬───────────────────┘
                │  Replication
                │  (RDS Global, DynamoDB Global,
                │   Blockchain peer in DR)
                ▼
┌───────────────────────────────────┐
│  eu-west-1 (DR / WARM STANDBY)    │
│                                   │
│  AZ-a    AZ-b                     │
│   │       │                       │
│   └───────┘                       │
│   Warm standby — promoted         │
│   in < 1 hour RTO                 │
└───────────────────────────────────┘
```

**Recovery objectives by component:**

| Component | DR Strategy | RTO | RPO |
|---|---|---|---|
| RDS Aurora (voter registry) | Aurora Global Database — synchronous replication | < 1 min failover | < 1 sec |
| DynamoDB (nullifier registry) | Global Tables — active-active | < 1 min | Near-zero |
| Amazon Managed Blockchain | Peer node in DR region | < 30 min | Zero (distributed) |
| Lambda / ECS | Deployed in DR region (warm) | < 5 min | N/A (stateless) |
| API Gateway | Regional endpoint in DR region | < 5 min | N/A |

### 8.2 Backup Strategy

- RDS: Automated daily snapshots + continuous transaction logs; 35-day retention
- DynamoDB: Point-in-time recovery enabled; 35-day window
- S3 (audit logs): Object Lock WORM; versioning enabled; cross-region replication
- KMS keys: Backed by CloudHSM — replicated across AZs; key material escrowed per governance protocol

---

## 9. NCSC Compliance Mapping

The architecture is designed against the NCSC Cloud Security Principles (14 principles) and Cyber Essentials Plus framework, appropriate for a system handling sensitive democratic infrastructure classified as OFFICIAL-SENSITIVE.

| NCSC Principle | Implementation |
|---|---|
| **1. Data in transit protection** | TLS 1.3 + mTLS; PrivateLink; no data traverses public internet between services |
| **2. Asset protection & resilience** | eu-west-2 primary (UK sovereign); Multi-AZ; AWS ISO 27001 / SOC 2 compliance |
| **3. Separation between users** | Multi-account AWS Organisation; SCPs enforce hard boundaries; VPC isolation |
| **4. Governance framework** | AWS Organisations + Config + Security Hub; documented RACI; named data owners per account |
| **5. Operational security** | GuardDuty + Security Hub; patching via AWS Systems Manager; immutable infrastructure |
| **6. Personnel security** | IAM Identity Centre + MFA; least privilege; JIT access; background-checked admin personnel |
| **7. Secure development** | IaC (Terraform/CDK) with mandatory peer review; SAST/DAST in CI pipeline; dependency scanning |
| **8. Supply chain security** | AWS Marketplace approved images only; container image signing (ECR + Sigstore); SBOM maintained |
| **9. Secure user management** | Cognito with MFA for all portals; session timeout; RBAC enforced at API Gateway |
| **10. Identity & authentication** | IAM roles only (no IAM users in production); CloudHSM-backed credentials; SAML federation |
| **11. External interface protection** | WAF (OWASP rules); Shield Advanced; API Gateway throttling; geo-restriction |
| **12. Secure service administration** | Bastion-free — all admin via AWS Systems Manager Session Manager; all sessions recorded |
| **13. Audit information** | Org CloudTrail + WORM S3; QLDB for ledger admin; 7-year retention; tamper-evident |
| **14. Secure use of the service** | Voter guidance documentation; hardware key device support; accessibility provisions |

### 9.1 Additional Framework Alignment

| Framework | Alignment |
|---|---|
| **NCSC Zero Trust Architecture** | Fully aligned — no implicit trust between services; continuous verification |
| **NCSC Logging Made Easy** | CloudWatch + OpenSearch SIEM; centralised log aggregation |
| **NCSC Cyber Essentials Plus** | Firewall, secure config, access control, malware protection, patch management — all implemented |
| **NCSC Supply Chain Guidance** | AWS as Tier 1 supplier; BAAs and DPAs in place; supplier assurance questionnaire completed |
| **UK GDPR / Data Protection Act 2018** | Voter PII confined to Identity accounts (UK region only); data minimisation by design; DPIAs conducted |

---

## 10. Key Management & Tally Ceremony

The most sensitive operational procedure in the system is the tally decryption key ceremony — the process by which encrypted ballots are decrypted to produce the final tally.

```
┌──────────────────────────────────────────────────────────────┐
│              TALLY KEY — SHAMIR SECRET SHARING               │
│                                                              │
│   Full key split into 5 shares (3-of-5 threshold to open)   │
│                                                              │
│   Share 1 → Ledger Authority                                 │
│   Share 2 → Issuance Firm A                                  │
│   Share 3 → Issuance Firm B                                  │
│   Share 4 → Regulatory Oversight Body                        │
│   Share 5 → Independent Legal Observer                       │
│                                                              │
│   Ceremony: In-person | Air-gapped hardware | Fully recorded │
│   Key used once, immediately after polls close               │
│   Key material securely destroyed after tally confirmed      │
└──────────────────────────────────────────────────────────────┘
```

This ensures no single party — including AWS — can decrypt ballots unilaterally.

---

## 11. Full System Summary Diagram

```
                              ┌─────────────┐
                              │   VOTER     │
                              └──────┬──────┘
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                                 ▼
         ┌─────────────────┐               ┌─────────────────┐
         │   FIRM A        │◄─cross-audit─►│   FIRM B        │
         │   Identity &    │               │   Identity &    │
         │   Registration  │               │   Registration  │
         │   (AWS Account) │               │   (AWS Account) │
         └────────┬────────┘               └────────┬────────┘
                  │  Eligibility credential          │
                  │  (M-of-2 required)               │
                  └──────────────┬───────────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │   ZK ISSUANCE BRIDGE         │
                  │   (Separate AWS Account)     │
                  │                              │
                  │   - ZK proof verification    │
                  │   - Nullifier registry       │
                  │   - Token minting            │
                  │   No identity data stored    │
                  └──────────────┬───────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │  Vote Token             │
                    │  (to voter device)      │
                    │  Token registration     │
                    │  (to ledger)            │
                    └────────────┬────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │   VOTE LEDGER                │
                  │   (Separate AWS Account)     │
                  │   Amazon Managed Blockchain  │
                  │   (Hyperledger Fabric)       │
                  │                              │
                  │   - Token validation         │
                  │   - Vote recording           │
                  │   - Nullifier enforcement    │
                  │   - Public tally             │
                  │   No identity data stored    │
                  └──────────────┬───────────────┘
                                 │
                    ┌────────────┴────────────┐
                    ▼                         ▼
         ┌─────────────────┐      ┌─────────────────────┐
         │  SECURITY &     │      │  PUBLIC             │
         │  AUDIT ACCOUNT  │      │  VERIFICATION       │
         │  CloudTrail     │      │  PORTAL             │
         │  GuardDuty      │      │  (Anyone can verify │
         │  Security Hub   │      │   tally integrity)  │
         │  Config         │      └─────────────────────┘
         └─────────────────┘
```

---

## 12. Next Steps

| Workstream | Owner | Priority |
|---|---|---|
| AWS Organisation provisioning and SCP design | Cloud Platform Team | Critical |
| Threat model and NCSC assurance review | CISO / Security Architect | Critical |
| ZK proof library selection and independent audit | Cryptography Lead | Critical |
| Amazon Managed Blockchain network design | Blockchain Engineer | High |
| Key ceremony procedure documentation | Governance Lead | High |
| Penetration testing scope definition | Security Team | High |
| Data Protection Impact Assessment (DPIA) | Data Protection Officer | High |
| Voter portal UX and accessibility design | Product Team | Medium |
| DR failover runbook | Operations Team | Medium |
| Independent security audit engagement | External Auditor | Medium |

---

*This document is a design and planning artefact. It does not constitute a final security assessment. Independent penetration testing, formal threat modelling, and NCSC assurance review are required before any production deployment.*
