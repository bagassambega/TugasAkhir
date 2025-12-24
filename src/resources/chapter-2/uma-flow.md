# UMA 2.0 Protocol Flow

## Abstract Protocol Flow

```mermaid
sequenceDiagram
    participant RO as Resource Owner
    participant RS as Resource Server
    participant AS as Authorization Server
    participant C as Client
    participant RqP as Requesting Party

    Note over RS,AS: 1. Resource Registration
    RS->>AS: 1. Register Protected Resource<br/>(Resource Metadata)
    AS->>RS: 2. Resource ID & Permission Ticket Endpoint

    Note over RO,AS: 2. Policy Definition
    RO->>AS: 3. Define Access Policy<br/>(Who, What, When)
    AS->>AS: 4. Store Policy

    Note over RqP,RS: 3. Resource Access Attempt
    RqP->>C: 5. Request Access
    C->>RS: 6. Request Resource<br/>(No Token)
    RS->>AS: 7. Request Permission Ticket
    AS->>RS: 8. Permission Ticket
    RS->>C: 9. Permission Ticket<br/>(+ AS Location)

    Note over C,AS: 4. Claims Gathering & Token Request
    C->>RqP: 10. Request Claims
    RqP->>C: 11. Provide Claims/Identity
    C->>AS: 12. Request RPT<br/>(Permission Ticket + Claims)
    AS->>AS: 13. Evaluate Policy & Claims
    
    alt Policy Satisfied
        AS->>C: 14a. Requesting Party Token (RPT)
    else Need More Claims
        AS->>C: 14b. Need Claims<br/>(Claims Required)
        C->>RqP: 15. Request Additional Claims
        RqP->>C: 16. Provide Claims
        C->>AS: 17. Retry with Claims
        AS->>C: 18. RPT
    end

    Note over C,RS: 5. Resource Access with RPT
    C->>RS: 19. Request Resource<br/>(with RPT)
    RS->>AS: 20. Validate RPT
    AS->>RS: 21. RPT Valid + Permissions
    RS->>C: 22. Protected Resource
    C->>RqP: 23. Deliver Resource
```

## Penjelasan Alur

### 1. Resource Registration

Resource Server mendaftarkan protected resource ke Authorization Server. Authorization Server mengembalikan resource ID dan endpoint untuk permission ticket yang akan digunakan dalam proses otorisasi selanjutnya.

### 2. Policy Definition

Resource Owner mendefinisikan kebijakan akses (policy) di Authorization Server, menentukan siapa (who), dapat mengakses apa (what), dan kapan (when). Authorization Server menyimpan kebijakan ini untuk evaluasi di masa mendatang.

### 3. Resource Access Attempt

Requesting Party mencoba mengakses resource melalui Client. Client meminta resource ke Resource Server tanpa token. Resource Server meminta permission ticket dari Authorization Server dan memberikannya ke Client bersama lokasi Authorization Server.

### 4. Claims Gathering & Token Request

Client meminta claims (bukti identitas/atribut) dari Requesting Party. Client kemudian meminta Requesting Party Token (RPT) dari Authorization Server dengan menyertakan permission ticket dan claims. Authorization Server mengevaluasi kebijakan yang telah didefinisikan Resource Owner terhadap claims yang diberikan.

Jika kebijakan terpenuhi, Authorization Server mengeluarkan RPT. Jika claims tidak cukup, Authorization Server meminta claims tambahan, dan proses diulang hingga kebijakan terpenuhi atau ditolak.

### 5. Resource Access with RPT

Client menggunakan RPT untuk mengakses protected resource dari Resource Server. Resource Server memvalidasi RPT dengan Authorization Server. Jika valid, Resource Server memberikan protected resource kepada Client, yang kemudian diteruskan ke Requesting Party.

## Perbedaan Utama dengan OAuth 2.0

1. **Asynchronous Authorization**: Resource Owner dapat mendefinisikan policy kapan saja, tidak harus saat access request
2. **Party-to-Party Sharing**: Fokus pada sharing antar individu/entitas, bukan user-to-app
3. **Claims-Based**: Menggunakan claims gathering untuk policy evaluation yang lebih granular
4. **Permission Ticket**: Mekanisme khusus untuk mengelola permission request
5. **Requesting Party Token (RPT)**: Token khusus yang membawa permission yang sudah dievaluasi

## Komponen Utama

- **Resource ID**: Identifier unik untuk protected resource
- **Permission Ticket**: Token sementara yang merepresentasikan permission request
- **Claims**: Atribut atau bukti identitas dari Requesting Party
- **Policy**: Aturan akses yang didefinisikan oleh Resource Owner
- **Requesting Party Token (RPT)**: Token yang membawa authorized permissions
