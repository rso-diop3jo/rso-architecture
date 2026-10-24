# Koncept in meje storitev (stanje 24. 10. 2026)

Osnova za DEADLINE 1 (rok pon 26. 10. 2026, 23.59).

## Koncept

- Skupina prijateljev = tenant. Na začetku izbere skupni cilj kampanje.
- Uporabnik aplikacije je traveler (raziskuje in potuje, ne samo igra).
- Daily questi so osebni. Vsak traveler ima svoj seznam ponavljajočih aktivnosti (npr. vsak dan, 3x na teden, 1x na mesec; ponovitev ne more biti več kot dni v obdobju, ker je na dan ena izbira). Poleg njih obstajajo default questi, skupni vsem (npr. 10.000 korakov, odnesi smeti, pokliči družinskega člana).
  - Traveler vsak dan izbere eno nalogo iz menija (default questi + njegove ponavljajoče aktivnosti, ki so ta dan na vrsti: opravljene so manjkrat v tekočem dnevu, tednu ali mesecu, kot določa ponavljanje). Izbira je za ta dan dokončna.
  - XP oceni ocenjevalec težavnosti (v MVP zunanji LLM API, zamenljiv z lokalnim modelom), celo število 1–100. Default questi imajo XP določen vnaprej in so primeri, po katerih se ocenjevalec zgleduje. Ocena je enkratna, ob dodajanju aktivnosti (UC-05). Traveler XP ne more spremeniti ročno.
  - Daily quest je v zgodbi upodobljen kot pošast ali ovira (npr. mračno morje). Tema kampanje vpliva samo na zgodbo, ne na naloge.
- Side quest: kateri koli prijatelj ga doda, velja 24 ur. Dva načina:
  - "Pokliči druge travelerje": traveler svoj današnji daily quest spremeni v side quest za druge. Sam obdrži XP za svoj daily quest, dodatnega XP ne dobi.
  - Prost side quest ("pridite zvečer na pijačo"): creator XP ne dobi. XP side questa nastavi creator z drsnikom med 2 in 10 (majhen razpon, da sistema ni mogoče izkoriščati).
  - Pravila: creator ne more opraviti lastnega side questa (pri "pokliči druge travelerje" svoj daily quest opravi običajno); prijatelj dobi dodatni XP samo enkrat, kot dodatek k svojemu daily questu: pri "pokliči druge travelerje" četrtino XP daily questa (navzdol, vsaj 1), pri prostem side questu XP z drsnika; dnevna omejitev 2–3 side questi na osebo; v MVP samo-potrditev.
- Zvečer skupina izbere pot naprej med predlogi aplikacije (npr. celina = fitnes, iskanje zakladov = plavanje).
- LLM je dungeon master (zunanji API, zahteva #9): napiše nadaljevanje zgodbe. Kontekst: karakterji in kaj so že naredili.
- Končan quest = trofeja v baraki, od tam nov quest.
- Skupno mesto z drugimi skupinami: izven prve različice. Pozneje kot javna read-only projekcija (ime skupine, trofeje), da ne prebije izolacije tenantov.
- Prva različica (MVP): jedro brez mesta.
- Lone wolf: ob prvi prijavi se ustvari solo skupina z enim članom, da `tenant_id` vedno obstaja. Side questi so skriti, prijatelje lahko povabi pozneje. `TravelerRegistered` poslušata `group`, ki ustvari solo skupino in njen `tenant_id`; `SoloGroupCreated` pa `character`, ki ustvari začetni lik v tej skupini.
- Ranjen (v MVP): stanje sproži samo prizadeti, brez razloga in brez zdravstvenih podatkov. Dogodek `CharacterWounded`: `character` zamrzne streak, `quest` ustavi njegove daily queste, `notification` pošlje prijateljem toplo sporočilo, `story` zgodbo obrne v taborjenje. Nikoli kazni za zamujene dni.
- Tabor skupine (po MVP): vsi se ustavijo, skupina glasuje in ima časovno omejitev, nato odloči, ali gre naprej ali počaka. Zapisan v domeni in use caseih, ne gradimo ga takoj.
- Predpostavka, izrecno nepotrjena: "dnevni kviz" je zgodbeni ovoj nad resničnimi nalogami, ne vprašanja z odgovori.

## Lone wolf in burnout: celoten predlog

### 4. Lone wolf

Predlog: ob prvi prijavi se ustvari solo skupina z enim članom. Tako `tenant_id` vedno obstaja (T1) in v kodi ni posebnih primerov. Side questi so skriti, daily questi in zgodba delujejo, prijatelje lahko povabi pozneje. Dogodek `TravelerRegistered` poslušata `group` (solo skupina in njen `tenant_id`), `SoloGroupCreated` pa `character` (začetni lik, kot v UC-01). Use case: “traveler brez prijateljev začne solo avanturo”.

### 5. Burnout: rane in tabor

Lepa ideja in se prilega dogodkom. Predlog:

1. Stanje “ranjen” sproži samo prizadeti, brez razloga in brez zdravstvenih podatkov. Skupina vidi samo, da je ranjen.
2. Dogodek `CharacterWounded`: `character` zamrzne streak, `quest` ustavi njegove daily queste, `notification` pošlje prijateljem toplo sporočilo, `story` zgodbo obrne v taborjenje.
3. Nikoli kazni za zamujene dni. Rana pride samo iz prijave prizadetega, ne iz neuspeha.
4. Tabor skupine (vsi se ustavijo): skupina glasuje in ima časovno omejitev, nato odloči, ali gre naprej ali počaka. Koliko dni, je odprto.
5. To ni terapija, samo umakne pritisk.

## 6 storitev + GUI (potrjeno)

| # | Storitev | Lastnik česa |
|---|---|---|
| 1 | identity | lastna Google prijava (OIDC odjemalec), registracija, profil travelerja, `TravelerRegistered` (UC-01) |
| 2 | group | skupine (tenanti, `tenant_id`), člani, povabila, ime skupine, lestvica skupine, pregled dneva (kaj je kdo izbral in opravil) |
| 3 | quest | seznam ponavljajočih aktivnosti travelerja, default questi, ocena XP (ocenjevalec težavnosti), izbira daily questa, side questi, 24 h potek |
| 4 | character | XP, level, streak, trofeje, oprema |
| 5 | story | kampanja, skupni cilj, izbira poti, LLM |
| 6 | notification | opomniki in obvestila |

GUI: React Native, svoja mikrostoritev.

Razlogi za združitev storitev iz prve predloge za opis projekta:

- task + quest sta ista domena: quest je paket nalog, storitvi se vzajemno kličeta. Se pravi oba čakata in bereta enaka sporočila iz NATs, kar pomeni da sta odvisna en od drugega.
- progress + reward sta ista domena: reward potrebuje gRPC `GetProgress`, da odloči o dosežku.
- social meša članstvo in lestvico.

## Identity: lastna storitev, ne Keycloak

- Predavanja (pravilo D3 v `architecture/rules.md`) priporočajo obstoječe rešitve, npr. Keycloak za identiteto. Vaje pa zahtevajo 6 mikrostoritev, ki jih implementiramo sami.
- Če bi uporabili Keycloak, bi imeli samo 5 lastnih storitev. Zato `identity` implementiramo sami. Utemeljitev gre v ADR (še ni napisan).
- Odprto za asistenta (vpisano v `_ToDo`): ali Keycloak, zagnan v gruči, šteje kot 6. storitev (5 lastnih + Keycloak). Če da, `identity` odpade, profil travelerja gre v `character`, prva prijava ustvari lik, dobimo še bonus #26.

## Kompleksen primer za DEADLINE 1

Traveler izbere daily quest (`DailyQuestChosen`, `group` pokaže pregled dneva), ga opravi in `quest` objavi `QuestCompleted`. `character` v eni transakciji obdela XP, level in trofeje ter objavi `XpAwarded`/`LevelUp`/`TrophyUnlocked`. Odzovejo se `group` (lestvica), `story` (napredek cilja) in `notification`. Sodeluje 5 od 6 storitev (UC-04).

## Odločeno

- GraphQL (#17): `character`, bogata poizvedba o profilu.
- Event sourcing + CQRS (#12): `character`. XP kot zaporedje dogodkov (`character_events`), bralni model `character_view`, demo `rebuild`. Glej ADR-010.
- gRPC (#17): `story` → `character`, `GetPartySnapshot`. Ob koncu dneva story potrebuje stopnjo, opremo in nedavna dejanja vseh članov za LLM: velik tipiziran odgovor z enim klicem, z rokom za LLM. Po pravilu E3 podrobnosti dobiš z API-jem objavitelja. Šibkejši kandidat `quest` → `group` zavrnjen.
- Serverless funkcija (#16): potek side questa po 24 h.
- Model daily questov: osebni seznam ponavljajočih aktivnosti + skupni default questi (UC-05). Traveler izbere eno nalogo na dan (`DailyQuestChosen`), izbira je dokončna. Tema kampanje ne vpliva na naloge (UC-02, UC-04).
- Side quest iz daily questa ("pokliči druge travelerje") in prost side quest: creator obdrži svoj daily XP in za side quest ne dobi dodatnega (UC-03).
- XP oceni ocenjevalec težavnosti: celo število 1–100, default questi z vnaprej znanim XP so primeri. Ocena enkrat, ob dodajanju aktivnosti (UC-05). Če ocenjevalec odpove, se uporabi privzeti XP (20).
- Odziv na tuj side quest je dodatek k lastnemu daily questu: pri "pokliči druge travelerje" četrtina XP daily questa, pri prostem side questu XP z drsnika (2–10, ga nastavi creator, ocenjevalec ni vključen). Bonus ni pogojen s tem, da je prijatelj opravil svoj daily quest (UC-03).
- Ocenjevalec težavnosti: v MVP zunanji LLM API, zaščiten z circuit breakerjem (ADR-007). `quest` je odvisen samo od porta za ocenjevalca (ADR-008), zato ga je pozneje mogoče zamenjati z lokalnim modelom brez spremembe domene in use caseov.
- Zaščita ocenjevalca pred prepričevanjem: sistemski poziv izrecno pove, da je besedilo aktivnosti samo podatek za oceno in da se ocenjevalec ne sme dati prepričati, naj oceni drugače. Odgovor mora biti samo število, ki se preveri (UC-05, 4b).
- Burnout: v MVP samo stanje ranjen, tabor skupine po MVP.
- Lone wolf: solo skupina z enim članom.
- `tenant_id` je UUID, ne ime skupine. Določi ga `group` ob ustvarjanju skupine, ime pa je ločen podatek, ki ga travelerji določijo, ko se zberejo. Ustanovitelj povabi ostale prek njihovega identity ID (kot Discord). Ker `tenant_id` določi `group`, solo skupina nastane pred začetnim likom.
- Tehnologije po storitvi: Node/NestJS iz sošolčevega predloga je dovoljen po ADR-005 (Go privzeto, odstopanja po storitvi dovoljena). GUI ostane React Native (ADR-009).

## Odprto

- Keycloak kot 6. storitev: vprašanje za asistenta (glej zgoraj in `_ToDo`).
- Use case-i: UC-01 (usklajen s travelerjem, skupino in solo skupino), UC-02 solo avantura, UC-03 side quest, UC-04 dokončanje questa, UC-05 dodajanje ponavljajoče aktivnosti so napisani. Še manjkajo: ustvarjanje skupine in povabilo travelerjev, konec dneva in izbira poti, ranjen.
- Ustvarjanje skupine: kdaj je zbiranje končano in kdo določi ime skupine.
- Ali je traveler lahko v več skupinah hkrati (vpliva na `tenant_id` v JWT, T2).
- Zloraba ocene XP: poziv za ocenjevalca ne daje popolne zaščite pred prepričevanjem, zato ostanejo trde meje (odgovor je samo število 1–100, ena izbira na dan) in metrika razpršenosti ocen. Odprto: škodljive aktivnosti (filter vhoda?) in dnevna zgornja meja XP iz side questov (prijatelj lahko v dnevu opravi veliko side questov).
- Aktivnosti: urejanje in brisanje (sprememba imena ali opisa zahteva novo oceno, sicer se XP da zaobiti); začetek tedna in časovni pas za štetje ponovitev.
- Ali se izbrani daily quest lahko zamenja (UC-04 predpostavlja: ne).
- Ali tema kampanje vpliva na naloge (UC-02 predpostavlja: ne, samo na zgodbo).
- Tabor skupine (po MVP): koliko dni traja in kako poteka glasovanje.
- Ime aplikacije: kandidati Campfire, Wayfarers, Dobra družba, Hearth. **TO SI IZBERITA, katero vama je najboljše ime!**
