pri mikrostoritvah vsa komunikacija poteka preko omrežja, kar je manj reliable kot monolitski klic. To je zaxto, ker je vsaka storitev svoja enota, ki deluje neodvisno od drugih. Storitev je lahko zagnana tudi na drugem serverju, v drugi državi, to ni odvisno od nas ampak od Kubernetesa in nastavitev. Zaradi tega razloga ne želimo imeti APIjev, ki omogočajo samo 1 POST/PATCH, kjer bi vpisali 1 transakcijo. Kaj se bo zgodilo je, da bo ogromno klicev, se pravi bo sistem deloval počasi. Namesto tega se dizajnira API, ki omogoča array zapis transakcij, kjer se pošlje več transakcij naenkrat. API se lahko nadaljno optimizira preko logov, ko vidimo kje se porabi največ časa, tako ad lahko uvedeš nov api klic, ali ga izboljšaš samo v najpočasnejših delih.

V REST API nujno vključujemo verzijo v URL, zato da lažje naredimo breaking changes! S tem ne podremo legacy kode, ker se za njo nič ne spremeni, npr. še zmeram uporablja .../V1/..., med tem ko mi uveljavljamo nove API klice na .../V2/...

Sistemskih logov se NE PIŠE V STORITVE! To je največja napaka svežega programerja. To je zato ker se storitve zaganjajo kot docker containers, ki so read only na underlying OSu. Log se že zapiše, ampak v docker sandboxed environment. Prav tako da bi dobil vse loge, bi potreboval se prijaviti na okolje vsake storitve in dobiti log iz nje. Najbolj problematična stvar pa je, da ko se docker zapre, se VSE IZGUBI, ker se pobriše celoten environment! Za zbiranje logov se uporablja Grafana Loki ali Fluentd, to so uveljavljene kubernetes rešitve. Za numerične metrike se uporablja Prometheus, text-based logs pa Grafana Loki.

Sklopljenost sistema z request/response klici je amaterska napaka. Ne tega delat, ker ko bo en service nehal respondad, se bo klicatelj zaklenil, ker ne bo dobil responsa.

Za opisovanje in dokumentacijo REST API uporabi OAS3.1.
REST API je standardiziral obliko napak (faultov). Uporabi ta standard.

Ker mikrostoritve komunicirajo preko omrežja moramo poskrbeti za idenpotentnost. READ je by design idenpotenten, UPDATE/WRITE pa NISO idenpotentni, zaradi česar potrebujemo uporabiti enolične indikatorje za te operacije, da vemo, če se je že izvedla. REST API nam ponuja e-tag, ki je standarizirani način za idenpotentnost. Uporabi to, namesto da razviješ svojo rešitev.

Standardiziran način za idenpotentnost je tudi retry in timeout, se pravi po timeout časa poskusi še enkrat, v kolikor nisi prejel odgovora. S tem poskrbimo, da se klici res izvedejo.

Kar NE delamo je skupna baza med mikrostoritvami. To je porazdeljeni monolit. Cloud native arhitektura prekinja to navado, saj so posledice hude: sprememba sheme zlomi druge storitve, ni neodvisne namestitve, baze ne moremo skalirati po storitvah, ena točka odpovedi (skupna baza) in skrita sklopljenost prek tabel. Vsi razvijalci na različnih ekipah različnih storitev morajo komunicirati medsebojno da uskladijo kako se podatki hranijo v bazi. To je ANTIPATTERN. Razbili smo kodo, sklopljenost pa je ostala preko skupne podatkovne baze. Če bi želeli zagnati še en service, tega ne moremo storiti neodvisno od ostalih storitev. Rabimo zagnati vse storitve, ki imajo to skupno bazo, zato da dobimo še eno željeno storitev.

Dobra praksa je da ima vsaka storitev svojo podatkovno bazo. To pripelje do logično izoliranih sklopov. Do podatkov drugih storitev sedaj dostopamo samo preko API-ja in dogodkov. Izgubimo pa ACID transakcije čez storitve. Sistem bo EVENTUELNO usklajen, saj rabi nekaj časa da se eventi propagirajo do vseh storitev. Na primer, ko vpišemo nov artikel potrebuje sistem nekaj časa da vsaka storitev prebere ta nov event.
Storitve syncamo preko jetstream, kafka... to je middlewear, ki skrbi za prenašanje dogodkov. Se pravi ko nek service na primer doda nov artikel, to zapiše v svojo interno bazo, HKRATI pa odda event na kafka kanal. Ta event ima enostavno JSON strukturo. Ostale storitve poslušajo kanale, za katere so zainteresirani in prebirajo ta sporočila. Se pravi preko eventov sinhroniziramo logiko, da vse storitve vejo, kaj se dogaja v drugih storitvah. Tako storitve niso več sklopljene preko skupne podatkovne baze, ampak preko poslovnih sporočil. Tako razklopimo sistem, s čimer ga lahko enostavno skaliramo. Prednost je tudi, da sedaj ko ima vsaka storitev svojo podatkovno bazo, da lahko izbiramo podatkovno bazo, ki je primerna za storitev. Za storitev kataloh lahko elastic search, vektorsko bazo... entitetna relacijska baza NI primerna, potrebujemo NoSQL bazo, ki ima močno semantično iskanje, ker katalog ni samo tekst, ampak so slike, videji, komentarji, kar bi v entitetni relacijski bazi delali na silo. Za storitev nakup pa je primerna entiteno relacijska baza, ker želimo integriteto zapisov.

API-ji NISO edini način za komunikacijo med storitvami. API endpointov se je treba lotiti tako, da ima vsak endpoint svoje zahteve, eni rabijo biti secure, eni rabijo obdelovati izjemno veliko zahtev vsako sekundo. Zato za vsak endpoint izbiramo drugo tehnologijo. Se pravi 
.../V1/.../client-api
.../V1/.../service-api
.../V1/.../agentic-api
Ima lahko vsak drugačno tehnologijo za seboj.
Dandanes pa se za komunikacijo uporablja EVENT STREAMING, ki je backbone, po njem pošiljamo poslovne evente. POSLOVNI del je zelo pomemben, saj se nanaša na "artikel x kupljen 2x", "vrnjen artikel y", "nov izdelek dodan v bazo" in NE na nek tehnični koncept "prišel je API klic x", "zapisal sem datoteko z". Se pravi namesto pri sklopljenem sistemu, kjer smo rabili vedeti kam v podatkovno bazo pogledati za katero stvar, je sedaj pomembno da vemo katere kanale poslušati.

POMEMBNO: pri začetnem projektnem planiranju je potrebno se uskladiti o PAYLOAD-u EVENTA. Se pravi se je treba uskladiti za vse ključne podatke, ki gredo v event. Potrebujejo biti stabilni event, da se ne spreminjajo skozi čas razvoja. API-je lahko updejtamo, saj imamo versioning (.../V1/..., .../V2/...), eventi pa so naša nova "sklopljenost" v smislu tega, da vse storitve uporabljajo isti event kanal, da akomunicirajo medsebojno.
V kolikor potrebuje storitev več kot samo event message, ker ne moreš vsega spraviti v event message, greš klicati API od storitve, ki je poslala event. To je dobra praksa, tako hierarhično urejeno klicanje. Event pove splošno poslovno novico, vsaka storitev pa preko API-ja pridobi bolj podrobne informacije.

Se pravi je kafka formatno sklopljena, preko skupnega dogovorjenega formata, kako izgledajo poslovni eventi, ki se pošiljajo. Ni pa več lokacijsko slopljena, niti časovno (sinhronizirani REST pa je časovno sklopljen na primer). Izogibamo se (kot antipatern) kakršnekoli arhitekture, ki je časovno ali lokacijsko sklopljena.

Kot alternativa Kafke je NATS, ki je leightweight in blazno hiter. Profesor ga preferira proti kafki. Je še za premisliti, katerega bi uporabljali.

Podatkovne baze znotraj mikrostoritev se ločijo na 3 ideje:
vse stranke v isti podatkovni bazi, kar NI secure. Znotraj ene podatkovne baze je vsaka stranka svoja shema. In pa najbolj secure način: nova podatkovna baza za vsako stranko/tennant. To programski jeziki ŽE PODPIRAJO z hybernate. Poveš katero strategijo bi uporabill, in ti sami skrbijo za izbran način.

Za ta projekt delamo multitanent arhitekturo. Kar pomeni da je že od začetka treba premisliti da se sabo vleče tanent ID, po celotni verigi.

To pomeni da lahko narediš tudi procesiranje secure, saj dobi vsak tenant svojo nit npr. ali pa svojo instanco storitve. API gateway pa nato routa na pravo storitev glede na tanent ID. Taka odločitev vpliva na celotno verigo od DevOps do CI/CD in je ZELO POMEMBNA ODLOČITEV med gradnjo arhitekture.

Se pravi bi izgledal flow:
tenant_id v žetonu JWT-> api gateway, ki omejuje kvote glede na najemnika-> storitev (kontekst v klicu)->dogodki (glava ali subjekt, NATS accounts, kafka ACL)-> podatki(filtriranje ali RLS) -> kubernetes (imenski prostor, ResourceQuota, NetworkPolicy), vendar hudo pozorni pri filtrih, saj lahko en napačno nastavljen filter privede do tega, da ima ena stranka dostop do druge stranke!

Za model izvajanja si lahko izberemo sodobne metode: async/awayt, ali reaktivno. Reaktivno je bolj kompleksno, a omogoča streaming podatkov, zato raje async/await. Ena nit na zahtevo ni več optimalen način pisanja, je ANTIPATERN. In to zato, ker ena nit zavzame približno 1MB pomnilnika, če je veliko api klicev, kjer vsak api klic sproži svojo nit, se to hitro nabere.

Vsaka storitev meri svoje metrike, ki se zbirajo v okolju Prometheus z grafano za prikaz. To dela preko /metrics endpointa. Zbirajo se metrike:
RED za storitve - rate (koliko zahtev na sekundo), errors (delež napak), duration (latenca, p50, p99)
USE za vire - utilization (izkoriščenost), saturation (nasičenost vrst), errors (napake naprave)
Zlati signali Google SRE - latenca, promet, napake, nasičenost


POMEMBNI PRINCIPI PRI GRADNJI ARHITEKTURE:

Idenpotentnost - ponovni poskusi, idenpotenten porabnik, idenpotency-key (e-tag?)
Deklerativnost in konvergenca - stanje iz dogodkov, konfiguracija kot podatki
Šibka slopljenost - dogodki, baza na storitev, kontekst najemnika/tanenta
Združljivost skozi čas - evolucija shem, razširi-preseli-skrči


Za vsako storitev imamo tudi HEALTH CHECKS: tri vrste preverjanj. To je izpostavljeno preko /health endpointa. Kubernetes, preko česar bomo deployali svoje storitve čekira 3 health pointe:
1. Liveness (Ali proces še živi?)
2. Readiness (ali je pripravljen sprejemati promet?)
3. Startup (ali se je zagon končal?)


Preden se lotimo programiranja, med načrtovanjem je zelo pomembno, da oseba, ki načrtuje dobro pozna domeno.

Pri dekompoziciji na storitve se vedno vprašaj "Ali lahko ena ekipa spremeni, preizkusi in namesti svojo storitev, ne da bi usklajevala z dugimi?"

Po razčlenitvi domene, se vprašamo nujno, ali za naš problem že obstaja rešitev. Na primer: rešeni problemi so že identiteta (identity, keycloak), plačila (stripe,bankcard), e-pošta (sendgrid). Generičnih storitev se ne splača graditi, ker je izguba virov.

Bolje kot poznamo domeno, lažje nam je narediti dekompozicijo in določiti meje.

Naša granularnost rešitve je 1 GUI vmesnik in 6 mikrostoritev.

Med pisanjem arhitekture se piše ADR! To je standardizirana oblika KAKO smo se odločili in ZAKAJ. Se pravi zapis arhitekturnih odločitev, ključne odločitve, ki se zapišejo v .md datoteke. Shranjujejo se ponavadi kot ADR-001, ADR-002...

Kdaj se odločiti za API ali dogodek? Če klicatelj (mikrostoritev) potrebuje rezultat da nadaljuje, je to sinhroni api klic (REST, gRPC) + timeout + retry (zato da smo res opravili klic. To je osnovna praksa z timeout in retry). Če gre za dejstvno, poslovni dogodek, ki zanima druge se odločimo za dogodek (kafka, NATS). V kolikor imamo dolgo operacijo, ki lahko počaka se odločimo za asihnron ukaz v vrsto + 202 accepted + status.

Za API je slab primer tak, kjer delamo CRUD nad bazo, se pravi dobimo status, ročno porapvimo v bazi... to je slab primer, ker imamo več klicev na enkrat, kar obremenjuje omrežje po nepotrebnem, porabnik prav tako obide pravila, ker direktno vpisuje v bazo. Dober primer uporabe pa je npr.

```
# usmerjen v namen
POST /narocila
  Idenpotency-key: 7c1e...
POST /narocila/42/preklic
GET /narocila/42    # vrne, da je bilo naročilo preklicano
```
Pri tako napisanem API-ju pravila ostanejo v storitvi, en klic pa je en primer uporabe.


Kar je potrebno narediti:
1. Zapisati domeno/temo
2. zapisati kaj bomo pokrili s storitvami
3. Zapisati arhitekturne odločitve in pričeti dizajnirati rešitev