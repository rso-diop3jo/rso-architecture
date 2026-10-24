# ZAHTEVE – Kriterij ocenjevanja (RSO 2026/27)

Povzetek iz `[SLO] Navodila za projekt in kriterij ocenjevanja.pdf` (datum dokumenta: 12. 10. 2026). Namen: pri zasnovi (DEADLINE 1) že odločiti, katere zahteve pokrijemo, da repo vsebuje vse, kar zahtevajo navodila.

## Pravila ocenjevanja

- Izpolniti je treba **vse obvezne zahteve** in **vsaj 50 % skupnih zahtev** (točk, glej `architecture/rules.md` §10).
- Zahteve se preverjajo med zagovorom in se ocenjujejo po **obsegu in kakovosti** implementacije.
- Minimalne zahteve veljajo **za vsakega člana** (primer iz PDF: minimum 2 mikrostoritvi → tričlanska skupina ima minimum 6 mikrostoritev).
- Ocenjujeta se GitHub repozitoriji in poročilo o projektu (navodila za poročilo bodo v ločenem dokumentu).
- Zahteve 23–26 so **dodatne točke**: ne štejejo v kvoto, končna ocena ostane največ 100 %.
- Ocena (iz vaj, 15. 10.): `OCENA = ⍺ · koda + (1 − ⍺) · končna predstavitev + vmesna predstavitev`.

## Točke

| Skupina | Točke | Od tega obvezno |
|---|---|---|
| Konfiguracijske (1–8) | 26 | 14 (1, 2, 3, 4, 8) |
| Funkcionalnost (9–15) | 30 | 4 (15) |
| Tehnološke (16–20) | 24 | 0 |
| Oddaja (21–22) | 20 | 10 (22) |
| **Skupaj** | **100** | **28** |
| Dodatne točke (23–26) | +15 | 0 |

Obvezno = 28 točk, torej je za prag 50 točk treba zbrati še **vsaj 22 točk** iz neobveznih (skupaj 72).

## Pregled vseh zahtev

Stolpec **Namera**: ✅ = obvezno · 🔵 = že odločeno ali omenjeno v pravilih/ADR · 🟢 = želimo vključiti v zasnovo · ❓ = še odločiti · ➕ = dodatne točke.

| # | Zahteva | Obv. | Točke | Namera | Kaj mora določiti zasnova |
|---|---|---|---|---|---|
| 1 | Repozitorij | Da | 1 | ✅ | README, strategija vej (main, dev); en repo na mikrostoritev v organizaciji `rso-diop3jo` |
| 2 | Mikrostoritve + cloud-native | Da | 4 | ✅ | Število mikrostoritev glede na člane, meje storitev, protokoli, K8s manifesti, clean architecture |
| 3 | Podatkovna baza na storitev | Da | 2 | ✅ | Katera storitev ima trajne podatke, katera baza (SQL/NoSQL), dostop drugih le prek API/dogodkov |
| 4 | Dokumentacija | Da | 2 | ✅ | Markdown v repozitorijih |
| 5 | Dokumentacija API | Ne | 3 | 🔵 | OpenAPI 3.1 (pravilo A5), napake RFC 9457 (A6) |
| 6 | CI/CD | Ne | 5 | 🟢 | Pipeline na repo; avtomatski test + deploy na K8s ob spremembi v main |
| 7 | Helm charts | Ne | 4 | 🟢 | Chart na mikrostoritev; vrednosti za dev in prod |
| 8 | Namestitev v oblak | Da | 5 | ✅ | Azure (priporočeno); javno dostopen K8s in aplikacija |
| 9 | Zunanji API | Ne | 3 | 🟢 | Kateri zunanji API, v kateri storitvi, kako avtentikacija; breaker + timeout + retry: ADR-007 |
| 10 | Večnajemništvo | Ne | 5 | 🟢 🔵 | Model izolacije (T3, še odprto), `tenant_id` skozi celotno verigo (T1–T4), K8s namespace |
| 11 | Preverjanje zdravja | Ne | 4 | 🔵 | Končne točke liveness/readiness/startup (pravilo R4) |
| 12 | Event sourcing + CQRS | Ne | 5 | 🟢 | `character` je event-sourced (`character_events`), bralni model `character_view`, demo ponovnega predvajanja (`rebuild`); ADR-010 |
| 13 | Izolacija in toleranca napak | Ne | 5 | 🟢 | Circuit Breaker (sony/gobreaker) + Retry (pravilo C1), demo izpada ene storitve, odstavek o izolaciji; lastnik še ni določen (_ToDo); ADR-007 |
| 14 | Upravljanje s konfiguracijo | Ne | 4 | ❓ | Seznam parametrov na storitev, ločitev kode in konfiguracije, več virov |
| 15 | Grafični vmesnik | Da | 4 | ✅ | Ogrodje, vsaj 2 podstrani, REST do mikrostoritev |
| 16 | Serverless funkcija | Ne | 5 | 🟢 | Kateri del postane funkcija (Azure Functions), kaj jo sproži |
| 17 | GraphQL in gRPC | Ne | 4 | 🟢 | GraphQL v eni storitvi, gRPC med dvema storitvama |
| 18 | Sporočilni sistemi | Ne | 5 | 🔵 | NATS JetStream (ADR-002), vsaj 2 storitvi komunicirata prek njega |
| 19 | Centralizirano beleženje | Ne | 5 | 🔵 | Zbiranje dnevnikov vseh storitev + opozorila na vzorce (pravilo R2) |
| 20 | Zbiranje metrik | Ne | 5 | 🔵 | Prometheus + Grafana (pravilo R3) |
| 21 | Vmesna predstavitev | Ne | 10 | ❓ | Kratek pitch na vajah (arhitektura + razvit del) |
| 22 | Oddaja projekta | Da | 10 | ✅ | Koda in poročilo na spletni učilnici |
| 23 | Terraform | Ne | +3 | ➕ | |
| 24 | API Gateway | Ne | +4 | ➕ | Povezano s T2 (kvote in usmerjanje po najemniku) |
| 25 | Ingress Controller | Ne | +4 | ➕ | |
| 26 | IAM, OAuth2, OIDC | Ne | +4 | ➕ | Povezano z UC-01/UC-02 (prijava z Google) |

### Točke po trenutni nameri

| Sklop | Točke |
|---|---|
| Obvezno (1, 2, 3, 4, 8, 15, 22) | 28 |
| 🔵 + 🟢 (5, 6, 7, 9, 10, 11, 12, 13, 16, 17, 18, 19, 20) | 58 |
| **Skupaj** | **86** (prag je 50) |
| ❓ (14) | 4 |
| ❓ vmesna predstavitev (21) | 10 |

## Podrobnosti zahtev (minimalne zahteve iz PDF)

### Konfiguracijske

**1. Repozitorij** (obvezno, 1 točka)

- [ ] Git repozitorij na GitHubu (lahko GitLab ali Bitbucket).
- [ ] README z informacijami o projektu (kratek opis, navodila za namestitev itd.).
- [ ] Strategija razvejitve (npr. main, dev).

**2. Mikrostoritve in Cloud-native aplikacija** (obvezno, 4 točke)

- [ ] IDE (npr. Visual Studio Code, IntelliJ IDEA).
- [ ] Osnovna struktura projekta po najboljših praksah (direktoriji za izvorno kodo, teste in konfiguracijske datoteke).
- [ ] Razvojno okolje z vsemi potrebnimi datotekami.
- [ ] Dokumentirana navodila za nastavitev za druge razvijalce.
- [ ] Vsaj dve mikrostoritvi poleg GUI na člana, vsaka odgovorna za določeno funkcionalnost (enočlanska ekipa: GUI + 2 mikrostoritvi).
- [ ] Mikrostoritve komunicirajo prek HTTP ali sporočilnega protokola.
- [ ] Kubernetes manifesti (YAML) za namestitev mikrostoritev.
- [ ] Kubernetes funkcionalnosti: storitve (Service), namestitve (Deployment), podi.
- [ ] Cloud-native koncepti.
- [ ] Koncepti clean architecture.

**3. Podatkovna baza za vsako mikrostoritev** (obvezno, 2 točki)

- [ ] Relacijska (PostgreSQL, MySQL) ali NoSQL (MongoDB) baza.
- [ ] Vsaka storitev s trajnimi podatki ima svojo bazo (storitve brez podatkov baze ne potrebujejo).
- [ ] Storitve ne dostopajo neposredno do baz drugih storitev, samo prek API-jev ali sporočil.
- [ ] Storitve v svoje baze shranjujejo in iz njih berejo podatke.
- [ ] V bazah je nekaj tabel in podatkov.

**4. Dokumentacija** (obvezno, 2 točki)

- [ ] Vse relevantne informacije o aplikaciji po primerih dobrih praks.
- [ ] Markdown ali drugo orodje za dokumentacijo za jasno organizacijo vsebine.

**5. Dokumentacija API** (neobvezno, 3 točke)

- [ ] OpenAPI ali Postman za dokumentiranje končnih točk.
- [ ] Za vsako mikrostoritev: formati zahtev/odgovorov, primeri tovorov in sporočila o napakah.

**6. Cevovod CI/CD** (neobvezno, 5 točk)

- [ ] Orodje CI/CD za avtomatizacijo gradnje in testiranja (priporočeno GitHub Actions; tudi GitLab CI, Jenkins).
- [ ] Koda se samodejno testira in namesti na Kubernetes gručo ob spremembah v glavni veji.

**7. Helm charts** (neobvezno, 4 točke)

- [ ] Helm Charts za mikrostoritve.
- [ ] Parametrizirane konfiguracije za različna okolja, vsaj dev in prod.
- [ ] Namestitev in posodobitev aplikacije v Kubernetesu s Helm Charts.

**8. Namestitev v oblak** (obvezno, 5 točk)

- [ ] Oblačna platforma (priporočeno Microsoft Azure; tudi AWS ali Google Cloud).
- [ ] Kubernetes gruča in aplikacija sta javno dostopni.

### Funkcionalnost

**9. Zunanji API** (neobvezno, 3 točke)

- [ ] Zunanji API uporabljen znotraj ene od mikrostoritev.
- [ ] Ustrezna potrditev verodostojnosti (authentication).

**10. Večnajemništvo (multitenancy)** (neobvezno, 5 točk)

- [ ] Večnajemništvo načrtovano in implementirano na nivoju mikrostoritev, Kubernetesa in podatkovne baze.
- [ ] Prikaz delovanja z več najemniki.

**11. Preverjanje zdravja** (neobvezno, 4 točke)

- [ ] Končne točke za preverjanje zdravja, ki vrnejo stanje vsake mikrostoritve.
- [ ] Kubernetes lahko spremlja te končne točke.

**12. Event sourcing in CQRS** (neobvezno, 5 točk)

- [ ] Event Sourcing za sledenje spremembam podatkov.
- [ ] CQRS: ločena modela branja in pisanja.
- [ ] Demonstracija ponovnega predvajanja dogodkov za obnovo stanja.

**13. Izolacija in toleranca napak** (neobvezno, 5 točk)

- [ ] Mehanizmi za toleranco napak, kot sta Circuit Breaker in Retry.
- [ ] Prikaz, kako sistem deluje tudi ob napaki ene storitve.
- [ ] Razlaga, kako izolacija storitev zmanjša vpliv napak na celoten sistem.

**14. Upravljanje s konfiguracijo** (neobvezno, 4 točke)

- [ ] Vsi konfiguracijski parametri vsake mikrostoritve (nizi za povezavo z bazo, URL-ji storitev in zunanjih API-jev, poverilnice in ključi API, vrata, feature flags, nivoji beleženja) so v dokumentaciji, z namenom in privzeto vrednostjo.
- [ ] Ločitev implementacije in konfiguracije: konfiguracijo je mogoče spreminjati brez ponovnega prevajanja in nameščanja.
- [ ] Uporaba različnih virov konfiguracije (konfiguracijske datoteke, okoljske spremenljivke, konfiguracijski strežnik ...).

**15. Grafični vmesnik** (obvezno, 4 točke)

- [ ] Ogrodje (React, Angular, Vue.js) ali mobilna aplikacija.
- [ ] Vsaj 2 podstrani.
- [ ] Sprednji del komunicira z mikrostoritvami prek REST API-jev.

### Tehnološke

**16. Serverless funkcija** (neobvezno, 5 točk)

- [ ] Azure Functions (tudi AWS Lambda ali Google Cloud Functions).
- [ ] Funkcija se proži na podlagi dogodkov (npr. HTTP zahtevek).
- [ ] Opazovanje delovanja funkcije in prilagajanje zmogljivosti glede na obremenitev.

**17. GraphQL in gRPC** (neobvezno, 4 točke)

- [ ] GraphQL za poizvedovanje po podatkih v eni mikrostoritvi.
- [ ] gRPC za komunikacijo med dvema storitvama.
- [ ] Prikaz uporabe obeh tehnologij v aplikaciji.

**18. Sporočilni sistemi** (neobvezno, 5 točk)

- [ ] Sporočilni posrednik (RabbitMQ, Kafka, NATS JetStream) za asinhrono komunikacijo.
- [ ] Vsaj dve mikrostoritvi komunicirata prek sporočilnega sistema.

**19. Centralizirano beleženje dnevnikov** (neobvezno, 5 točk)

- [ ] Orodja, kot sta ELK Stack ali Fluentd, za zbiranje dnevnikov.
- [ ] Dnevniki vseh storitev so centralizirani.
- [ ] Opozorila na podlagi zaznanih vzorcev v dnevnikih.

**20. Zbiranje metrik** (neobvezno, 5 točk)

- [ ] Prometheus, Grafana ali podobno orodje za zbiranje in vizualizacijo.
- [ ] Spremljanje ključnih metrik, kot sta odzivnost in obremenitev.
- [ ] Prikaz uporabe zbranih metrik za optimizacijo delovanja.

### Oddaja

**21. Vmesna predstavitev** (neobvezno, 10 točk)

- [ ] Kratek javni pitch projekta na vajah: arhitekturna zasnova in že razviti deli (več informacij v ločenem dokumentu).

**22. Oddaja projekta** (obvezno, 10 točk)

- [ ] Koda in poročilo na spletni učilnici (več informacij v ločenem dokumentu).

### Dodatne točke (ne štejejo v kvoto)

**23. Terraform** (+3)

- [ ] Terraform konfiguracijske datoteke za potrebne vire (virtualni stroji, omrežja, baze).
- [ ] Moduli za ponovno uporabo konfiguracij.
- [ ] Terraform plan in apply za izpeljavo sprememb infrastrukture.

**24. API Gateway** (+4)

- [ ] API Gateway (Kong, AWS API Gateway, Istio) za usmerjanje API klicev.
- [ ] Ustrezna avtentikacija in avtorizacija prek API Gateway.
- [ ] Enotna točka dostopa za različne API-je.
- [ ] Prikaz delovanja z več mikrostoritvami.

**25. Ingress Controller** (+4)

- [ ] Ingress Controller (NGINX, Traefik) v Kubernetes okolju.
- [ ] Pravila Ingress za usmerjanje prometa do storitev.
- [ ] TLS/SSL za varno komunikacijo.
- [ ] Preverjeno delovanje z dostopom do storitev prek URL-jev.

**26. IAM, OAuth2, OIDC** (+4)

- [ ] IAM rešitev za upravljanje uporabniških dostopov (priporočeno Azure AD; tudi AWS IAM, Keycloak).
- [ ] OAuth2 za avtorizacijo dostopa do API-jev.
- [ ] OpenID Connect (OIDC) za avtorizacijo in pridobivanje informacij o uporabnikih.
- [ ] Preverjeno delovanje s prijavo uporabnika in dodelitvijo pravic dostopa do aplikacije.

## Odprta vprašanja

- Število mikrostoritev: PDF zahteva 2 na člana, v `README.md` in `rules.md` (D4) je 6 mikrostoritev. To velja za 3 člane; za 6 članov bi bil minimum 12.
- Ali Grafana Loki (pravilo R2) šteje pri zahtevi 19? PDF navaja »orodja, kot so ELK Stack ali Fluentd«, torej verjetno da, a potrdi na vajah.
- Event sourcing (12) ni isto kot dogodkovna arhitektura (ADR-001): potrebuje lastno odločitev, katera storitev hrani stanje kot tok dogodkov. Rešeno: `character`, glej ADR-010.
