# UMA 2.0 Architecture Diagram

```mermaid
flowchart TD
    %% Definisikan Style
    classDef ownerStyle fill:#f9f,stroke:#333,stroke-width:2px
    classDef requestingStyle fill:#ffd700,stroke:#333,stroke-width:2px
    classDef serverStyle fill:#e1f5fe,stroke:#0277bd,stroke-width:2px
    classDef asStyle fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    classDef dataStyle fill:#fff3e0,stroke:#ef6c00,stroke-width:2px

    %% Actors
    RO((Resource Owner<br/>Pasien)):::ownerStyle
    RqP((Requesting Party<br/>Dokter)):::requestingStyle

    %% Components
    subgraph ClientLayer [Client Layer]
        C[Client Application]:::serverStyle
    end

    subgraph ResourceLayer [Resource Server Layer]
        RS[Resource Server<br/>RME System]:::serverStyle
        PDB[(Protected Resources<br/>Medical Data)]:::dataStyle
    end

    subgraph AuthorizationLayer [Authorization Server Layer]
        AS[Authorization Server]:::asStyle
        PolicyDB[(Policy Store)]:::dataStyle
        ResourceReg[(Resource Registry)]:::dataStyle
    end

    %% Alur Protokol dengan Nomor
    
    %% Phase 1: Resource Protection
    RS -->|1. Register Resource| AS
    AS -->|2. Resource ID| RS
    
    %% Phase 2: Policy Management
    RO -->|3. Define Policy| AS
    AS -->|4. Store Policy| PolicyDB
    AS -->|5. Policy Stored| RO
    
    %% Phase 3: Access Request
    RqP -->|6. Request Access| C
    C -->|7. Access Request<br/>(No Token)| RS
    RS -->|8. Check Registration| ResourceReg
    RS -->|9. Request Permission Ticket| AS
    AS -->|10. Permission Ticket| RS
    RS -->|11. Permission Ticket<br/>+ AS Location| C
    
    %% Phase 4: Token Request
    C -->|12. Request Claims| RqP
    RqP -->|13. Provide Claims/Identity| C
    C -->|14. Request RPT<br/>(Ticket + Claims)| AS
    AS -->|15. Check Policy| PolicyDB
    AS -->|16. Evaluate Claims| AS
    AS -->|17. RPT<br/>(if authorized)| C
    
    %% Phase 5: Resource Access
    C -->|18. Access with RPT| RS
    RS -->|19. Introspect RPT| AS
    AS -->|20. RPT Valid<br/>+ Permissions| RS
    RS -->|21. Fetch Data| PDB
    PDB -->|22. Data| RS
    RS -->|23. Protected Resource| C
    C -->|24. Deliver Resource| RqP
    
    %% Styling
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23 stroke-width:2px
```

## Penjelasan Arsitektur dan Alur Kerja

### Layer Arsitektur

1. **Client Layer**: Aplikasi yang digunakan oleh Requesting Party untuk mengakses resources
2. **Resource Server Layer**: Server yang menyimpan dan melindungi resources (RME System)
3. **Authorization Server Layer**: Server yang mengelola otorisasi, policy, dan registrasi resource

### Alur Kerja Detail

#### Phase 1: Resource Protection (Step 1-2)

1. Resource Server mendaftarkan protected resource ke Authorization Server
2. Authorization Server mengembalikan Resource ID untuk resource yang terdaftar

#### Phase 2: Policy Management (Step 3-5)

3. Resource Owner (Pasien) mendefinisikan policy akses ke Authorization Server
2. Authorization Server menyimpan policy ke Policy Store
3. Konfirmasi bahwa policy telah tersimpan

#### Phase 3: Access Request (Step 6-11)

6. Requesting Party (Dokter) meminta akses melalui Client
2. Client meminta resource ke Resource Server tanpa token
3. Resource Server mengecek registrasi resource
4. Resource Server meminta permission ticket dari Authorization Server
5. Authorization Server mengeluarkan permission ticket
6. Resource Server memberikan permission ticket dan lokasi AS ke Client

#### Phase 4: Token Request (Step 12-17)

12. Client meminta claims dari Requesting Party
2. Requesting Party menyediakan claims/identitas
3. Client meminta RPT dari Authorization Server dengan ticket dan claims
4. Authorization Server mengecek policy yang relevan
5. Authorization Server mengevaluasi claims terhadap policy
6. Jika authorized, Authorization Server mengeluarkan RPT

#### Phase 5: Resource Access (Step 18-24)

18. Client mengakses resource dengan RPT
2. Resource Server melakukan introspeksi RPT ke Authorization Server
3. Authorization Server memvalidasi RPT dan mengirim permissions
4. Resource Server mengambil data dari Protected Resources
5. Data dikirim ke Resource Server
6. Resource Server memberikan protected resource ke Client
7. Client menyampaikan resource ke Requesting Party

### Keunggulan Arsitektur UMA 2.0

- **Decoupled Authorization**: Policy definition terpisah dari access request
- **Centralized Policy Management**: Semua policy dikelola terpusat di AS
- **Fine-grained Access Control**: Policy dapat sangat granular berdasarkan claims
- **Asynchronous**: Resource Owner tidak perlu online saat access request
- **Audit Trail**: Semua akses tercatat untuk keperluan audit
