2. Učenje v obliki RPG-igre
Osnovna ideja:
Aplikacija bi uporabnikom omogočala učenje skozi igranje. Namesto da samo berejo zapiske in rešujejo kvize, bi raziskovali ječe, se borili proti pošastim, zbirali nagrade in odklepali nove stopnje.
Aplikacija bi lahko bila namenjena študentom, dijakom ali vsem, ki se želijo naučiti novega znanja. Uporabnik bi izbral predmet, na primer programiranje, matematiko, računalniška omrežja ali angleščino, in začel svojo pustolovščino.
Glavna ideja je preprosta: za napredovanje v igri mora uporabnik pokazati, da razume snov.
Kako bi aplikacija delovala?
- Učne sobe: vsaka soba predstavlja določeno temo. V programiranju bi bila to na primer zanka for, v omrežjih pa model OSI.
- Vprašanja in naloge: za napredovanje mora uporabnik pravilno odgovoriti na vprašanja ali rešiti praktične naloge.
- Bitke proti pošastim: pravilni odgovori povzročijo škodo pošasti, napačni pa uporabniku vzamejo nekaj življenjskih točk.
- Večji izzivi: ob koncu poglavja uporabnika čaka težji izziv, ki preveri več različnih delov snovi.
- Oprema in sposobnosti: uporabnik odklepa nove sposobnosti, predmete in kozmetične dodatke.
- Sistem ponavljanja: aplikacija ponovno ponudi vprašanja, na katera je uporabnik prej napačno odgovoril.
- Prilagajanje težavnosti: če uporabnik dobro obvlada določeno temo, dobi težja vprašanja. Če ima težave, dobi lažje naloge in več ponovitev.
- Statistika znanja: uporabnik vidi, katere teme dobro obvlada in katere mora še vaditi.
Primer uporabe
Uporabnik se želi naučiti računalniških omrežij.
1. Izbere svet >>Računalniška omrežja<<.
2. V prvi sobi se uči osnov modela OSI.
3. Nato mora odgovoriti na nekaj vprašanj, da premaga šibkejšo pošast.
4. Z uspešnim reševanjem pridobiva izkušnje in opremo.
5. V naslednji sobi ga čakajo zahtevnejša vprašanja o TCP, UDP in usmerjanju.
6. Na koncu poglavja ga čaka večji izziv, v katerem mora povezati znanje iz več tem.
7. Če pri določeni temi pogosto odgovarja napačno, mu aplikacija pozneje ponudi dodatne vaje.
Na ta način je učenje predstavljeno kot napredovanje skozi igro, ne samo kot zbiranje točk.
Možne mikrostoritve
Mikrostoritev	Kaj počne?
user-service	Uporabniki, profili in nastavitve.
content-service	Predmeti, poglavja, vprašanja in odgovori.
quiz-service	Izvajanje kvizov in preverjanje odgovorov.
game-service	Bitke, življenjske točke, pošasti, sobe in napredovanje.
progress-service	Spremljanje znanja in rezultatov uporabnika.
reward-service	Izkušnje, stopnje, oprema in dosežki.
recommendation-service	Predlogi za ponavljanje in naslednje učne teme.
Kaj bi bilo zanimivo pri RSO?
Ko uporabnik odgovori na vprašanje, quiz-service preveri odgovor in objavi dogodek. game-service uporabi rezultat za izračun škode pošasti, progress-service shrani rezultat, reward-service pa dodeli izkušnje.
Poleg tega bi lahko dodali priporočila, ki uporabniku predlagajo, kaj naj ponavlja na podlagi njegovih preteklih rezultatov.
Prednost: aplikacija ima jasen namen, zabavno uporabniško izkušnjo in zanimivo povezavo med znanjem ter igralno logiko.
Izziv: treba je pripraviti dovolj kakovostnih vprašanj in dobro zasnovati sistem preverjanja znanja. Za začetek bi bilo smiselno podpreti en predmet in nekaj poglavij.