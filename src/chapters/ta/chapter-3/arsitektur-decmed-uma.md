# Arsitektur DecMed dengan Decentralized UMA

```mermaid
flowchart TD
    %% Definisikan Style
    classDef actorStyle fill:#f9f,stroke:#333,stroke-width:2px
    classDef componentStyle fill:#e1f5fe,stroke:#0277bd,stroke-width:2px
    classDef blockchainStyle fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,stroke-dasharray: 5 5
    classDef storageStyle fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px

    %% Actors
    Pasien((Resource Owner)):::actorStyle
    Dokter((Requesting Party)):::actorStyle

    %% Layer Aplikasi
    subgraph ClientLayer [Layer Aplikasi Client]
        PatientApp[Aplikasi Pasien / PAP]:::componentStyle
        DoctorApp[Aplikasi Dokter]:::componentStyle
    end

    %% Layer Blockchain / Decentralized AS
    subgraph IOTA_Network [Jaringan IOTA - Decentralized AS]
        direction TB
        SmartContract["Smart Contract AS<br/>(Distributed PDP)"]:::blockchainStyle
        Validator1((Node Validator 1))
        Validator2((Node Validator 2))
        Validator3((Node Validator 3))
        
        %% Hubungan Internal Blockchain
        SmartContract -.-> Validator1
        SmartContract -.-> Validator2
        SmartContract -.-> Validator3
        
        EventLog[("Event Log / Ledger State")]:::blockchainStyle
    end

    %% Layer Data / Enforcement
    subgraph DataLayer [Resource Server & Enforcement]
        PEP_Node[PEP Gateway / Node RS]:::componentStyle
        EncryptedDB[("Encrypted Health Data")]:::storageStyle
    end

    %% ALUR PROSES
    %% 1. Definisi Policy
    Pasien -->|1. Input Aturan Akses| PatientApp
    PatientApp -->|"2. Register Policy Transaction<br/>(TargetDID, ResourceID, Permission)"| SmartContract

    %% 2. Request Akses
    Dokter -->|3. Request Data| DoctorApp
    DoctorApp -->|"4. Call SC: RequestAccess()"| SmartContract

    %% 3. Verifikasi Terdistribusi
    SmartContract -->|"5. Konsensus Verifikasi Logic"| Validator1
    SmartContract -->|"5. Konsensus Verifikasi Logic"| Validator2
    SmartContract -->|"5. Konsensus Verifikasi Logic"| Validator3
    Validator1 -->|"6. Validasi Sukses"| EventLog
    Validator2 -->|"6. Validasi Sukses"| EventLog
    Validator3 -->|"6. Validasi Sukses"| EventLog

    %% 4. Enforcement
    EventLog -.->|"7. Emit Event: AccessGranted"| PEP_Node
    PEP_Node -->|8. Cek Event Signature| EventLog
    PEP_Node -->|9. Release Data/Key| DoctorApp
    DoctorApp -->|10. Decrypt & View| Dokter

    %% Koneksi Storage
    PEP_Node <--> EncryptedDB
```

## Penjelasan Komponen

### Actors
- **Resource Owner (Pasien)**: Pemilik data rekam medis yang mendefinisikan kebijakan akses
- **Requesting Party (Dokter)**: Pihak yang meminta akses ke data rekam medis

### Layer Aplikasi Client
- **Aplikasi Pasien/PAP**: Portal consent untuk pasien mengelola kebijakan akses
- **Aplikasi Dokter**: Interface untuk tenaga medis mengakses data pasien

### Jaringan IOTA - Decentralized AS
- **Smart Contract AS**: Authorization Server terdistribusi yang menjalankan Policy Decision Point (PDP)
- **Node Validator**: Node-node yang melakukan konsensus untuk verifikasi kebijakan
- **Event Log**: Ledger state yang menyimpan log akses dan keputusan otorisasi

### Resource Server & Enforcement
- **PEP Gateway**: Policy Enforcement Point yang menegakkan keputusan otorisasi
- **Encrypted Health Data**: Storage terenkripsi untuk data rekam medis

## Alur Kerja

1. **Definisi Policy**: Pasien mendefinisikan aturan akses melalui aplikasi
2. **Register Transaction**: Policy didaftarkan ke smart contract di IOTA
3. **Request Access**: Dokter meminta akses data melalui aplikasinya
4. **Smart Contract Verification**: Smart contract memverifikasi permintaan
5. **Distributed Consensus**: Node validator melakukan konsensus
6. **Log Event**: Hasil validasi dicatat di event log
7. **Emit Event**: Event aksess granted dikirim ke PEP
8. **Signature Check**: PEP memverifikasi signature event
9. **Release Data**: PEP memberikan akses/key ke dokter
10. **View Data**: Dokter dapat dekripsi dan melihat data
