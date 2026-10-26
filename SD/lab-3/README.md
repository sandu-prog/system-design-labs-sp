# Laboratorul 3 - Arhitectura sistemului

**Student:** Pușca Sandu  
**Grupa:** CIB 241  
**Limba:** Română

## Obiectiv

Pentru fiecare proiect sunt prezentate trei diagrame de arhitectura (System Context, Container, Component) si trei diagrame de secventa, cu fluxuri normale si alternative. Diagramele sunt in format Mermaid si pot fi vizualizate direct pe GitHub.

> Nota: Personal Investment Dashboard este tema principala. Student Skill Exchange este un proiect alternativ, a carui includere necesita acordul profesorului.

---

## 1. Personal Investment Dashboard

**Statut:** Tema principala  
**Tinta de capacitate din Laboratorul 2:** 10.296 RPS la varf, pentru 30.000 utilizatori simultani (estimare, nu masuratoare).

**Decizie arhitecturala:** Colectarea datelor de piata este separata de citirile utilizatorilor. Datele invalide nu suprascriu valorile acceptate.

### 1.1. System Context

Prezinta utilizatorul autentificat, sistemul Dashboard si furnizorul extern de date de piata.

```mermaid
graph LR
U[Utilizator autentificat]
D[Personal Investment Dashboard]
P[Market Data Provider]
U --> D
D --> U
D --> P
P --> D
```

### 1.2. Container

Web Application gestioneaza interfata, Dashboard Application cererile, Market Data Store preturile, User Data Store datele private, iar Market Data Collector actualizeaza preturile periodic.

```mermaid
graph LR
U[Utilizator autentificat]
P[Market Data Provider]
subgraph Dashboard
W[Web Application]
A[Dashboard Application]
M[Market Data Store]
S[User Data Store]
C[Market Data Collector]
end
U --> W
W --> A
A --> M
A --> S
C --> M
C --> P
P --> C
```

### 1.3. Component

Descompune Dashboard Application in IdentityService, StockCatalogService, StockPriceService, PriceHistoryService si WatchlistService.

```mermaid
graph LR
W[Web Application]
M[Market Data Store]
S[User Data Store]
subgraph Dashboard_Application
I[IdentityService]
C[StockCatalogService]
P[StockPriceService]
H[PriceHistoryService]
L[WatchlistService]
end
W --> C
W --> P
W --> H
W --> L
C --> I
P --> I
H --> I
L --> I
I --> S
C --> M
P --> M
H --> M
L --> S
```

### 1.4. Secventa A - Sincronizare si publicare

Collectorul preia periodic observatii de la furnizor, valideaza datele si le salveaza atomic. Sunt tratate date invalide, erori de stocare si rate limit.

```mermaid
sequenceDiagram
participant C as Market Data Collector
participant P as Market Data Provider
participant M as Market Data Store
Note over C: Sincronizare la fiecare 60 secunde
C->>P: Solicita datele financiare
alt Date valide
P-->>C: Returneaza observatiile
C->>C: Valideaza simbolul, pretul si timpul
C->>M: Salveaza atomic datele validate
alt Salvare reusita
M-->>C: Confirmare commit
Note over M: Datele noi sunt disponibile
else Eroare la salvare
M-->>C: Eroare si rollback
Note over M: Datele anterioare raman salvate
end
else Date invalide
P-->>C: Returneaza date invalide
C->>C: Respinge datele si inregistreaza eroarea
Note over M: Se pastreaza datele anterioare
else Timeout sau limita de cereri
P-->>C: Timeout sau eroare 429
C->>C: Asteapta inainte de reincercare
Note over M: Datele existente pot deveni invechite
end
```

### 1.5. Secventa B - Citire si cautare

Utilizatorul cere pretul unui simbol. Serviciul verifica sesiunea si simbolul, apoi citeste doar din Market Data Store; datele vechi nu sunt prezentate drept actuale.

```mermaid
sequenceDiagram
actor U as Utilizator
participant W as Web Application
participant P as StockPriceService
participant I as IdentityService
participant M as Market Data Store
U->>W: Cere pretul unei actiuni
W->>P: Solicita pretul
P->>I: Verifica identitatea
alt Sesiune invalida
I-->>P: Neautentificat
P-->>W: Eroare 401
W-->>U: Autentificare necesara
else Sesiune valida
I-->>P: User ID
P->>P: Valideaza simbolul
alt Simbol invalid
P-->>W: Eroare 400
W-->>U: Simbol invalid
else Simbol valid
P->>M: Citeste ultimul pret
alt Stocare indisponibila
M-->>P: Eroare
P-->>W: Eroare 503
W-->>U: Serviciu indisponibil
else Pret lipsa
M-->>P: Fara observatie
P-->>W: Pret indisponibil
W-->>U: Pret indisponibil
else Pret mai vechi de 15 minute
M-->>P: Observatie expirata
P-->>W: Date invechite
W-->>U: Pretul nu este actual
else Pret valid
M-->>P: Pret si ora furnizorului
P-->>W: Pret si intarziere
W-->>U: Afiseaza pretul si ora
end
end
end
Note over P,M: Citirea nu contacteaza furnizorul extern
```

### 1.6. Secventa C - Modificare privata

Utilizatorul modifica watchlist-ul propriu. Sunt verificate sesiunea, datele si proprietarul; succesul este comunicat numai dupa commit.

```mermaid
sequenceDiagram
actor U as Utilizator
participant W as Web Application
participant L as WatchlistService
participant I as IdentityService
participant S as User Data Store
U->>W: Adauga actiune in Watchlist
W->>L: Trimite cererea
L->>I: Verifica sesiunea
alt Sesiune invalida
I-->>L: Neautentificat
L-->>W: Eroare 401
W-->>U: Autentificare necesara
else Sesiune valida
I-->>L: User ID
L->>L: Valideaza Stock ID si operatia
alt Date invalide
L-->>W: Eroare 400
W-->>U: Cerere invalida
else Date valide
L->>S: Verifica proprietarul si incepe tranzactia
alt Proprietar diferit
S-->>L: Acces interzis
L-->>W: Eroare 403
W-->>U: Acces refuzat
else Eroare de stocare
S-->>L: Eroare
L-->>W: Eroare 503
W-->>U: Modificare neconfirmata
else Proprietar valid
S-->>L: Proprietar confirmat
L->>S: Aplica modificarea si commit
alt Commit esuat
S-->>L: Rollback sau rezultat incert
L-->>W: Succes neconfirmat
W-->>U: Verifica starea inainte de retry
else Commit confirmat
S-->>L: Commit reusit
Note over L,S: Succesul este confirmat dupa commit
L-->>W: Status 200
W-->>U: Modificare salvata
end
end
end
end
Note over U,S: Citirile urmatoare reflecta modificarea salvata
```

---

## 2. Student Skill Exchange

**Statut:** Proiect alternativ, supus acceptarii profesorului.  
**Tinta de capacitate din Laboratorul 2:** 5.016 RPS la varf, pentru 30.000 de utilizatori simultani (estimare de proiectare).

**Decizie arhitecturala:** Profilurile canonice sunt separate de indexul de cautare. Operatiile private sunt autorizate si confirmate numai dupa commit.

### 2.1. System Context

Studentul si moderatorul interactioneaza cu Student Skill Exchange.

```mermaid
graph LR
    A[Student] --> C[Student Skill Exchange]
    B[Moderator] --> C
    C --> A
    C --> B
```

### 2.2. Container

Web Application trimite cererile catre Skill Exchange Application. Datele sunt salvate in Exchange Data Store, iar Search Index Worker actualizeaza asincron Skill Search Index.

```mermaid
graph LR
    S[Student]
    M[Moderator]
    subgraph Exchange
        W[Web Application]
        A[Skill Exchange Application]
        D[Exchange Data Store]
        I[Skill Search Index]
        J[Search Index Worker]
    end
    S --> W
    M --> W
    W --> A
    A --> D
    A --> I
    A --> J
    J --> I
```

### 2.3. Component

Skill Exchange Application contine IdentityService, StudentProfileService, SkillSearchService, ExchangeRequestService si ModerationService.

```mermaid
graph LR
    W[Web Application]
    D[Exchange Data Store]
    I[Skill Search Index]
    J[Search Index Worker]
    subgraph Skill_Exchange_Application
        Auth[IdentityService]
        Profile[StudentProfileService]
        Search[SkillSearchService]
        Request[ExchangeRequestService]
        Report[ModerationService]
    end
    W --> Profile
    W --> Search
    W --> Request
    W --> Report
    Profile --> Auth
    Search --> Auth
    Request --> Auth
    Report --> Auth
    Auth --> D
    Profile --> D
    Request --> D
    Report --> D
    Search --> I
    Profile --> J
```

### 2.4. Secventa A - Sincronizare si publicare

Actualizarea profilului se salveaza atomic impreuna cu evenimentul pentru indexare. Indexarea ulterioara poate intarzia si poate fi reincercata.

```mermaid
sequenceDiagram
    actor S as Student
    participant W as Web Application
    participant P as StudentProfileService
    participant A as IdentityService
    participant D as Exchange Data Store
    participant J as Search Index Worker
    participant I as Skill Search Index
    S->>W: Modifica abilitatile
    W->>P: Salveaza profilul
    P->>A: Verifica sesiunea
    alt Sesiune invalida
        A-->>P: Acces refuzat
        P-->>W: Eroare 401
        W-->>S: Autentificare necesara
    else Sesiune valida
        A-->>P: Student ID
        P->>P: Valideaza datele
        alt Date invalide
            P-->>W: Eroare 400
            W-->>S: Corecteaza datele
        else Date valide
            P->>D: Salveaza profilul si evenimentul
            alt Eroare la salvare
                D-->>P: Rollback
                P-->>W: Eroare 503
                W-->>S: Modificare neconfirmata
            else Salvare reusita
                D-->>P: Confirmare commit
                P-->>W: Profil actualizat
                W-->>S: Salvare confirmata
                J->>D: Citeste evenimentul
                J->>I: Actualizeaza indexul
                alt Indexare reusita
                    I-->>J: Confirmare
                else Index indisponibil
                    I-->>J: Eroare
                    J->>J: Reincearca indexarea
                end
            end
        end
    end
```

### 2.5. Secventa B - Citire si cautare

Studentul cauta parteneri folosind filtre validate. Sistemul verifica autentificarea, disponibilitatea indexului si accesul la profiluri.

```mermaid
sequenceDiagram
    actor S as Student
    participant W as Web Application
    participant Q as SkillSearchService
    participant A as IdentityService
    participant I as Skill Search Index
    participant D as Exchange Data Store
    S->>W: Cauta Python si disponibilitate
    W->>Q: Trimite filtrele
    Q->>A: Verifica identitatea
    alt Sesiune invalida
        A-->>Q: Neautentificat
        Q-->>W: Eroare 401
        W-->>S: Autentificare necesara
    else Sesiune valida
        A-->>Q: Student ID
        Q->>Q: Valideaza filtrele
        alt Filtre invalide
            Q-->>W: Eroare 400
            W-->>S: Filtre invalide
        else Filtre valide
            Q->>I: Cauta profiluri relevante
            alt Index indisponibil
                I-->>Q: Eroare
                Q-->>W: Eroare 503
                W-->>S: Cautare indisponibila
            else Index intarziat
                I-->>Q: Rezultate posibil invechite
                Q-->>W: Rezultate si avertisment
                W-->>S: Afiseaza avertisment
            else Index actualizat
                I-->>Q: ID profiluri
                Q->>D: Citeste profilurile
                alt Stocare indisponibila
                    D-->>Q: Eroare
                    Q-->>W: Eroare 503
                    W-->>S: Rezultate indisponibile
                else Citire reusita
                    D-->>Q: Profiluri vizibile
                    Q-->>W: Rezultate filtrate
                    W-->>S: Afiseaza partenerii
                end
            end
        end
    end
```

### 2.6. Secventa C - Modificare privata

Acceptarea unei cereri necesita sesiune valida, destinatar corect si stare pending. Succesul este confirmat numai dupa commit.

```mermaid
sequenceDiagram
    actor S as Student
    participant W as Web Application
    participant R as ExchangeRequestService
    participant A as IdentityService
    participant D as Exchange Data Store
    S->>W: Accepta cererea primita
    W->>R: Accept requestId
    R->>A: Verifica sesiunea
    alt Sesiune invalida
        A-->>R: Neautentificat
        R-->>W: Eroare 401
        W-->>S: Autentificare necesara
    else Sesiune valida
        A-->>R: Student ID
        R->>R: Valideaza cererea
        alt Date invalide
            R-->>W: Eroare 400
            W-->>S: Cerere invalida
        else Date valide
            R->>D: Verifica destinatarul si starea
            alt Studentul nu este destinatarul
                D-->>R: Acces interzis
                R-->>W: Eroare 403
                W-->>S: Acces refuzat
            else Cererea nu este pending
                D-->>R: Conflict de stare
                R-->>W: Eroare 409
                W-->>S: Cerere deja procesata
            else Stocare indisponibila
                D-->>R: Eroare
                R-->>W: Eroare 503
                W-->>S: Rezultat neconfirmat
            else Cerere eligibila
                D-->>R: Operatie permisa
                R->>D: Actualizeaza statusul la accepted
                alt Commit esuat
                    D-->>R: Rollback
                    R-->>W: Eroare la salvare
                    W-->>S: Verifica starea inainte de retry
                else Commit confirmat
                    D-->>R: Salvare confirmata
                    R-->>W: Status 200 accepted
                    W-->>S: Cerere acceptata
                end
            end
        end
    end
```

---

## Concluzie

Ambele proiecte separa responsabilitatile intre interfata, logica de aplicatie si stocare. Diagramele de secventa descriu atat traseele reusite, cat si validarea intrarilor, autorizarea si comportamentul in caz de eroare.
