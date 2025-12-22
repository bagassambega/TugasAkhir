# DecMed UMA Architecture

This document describes the system architecture for integrating User-Managed Access (UMA 2.0) with Proxy Re-Encryption (PRE) in the DecMed medical record management framework.

## System Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        PC[("Patient Client<br/>(Mobile App)")]
        HC[("Hospital Client<br/>(Desktop App)")]
        MC[("Ministry Client<br/>(Web App)")]
    end

    subgraph "UMA Authorization Layer"
        AS["Authorization Server<br/>(UMA 2.0)"]
        CD["Consent Dashboard<br/>(Patient UI)"]
        PE["Policy Engine"]
        
        AS --- PE
        CD --> AS
    end

    subgraph "Resource Server Layer"
        RS["Resource Server<br/>(API Gateway)"]
        PRE["PRE Proxy Server"]
        GSS["Gas Station Server"]
        
        RS --> PRE
        RS --> GSS
    end

    subgraph "Storage Layer"
        subgraph "On-Chain (IOTA)"
            IOTA[("IOTA Network")]
            SC["Smart Contracts<br/>(Move)"]
            AL["Audit Logs"]
            PR["Policy Registry"]
            
            IOTA --- SC
            IOTA --- AL
            IOTA --- PR
        end
        
        subgraph "Off-Chain"
            IPFS[("IPFS Cluster<br/>(Encrypted Clinical Data)")]
            REDIS[("Redis<br/>(Session/kfrag Cache)")]
        end
    end

    subgraph "External Services"
        HSM["HSM/Secure Enclave<br/>(Key Storage)"]
        NIK["NIK Validation API<br/>(Dukcapil)"]
    end

    %% Client connections
    PC <--> RS
    PC <--> CD
    HC <--> RS
    MC <--> RS

    %% Authorization flow
    RS <--> AS
    AS <--> IOTA
    PE --> PR

    %% Resource Server to Storage
    PRE <--> REDIS
    PRE <--> IPFS
    GSS <--> IOTA

    %% Security
    PC -.-> HSM
    HC -.-> HSM
    
    %% External
    AS -.-> NIK

    classDef client fill:#e1f5fe,stroke:#01579b
    classDef auth fill:#fff3e0,stroke:#e65100
    classDef resource fill:#e8f5e9,stroke:#1b5e20
    classDef storage fill:#f3e5f5,stroke:#4a148c
    classDef external fill:#fce4ec,stroke:#880e4f

    class PC,HC,MC client
    class AS,CD,PE auth
    class RS,PRE,GSS resource
    class IOTA,SC,AL,PR,IPFS,REDIS storage
    class HSM,NIK external
```

## Component Description

| Component | Purpose | Technology |
|-----------|---------|------------|
| **Patient Client** | Mobile app for patients to manage consent, view EMR | React Native / Flutter |
| **Hospital Client** | Desktop app for healthcare personnel | Tauri / Electron |
| **Ministry Client** | Web app for Kemenkes to register hospitals | Next.js |
| **Authorization Server** | UMA 2.0 compliant token issuer and policy evaluator | Keycloak / Custom |
| **Consent Dashboard** | Patient-facing UI for consent management | Embedded in Patient Client |
| **Policy Engine** | Evaluates access policies against claims | OPA (Open Policy Agent) |
| **Resource Server** | API Gateway protecting medical record endpoints | Go / Rust |
| **PRE Proxy Server** | Performs proxy re-encryption operations | Python (umbral library) |
| **Gas Station Server** | Sponsors IOTA transactions | IOTA SDK |
| **IOTA Network** | DLT for immutable audit logs and policy storage | IOTA Rebased |
| **IPFS Cluster** | Distributed storage for encrypted clinical data | IPFS |
| **Redis** | Cache for sessions, kfrags, nonces | Redis |

---

## Sequence Diagrams

### 1. Authentication Flow

Patient and healthcare personnel authentication using IOTA-based identity.

```mermaid
sequenceDiagram
    autonumber
    participant U as User (Patient/Personnel)
    participant C as Client App
    participant SE as Secure Enclave
    participant AS as Authorization Server
    participant IOTA as IOTA Network

    U->>C: Enter PIN
    C->>SE: Decrypt stored seed with PIN
    SE-->>C: Return seed
    
    C->>C: Derive IOTA keypair from seed
    C->>C: Derive PRE keypair from seed
    
    C->>AS: POST /auth/challenge<br/>{iota_address}
    AS->>IOTA: Verify identity exists<br/>(inspect transaction)
    IOTA-->>AS: Identity confirmed
    AS->>AS: Generate random nonce
    AS-->>C: {nonce, challenge_id}
    
    C->>SE: Sign nonce with IOTA private key
    SE-->>C: {signature}
    
    C->>AS: POST /auth/verify<br/>{challenge_id, signature}
    AS->>AS: Verify signature against IOTA public key
    AS->>AS: Generate session tokens
    AS-->>C: {access_token, refresh_token, id_token}
    
    C->>C: Store tokens securely
    C-->>U: Authentication successful
```

### 2. Authorization Flow (UMA Grant)

Healthcare personnel requesting access to patient's medical record.

```mermaid
sequenceDiagram
    autonumber
    participant RP as Requesting Party<br/>(Doctor Client)
    participant RS as Resource Server
    participant AS as Authorization Server
    participant PE as Policy Engine
    participant RO as Resource Owner<br/>(Patient Client)
    participant IOTA as IOTA Network

    RP->>RS: GET /patient/{id}/record<br/>(no token)
    RS->>RS: Resource is protected
    RS->>AS: POST /permission<br/>{resource_id, scopes: [read:clinical]}
    AS-->>RS: {permission_ticket}
    RS-->>RP: 401 Unauthorized<br/>{permission_ticket, as_uri}

    RP->>AS: POST /token<br/>{grant_type: uma-ticket,<br/>ticket: permission_ticket,<br/>claim_token: id_token}
    
    AS->>PE: Evaluate policy<br/>{resource, requestor_claims}
    PE->>IOTA: Fetch policy from registry
    IOTA-->>PE: {policy_rules}
    PE->>PE: Check: role=MedicalPersonnel ✓<br/>Check: pre-authorized? ✗
    PE-->>AS: Need resource owner consent

    AS->>AS: Create pending request
    AS->>RO: Push notification<br/>"Dr. X requests access to your record"
    
    RO->>AS: GET /consent/pending
    AS-->>RO: {request_details, requestor_info}
    
    RO->>RO: Review request details
    RO->>AS: POST /consent/approve<br/>{request_id, duration: 15min,<br/>scopes: [read:clinical, read:admin]}
    
    AS->>IOTA: Log consent decision
    IOTA-->>AS: {tx_hash}
    AS->>AS: Issue RPT with approved scopes
    AS-->>RO: Consent recorded

    RP->>AS: POST /token (retry with same ticket)
    AS-->>RP: {access_token: RPT,<br/>token_type: Bearer}
    
    Note over RP: Doctor now has valid RPT
```

### 3. Providing Access Flow (with PRE)

After authorization, the actual data access with proxy re-encryption.

```mermaid
sequenceDiagram
    autonumber
    participant RP as Requesting Party<br/>(Doctor Client)
    participant RS as Resource Server
    participant AS as Authorization Server
    participant PRE as PRE Proxy
    participant REDIS as Redis Cache
    participant IOTA as IOTA Network
    participant IPFS as IPFS

    Note over RP: Doctor has valid RPT from authorization flow

    RP->>RS: GET /patient/{id}/record<br/>Authorization: DPoP {RPT}<br/>DPoP: {proof}
    
    RS->>AS: POST /introspection<br/>{token: RPT}
    AS-->>RS: {active: true,<br/>scopes: [read:clinical, read:admin],<br/>exp: timestamp}

    RS->>RS: Validate DPoP proof
    
    RS->>PRE: Request re-encryption<br/>{patient_address, requestor_address, RPT}
    
    PRE->>REDIS: GET kfrag:{patient}:{requestor}
    
    alt kfrag not found (first access)
        REDIS-->>PRE: null
        PRE-->>RS: Need kfrag from patient
        RS-->>RP: 202 Accepted, await kfrag
        
        Note over RP,RS: Patient client generates kfrag asynchronously
        
        RP->>RS: Notify patient to generate kfrag
        RS->>AS: Request kfrag generation
        AS->>RO: Push: Generate access key for Dr. X
        
        RO->>RO: Generate random seed (64 bytes)
        RO->>RO: Derive delegated PRE keypair
        RO->>RO: Compute kfrag using:<br/>- Patient PRE secret key<br/>- Delegated PRE public key<br/>- Signer key
        RO->>RO: Encrypt seed half with Doctor's PRE pubkey
        
        RO->>PRE: POST /keys<br/>{kfrag, enc_seed, capsule,<br/>delegated_pubkey, signer_pubkey}
        PRE->>REDIS: SETEX kfrag:{patient}:{doctor}<br/>TTL: 15min
        REDIS-->>PRE: OK
        PRE-->>RO: kfrag stored
    else kfrag found
        REDIS-->>PRE: {kfrag, capsules, keys}
    end

    PRE->>IOTA: Query patient metadata<br/>(admin + clinical)
    IOTA-->>PRE: {enc_admin_data, enc_aes_key_nonce,<br/>aes_capsule, clinical_cid}
    
    PRE->>IPFS: GET {clinical_cid}
    IPFS-->>PRE: {enc_clinical_data}
    
    PRE->>PRE: Re-encrypt aes_key_nonce<br/>using kfrag → cfrag
    
    PRE-->>RS: {cfrag, enc_admin_data,<br/>enc_clinical_data, capsules,<br/>delegated_pubkey, signer_pubkey}
    
    RS->>IOTA: Log access event<br/>{accessor, patient, timestamp, tx_hash}
    
    RS-->>RP: 200 OK<br/>{encrypted_payload}
    
    RP->>RP: Decrypt enc_seed with own PRE privkey
    RP->>RP: Derive delegated PRE keypair from seed
    RP->>RP: Decrypt aes_key_nonce using cfrag
    RP->>RP: Decrypt admin + clinical data with AES-GCM
    
    RP-->>RP: Display patient record
```

### 4. Revoking Access Flow

Patient revoking previously granted access.

```mermaid
sequenceDiagram
    autonumber
    participant RO as Resource Owner<br/>(Patient Client)
    participant CD as Consent Dashboard
    participant AS as Authorization Server
    participant REDIS as Redis Cache
    participant IOTA as IOTA Network
    participant GSS as Gas Station Server

    RO->>CD: Open "Active Access" page
    
    CD->>AS: GET /consent/active<br/>Authorization: Bearer {patient_token}
    AS->>IOTA: Query active access entries<br/>for patient address
    IOTA-->>AS: [{accessor: Dr.X, scope: read,<br/>granted_at, expires_at, tx_hash}, ...]
    AS-->>CD: {active_accesses: [...]}
    
    CD-->>RO: Display list of active accesses

    RO->>CD: Select access to revoke<br/>(Dr. X - read access)
    CD->>RO: Confirm revocation dialog
    RO->>CD: Confirm

    CD->>AS: POST /consent/revoke<br/>{access_id, reason: "patient_request"}
    
    par Immediate Token Revocation
        AS->>AS: Mark RPT as revoked<br/>(token blacklist)
    and Cache Invalidation
        AS->>REDIS: DEL kfrag:{patient}:{doctor}
        REDIS-->>AS: OK
    and Blockchain Logging
        AS->>GSS: Initiate sponsored transaction
        GSS->>IOTA: Execute revoke_access<br/>{patient, accessor, timestamp}
        IOTA-->>GSS: {tx_hash, status: success}
        GSS-->>AS: Revocation logged
    end

    AS-->>CD: {status: revoked,<br/>tx_hash, revoked_at}
    CD-->>RO: Access revoked successfully

    Note over RO,IOTA: Subsequent access attempts will fail

    rect rgb(255, 230, 230)
        Note right of RP: Doctor attempts to use old RPT
        RP->>RS: GET /patient/{id}/record<br/>Authorization: DPoP {old_RPT}
        RS->>AS: POST /introspection<br/>{token: old_RPT}
        AS-->>RS: {active: false,<br/>reason: "revoked"}
        RS-->>RP: 401 Unauthorized<br/>{error: "access_revoked"}
    end
```

---

## Security Considerations

### Token Security

| Mechanism | Purpose |
|-----------|---------|
| **DPoP** | Binds tokens to client keypair, prevents token theft |
| **Short TTL** | Limits exposure window (5-15 min access tokens) |
| **Refresh Rotation** | New refresh token on each use, detects theft |
| **Introspection** | Real-time token validation before access |

### Cryptographic Guarantees

| Layer | Mechanism |
|-------|-----------|
| **Transport** | mTLS between all services |
| **Data at Rest** | AES-256-GCM encryption |
| **Delegation** | Proxy Re-Encryption (Umbral) |
| **Audit** | IOTA immutable ledger |

### Consent Properties

| Property | Implementation |
|----------|----------------|
| **Explicit** | Push notification + manual approval |
| **Granular** | Per-resource, per-scope consent |
| **Time-bound** | Configurable expiration |
| **Revocable** | Immediate via AS + delayed via blockchain |
| **Auditable** | All decisions logged to IOTA |

---

## Policy Example

```json
{
  "policy_id": "emr_access_policy_v1",
  "resource_type": "patient_emr",
  "rules": [
    {
      "effect": "permit",
      "conditions": {
        "requestor_role": ["MedicalPersonnel"],
        "scopes": ["read:clinical", "read:admin", "write:clinical"],
        "requires_consent": true,
        "max_duration_minutes": 120
      }
    },
    {
      "effect": "permit", 
      "conditions": {
        "requestor_role": ["AdministrativePersonnel"],
        "scopes": ["read:admin"],
        "requires_consent": true,
        "max_duration_minutes": 15
      }
    },
    {
      "effect": "deny",
      "conditions": {
        "time_of_day": {"before": "06:00", "after": "22:00"},
        "unless": {"context": "emergency"}
      }
    }
  ]
}
```
