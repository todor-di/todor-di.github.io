---
title: Todor Dimitrov
---

# Todor Dimitrov

_Selected work in payments security, compliance and platform architecture._

[GitHub](https://github.com/todor-di) · [LinkedIn](https://www.linkedin.com/in/todordim/)

This page covers three products I've built, each as a short case study: the problem, the solution, how it works, and its impact.

1. [Delegated Authentication Solution (FIDO2)](#1-delegated-authentication-solution-fido2)
2. [Rule Engine Micro-service](#2-rule-engine-micro-service)
3. [PCI Proxy Tokenization Component](#3-pci-proxy-tokenization-component)

---

## 1. Delegated Authentication Solution (FIDO2)

**Introduce a SCA solution for frictionless checkout for a major Nordic retail chain.**

### The problem

3-D Secure is mandatory for card payments within Europe, however for non-card payment methods, the authentication is standardized and is done via multiple channels. This causes friction and abandonment during payment sessions.

### The solution

I led the design and delivery of a FIDO2-based **delegated authentication** flow. The solution marries the transaction data, created by the merchant with the specific cryptographic proof, created on the customer's device. The result is a SCA compatible cryptogram, which is time bound, transaction bound and can be validated by any stakeholder that takes part in the payment process.

### How it works

The API handshake between the merchant app, the FIDO2 server, the e-wallet and the issuer:

```mermaid
sequenceDiagram
    autonumber
    actor C as Customer
    participant MBE as Merchant server
    participant MFE as Merchant App<br/>(device authenticator)
    participant F as FIDO2 Server
    participant PM as Payment Method Provider

    rect rgba(128,128,128,0.08)
    Note over C, F: Assume cryptographic keys were setup earlier during enrollment phase
    C->>MFE: Select a payment method and click "Pay"
    MFE->>MBE: Initiate payment 
    MBE->>F: POST /transactions (amount, currency, references)
    F->>F: Validate the data, store it and create timebound session.
    F->>MBE: Return transaction data along with session details
    MBE->>MFE: Pass the session data.
    MFE->>F: POST /sessions/{public id}/attestation
    F->>F: Validate session
    F->>F: Create unique cryptographic challange
    F->>MFE: Return challange
    MFE->>C: Render biometric challange
    C->>MFE: Approve challange with FaceId or fingerprint
    MFE->>F: POST /sessions/{public id}/result and pass encrypted challange
    F->>F: Validate encrypted data
    F->>F: Issue a one time JWT (oJWT), containing encrypted data
    F->>MFE: Return oJWT and result code
    MFE->>MBE: Pass the oJWT 
    MBE->>F: POST /transactions/{id}/token with oJWT
    F->>F: Validate the oJWT, produce cryptogram
    F->>MBE: Return cryptogram
    MBE->>PM: Create transaction with payment data and cryptogram as proof.
    PM->>PM: Process transaction
    PM->>MBE: Return result
    MBE->>MFE: Return result
    MFE->>C: Show result and redirect customer to success page
    end
```

### Impact

- **User experience:** No waiting, no redirects, no confusing windows opening and closing.
- **Authorisation rate:** Improved by 10% for non-card payments. Currently being discussed with issuers to accept it instead of 3-D Secure.
- **Conversion:** Reduce the dropout and error rate by 15%
- **Compliance:** FIDO2 certified and compliant. Currently being discussed for SCA replacement for issuers in Nordics.

---

## 2. Rule Engine Micro-service

**Systems thinking: a decoupled, scalable architecture that lets the business change its own rules.**

### The problem

Payment routing, fraud detection and fee calculation logic is often hard-coded into monoliths. Changing even a simple business rule then costs developer time and a full deployment cycle.

### The solution

I architected an isolated rule engine micro-service. It evaluates business logic dynamically against each incoming transaction payload. Rules are stored and versioned as code so the business has flexibility to do whatever it needs.

### How it works

The structure follows the [C4 model](https://c4model.com). The Apple Pay and SEPA connectors are example clients.

**Level 1: System context.** Who uses the rule engine, and which systems depend on it:

```mermaid
flowchart TB
    OPS["<b>Business Operations</b><br/><small>[Person]</small><br/>Configures routing, fraud<br/>and fee rules"]
    APPLE["<b>Apple Pay Connector</b><br/><small>[Software System]</small><br/>Processes Apple Pay<br/>tokenised card payments"]
    SEPA["<b>SEPA Connector</b><br/><small>[Software System]</small><br/>Processes SEPA credit<br/>transfers and direct debits"]
    ENGINE["<b>Rule Engine</b><br/><small>[Software System]</small><br/>Evaluates business rules against<br/>transaction payloads and<br/>returns decisions"]

    APPLE -- "Requests routing, fraud<br/>and fee decisions<br/><small>[REST/JSON]</small>" --> ENGINE
    SEPA -- "Requests scheme, limit<br/>and fee decisions<br/><small>[REST/JSON]</small>" --> ENGINE
    OPS -- "Manages rules<br/><small>[HTTPS]</small>" --> ENGINE

    classDef person fill:#08427b,stroke:#052e56,color:#fff
    classDef system fill:#1168bd,stroke:#0b4884,color:#fff
    classDef external fill:#999,stroke:#6b6b6b,color:#fff
    class OPS person
    class ENGINE system
    class APPLE,SEPA external
```

**Level 2: Containers.** What's inside the rule engine, and how a decision request and a rule change flow through it:

```mermaid
flowchart TB
    APPLE["<b>Apple Pay Connector</b><br/><small>[Software System]</small><br/>Card payments via Apple Pay"]
    SEPA["<b>SEPA Connector</b><br/><small>[Software System]</small><br/>SCT, SCT Inst and SDD payments"]
    OPS["<b>Business Operations</b><br/><small>[Person]</small><br/>Configures rules"]

    subgraph ENGINE["Rule Engine [Software System]"]
        EVAL["<b>Evaluation API</b><br/><small>[Container: REST, stateless]</small><br/>Evaluates a payload against the<br/>active rule set, returns a decision"]
        ADMIN["<b>Rule Management API</b><br/><small>[Container: REST]</small><br/>Validates, versions and<br/>publishes rule sets"]
        CACHE[("<b>Compiled Rule Cache</b><br/><small>[Container: In-memory]</small><br/>Pre-compiled decision trees")]
        AUDIT[("<b>Decision Audit Log</b><br/><small>[Container: Append-only store]</small><br/>Decision, rule version<br/>and evaluation path")]
        BUS{{"<b>Rule Events</b><br/><small>[Container: Event bus]</small><br/>Rule-published events"}}
        RULES[("<b>Rule Repository</b><br/><small>[Container: Versioned store]</small><br/>Rule sets and version history")]
    end

    APPLE -- "Evaluate payment<br/><small>[REST/JSON]</small>" --> EVAL
    SEPA -- "Evaluate transfer<br/><small>[REST/JSON]</small>" --> EVAL
    OPS -- "Edits and publishes rules<br/><small>[HTTPS]</small>" --> ADMIN
    EVAL -- "Records decision" --> AUDIT
    EVAL -- "Reads active rules" --> CACHE
    ADMIN -- "Stores new version" --> RULES
    ADMIN -- "Publishes rule change" --> BUS
    BUS -- "Invalidates / reloads" --> CACHE

    classDef person fill:#08427b,stroke:#052e56,color:#fff
    classDef container fill:#438dd5,stroke:#2e6295,color:#fff
    classDef external fill:#999,stroke:#6b6b6b,color:#fff
    class OPS person
    class EVAL,ADMIN,CACHE,AUDIT,BUS,RULES container
    class APPLE,SEPA external
    style ENGINE fill:none,stroke:#0b4884,stroke-dasharray:6 4
```

### Main concepts

**Rule** - A javascript code, which is executed in a secure environment on the server. The code and the server belong to the same entity and access to both is restricted. <br>
**Rule set** - A collection of rules with sequence of execution. <br>
**Triggers** - Defined via rule management API. Each trigger is created to reflect a specific processing step and service. <br>
**Contracts** - The format of messages being exchanged for each trigger. <br>
**Rule test** - A specific execution environment, which allows rule creators to test their code execution. Each rule must comply with specific latency requirement, otherwise it cannot be included in a rule-set.
**Connectors** - A list of pre-defined functions, which rule creators can invoke in their javascript code (such as external systems, caching or data storage). Some example connectors are "list", "dictionary" or "call". 

**Example: Making a SEPA payment on sepa-input trigger**:

**Request**
```json
POST /triggers/sepa-input/
{
  "transactionId": "txn_8f2c1a",
  "amount": { "value": 1250.00, "currency": "EUR" },
  "reference": "Your_Order_12345",
  "paymentMethod": {
    "type": "sepadirectdebit",
    "ownerName": "A. Klaassen",
    "ibanNumber": "NL98ABNA0410108103"
  },
  "returnUrl": "https://your-company.com/checkout/success",
  "merchantAccount": "YourMerchantAccount",
  "channel": "ECOM"
}
```
**Response**

```json
{
  "transactionId": "txn_8f2c1a",
  "amount": { "value": 1250.00, "currency": "EUR" },
  "reference": "Your_Order_12345",
  "paymentMethod": {
    "type": "sepadirectdebit",
    "ownerName": "A. Klaassen",
    "ibanNumber": "NL98ABNA0410108103"
  },
  "returnUrl": "https://your-company.com/checkout/success",
  "merchantAccount": "YourMerchantAccount",
  "channel": "ECOM",
  "evaluation": {
    "ruleSet": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "ruleSetVersionStamp": "hiu597ag...",
    "startProcessingAt": "17907785582312",
    "finishProcessingAt": "17907785583711",
    "isSuccess": true,
    "isStopped": false
  },
  "actions": [
    {
      "op": "add",
      "path": "riskScore",
      "value": 75
    },
    {
      "op": "add",
      "path": "multipayment",
      "value": true
    },
    {
      "op": "add",
      "path": "merchantAccounts",
      "value": [{"accountNumber":"YourMerchantAccount", "merchantName":"myMerchantName"}, {"accountNumber":"Somenumber", "merchantName":"someName"}]
    }
  ]
}
```

### Impact

- **Time-to-market for new rules:** from _[weeks]_ of developer effort and a release cycle to _[minutes]_ of operational configuration.
- **Decoupling:** routing, fraud, and fee logic moved out of the core services, so rule changes no longer need a deployment.
- **Traceability:** every decision is recorded with the rule-set version and evaluation path, for audit and dispute handling.

---

## 3. PCI Proxy Tokenization Component

**High-stakes compliance, risk reduction and enterprise data security.**

### The problem

Any system that stores, processes or transmits a raw Primary Account Number (PAN) falls within PCI-DSS scope. Handling raw PANs puts merchants and payment providers into the highest and most expensive compliance tiers.

### The solution

I designed a proxy that intercepts PAN data before it reaches the merchant's backend. It swaps the PAN for a **network token** (EMVCo, via Mastercard MDES / Visa VTS), so the core systems only ever handle desensitised data.

### How it works

Raw card data stays inside the red zone. Only tokens cross into the merchant's green zone.

```mermaid
flowchart TB
    subgraph RED["🔴 Red zone: PCI-DSS scope (raw PAN)"]
        direction LR
        C[Cardholder<br/>browser / app]
        P[PCI Proxy<br/>Tokenization]
        TSP[Token Service Provider<br/>MDES / VTS]
    end

    subgraph GREEN["🟢 Green zone: desensitised scope (tokens only)"]
        direction LR
        B[Merchant backend]
        DB[(Merchant databases)]
        AN[Analytics / CRM]
    end

    PSP[PSP / Acquirer]

    C -- "① raw PAN (TLS)" --> P
    P -- "② PAN → token request" --> TSP
    TSP -- "③ network token" --> P
    P -- "④ network token only" --> B
    B --> DB
    B --> AN
    B -- "⑤ token + cryptogram" --> PSP

    classDef red fill:#fde2e2,stroke:#c0392b,color:#000
    classDef green fill:#e3f6e8,stroke:#27ae60,color:#000
    class C,P,TSP red
    class B,DB,AN green
    style RED fill:#fff5f5,stroke:#c0392b
    style GREEN fill:#f5fff7,stroke:#27ae60
```

### Impact

- **Reduced compliance scope:** the merchant's PCI-DSS scope is downgraded, e.g. to **SAQ A** or **SAQ A-EP** instead of a full SAQ D / Report on Compliance.
- **Cost savings:** _[e.g. annual audit and compliance cost reduced by X]_
- **Risk reduction:** a breach of the merchant's systems exposes only tokens, which are useless outside their domain.
- **Authorisation uplift:** network tokens stay valid when cards are reissued _[e.g. +X% authorisation rate]_.
