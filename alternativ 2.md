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

Protokoll mellom hjørne 2 og 3 er bestemt av PEPPOL, og heter AS4.
Protokoll mellom hjørne 3 og Postkasse-leverandør bør avtales bilateralt mellom disse aktørene.



# Revidert meldingsformat- og transportformat i DPI

Motivasjon bak forslaget:

* Minst mulig endringer på eksisterende format, da både avsender-systemer og postkasse-leverandører støtter dette.
* Må ta høyde for at en aksesspunktleverandør i PEPPOL kan bli kompromittert av en angriper som vil forsøke å injisere falske meldinger
* Fjerne ebMS 3.0 som transportformat mellom Databehandler og Meldingsformidler, da erfaring viser at denne er relativt komplisert å ta i bruk, og vil være lite attraktiv for potensielle aksesspunktleverandører å måtte implementere,  og heller innføre en moderne og sikker lettvektsprotokoll som REST.
* Gjenbrukbart format også for andre meldinger i eMeldingsinfrastrukturen
* Utviklervenlig format med høyt rammeverk- og produktstøtte, for å gjøre det så lett som mulig å sende meldinger fra hjørne 1.

Forslaget er derfor:
1. [*Dokumentpakken*, dvs ASiC-pakken](https://docs.digdir.no/dokumentpakke_index.html) beholdes uendret
2. [*Forretningsmeldingene*](https://docs.digdir.no/sdp_index.html) beholdes også stort sett uendret, men:
  * Formatet skal endres fra XML til JSON for å bli mer tilpasset vanlig REST-bruk
    * json-schema følge dagens xsd så langt det lar seg gjøre.
  * Strukturen er fremdeles en SBD, dvs. består av  
    * Først en [*SBDH*](https://docs.digdir.no/standardbusinessdocument_index.html), nå JSON-ifisert.
    * Så selve forretningsmeldinga (eks. digitalpostmelding), også JSON-ifisert
 * Maskinporten-token må inkluderes i forretningsmeldinga slik at PK-leverandør kan motta dette som bevis på tillatelse til å sende post
 * Hele SBD'en må signeres på meldingsnivå for å sikre ende-til-ende integritet, og den må defor da bli en JWT.


# Grensesnittsdefinisjoner

## REST-api mellom Avsender og Hjørne 2

Aksesspunkt-leverandør i Hjørne 2 skal tilby et enkelt REST-endepunkt som Avsender bruker å sende post og hente kvitteringar.

REST-grensesnittet skal sikres med Bearer tokens fra Maskinporten, se: https://docs.digdir.no/maskinporten_auth_server-to-server-oauth2.html.  Informasjon om godkjente avsendere og deres eventuell databehandlere blir kodet inn i dette tokenet.

Aksesspunkt-leverandør skal tilby 2 endepunkt:

### Sende post

Post sendes i to steg - i første steg sendes forretningsmeldinga:
```
POST /sendmelding/{meldingsid}
Authorization: Bearer <maskinporten_token>

Body:
<forretningsmelding i JWT-format>
```

og i andre steg sendes / strømmes selve dokumentet.
```
PUT /sendmelding/{meldingsid}
Authorization: Bearer <maskinporten_token>

Body:
<ASCI-e>
```
Oppdeling i to steg skaper fleksibilitet og sikrer støtte for store dokumenter, evt. flere dokumenter på samme melding  (for andre meldingstyper enn DigitalPostMelding)

### Hente kvitteringer:

URL på formen
```
GET /kvittering/{conversationid}
GET /kvittering/avsenderidentifikator/{conversationid}
```
### Se status på en melding
Gir en statuskode på hva som har skjedd med den sendte meldinga.
```
GET /status/{conversationid}
```

### API-definisjoner:

| Type | Definisjon |
|-|-|
| SBDH | https://github.com/joergenb/dpi_transport/blob/main/Schemas/sbd.schema.json  (Felles for alle meldinger) |
| *Forretningsmeldinger* ||
| Digital og fysisk post | https://github.com/joergenb/dpi_transport/blob/main/Schemas/digitalpost_dpi_1_0.schema.json |
| Kvittering | TODO |
| API |   TODO: skrive openapi-definisjon |

## PEPPOL (hjørne 2-> 3)

tbd

## Hjørne 3 -> Postkasse-leverandør

Dette avtales bilateralt mellom de to partene.


# Forutsetninger:

### System-oppsett

Digdir oppretter maskinporten-scopet `dpi:send`. Tilgang til dette scopet betyr at Avsender har inngår bruksvilkår for Digital Postkasse til Innbygger.  Digdir settes som eier av Maskinporten-scopet, som betyr at det er Digdir som administrerer hvem som får tilgang. Faktura for konsumentene (= alle avsendere) går til Digdir selv og faktureres ikke.

Ved bruk av Altinn Autorisasjon til delegering, må det opprettes et "delegationScheme" i Altinn som muliggjør at Behandlingsansvarlig kan delegere til Databehandler.   Digdir blir eier av delegationSchemet, og Digdir vil motta faktura for bruk av Altinn.

Det opprettes `processid` i ELMA for de dokumenttyper som trengs støttes.
- digitalpost
- fysiskpost  (eller er denne eigentleg berre ein type digitalpost, ref dok?)
- dpi-kvittering
- flytt-digitalpost



### Oppsett av PK-leverandør

PK-leverandørene registreres som mottakere av aktuelle processid i ELMA. Må utføres av Hjørne 3.


### Oppsett av ny Avsender

Digdir gjev Avsender tilgang i Maskinporten til oauth2-scopet `dpi:send`  når bruksvilkår for Digital Postkasse er inngått.

Alt 1: Avsender som skal setje opp sitt eige system, må få tilgang til Samarbeidsportalen (i Prod) og registere ein maskinporten-integrasjon med det aktuelle scopet.  Fagsystemet/oauth2-klieten må konfigurerast med den genererte `client_id` og Avsender sitt virksomheitssertifikat.  Avsender som bruker integrasjonspunktet, trenger ikke gjennomfør dette, men må følge de retningslinjer som gjelder for å få satt opp et integrasjonspunkt.

Alt 2: Avsender som nyttar systemleverandør/databehandlar, må istaden logge inn i Altinn og delegere tilgangen til å sende DPI, vidare til systemleverandør.   Systemleverandør må få tilgang til Samarbeidsportalen og lage Maskinporten-integrasjon på same måte som sjølvstendige avsendere. Merk at systemleverandør autentiserer seg mot Maskinporten med sitt eige virksomheitsertifikat, og treng ikkje ha Avsender sitt.

Avsender (evt. systemleverandør) inngår so ein avtale med ein aksesspunkt-leverandør.  Fagsystemet blir konfigureret og integerert mot aksesspunktleverandør.

Avsender må bli satt opp i ELMA som mottaker av DPI-kvitteringsmeldinger. Dette gjør aksesspunktleverandør.

PK-leverandør mottar beskjed manuelt fra Digdir om at det er etablert en ny Avsender.

TODO:  kva viss avsender har fleire fagsystemer som sender digital post (/Avsender/avsenderidentifiktor/ ) - må alle Oslo Kommune sine systemer være kobla til samme aksesspunkt ?


# Meldingsflyt

**(dersom du ikkje ser eit sekvensdiagram under her, må du få opna dokumentet i noko som kan vise mermaid inline grafikk)**


<div class="mermaid">
sequenceDiagram
  participant A as Avsender
  participant F as Fagsystem
  participant MP as Maskinporten
  participant ELMA
  participant AA as Altinn Autorisasjon
  participant K as Kontaktregisteret

  participant C2 as Hjørne 2
  participant C3 as Hjørne 3
  participant PK as Postkasse-leverandør

  note over F: signeringssertikat (tilhører enten Avsender eller Databehandler)

  note over A, PK: Oppsett
  A->>MP: Digdir gir tilgang til Avsender
  A->>C2: Inngå avtale med aksesspunktleverandør
  C2->>ELMA: Hjørne 2 oppretter Avsender i ELMA
  A->>PK: Digdir forteller PK at ny Avsender er opprettet (manuell prosess)

  opt Dersom Avsender ønsker å bruke en leverandør/Databehandler
    A->>AA: Bemyndiget ansatt hos Avsender delegerer til Databehandler (valgfritt, asynkront)
  end
  note over A, PK: Runtime

  activate F
  F->>K: Forespør innbyggers postkasse
  activate K
  K-->>F: orgno pk-leverandør, krypteringssertifikat
  deactivate K
  F->>MP: Forespør token
  activate MP
  opt Dersom Fagsystem ikke tilhører Avsender
    MP->>AA: Sjekker om tilgang er delegert Databehandler
  end
  MP-->>F: maskinporten_token
  deactivate MP
  F->>C2: POST /sendmelding/{meldingsid} (Forretningmelding)
  activate C2
  C2-->>F: 200 OK
  F->>C2: PUT  /sendmelding/{meldingsid} (Dokumentpakke)
  C2->>C2: validere
  C2-->>F: 200 OK
  deactivate F
  C2->>ELMA: oppslag (meldingstype + orgno til postkasse)
  activate ELMA
  ELMA-->>C2: adresse til Hjørne 3
  deactivate ELMA

  C2->>C2: pakke om melding til PEPPOL-format
  C2->>C3: PEPPOL-melding over AS4
  activate C3
  C3-->>C2: akseptert
  deactivate C2
  C3->>PK: levere melding (bilateral protokoll)
  deactivate C3

  note over  PK: validere og putte i innbyggers postkasse

  note over PK,A: kvitteringer

  PK->>C3: kvittering til Avsender (bilateral protokoll)
  activate C3
  C3->>ELMA: oppslag (meldingstype=kvittering, orgno til avsender)
  ELMA-->>C3: adresse til hjørne 2

  C3->>C2: PEPPOL-melding over AS4
  activate C2
  C2-->>C3: akseptert
  deactivate C3
  F->>MP: Forespør token
  activate F
  activate MP
  opt Dersom Fagsystem ikke tilhører Avsender
    MP->>AA: Sjekker om tilgang er delegert Databehandler
  end
  MP-->>F: maskinporten_token
  deactivate MP
  F->>C2: poller på kvitteringer (GET /status/{conversationid} )
  deactivate C2
  deactivate F

</div>

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
      "ID": "0192:999888777"      // Eventuell Databehandler.
  }
  "iat": <timestamp>
  "exp": <iat-verdi + 30 sec>
  }
```

Dersom avsender er sin egen Databehandler så mangler tokenet `supplier`-claimet.


## 2:   Avsender sender post-melding

Avsender slår opp i KRR og finner hvilken PK-leverandør som innbygger benytter - og vet derigjennom om det skal sendes digital eller fysisk post.

Avsender kan nå konstruere korrekt **forretningsmelding** (DigitalPostMelding) etter [dagens regler](https://docs.digdir.no/sdp_digitalpostmeldinger.html), men med følgende tillegg/endringer:

* Format skal være JSON, og følge skjema-definisjonen her: https://github.com/joergenb/dpi_transport/blob/main/Schemas/digitalpost_dpi_1_0.schema.json
* Strukturen er fremdeles en SBD, dvs. består av  
  * Først en [*SBDH*](https://docs.digdir.no/standardbusinessdocument_index.html), nå JSON-ifisert.
  * Så selve forretningsmeldinga (eks. digitalpostmelding), også JSON-ifisert
* Token mottatt fra Maskinporten inkluderes i et felt `maskinporten_token` under `digitalpost`
* Hele SBD'en må signeres på meldingsnivå for å sikre ende-til-ende integritet, og den må defor da bli en JWT.
  * Databehandler må signere forretningsmeldingen med samme sertifikat som benyttes til å signere Dokumentpakke.
  * TODO: hvordan får PK-leverandør vite sertifikatet slik at full X509-validering + orgno-sjekk kan skje?

Regler for hvilke organisasjonsnummer som skal på ulike steder i meldingsstrukturen videreføres, se [eksempelet om Bunadsrådet og Acos i eksisterende dokumentasjon](https://docs.digdir.no/sdp_meldingsformat.html?h=bunadsr%C3%A5det#p%C3%A5vegne-eksempel).

**Eksempel på forretningsmeldinger**:
* Se https://github.com/joergenb/dpi_transport/blob/main/Samples/DIGITALPOST_DPI_1_0_Sample.json for et eksempel med digital post. `receiver` er her PK-leverandør sitt orgno.
* Se https://github.com/joergenb/dpi_transport/blob/main/Samples/FYSISKPOST_PRINT_1_0_Sample.json for eksempel med fysisk post. `receiver` er her org.no til printtjeneste-leverandør.



Avsender lager nå til slutt en unik meldingsid, og sender så posten  i to steg - i første steg sendes forretningsmeldinga:
```
POST /sendmelding/{meldingsid}
Host: api.aksesspunktleveradandør.no
Content-Type: application/jwt
Authorization: Bearer <maskinporten_token>

Body:
<forretningsmelding i JWT-format>
```

og i andre steg sendes / strømmes selve dokumentpakka.
```
PUT /sendmelding/{meldingsid}
Host: api.aksesspunktleveradandør.no
Content-Type: vnd.etsi.asic-e+zip
Authorization: Bearer <maskinporten_token>

Body:
<ASCI-e>
```







### 3: Aksesspunkt-leverandør mottek melding

Aksesspunktleverandør (APL) må gjennomføre ei teknisk validering av Maskinporten-tokenet ihht Oauth-standarden (ustedet av Maskinporten, gyldig signatur, ikkje utløpt).  
- Utgått token medfører 401-respons
- Gyldig token men som mangler "dpi:send"-scope eller avsender med manglande avtale med APL -> 403 repons

Aksesspunktleverandør må videre validere at:
* `consumer`-claimet i token stemmer med Avsender i forretningsmelding.
* [`aud`-claimet i token](ttps://docs.digdir.no/maskinporten_func_audience_restricted_tokens.html) stemmer med eget API-endepunkt
* `maskinporten_token` i forretningsmelding er identisk med det som ble brukt som Bearer token det aktuelle API-kallet
* `meldingsid` ikkje er forsøkt brukt tidlegare.


Aksesspunktleverandør lagrer converstation-id og tilhørende avsender/databehandler og avsenderidentifikator, slik at kvitteringsmeldinger og feilmeldinger relatert til meldingen kan håndteres, og fagsystemet kan etterspørre status.


### 4: Aksesspunkt-leverandør i hjørne 2 sender meldinga vidare til hjørne 3

APL mapper `documentIdentification/type` i SBDH-delen av forretningmelding til riktig processid ihht PEPPOL
APL slår opp i ELMA på processid og PK-leverandørs orgno (=receiver i SBDH-delen av forretningsmeldinga) og får hvem som er hjørne 3 for den aktuelle PK-leverandøren.

APL kan nå konstrurere en PEPPOL-melding. Dvs:  
* Pakke forretningmelding og dokumentpakke om til avtalt payload-format
  *
* Lage PEPPOL-konvolutt ("ytre" SBDH)
  * SBDH `Receiever` settes lik pk-leverandør orgno (?)
  * SBDH `processid` settes lik `processid`
  * SBDH `Sender` settes lik  (TODO: Avsender eller Databehandler)?

**Eksempel**: Et mulig transport-format i PEPPOL kan se ut som her: https://github.com/joergenb/dpi_transport/blob/main/Samples/DIGITALPOST_DPI_1_0_Minimal_Sample.xml




### 5: Hjørne 3 mottek meldinga, og sender vidare til hjørne 4

C3 og PK-leverandør avtaler selv protokoll seg i mellom.  

De står fritt til å bruke PEPPOL payload-formatet direkte, eller dele opp forretningsmelding/dokumentpakke slik det er gjort over Avsender->Hjørne 2-grensesnittet, eller andre hensiktsmessige protokoller.  Det må dog være enkelt og entydig for PK-leverandør å koble dokumentpakke og forretningsmelding sammen.

Partene bestemmer selv hvilken sikringsmekanisme de vil ha (feks egen oauth2 autorisasjonsserver, bruke 2-vegs tls, eller noe annet). Vi anbefaler ikke at de bruker maskinporten-tokenet, da dette kan ha utløpt undervegs.


### 6: Hjørne 4 mottek meldinga og puttar i postkassen til innnbygger

Ved mottak av melding, må postkasse-leverandør validere ende-til-ende integritet, dvs:

a: at DigitalPostMelding er signert av Avsender(eller Databehandler) og inneholder en digest for tilhørende dokumentpakke
b: at dokumentpakken er signert av Avsender(eller Databehandler)
c: regne ut digest av dokumentpakken og kontrollere at utrekna digest stemmer med påstått verdi i forretningsmeldinga


d: validere at Avsender i forretningsmeldinger stemmer med `consumer`-claimet i `maskinporten_token` i forretningsmeldinga.
e: validere at virksomhetssertifikatet som er brukt til å signere både Dokumentpakke og DigitalPostMelding stemmer med autorisert avsender (dvs maskinporten-token)
  * lik `supplier`s orgno, dersom dette claimet finnes i maskinporten_token
  * lik `consumer`s orgno ellers


# Kvitteringer

PK-leverandør må kunne asynkront sende kvittering tilbake i C3 -> C2 -> C1.  (dvs C3 må tilby et API eller mekanisme der PK-leverandør kan overføre kvitteringer, og dette må ta høyde for at slik kvitteringslevering skal kunne skje asynkront ifht levering av digitalpostmelding C3->C4.)

Feltet `conversation_id` kobler sammen en kvittering med tilhørende DigitalPostMelding. Det er lov å sende flere kvitteringer tilhørene en og samme conversation_id.

PK-leverandør lager en **forretningsmelding** (LeveringsKvittering, VarslingFeilet, Mottak, ReturPost) etter dagens regler, men med endring tilsvarende det som ble skissert i steg 2. Dvs.:
* Format skal være JSON, og følge skjema-definisjonen.
  * PK-leverandør legger til Avsenders orgno i feltet `Avsender`
  * PK-leverandør legger til evt. Databehandlers orgno i feltet `Databehandler`

* Strukturen er fremdeles en SBD, dvs. består av
    * Først en SBDH, nå JSON-ifisert.
    * Så selve forretningsmeldinga (eks. leveringskvittering), også JSON-ifisert
* Hele SBD'en må signeres på meldingsnivå for å sikre ende-til-ende integritet, og den må defor da bli en JWT.
* PK-Leverandør må signere forretningsmeldingen med sitt virksomhetssertifikat.


PK-leverandør ber så om at C3 sender kvitteringa til Avsender.



C3 slår opp i ELMA og finner hvem som er C2 for Avsender.
C3 lager en PEPPOL-melding til C2.
C2 mottar kvitteringa, og legger den i kø.  Venter på at Avsenders system poll'er på kvitteringer med riktig conversation-id.  Verifiserer at Avsender som forsøker å hente kvittering, er den samme som sendte digitalpostmeldinga.


TODO: kva viss det er fleire fagsystemer hjå Avsender som sender Digtal Post - korleis identifisere og adressere det rette?




# 2-vegs svar

TODO

# Annet

TODO: Bør man legge til mer info i kvitteringer for routing tilbake til avsender systeme. Feks avsenderidentifikator i retur

TODO: Trenger man et statusgrensesnitt der man kan se transportstatus på meldingen fra C2-C3
