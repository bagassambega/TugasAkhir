# Arsitektur DecMed dengan UMA 2.0 dan Patient Consent Portal

## 1. Diagram Arsitektur Sistem

```mermaid
graph TB
    subgraph "Patient"
        Patient[("Pasien<br/>(Resource Owner)")]
    end
    
    subgraph "Medical Personnel"
        Doctor[("Dokter")]
        Nurse[("Perawat")]
        Admin[("Admin Fasyankes")]
    end
    
    subgraph "External Entity"
        MinHealth[("Kementerian<br/>Kesehatan")]
        OtherFasyankes[("Fasyankes<br/>Lain")]
    end
    
    subgraph "Client Layer"
        ConsentPortal["Patient Consent Portal<br/>(PAP - Policy Admin Point)<br/>[Web/Mobile App]"]
        HospitalClient["Hospital Client<br/>[Desktop App]"]
        MinistryClient["Ministry Client<br/>[Desktop App]"]
    end
    
    subgraph "IOTA Network - Smart Contract Layer"
        direction TB
        AuthzSC["Authorization Server<br/>Smart Contract<br/>(PDP - Policy Decision Point)"]
        ResourceSC["Resource Registration<br/>Smart Contract"]
        PolicySC["Policy Storage<br/>Smart Contract"]
        AuditSC["Audit Trail<br/>Smart Contract"]
    end
    
    subgraph "Backend Services"
        ProxyReEnc["Proxy Re-encryption<br/>Server<br/>(PEP - Policy Enforcement Point)<br/>[REST Backend]"]
        TokenService["Token Service<br/>(Issue PAT/AAT/RPT)"]
        ClaimsGathering["Claims Gathering<br/>Service"]
    end
    
    subgraph "Storage Layer"
        Redis[("Redis<br/>(Session & Cache)")]
        IPFS[("IPFS<br/>(Medical Records<br/>Encrypted Storage)")]
        OffChainDB[("Off-Chain Database<br/>(Policy Metadata)")]
    end
    
    subgraph "IOTA Infrastructure"
        GasStation1["IOTA Gas Station<br/>Server"]
        GasStation2["IOTA Gas Station<br/>Server"]
        IOTANetwork["IOTA Network<br/>(DLT)"]
    end
    
    %% Patient Interactions
    Patient -->|"1. Login &<br/>Define Policy"| ConsentPortal
    Patient -->|"8. Sponsored Tx"| GasStation1
    
    %% Medical Personnel Interactions
    Doctor -->|"Request Access"| HospitalClient
    Nurse -->|"Request Access"| HospitalClient
    Admin -->|"Register Resources"| HospitalClient
    
    %% Ministry Interactions
    MinHealth -->|"Monitor & Audit"| MinistryClient
    
    %% External Requesting Party
    OtherFasyankes -->|"Request Access<br/>via Claims"| HospitalClient
    
    %% Consent Portal Flow
    ConsentPortal -->|"2. Create/Update<br/>Policy"| AuthzSC
    ConsentPortal -->|"Store Policy<br/>Metadata"| OffChainDB
    ConsentPortal -->|"View Resources"| ResourceSC
    
    %% Hospital Client Flow
    HospitalClient -->|"3. Register<br/>Resource (PAT)"| ResourceSC
    HospitalClient -->|"4. Request<br/>Permission (AAT)"| AuthzSC
    HospitalClient -->|"5. Access<br/>Resource (RPT)"| ProxyReEnc
    HospitalClient -->|"Sponsored Tx"| GasStation2
    
    %% Ministry Client Flow
    MinistryClient -->|"Query Audit<br/>Trail"| AuditSC
    MinistryClient -->|"Sponsored Tx"| GasStation2
    
    %% Authorization Server Smart Contract
    AuthzSC -->|"Validate Policy"| PolicySC
    AuthzSC -->|"Check Resource"| ResourceSC
    AuthzSC -->|"Log Access"| AuditSC
    AuthzSC -->|"Retrieve Policy<br/>Metadata"| OffChainDB
    AuthzSC -->|"Issue Tokens"| TokenService
    
    %% Token Service
    TokenService -->|"Cache Tokens"| Redis
    TokenService -->|"Validate Claims"| ClaimsGathering
    
    %% Claims Gathering
    ClaimsGathering -->|"Verify Attributes"| OffChainDB
    
    %% Proxy Re-encryption Server (PEP)
    ProxyReEnc -->|"6. Validate Token<br/>(RPT)"| AuthzSC
    ProxyReEnc -->|"7. Retrieve<br/>Encrypted Data"| IPFS
    ProxyReEnc -->|"Re-encrypt &<br/>Return Data"| HospitalClient
    ProxyReEnc -->|"Cache"| Redis
    
    %% IOTA Network Integration
    GasStation1 -->|"Submit Transaction"| IOTANetwork
    GasStation2 -->|"Submit Transaction"| IOTANetwork
    IOTANetwork -->|"Deploy/Execute"| AuthzSC
    IOTANetwork -->|"Deploy/Execute"| ResourceSC
    IOTANetwork -->|"Deploy/Execute"| PolicySC
    IOTANetwork -->|"Deploy/Execute"| AuditSC
    
    %% Styling
    classDef actor fill:#e1f5ff,stroke:#01579b,stroke-width:2px
    classDef client fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    classDef smartcontract fill:#e8eaf6,stroke:#3f51b5,stroke-width:2px
    classDef backend fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef storage fill:#e0f2f1,stroke:#00695c,stroke-width:2px
    classDef infra fill:#fce4ec,stroke:#c2185b,stroke-width:2px
    
    class Patient,Doctor,Nurse,Admin,MinHealth,OtherFasyankes actor
    class ConsentPortal,HospitalClient,MinistryClient client
    class AuthzSC,ResourceSC,PolicySC,AuditSC smartcontract
    class ProxyReEnc,TokenService,ClaimsGathering backend
    class Redis,IPFS,OffChainDB storage
    class GasStation1,GasStation2,IOTANetwork infra
```

## 2. Data Flow Diagram (DFD) Level 0 - Context Diagram

```mermaid
graph LR
    subgraph External_Entities["External Entities"]
        Patient[("Pasien<br/>(Resource Owner)")]
        MedicalStaff[("Tenaga Medis<br/>(Requesting Party)")]
        Ministry[("Kementerian<br/>Kesehatan")]
    end
    
    System["DecMed-UMA System<br/>(Medical Records Management<br/>with UMA 2.0)"]
    
    %% Patient Flows
    Patient -->|"Login Credentials"| System
    Patient -->|"Define Access Policy"| System
    Patient -->|"View Access History"| System
    Patient -->|"Revoke Access"| System
    System -->|"Policy Confirmation"| Patient
    System -->|"Access Request Notification"| Patient
    System -->|"Audit Report"| Patient
    
    %% Medical Staff Flows
    MedicalStaff -->|"Authentication"| System
    MedicalStaff -->|"Resource Access Request"| System
    MedicalStaff -->|"Medical Record Data"| System
    System -->|"Access Token (RPT)"| MedicalStaff
    System -->|"Medical Records"| MedicalStaff
    System -->|"Access Denied Notice"| MedicalStaff
    
    %% Ministry Flows
    Ministry -->|"Audit Query"| System
    System -->|"Audit Trail Report"| Ministry
    System -->|"Compliance Report"| Ministry
    
    classDef entity fill:#e1f5ff,stroke:#01579b,stroke-width:3px
    classDef system fill:#fff9c4,stroke:#f57f17,stroke-width:4px
    
    class Patient,MedicalStaff,Ministry entity
    class System system
```

## 3. Data Flow Diagram (DFD) Level 1 - Process Decomposition

```mermaid
graph TB
    subgraph External_Entities["External Entities"]
        Patient[("Pasien")]
        MedicalStaff[("Tenaga Medis")]
        Ministry[("Kementerian<br/>Kesehatan")]
    end
    
    subgraph Processes["Main Processes"]
        P1["1.0<br/>Authentication &<br/>Identity Management"]
        P2["2.0<br/>Resource<br/>Registration"]
        P3["3.0<br/>Policy<br/>Management"]
        P4["4.0<br/>Authorization<br/>& Token Issuance"]
        P5["5.0<br/>Resource<br/>Access Control"]
        P6["6.0<br/>Audit Trail<br/>& Monitoring"]
    end
    
    subgraph Data_Stores["Data Stores"]
        D1[("D1: User Credentials<br/>(Off-Chain)")]
        D2[("D2: Resource Registry<br/>(Smart Contract)")]
        D3[("D3: Policy Store<br/>(Smart Contract +<br/>Off-Chain Metadata)")]
        D4[("D4: Token Cache<br/>(Redis)")]
        D5[("D5: Medical Records<br/>(IPFS Encrypted)")]
        D6[("D6: Audit Log<br/>(Smart Contract)")]
    end
    
    %% Patient Flows
    Patient -->|"Login Request"| P1
    P1 -->|"Authentication Token"| Patient
    Patient -->|"Define/Update Policy"| P3
    P3 -->|"Policy Confirmation"| Patient
    Patient -->|"View Audit History"| P6
    P6 -->|"Access Report"| Patient
    
    %% Medical Staff Flows
    MedicalStaff -->|"Login Request"| P1
    P1 -->|"Authentication Token"| MedicalStaff
    MedicalStaff -->|"Access Request + Claims"| P4
    P4 -->|"Permission Ticket"| MedicalStaff
    MedicalStaff -->|"Permission Ticket + Claims"| P4
    P4 -->|"RPT Token"| MedicalStaff
    MedicalStaff -->|"Resource Request + RPT"| P5
    P5 -->|"Medical Records"| MedicalStaff
    MedicalStaff -->|"Upload Medical Record"| P2
    
    %% Ministry Flows
    Ministry -->|"Audit Query"| P6
    P6 -->|"Audit Trail Report"| Ministry
    
    %% Process to Data Store Flows
    P1 -->|"Verify Credentials"| D1
    D1 -->|"User Info"| P1
    
    P2 -->|"Register Resource"| D2
    P2 -->|"Store Encrypted Data"| D5
    D2 -->|"Resource ID"| P2
    
    P3 -->|"Store Policy"| D3
    D3 -->|"Retrieve Policy"| P3
    P3 -->|"Query Resources"| D2
    
    P4 -->|"Validate Policy"| D3
    P4 -->|"Check Resource"| D2
    P4 -->|"Cache Token"| D4
    D4 -->|"Retrieve Token"| P4
    P4 -->|"Log Authorization"| D6
    
    P5 -->|"Validate Token"| D4
    P5 -->|"Retrieve Encrypted Data"| D5
    P5 -->|"Check Policy"| D3
    P5 -->|"Log Access"| D6
    
    P6 -->|"Query Logs"| D6
    D6 -->|"Audit Records"| P6
    
    %% Inter-Process Flows
    P1 -->|"Authenticated User"| P2
    P1 -->|"Authenticated User"| P3
    P1 -->|"Authenticated User"| P4
    P2 -->|"Resource Registered Event"| P3
    P3 -->|"Policy Rules"| P4
    P4 -->|"Authorization Decision"| P5
    P5 -->|"Access Event"| P6
    
    classDef entity fill:#e1f5ff,stroke:#01579b,stroke-width:2px
    classDef process fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    classDef datastore fill:#e0f2f1,stroke:#00695c,stroke-width:2px
    
    class Patient,MedicalStaff,Ministry entity
    class P1,P2,P3,P4,P5,P6 process
    class D1,D2,D3,D4,D5,D6 datastore
```

## 4. Sequence Diagram - Asynchronous Policy Definition & Access Flow

```mermaid
sequenceDiagram
    actor Patient as Pasien (RO)
    participant Portal as Consent Portal (PAP)
    participant AuthzSC as Authorization Server SC (PDP)
    participant PolicyDB as Policy Store (SC + Off-Chain)
    participant ResourceSC as Resource Registry SC
    actor Doctor as Dokter (RqP)
    participant HospitalApp as Hospital Client
    participant PEP as Proxy Re-encryption Server (PEP)
    participant IPFS as IPFS Storage
    
    Note over Patient,IPFS: Fase 1: Policy Definition (Asynchronous)
    Patient->>Portal: 1. Login
    Portal->>AuthzSC: Authenticate
    AuthzSC-->>Portal: Auth Token
    
    Patient->>Portal: 2. Define Access Policy<br/>(Dokter Spesialis Penyakit Dalam<br/>di RS Harapan, Read Access,<br/>Valid: 1 bulan)
    Portal->>ResourceSC: Query Available Resources
    ResourceSC-->>Portal: Resource List
    Portal->>AuthzSC: Create Policy
    AuthzSC->>PolicyDB: Store Policy (On-Chain Hash)
    AuthzSC->>PolicyDB: Store Metadata (Off-Chain)
    PolicyDB-->>AuthzSC: Policy ID
    AuthzSC-->>Portal: Policy Created
    Portal-->>Patient: Confirmation
    
    Note over Patient,IPFS: Fase 2: Resource Access (Days/Weeks Later)
    Doctor->>HospitalApp: 3. Request Patient Medical Record
    HospitalApp->>AuthzSC: Request Permission Ticket (AAT)
    AuthzSC->>ResourceSC: Verify Resource Exists
    ResourceSC-->>AuthzSC: Resource Info
    AuthzSC-->>HospitalApp: Permission Ticket
    
    HospitalApp->>AuthzSC: 4. Request RPT with Claims<br/>(NIK, Spesialisasi, RS ID)
    AuthzSC->>PolicyDB: Check Policy
    PolicyDB-->>AuthzSC: Policy Rules
    AuthzSC->>AuthzSC: Evaluate Claims vs Policy
    
    alt Policy Allows Access
        AuthzSC-->>HospitalApp: 5. Issue RPT Token
        HospitalApp->>PEP: 6. Access Resource + RPT
        PEP->>AuthzSC: Validate RPT
        AuthzSC-->>PEP: Valid
        PEP->>IPFS: 7. Retrieve Encrypted Data
        IPFS-->>PEP: Encrypted Medical Record
        PEP->>PEP: Re-encrypt for Doctor
        PEP-->>HospitalApp: Medical Record
        HospitalApp-->>Doctor: Display Record
        
        PEP->>AuthzSC: Log Access Event
        AuthzSC->>PolicyDB: Store Audit Trail
    else Policy Denies Access
        AuthzSC-->>HospitalApp: Access Denied
        HospitalApp-->>Doctor: Request Additional Authorization
    end
```

## 5. Komponen Utama UMA 2.0 dalam Arsitektur DecMed

### 5.1 Pemetaan Komponen UMA ke DecMed

| Komponen UMA 2.0                    | Implementasi DecMed                     | Teknologi                   |
| ----------------------------------- | --------------------------------------- | --------------------------- |
| Resource Owner (RO)                 | Pasien                                  | Mobile/Web App              |
| Resource Server (RS)                | Fasyankes/Hospital System               | Desktop Client + IPFS       |
| Authorization Server (AS)           | Smart Contract on IOTA                  | IOTA Smart Contract         |
| Requesting Party (RqP)              | Tenaga Medis/Fasyankes Lain             | Desktop/Mobile Client       |
| Policy Administration Point (PAP)   | Patient Consent Portal                  | Web/Mobile App              |
| Policy Decision Point (PDP)         | Authorization Server Smart Contract     | IOTA Smart Contract         |
| Policy Enforcement Point (PEP)      | Proxy Re-encryption Server              | REST Backend                |
| Policy Storage                      | Hybrid: On-Chain + Off-Chain            | Smart Contract + Database   |

### 5.2 Token Flow dalam Sistem

1. **PAT (Protection API Token)**: Diterbitkan untuk Resource Server (Fasyankes) saat mendaftarkan resource
2. **AAT (Authorization API Token)**: Diterbitkan untuk Requesting Party yang terautentikasi
3. **Permission Ticket**: Diterbitkan saat Requesting Party meminta akses ke resource
4. **RPT (Requesting Party Token)**: Token akhir yang memberikan akses setelah policy evaluation

### 5.3 Granularitas Akses

Patient dapat mendefinisikan policy dengan granularitas:

- **Data**: Specific FHIR resources (Observation, Medication, DiagnosticReport, dll)
- **Pihak**: Individual (NIK), Role (Dokter, Perawat), Organization (RS ID)
- **Aksi**: Read, Write, Delete
- **Waktu**: Start date, End date, Duration
- **Kondisi**: Emergency access, Referral code, Specialization

### 5.4 Desentralisasi

- **Authorization Server**: Smart Contract on IOTA (terdesentralisasi)
- **Resource Storage**: IPFS (decentralized storage)
- **Policy Metadata**: Off-chain database (dapat dipilih per fasyankes)
- **Audit Trail**: Immutable record on IOTA blockchain

## 6. Keunggulan Arsitektur

1. **Asynchronous Authorization**: Pasien dapat mendefinisikan policy di masa lalu tanpa harus hadir saat akses
2. **Granular Access Control**: Fine-grained policy dengan claims-based evaluation
3. **Decentralized**: Authorization logic di smart contract, data di IPFS
4. **Immutable Audit**: Semua akses tercatat di blockchain
5. **Scalable**: Off-chain metadata untuk policy yang kompleks
6. **Interoperable**: Mendukung standar FHIR dan UMA 2.0
7. **Privacy-Preserving**: Proxy re-encryption untuk data protection
8. **Cost-Effective**: Sponsored transaction via IOTA Gas Station
