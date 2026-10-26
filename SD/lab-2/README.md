# Laboratorul 2 — Cerințe de calitate, workload și capacitate

**Student:** Pușca Sandu  
**Grupa:** CIB 241  
**Limba:** Română

Acest laborator documentează două produse: **Personal Investment Dashboard** (tema principală) și **Student Skill Exchange** (proiect alternativ, supus acceptării profesorului). Valorile de trafic și stocare sunt estimări de proiectare, nu rezultate de test.

---

## Personal Investment Dashboard

Continuare a Laboratorului 1 • Data analizei: 19 octombrie 2026 • Toate capacitățile și dimensiunile sunt estimări de proiectare, nu rezultate de teste.

### 1. Cerințe de calitate măsurabile

Domeniu: Overview, Filter, Stock price, History, Watchlist privat și Search. Actor direct: utilizator autentificat. Dependență externă: Market Data Provider. O cerere indisponibilă este distinctă de un rezultat corect; un preț lipsă nu devine niciodată zero.

1.1 Latență

Măsură: durata dintre recepționarea cererii HTTP de către Dashboard și expedierea completă a răspunsului HTTP, măsurată server-side; pentru experiența completă se monitorizează separat și timpul perceput în browser. Țintă: p95 ≤ 2.000 ms pentru fiecare tip de citire validă, inclusiv în fereastra de vârf de la deschiderea pieței și la încărcarea proiectată de până la 30.000 utilizatori concurenți. Exact 2.000 ms este conform; > 2.000 ms nu este. Erorile, timeout-urile și răspunsurile «indisponibil» se contabilizează separat pentru corectitudine/disponibilitate; latența lor se măsoară de asemenea, fără a le considera citiri corecte.

1.2 Disponibilitate

Dashboard utilizabil = un utilizator autentificat poate deschide Overview, căuta și citi Watchlist-ul propriu, cu date valide sau cu o stare explicită «delayed/unavailable»; erorile 5xx, indisponibilitatea autentificării și lipsa accesului la funcțiile de bază sunt indisponibilitate. O etichetă de date întârziate nu transformă o informație nevalidă în date corecte.

Ore de tranzacționare: luni–vineri, 09:30–16:00 America/New_York, în zilele cu sesiune normală Nasdaq; sărbătorile și sesiunile scurtate sunt tratate după calendarul bursei. Restul timpului aparține intervalului «în afara orelor». Pentru bugetul demonstrativ de 30 zile presupunem 22 sesiuni normale × 6,5 h = 143 h; restul = 720 − 143 = 577 h. Numărul efectiv de ore trebuie recalculat pentru fereastra calendaristică testată.

| Interval | Țintă de uptime | Ore / 30 zile | Buget indisponibilitate |
| --- | --- | --- | --- |
| Trading hours | 99,9% (3 nouari) | 143 h | 0,143 h = 8,58 min |
| În afara sesiunii | 99,5% | 577 h | 2,885 h = 173,1 min |

Justificare: în timpul tranzacționării deciziile sunt mai sensibile la întreruperi, de aceea se impune 99,9%; în afara sesiunii sunt acceptabile ferestre de mentenanță mai mari. Calcul: downtime = durata intervalului × (1 − uptime). Se măsoară separat, prin probe sintetice la fiecare minut și monitorizarea cererilor reale; se publică procentul pe fiecare fereastră de 30 zile.

1.3 Consistență și prospețimea datelor

Stock price: afișăm provider_timestamp și întârzierea calculată față de ceasul de referință. În sesiunea activă, un preț cu vechime ≤ 15 minute este acceptat și etichetat «delayed» (nu «real-time»); > 15 minute este «unavailable» pentru cotația curentă. Dacă timestamp lipsește sau este invalid, prețul curent este indisponibil. În afara sesiunii, ultima cotație de închidere este permisă numai cu eticheta «last close» și timestamp, nu ca preț live. Această regulă este o alegere de produs pe baza întârzierii așteptate de circa 15 minute, nu o garanție contractuală a furnizorului.

Watchlist: read-your-writes. După confirmarea cu succes a adăugării/ștergerii, următoarea citire a aceluiași utilizator trebuie să reflecte modificarea (100% dintre citirile reușite), inclusiv după refresh; alt utilizator nu poate citi sau modifica lista (0 accesări neautorizate acceptate). Modificările neconfirmate nu se prezintă ca salvate.

1.4 Debit

Țintă: serviciul trebuie să susțină ≥ 10.296 cereri de citire/s în scenariul de 30.000 utilizatori concurenți, incluzând burst-ul de deschidere și marja de 10%, menținând p95 ≤ 2 s pentru citirile corecte. Se validează prin test de încărcare de minimum 10 minute la vârf, fără presupunerea că fiecare cerere produce o cerere către furnizor.

### 2. Estimări RPS în regim stabil

Formula pentru fiecare citire: RPS = utilizatori concurenți × pondere participanți × acțiuni per utilizator / secunde. Rezultatele sunt nerotunjite; toate cererile merg către Dashboard.

| Citire | Calcul (N utilizatori) | 300: RPS | 3.000: RPS | 30.000: RPS |
| --- | --- | --- | --- | --- |
| Overview | N × 0,70 / 30 | 7 | 70 | 700 |
| Filter | N × 0,50 × 3 / 60 | 7,5 | 75 | 750 |
| Stock price | N × 0,20 / 1 | 60 | 600 | 6.000 |
| History | N × 0,20 / 300 | 0,2 | 2 | 20 |
| Watchlist | N × 0,60 / 60 | 3 | 30 | 300 |
| Search | N × 0,10 × 3 / 60 | 1,5 | 15 | 150 |
| TOTAL | Suma rândurilor | 79,2 | 792 | 7.920 |

Exemplu: History la 300 utilizatori = 300 utilizatori × 0,20 × (1 cerere / 300 s) = 0,2 cereri/s. Pentru Stock price la 30.000 utilizatori = 30.000 × 0,20 × 1/s = 6.000 RPS.

### 3. Estimări RPS la deschiderea pieței

Traficul suplimentar este ADĂUGAT la traficul stabil, fără a recalcula sau dubla separat componentele deja incluse în totalul stabil. Overview suplimentar = N × 0,30 / 10 s; Watchlist suplimentar = N × 0,30 × 0,60 / 10 s. Marja = 10% × subtotal; numai ținta finală se rotunjește în sus.

| Calcul | 300 | 3.000 | 30.000 |
| --- | --- | --- | --- |
| Stabil (RPS) | 79,2 | 792 | 7.920 |
| Overview suplimentar | 9 | 90 | 900 |
| Watchlist suplimentar | 5,4 | 54 | 540 |
| Subtotal (RPS) | 93,6 | 936 | 9.360 |
| Marjă 10% (RPS) | 9,36 | 93,6 | 936 |
| Subtotal + marjă | 102,96 | 1.029,6 | 10.296 |
| ȚINTĂ FINALĂ (RPS) | 103 | 1.030 | 10.296 |

Exemplu 300: 79,2 + (300 × 0,30 / 10) + (300 × 0,30 × 0,60 / 10) = 93,6 RPS; 93,6 × 1,10 = 102,96 → ceil = 103 RPS. Modelul este pentru traficul către Dashboard; traficul către Market Data Provider depinde de cache, batching, licență și strategia de sincronizare și NU este estimat aici.

### 4. Domeniul Stock și estimarea stocării

4.1 Definiție și selecția instrumentelor

Un stock este o participație de proprietate într-o companie, conform Investor.gov [S1]. Pentru această primă versiune, «Stock» = o componentă de tip common stock / common-type equity a indicelui Nasdaq Composite, identificată prin simbol + piață + identificator intern. Acoperire: SUA, Nasdaq Stock Market, doar componentele eligibile ale Nasdaq Composite; excludem preferred stock, ADR-uri care nu intră în selecția definită, ETF-uri, fonduri, obligațiuni, opțiuni și criptomonede. Instrumentele delistate/inactive se păstrează ca referințe istorice numai dacă au fost anterior acceptate, dar nu mai apar în căutarea pentru noi selecții și nu primesc cotație curentă.

La 16 octombrie 2026, pagina Nasdaq Composite indică 3.397 componente [S2]. Adoptăm N = 3.397 Stocks ca snapshot de capacitate la acea dată. Este numărul componentelor indicelui, nu numărul tuturor listărilor Nasdaq; compoziția poate varia și trebuie reverificată la sincronizare. Nu se susține că toate componentele sunt instrumente juridic identice cu «US domestic common stock»; eligibilitatea finală se validează din metadata furnizorului.

4.2 Date sincronizate și retenție

| Set | Câmpuri și motiv | Frecvență / retenție |
| --- | --- | --- |
| Stock reference | stock_id, symbol, name, exchange, currency, sector, status, last_verified; Search și Filter | Sincronizare zilnică; active + arhivă delistate |
| Latest price | stock_id, price, provider_timestamp, fetched_at, status; Overview și Stock price | Sincronizare presupusă la 60 s în sesiune; 1 rând/Stock |
| Price history | stock_id, interval_start, close_price, provider_timestamp, quality_flag; grafic History | Puncte la 15 min × 26/sesiune; 252 sesiuni (~1 an bursier) |
| Other selected | stock_id, previous_close, currency, updated_at; variația față de închidere | Actualizare la închidere; 1 rând/Stock |

Ipoteze: 6,5 h × 60 / 15 = 26 puncte pe zi de tranzacționare; 252 sesiuni bursiere/an. Istoricul se păstrează pentru 252 zile de tranzacționare, nu 252 zile calendaristice. Prețul de închidere și sectorul sunt suficiente pentru scopul V1; nu stocăm order book, tranzacții, știri sau toate câmpurile furnizorului.

4.3 Exemplu de înregistrare și octeți

Eșantion logic History: {stock_id: 1234, interval_start: "2026-10-16T14:30:00Z", close_price: 250.35, provider_timestamp: "2026-10-16T14:29:00Z", quality_flag: 1}. Model binar estimat: stock_id 8 B + interval_start 8 B + close_price 8 B + provider_timestamp 8 B + quality_flag 1 B + padding/overhead 31 B = 64 B/record. Celelalte medii estimate: reference 160 B, latest price 48 B, other 32 B; nu sunt dimensiuni măsurate într-o bază de date reală.

| Set de date | Decizie / retenție | Număr de rânduri | B / rând | Stocare brută |
| --- | --- | --- | --- | --- |
| Stock reference | 3.397 active; snapshot inițial | 3 397 | 160 | 543 520 B |
| Latest prices | 1 per Stock | 3 397 | 48 | 163 056 B |
| Price history | 26/zi × 252 zile | 22 257 144 | 64 | 1 424 457 216 B |
| Other selected | Previous close, 1 per Stock | 3 397 | 32 | 108 704 B |
| TOTAL după 1 an | Fără replici/indecși/backups | — | — | 1 425 272 496 B |

Inițial (fără istoric): 815 280 B = 0.815 MB. Creștere brută/zi de tranzacționare: 3.397 × 26 × 64 = 5 652 608 B = 5.653 MB. După 252 sesiuni (~1 an): 1 425 272 496 B = 1.425 GB = 1.327 GiB. La retenție glisantă, istoricul ajunge la acest ordin de mărime și se stabilizează; metadatele se pot modifica.

Atenție: estimarea este pentru payload brut reprezentativ; exclude indexuri, pagini DB, WAL, replici, backupuri, compresie, cost de obiecte/JSON și datele utilizatorilor/Watchlist. Spațiul fizic real se măsoară ulterior cu schema și motorul alese. Dacă păstrăm delistate istorice, creșterea lor necesită buget separat.

### 5. Posibile blocaje și plan de validare

| Calitate | Blocaj posibil | Dovezi / efect | Măsurare de confirmare |
| --- | --- | --- | --- |
| Latență | Calea Stock price: cozi/cache miss/DB | 6.000 RPS stabil la 30k; p95 poate depăși 2 s | p50/p95/p99 pe endpoint; cache hit; timp DB; queue wait la 10.296 RPS |
| Consistență | Date întârziate la Market Data Provider; replicare Watchlist | Țintă preț ≤15 min; următoarea citire Watchlist trebuie să reflecte scrierea | Distribuție age(provider_timestamp); % unavailable; read-after-write test |
| Debit | Saturație procesare cereri și conexiuni | Vârf calculat 10.296 RPS; nu dovedește automat un blocaj | Load test 10 min, throughput susținut, CPU, memorie, conexiuni, 429/5xx |
| Disponibilitate | Dependență externă, autentificare, eșec de serviciu | 99,9% în sesiune => doar 8,58 min/30 zile; degradare poate afecta funcțiile | Sonde sintetice per minut; uptime separat; error budget; scenarii provider outage |

Interpretare: cifrele indică traseele de investigat, nu demonstrează existența unui blocaj. Un rezultat «unavailable» în 100 ms poate respecta latența, dar poate încălca prospețimea/funcționalitatea. Separăm în rapoarte răspunsurile corecte, întârziate, indisponibile și erorile.

### 6. Verificare finală și ipoteze

✓ Patru calități: latență, disponibilitate, consistență, debit. ✓ 2 ferestre de uptime și bugete/30 zile. ✓ RPS pentru 6 citiri × 3 niveluri. ✓ Vârf cu marjă 10% și rotunjire doar la final. ✓ Definiție Stock și număr datat. ✓ Stocare inițială, pe zi, după 1 an. ✓ 4 ipoteze de blocaj cu măsurători.

Ipoteze importante: 22 sesiuni normale/30 zile; 252 sesiuni/an; 3.397 componente ca snapshot; 15 min întârziere acceptată, 60 s sincronizare propusă; medii 160/48/64/32 B. Toate trebuie confirmate prin date reale, condițiile furnizorului și teste de încărcare. Nu se presupune raport 1:1 între cererile utilizatorilor și cererile către furnizor.

Surse

[S1] Investor.gov, «Stock», accesat 19.10.2026: https://www.investor.gov/introduction-investing/investing-basics/glossary/stock

[S2] Nasdaq, «Nasdaq Composite Index — Overview», număr de componente 3.397, date as of 16.10.2026, accesat 19.10.2026: https://indexes.nasdaq.com/Index/Overview/COMP

[S3] NasdaqTrader, «Symbol Directory Definitions», accesat 19.10.2026: https://nasdaqtrader.com/Trader.aspx?id=SymbolDirDefs

[S4] Enunțul Laboratorului 2 și Lecture 3: https://github.com/metreniuk/system-design-labs/blob/main/lectures/lecture-03-quality-requirements-workload-and-capacity.md

---

## Student Skill Exchange

### 0. Domeniul produsului și comportamentul analizat

Actorii direcți sunt Studentul autentificat și Moderatorul. Studentul publică abilități oferite și dorite, caută parteneri, vede profiluri, trimite cereri de schimb, consultă cererile proprii și disponibilitatea. Moderatorul consultă raportările. V1 nu procesează plăți, nu oferă certificări și nu depinde de integrări externe obligatorii.

Pentru comparație cu modelul Lecției 3, păstrăm aceleași trei niveluri de concurență: 300, 3.000 și 30.000 de studenți. În locul deschiderii bursei modelăm vârful de trafic de la începutul semestrului.

### 1. Cerințe de calitate măsurabile

| Calitate | Măsură + țintă + condiție |
| --- | --- |
| Latență | Timp de la recepția HTTP până la trimiterea completă a răspunsului; p95 ≤ 2,00 s pentru citiri valide în ora de vârf; exact 2,00 s este acceptat. Răspunsurile indisponibile/eroare sunt urmărite separat. |
| Disponibilitate | Utilizabil = studentul autentificat poate căuta parteneri, citi profiluri și vedea cererile proprii, sau primește stare explicită de indisponibilitate a unei funcții. 99,9% în intervalul de vârf 08:00–22:00 și 99,5% în 22:00–08:00, ora Europe/Chisinau, măsurate separat pe 30 de zile. |
| Consistență | Read-your-writes: după confirmarea actualizării profilului ori acceptării unei cereri, următoarea citire a aceluiași utilizator reflectă schimbarea în 100% dintre cazuri; o cerere nu poate fi simultan acceptată și respinsă. |
| Prospețime | Rezultatele căutării pot avea index întârziat ≤ 60 s de la o schimbare confirmată; peste 60 s, sistemul semnalează că indexul nu este actualizat și oferă acces la profilul canonic. |
| Debit | Sistemul suportă ținta finală de vârf pentru 30.000 utilizatori, 5.016 RPS, cu p95 ≤ 2 s și rată de eroare tehnică < 1%, în test de sarcină de 15 minute. |

Buget indisponibilitate, 30 zile: vârf 14 h/zi × 30 = 420 h; (1−0,999) × 420 = 0,42 h = 25,2 minute. Rest 10 h/zi × 30 = 300 h; (1−0,995) × 300 = 1,5 h = 90 minute. Acestea sunt bugete distincte, nu un procent unic.

### 2. Estimări RPS în regim stabil

Ipoteze de comportament alese pentru acest produs (nu sunt măsurători):

| Citire | Pondere | Frecvență | Formula per utilizator |
| --- | --- | --- | --- |
| Feed / recomandări | 70% | 1 acțiuni / 30 s | 0.7 × 1/30 |
| Filtrare abilități | 50% | 3 acțiuni / 60 s | 0.5 × 3/60 |
| Profil partener | 20% | 1 acțiuni / 5 s | 0.2 × 1/5 |
| Detalii cereri | 20% | 1 acțiuni / 300 s | 0.2 × 1/300 |
| Cereri primite / trimise | 60% | 1 acțiuni / 60 s | 0.6 × 1/60 |
| Căutare | 10% | 3 acțiuni / 60 s | 0.1 × 3/60 |
| Citire | 300 utilizatori | 3.000 utilizatori | 30.000 utilizatori |
| Feed / recomandări | 7.00 | 70.00 | 700.00 |
| Filtrare abilități | 7.50 | 75.00 | 750.00 |
| Profil partener | 12.00 | 120.00 | 1 200.00 |
| Detalii cereri | 0.20 | 2.00 | 20.00 |
| Cereri primite / trimise | 3.00 | 30.00 | 300.00 |
| Căutare | 1.50 | 15.00 | 150.00 |
| TOTAL RPS | 31.20 | 312.00 | 3 120.00 |

Formula: RPS = utilizatori concurenți × pondere participantă × acțiuni / secunde. Exemplu: Profil partener, 300 utilizatori = 300 × 0,20 × 1/5 = 12 RPS. Valorile sunt prezentate înainte de orice rotunjire a capacității.

### 3. Estimări RPS la început de semestru

Scenariu suplimentar: 30% dintre utilizatori reîmprospătează Feed o dată în 10 secunde; 60% din acest grup deschid și lista cererilor în aceleași 10 secunde. Aceste acțiuni se adaugă traficului stabil; nu îl înlocuiesc. Se adaugă 10% rezervă; numai ținta finală se rotunjește în sus.

| Calcul | 300 | 3.000 | 30.000 |
| --- | --- | --- | --- |
| Trafic stabil | 31.20 | 312.00 | 3120.00 |
| Feed suplimentar | 9.00 | 90.00 | 900.00 |
| Cereri suplimentare | 5.40 | 54.00 | 540.00 |
| Subtotal | 45.60 | 456.00 | 4560.00 |
| Rezervă 10% | 4.56 | 45.60 | 456.00 |
| Țintă finală (ceil) | 51 | 502 | 5016 |

### 4. Estimarea stocării

În Student Skill Exchange nu există „Stock”, furnizor de prețuri sau istoric bursier. Înlocuim stocarea de piață cu date de profil, abilități, cereri și feedback, deoarece acestea susțin fluxurile din Laboratorul 1. Numerele de mai jos sunt presupuneri explicite, nu statistici despre piață.

Model: 30.000 studenți înregistrați inițial; fiecare are în medie 5 abilități declarate; 2 cereri de schimb noi pe student pe lună; 1 feedback pentru fiecare cerere finalizată, cu rată de finalizare 50%; 12 luni de retenție pentru cereri și feedback; profilurile se păstrează cât timp contul este activ. Pentru estimare, un an = 12 luni.

Exemple logice de înregistrări: student={id:123, display_name:"Ana", university:"...", availability:"weekend", status:"active"}; skill={student_id:123, skill:"Python", direction:"teach"}; request={id:987, from:123, to:456, status:"pending", created_at:"..."}; feedback={request_id:987, rating:5, text:"..."}. Dimensiunile de mai jos sunt medii presupuse pentru reprezentarea brută serializată, nu mărimi măsurate ale bazei de date.

| Set de date | Număr înregistrări | Bytes / înregistrare | Stocare brută (B) |
| --- | --- | --- | --- |
| Profil student | 30.000 | 512 | 15.360.000 |
| Abilități declarate | 30.000 × 5 = 150.000 | 160 | 24.000.000 |
| Cereri de schimb / an | 30.000 × 2 × 12 = 720.000 | 256 | 184.320.000 |
| Feedback / an | 720.000 × 50% = 360.000 | 256 | 92.160.000 |

Total inițial (profil + abilități): 39.360.000 B = 39,36 MB. Creștere brută pe lună: 60.000 × 256 B + 30.000 × 256 B = 23.040.000 B = 23,04 MB. Creștere brută pe an: 276.480.000 B = 276,48 MB. Total după 12 luni: 315.840.000 B = 315,84 MB (≈ 301,2 MiB).

Aceste calcule EXCLUD indexuri, replici, backupuri, loguri, atașamente, overhead de bază de date și compresie. Pentru planificare, un factor suplimentar de 2–4× trebuie validat ulterior prin măsurarea înregistrărilor reale. Raportările de moderare pot fi adăugate într-o iterație ulterioară; nu sunt incluse în totalul brut pentru a evita o rată inventată de incidente.

### 5. Posibile blocaje și verificare

| Calitate | Ipoteză de blocaj | Efect posibil | Ce măsurăm |
| --- | --- | --- | --- |
| Latență | Căutarea/filtrarea profilurilor cu multe criterii | p95 > 2 s la vârf | p95/p99 pe endpoint, query plan, timp DB, rate cache-hit |
| Consistență | Actualizări concurente ale cererilor și indexare întârziată | Stări contradictorii sau profil vechi | Teste de concurență, procent read-your-writes, întârziere index |
| Debit | Fluxul de citire a profilurilor la 30.000 concurenți | Cozi, timeout, erori | RPS susținut, CPU, conexiuni, cozi, erori la ținta de vârf |
| Disponibilitate | Baza de date/autentificarea ca dependențe critice | Căutarea și cererile devin inaccesibile | Uptime pe intervale, health-check sintetic, rata erorilor |

Un RPS ridicat indică o cale de investigat, nu dovedește un blocaj. Fiecare ipoteză trebuie confirmată sau infirmată prin test de sarcină și observabilitate.

### 6. Concluzie și surse

Pentru 30.000 de studenți concurenți, modelul estimează 3,120 RPS în regim stabil și 5,016 RPS la vârful începutului de semestru, cu rezervă de 10%. Datele brute estimate sunt 39,36 MB inițial și 315,84 MB după 12 luni. Valorile trebuie validate cu trafic real și probe de stocare.

Surse metodologice: Lecția 3 — https://github.com/metreniuk/system-design-labs/blob/main/lectures/lecture-03-quality-requirements-workload-and-capacity.md ; Laboratorul 1 — Student Skill Exchange (documentul de produs realizat anterior). Toate proporțiile de utilizare și dimensiunile înregistrărilor sunt ipoteze proprii.

---
