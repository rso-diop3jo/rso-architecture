# Koncept in meje storitev (stanje 24. 10. 2026)

Osnova za DEADLINE 1 (rok pon 26. 10. 2026, 23.59).

## Koncept

- Skupina prijateljev = tenant. Na začetku izbere skupni cilj kampanje.
- Daily questi so v zgodbi upodobljeni kot pošast ali ovira (npr. mračno morje).
- Side quest: kateri koli prijatelj ga doda, velja 24 ur, creator ne dobi XP.
  - Pravila: creator ne more opraviti lastnega side questa; XP je fiksen po težavi; dnevna omejitev 2–3 side questi na osebo; v MVP samo-potrditev.
- Zvečer skupina izbere pot naprej med predlogi aplikacije (npr. celina = fitnes, iskanje zakladov = plavanje).
- LLM je dungeon master (zunanji API, zahteva #9): napiše nadaljevanje zgodbe. Kontekst: karakterji in kaj so že naredili.
- Končan quest = trofeja v baraki, od tam nov quest.
- Skupno mesto z drugimi skupinami: izven prve različice. Pozneje kot javna read-only projekcija (ime skupine, trofeje), da ne prebije izolacije tenantov.
- Prva različica (MVP): jedro brez mesta.
- Lone wolf: ob prvi prijavi se ustvari solo skupina z enim članom, da `tenant_id` vedno obstaja. Side questi so skriti, prijatelje lahko povabi pozneje. `LearnerRegistered` poslušata `character` (začetni lik) in `group` (solo skupina).
- Ranjen (v MVP): stanje sproži samo prizadeti, brez razloga in brez zdravstvenih podatkov. Dogodek `CharacterWounded`: `character` zamrzne streak, `quest` ustavi njegove daily queste, `notification` pošlje prijateljem toplo sporočilo, `story` zgodbo obrne v taborjenje. Nikoli kazni za zamujene dni.
- Tabor skupine (po MVP): vsi se ustavijo, skupina glasuje in ima časovno omejitev, nato odloči, ali gre naprej ali počaka. Zapisan v domeni in use caseih, ne gradimo ga takoj.
- Predpostavka, izrecno nepotrjena: "dnevni kviz" je zgodbeni ovoj nad resničnimi nalogami, ne vprašanja z odgovori.

## Lone wolf in burnout: celoten prvotni predlog

### 4. Lone wolf

Predlog: ob prvi prijavi se ustvari solo skupina z enim članom. Tako `tenant_id` vedno obstaja (T1) in v kodi ni posebnih primerov. Side questi so skriti, daily questi in zgodba delujejo, prijatelje lahko povabi pozneje. Dogodek `LearnerRegistered` poslušata `character` (začetni lik, kot v UC-01) in `group` (solo skupina). Use case: “igralec brez prijateljev začne solo avanturo”.

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
| 1 | identity | lastna Google prijava (OIDC odjemalec), registracija, profil, `LearnerRegistered` (UC-01) |
| 2 | group | skupine (tenanti), člani, povabila, lestvica skupine |
| 3 | quest | daily + side questi, dodelitev, 24 h potek |
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
- Odprto za asistenta (vpisano v `_ToDo`): ali Keycloak, zagnan v gruči, šteje kot 6. storitev (5 lastnih + Keycloak). Če da, `identity` odpade, profil igralca gre v `character`, prva prijava ustvari lik, dobimo še bonus #26.

## Kompleksen primer za DEADLINE 1

`quest` objavi `QuestCompleted`. `character` v eni transakciji obdela XP, level in trofeje ter objavi `XpAwarded`/`LevelUp`. Odzovejo se `group` (lestvica), `story` (napredek cilja) in `notification`. Sodeluje 5 od 6 storitev.

## Odločeno

- GraphQL (#17): `character`, bogata poizvedba o profilu.
- Event sourcing + CQRS (#12): `character`. XP kot zaporedje dogodkov (`character_events`), bralni model `character_view`, demo `rebuild`. Glej ADR-010.
- gRPC (#17): `story` → `character`, `GetPartySnapshot`. Ob koncu dneva story potrebuje stopnjo, opremo in nedavna dejanja vseh članov za LLM: velik tipiziran odgovor z enim klicem, z rokom za LLM. Po pravilu E3 podrobnosti dobiš z API-jem objavitelja. Šibkejši kandidat `quest` → `group` zavrnjen.
- Serverless funkcija (#16): potek side questa po 24 h.
- Burnout: v MVP samo stanje ranjen, tabor skupine po MVP.
- Lone wolf: solo skupina z enim članom.
- Tehnologije po storitvi: Node/NestJS iz sošolčevega predloga je dovoljen po ADR-005 (Go privzeto, odstopanja po storitvi dovoljena). GUI ostane React Native (ADR-009).

## Odprto

- Keycloak kot 6. storitev: vprašanje za asistenta (glej zgoraj in `_ToDo`).
- UC-01 uskladiti: "learner" in "worlds" nadomestita igralec in skupina/kampanja.
- Tabor skupine (po MVP): koliko dni traja in kako poteka glasovanje.
- Ime aplikacije: kandidati Campfire, Wayfarers, Dobra družba, Hearth. **TO SI IZBERITA, katero vama je najboljše ime!**
