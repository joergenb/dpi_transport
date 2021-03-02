Dette dokumentet er eit framlegg til nytt, proprietært REST-api sikret med Maskinporten, for å sende Digital Post til Innbygger.


# Bakgrunn

Den proprietære transportinfrastrukturen for Digital Postkasse til innbyggere skal erstattes med en standard-infrastruktur for meldingsutvekling i det offentlege, dvs 4-hjørnes-modell med CEF eDelivery/PEPPOL. Følgende aktører inngår:
- Hjørne 1: Avsender (og evt. avsender sin leverandør/databehandler)
- Hjørne 2: Avsenders aksesspunkt-leverandør
- Hjørne 3: Postkasse-leverandørs aksesspunktleverandør
- Hjørne 4: Postkasse-leverandør

Vi ser altså at den sentraliserte Meldingsformidleren blir erstattet av et distribuert nettverk av aksesspunkt-leverandører.

Av sikkerhetsgrunner må derfor:
- meldingsformatet i DPI endres noe

samt noen av Meldingsformidlers oppgaver flyttes til Postkassene:
- validere ende-til-ende integritet på sendte meldinger
- validere at avsender har tillatelse til å sende digital post
- dersom en melding er sendt av en databehandler, validere at databehandler har lov til å opptre på vegne av avsnder
- sørge for at kvitteringer blir sendt tilbake til rett system uavhengig av om avsender bruker databehandler eller ikke


Det må etableres en ny transport-protokoll mellom hjørne 1 og hjørne 2.  I tradisjonell PEPPOL-tenking er dette noe som markedsaktørene selv skal ta fram - dvs i prinsippet er dette opp til aksesspunktleverandørene selv å bestemme. Siden Digdir ønsker å gjøre en anskaffelse av aksesspunktleverandørtjenester for formidling av digital post, som de fleste avsendere kommer til å benytte, er det hensiktismessig at Digdir kravstiller en protokoll som skal brukes av denne leverandøren.  Det hindrer ikke andre aktører å implementere andre, egne protokoller.



# Revidert meldingsformat- og transportformat i DPI

Motivasjon bak forslaget:

* Minst mulig endringer på eksisterende format, da både avsender-systemer og postkasse-leverandører støtter dette.
* Må ta høyde for at en aksesspunktleverandør i PEPPOL kan bli kompromittert av en angriper som vil forsøke å injisere falske meldinger
* Fjerne ebMS 3.0 som transportformat mellom Databehandler og Meldingsformidler, da erfaring viser at denne er relativt komplisert å ta i bruk, og vil være lite attraktiv for potensielle aksesspunktleverandører å måtte implementere,  og heller innføre en moderne og sikker lettvektsprotokoll som REST.
* Gjenbrukbart format også for andre meldinger i eMeldingsinfrastrukturen
* Utviklervenlig format med høyt rammeverk- og produktstøtte, for å gjøre det så lett som mulig å sende meldinger fra hjørne 1.

Forslaget er derfor:
1. [*Dokumentpakken*, dvs ASiC-pakken](https://docs.digdir.no/dokumentpakke_index.html) beholdes uendret
2. [*Forretningsmeldingene*](https://docs.digdir.no/sdp_index.html) beholdes også stort sett uendret.
 * Formatet skal endres fra XML til JWT for å bli mer tilpasset vanlig REST-bruk
3. [*SBDH*](https://docs.digdir.no/standardbusinessdocument_index.html) flyttes inn i forretningsmeldinga


# Grensesnittsdefinisjon REST-api

Aksesspunkt-leverandør i Hjørne 2 skal tilby et enkelt REST-endepunkt som Avsender bruker å sende post og hente kvitteringar.

REST-grensesnittet skal sikres med Bearer tokens fra Maskinporten, se: https://docs.digdir.no/maskinporten_auth_server-to-server-oauth2.html.  Informasjon om godkjente avsendere og deres eventuell databehandlere blir kodet inn i dette tokenet.

Aksesspunkt-leverandør skal tilby 2 endepunkt:

### Sende post

Post sendes i to steg - i første steg sendes forretningsmeldinga:
```
POST /sendmelding/{meldingsid}

<forretningsmelding i JWT-format>
```

og i andre steg sendes / strømmes selve dokumentet.
```
PUT /sendmelding/{meldingsid}

<ASCI-e>
```
Oppdeling i to steg skaper fleksibilitet og sikrer støtte for store dokumenter, evt. flere dokumenter på samme melding.

### Hente kvitteringer:

URL på formen
```
GET /kvittering/{conversationid}
GET /kvittering/avsenderidentifikator/{conversationid}
```

TODO: openapi-definisjon


# Forutsetninger:

### System-oppsett

Digdir oppretter maskinporten-scopet `dpi:send`. Tilgang til dette scopet betyr at Avsender har inngår bruksvilkår for Digital Postkasse til Innbygger.  Digdir settes som eier av Maskinporten-scopet, som betyr at det er Digdir som administrerer hvem som får tilgang. Faktura for konsumentene (= alle avsendere) går til Digdir selv og faktureres ikke.

Ved bruk av Altinn Autorisasjon til delegering, må det opprettes et "delegationScheme" i Altinn som muliggjør at Behandlingsansvarlig kan delegere til Databehandler.   Digdir blir eier av delegationSchemet, og Digdir vil motta faktura for bruk av Altinn.

Det opprettes `processid` i ELMA for de dokumenttyper som trengs støttes.
- digitalpost
- fysiskpost
- dpi-kvittering
- flytt-digitalpost


TODO: trengs det opprettes noe mer ?


### Oppsett av PK-leverandør

PK-leverandørene registreres som mottakere av aktuelle processid i ELMA.


### Oppsett av ny Avsender

Digdir gjev Avsender tilgang i Maskinporten til oauth2-scopet `dpi:send`  når bruksvilkår for Digital Postkasse er inngått.

Alt 1: Avsender som skal setje opp sitt eige system, må få tilgang til Samarbeidsportalen (i Prod) og registere ein maskinporten-integrasjon med det aktuelle scopet.  Fagsystemet/oauth2-klieten må konfigurerast med den genererte `client_id` og Avsender sitt virksomheitssertifikat.  Avsender som bruker integrasjonspunktet, trenger ikke gjennomfør dette, men må følge de retningslinjer som gjelder for å få satt opp et integrasjonspunkt.

Alt 2: Avsender som nyttar systemleverandør/databehandlar, må istaden logge inn i Altinn og delegere tilgangen til å sende DPI, vidare til systemleverandør.   Systemleverandør må få tilgang til Samarbeidsportalen og lage Maskinporten-integrasjon på same måte som sjølvstendige avsendere. Merk at systemleverandør autentiserer seg mot Maskinporten med sitt eige virksomheitsertifikat, og treng ikkje ha Avsender sitt.

Avsender (evt. systemleverandør) inngår so ein avtale med ein aksesspunkt-leverandør.  Fagsystemet blir konfigureret og integerert mot aksesspunktleverandør.

TODO: noko om oppsett av Integrasjonspunktet?

Avsender må bli satt opp i ELMA som mottaker av DPI-kvitteringsmeldinger.
TODO: Kva med bruk av Databehandler - er det avsender eller databehdnler som skal registreres i ELMA?
TODO:  kva viss avsender har fleire fagsystemer som sender digital post (/Avsender/avsenderidentifiktor/ ) - må alle Oslo Kommune sine systemer være kobla til samme aksesspunkt ?


# Meldingsflyt sende post

## 1:  Avsender henter token fra maskinporten

Fagsystem lager en JWT grant som etterspør `dpi:send`-scopet, signerer denne ( se https://docs.digdir.no/maskinporten_protocol_jwtgrant.html, sender dette til Maskinporten og får  et access_token i retur.

Ved utstedelse av token vil Maskinporten kontrollere:
- at Avsender har lov til å bruka DPI.
- Viss Avsender har databehandler, sjekkar Maskinporten mot Altinn Autorisasjon at aktuell autentisert Databehandlar har fått lov til å opptre på vegne av Avsender for sending av DPI.


Døme på access_token
```
  "iss": "https://maskinporten.no",
  "scope": "dpi:send",
  "aud": "https://api.aksesspunktleverandør.no/"
  "consumer": {
      "Authority": "iso6523-actorid-upis",
      "ID": "0192:991825827"      // Alltid Avsenders orgno
  }
  "supplier": {
      "Authority": "iso6523-actorid-upis",
      "ID": "0192:991825827"      // Eventuell Databehandler.
  }
  "iat": <timestamp>
  "exp": <iat-verdi + 30 sec>
  }
```

Dersom avsender er sin egen Databehandler så mangler tokenet `supplier`-claimet.

Avsender slår opp i KRR og finner hvilken PK-leverandør som innbygger benytter - og vet derigjennom om det skal sendes digital eller fysisk post

Avsender kan nå konstruere korrekt **forretningsmelding** (DigitalPostMelding) etter [dagens regler](https://docs.digdir.no/sdp_digitalpostmeldinger.html), men med følgende tillegg:

* token mottatt fra Maskinporten inkluderes i et felt `<maskinporten_token>`.
* en identifikator for meldingen inkluderes i et felt `<conversation_id>`
* Databehandler må signere forretningsmeldingen med samme sertifikat som benyttes til å signere Dokumentpakke.
  * TODO: er denne signert idag, hvilke regler for XML-signering gjelder?


TODO: er det lov for Avsender å sende flere meldinger på samme conversation-id ?

TODO: hvordan finnes print-leverandør?  kan det være flere? dersom ja,  er det avsender eller aksesspunktleverandør som bestemmer hvilken som skal brukes?


## 2:   Avsender sender post-melding

Avsender kan konstrukere korrekt URL mot aksespunktvet nå også hvilken forretningsmelding som skal sendes (fysisk eller digital).

```
POST /dpi/send/digitalpost/{postkasseleverandør-orgno}/{conversationid}
Host: api.aksesspunktleveradandør.no
Content-Type: application/xml
Authorization: Bearer <maskinporten_token m/ dpi:send scope>

<DigitalPostMelding>

--DPI-separator
Content-Disposition: form-data; name="dokumentpakke"
Content-Transfer-Encoding: chunked

<ASiC dokumentpakke; stor klump med alle dokument og vedlegg>

```


TODO: finst det betre handtering av store dok og/eller mange vedlegg ?  Gjenbruk Espen sine prosseser som integrasjonspunktet allereie har implementert.


### 3: Aksesspunkt-leverandør mottek melding

Aksesspunktleverandør (APL) må gjennomføre ei teknisk validering av Maskinporten-tokenet (ustedet av Maskinporten, gyldig signatur, ikkje utløpt).  
- Utgått token medfører 401-respons
- Gyldig token men som mangler "dpi:send"-scope eller avsender med manglande avtale med APL -> 403 repons

Aksesspunktleverandør må videre validere at:
* `consumer`-claimet i token stemmer med Avsender i forretningsmelding.
* [`aud`-claimet i token](ttps://docs.digdir.no/maskinporten_func_audience_restricted_tokens.html) stemmer med eget API-adresse
* `maskinporten_token` i forretningsmelding er identisk med det som ble brukt som Bearer token det aktuelle API-kallet
* `meldingsid` ikkje er forsøkt brukt tidlegare.


TODO: skal APL kva med PUT og DELETE på tidlegare innsendt melding


Aksesspunktleverandør lagrer converstation-id og tilhørende avsender/databehandler, slik at kvitteringsmeldinger og feilmeldinger relatert til meldingen kan håndteres.


### 4: Aksesspunkt-leverandør i hjørne 2 sender meldinga vidare til hjørne 3

APL mapper `meldingstype` i URL til riktig processid.  
APL slår opp i ELMA på processid og PK-leverndørs orgno.

APL kan nå konstrurere en PEPPOL-SBDH:

TODO: hvordan adresseres C3?
TODO: hvordan får C3 vite at denne meldingen skal til C4 ?

Dvs:  
* SBDH `Receiever` settes lik ?
* SBDH `processid` settes lik `processid`
* SBDH `Sender` settes lik  (Avsender / Databehandler)?

TODO:  Er det behov for å kunne spore at melding er mistet mellom hjørne 2 og hjørne 3.  Hva vet Digdir i såfall som kan hjelpe slik feilsøking?

### 5: Hjørne 3 mottek meldinga, og sender vidare til hjørne 4

TODO: C3 trenger å vite om meldingen skal til Digipost eller eBoks (Er dette Receiver i SBDH?)

C3 og C4 avtaler selv protokoll seg i mellom.  De kan gjerne gjenbruke C1-C2 REST-grensesnittet definert over, men må finne egen sikringsmekanisme (egen oauth2 autorisasjonsserver, eller bare bruke 2-vegs tls).


### 6: Hjørne 4 mottek meldinga og puttar i postkassen til innnbygger

Ved mottak av melding, må postkasse-leverandør validere ende-til-ende integritet, dvs:
a: at DigitalPostMelding er signert av Avsender(eller Databehandler) og inneholder en digest for tilhørende dokumentpakke
b: at dokumentpakken er signert av Avsender(eller Databehandler)
c: regne ut digest av dokumentpakken og kontrollere at utrekna digest stemmer med påstått verdi i forretningsmeldinga
d: validere at Avsender i forretningsmeldinger stemmer med `consumer` i `maskinporten_token` i forretningsmeldinga.
e: validere at virksomhetssertifikatet som er brukt til å signere både Dokumentpakke og DigitalPostMelding stemmer med autorisert avsender (dvs maskinporten-token)
  * lik `supplier`s orgno, dersom dette claimet finnes i maskinporten_token
  * lik `consumer`s orgno ellers

d: på-en-eller-annen-måte validere at virksomhetssertifikatet som er brukt til å signere i pkt a-b tilhører avsender.

# Kvitteringer

PK-leverandør må kunne asynkront sende kvittering tilbake i C3 -> C2 -> C1.  (dvs C3 må tilby et API eller mekanisme der PK-leverandør kan overføre kvitteringer, og dette må ta høyde for at slik kvitteringlevering skal kunne skje asynkront ifht levering av digitalpostmelding C3->C4.)

Feltet `converstation_id` kobler sammen en kvittering med tilhørende DigitalPostMelding.  

PK-leverandør lager en **forretningsmelding** (<LeveringKvittering>, <VarslingFeilet>, <Mottak>, <ReturPost>) etter dagens regler, med følgende tillegg:
* PK-leverandør legger til `conversation_id` i LeveringsKvittering
* PK-leverandør legger til Avsenders orgno i feltet `Avsender`
* PK-leverandør legger til Databehandlers orgno i feltet `Databehandler`


PK-leverandør ber så om at C3 sender kvitteringa til Avsender.

Eksempel
```
POST /send/kvittering/{avsender_orgno}/{conversationid}/{meldingsid}
```

C3 slår opp i ELMA og finner hvem som er C2 for Avsender.
C3 lager en PEPPOL-melding til C2.
C2 mottar kvitteringa, og legger den i kø.  Venter på at Avsenders system poll'er på kvitteringer med riktig conversation-id.  Verifiserer at Avsender som forsøker å hente kvittering, er den samme som sendte digitalpostmeldinga.


TODO: kva viss det er fleire fagsystemer hjå Avsender som sender Digtal Post - korleis identifisere og adressere det rette?




# 2-vegs svar

TODO
