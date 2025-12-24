# OAuth 2.0 Protocol Flow

## Abstract Protocol Flow

```mermaid
sequenceDiagram
    participant RO as Resource Owner
    participant C as Client
    participant AS as Authorization Server
    participant RS as Resource Server

    Note over C,RO: (A) Authorization Request
    C->>RO: 1. Request Authorization
    
    Note over RO,C: (B) Authorization Grant
    RO->>C: 2. Authorization Grant

    Note over C,AS: (C) Access Token Request
    C->>AS: 3. Request Access Token<br/>(Authorization Grant)
    
    Note over AS,C: (D) Access Token Response
    AS->>AS: 4. Authenticate Client &<br/>Validate Grant
    AS->>C: 5. Access Token

    Note over C,RS: (E) Protected Resource Request
    C->>RS: 6. Request Protected Resource<br/>(Access Token)
    
    Note over RS,C: (F) Protected Resource Response
    RS->>RS: 7. Validate Access Token
    RS->>C: 8. Protected Resource
```

## Penjelasan Alur

### (A) Authorization Request

Client meminta otorisasi dari Resource Owner. Permintaan otorisasi dapat dilakukan secara langsung kepada Resource Owner atau secara tidak langsung melalui Authorization Server sebagai perantara.

### (B) Authorization Grant

Client menerima authorization grant, yang merupakan kredensial yang mewakili otorisasi Resource Owner. Authorization grant dapat menggunakan salah satu dari empat tipe grant yang didefinisikan dalam spesifikasi OAuth 2.0 atau menggunakan extension grant type.

### (C) Access Token Request

Client meminta access token dengan mengautentikasi diri ke Authorization Server dan menyajikan authorization grant.

### (D) Access Token Response

Authorization Server mengautentikasi Client dan memvalidasi authorization grant. Jika valid, Authorization Server mengeluarkan access token.

### (E) Protected Resource Request

Client meminta protected resource dari Resource Server dan mengautentikasi diri dengan menyajikan access token.

### (F) Protected Resource Response

Resource Server memvalidasi access token, dan jika valid, melayani permintaan dengan memberikan protected resource.

## Komponen Utama

- **Authorization Grant**: Kredensial yang mewakili otorisasi Resource Owner
- **Access Token**: Token yang digunakan untuk mengakses protected resource
- **Client Authentication**: Proses autentikasi Client oleh Authorization Server
- **Token Validation**: Proses validasi access token oleh Resource Server
