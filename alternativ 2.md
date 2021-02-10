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

Forslaget er derfor:
1. [*Dokumentpakken*, dvs ASiC-pakken](https://docs.digdir.no/dokumentpakke_index.html) beholdes uendret
2. [*Forretningsmeldingene*](https://docs.digdir.no/sdp_index.html) beholdes også stort sett uendret.
 * TBD om formatet skal endres fra XML til JSON for å bli mer tilpasset vanlig REST-bruk
3. [*SBDH*](https://docs.digdir.no/standardbusinessdocument_index.html) fjernes
 * Informasjonen som finnes her idag, flyttes delvis inn forretningsmelding, eller som del av REST-URLene.
 

# Grensesnittsdefinisjon REST-api

Aksesspunkt-leverandør i Hjørne 2 skal tilby et enkelt REST-endepunkt som Avsender bruker å sende post og hente kvitteringar.

REST-grensesnittet skal sikres med Bearer tokens fra Maskinporten, se: https://docs.digdir.no/maskinporten_auth_server-to-server-oauth2.html.  Informasjon om godkjente avsendere og  En aksesspunktleverandør kan stole på at slike tokens betyr at Avsender har lov til å sende digital post.  okens utsted av Maskinporten Kun godkjente Avsendere, og deres eventuelle Databehandlere 

Følgende endepunkt skal tilbys:

- POST /send/{meldingstype}/{postkasseleverandørorgno}/{conversationid}/{meldingsid}
- GET /kvittering/{conversationid}

TODO: vurdere om c2 skal PUSHE kvitteringer tilbake til avsender / systemleverandør, istedenfor pull.
TODO: trengs serializering / canonicalization  av JSON/XML ?
TODO: kva med PUT og DELETE på tidlegare innsendt melding
TODO: kva med store filer /mange vedlegg?
TODO: kva med POST /flytt/digitalpost/



# 1: System-oppsett

Digdir oppretter maskinporten-scopet `dpi:send`. Tilgang til dette scopet betyr at Avsender har inngår bruksvilkår for Digital Postkasse til Innbygger.  Digdir settes som eier, som betyr at det er Digdir som administrerer hvem som får tilgang. Faktura for konsumentene (= alle avsendere) går til Digdir selv og faktureres ikke.

Det opprettes `processid` i ELMA for de dokumenttyper som trengs støttes.
- digitalpost:
- fysiskpost:
- dpi-kvittering:


TODO: trengs det opprettes noe mer ?


# 2: Oppsett av PK-leverandør

PK-leverandørene registreres som mottakere av aktuelle processid i ELMA.


# 3: Oppsett av ny Avsender

Digdir gjev Avsender tilgang i Maskinporten til oauth2-scopet `dpi:send`  når bruksvilkår for Digital Postkasse er inngått.

Alt 1: Avsender som skal setje opp sitt eige system, må få tilgang til Samarbeidsportalen (i Prod) og registere ein maskinporten-integrasjon med det aktuelle scopet.  Fagsystemet/oauth2-klieten må konfigurerast med den genererte `client_id` og Avsender sitt virksomheitssertifikat.  Avsender som bruker integrasjonspunktet, trenger ikke gjennomfør dette, men må følge de retningslinjer som gjelder for å få satt opp et integrasjonspunkt.

Alt 2: Avsender som nyttar systemleverandør/databehandlar, må istaden logge inn i Altinn og delegere tilgangen til å sende DPI, vidare til systemleverandør.   Systemleverandør må få tilgang til Samarbeidsportalen og lage Maskinporten-integrasjon på same måte som sjølvstendige avsendere. Merk at systemleverandør autentiserer seg mot Maskinporten med sitt eige virksomheitsertifikat, og treng ikkje ha Avsender sitt.

Avsender (evt. systemleverandør) inngår so ein avtale med ein aksesspunkt-leverandør.  Fagsystemet blir konfigureret og integerert mot aksesspunktleverandør.

TODO: noko om oppsett av Integrasjonspunktet?

TODO: Avsender (eller Databehandler???) blir satt opp i ELMA?   som mottaker av DPI-kvitteringsmeldinger ?




## Eksempel på å sende melding

**1:  Avsender henter token fra maskinporten**

Fagsystem signerer eit JWT grant ihht https://docs.digdir.no/maskinporten_protocol_jwtgrant.html, sender dette til Maskinporten og får  eit access_token i retur.

Ved utstedelse av token vil Maskinpoten kontrollere:
- at Avsender har lov til å bruka DPI.
- Viss Avsender har databehandler, sjekkar Maskinporten mot Altinn Autorisasjon at aktuell autentisert Databehandlar har fått lov til å opptre på vegne av Avsender for sending av DPI.


Døme på access_token
```
  "iss": "https://maskinporten.no",
  "scope": "dpi:send",
  "aud": "https://api.aksesspunktleverandør.no/"
  "consumer": {
    "Identifier": {
      "Authority": "iso6523-actorid-upis",
      "ID": "0192:991825827"
    }
  "iat": <timestamp>
  "exp": <iat-verdi + 30 sec>
  }
```

Dersom avsender nyttar ein Databehandler, so vil tokenet ogso ha eit `supplier`-claim som inneheld systemleverandørens organisasjonsnummer.


Avsender konstruerer en **forretningsmelding**
* token mottatt fra Maskinporten puttes i `maskinporten_token`.
* Avsender må sjekke KRR for å finne hvilken postkasse avsender benytter, og sette `postkasseorg` lik postkassens orgno.
* Avsender må sette `processID` til korrekt verdi, alt etter om det er fysisk post (TODO: verdi) eller digital post (TODO: verdi)
* Avsender regner ut en digest av dokumentpakke, og inkluderer som `dokumentpakkefingeravtrykk`
* `meldingsid` og `ConversationId` må ha tilstrekkeleg entropi til at den vil vere unik over alle meldinger frå alle avsendere over tid.


Avsender signerer forretningsmelding slik at den blir en JWS.



**2:   Avsender sender post-melding**



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


**3: Aksesspunkt-leverandør mottek melding**

Aksesspunktleverandør (APL) må gjennomføre ei teknisk validering av Maskinporten-tokenet (ustedet av Maskinporten, gyldig signatur, ikkje utløpt).  
- Utgått token medfører 401-respons
- Gyldig token men som mangler "dpi:send"-scope eller avsender med manglande avtale med APL -> 403 repons

Aksesspunktleverandør må videre validere at:
* `consumer`-claimet i token stemmer med Avsender i forretningsmelding.
* `maskinporten_token` i forretningsmelding er samme som ble brukt som Bearer token på API-kallet
* `meldingsid` ikkje er forsøkt brukt tidlegare.




Aksesspunkt-leverandørar *bør* bruke [audience-begrensa](https://docs.digdir.no/maskinporten_func_audience_restricted_tokens.html) token, slik at ikkje avsender sitt fagsystem ved uhell eller forsett sender post til ein annan aksesspunkt-leverandør enn dei har avtale med.




**4: Aksesspunkt-leverandør i hjørne 2 sender meldinga vidare til hjørne 3**

APL konstrurerer en PEPPOL-SBDH basert på :
- innkommende URL
- innholdet i <DigitalPostMelding>
- oppslag i ELMA for processid og postkasseleverandør-orgno.
  
TODO: hvordan adresseres C3?
TODO: hvordan får C3 vite at denne meldingen skal til C4 ?
TODO: må vi her ta vite hvem som skal ha tilhørende kvitteringer (avsender selv, eller databehandlers system)?

Dvs:  
* SBDH `Receiever` settes lik ? 
* SBDH `processid` settes lik `processid`
* SBDH `Sender` settes lik  ? 



**5: Hjørne 3 mottek meldinga, og sender vidare til hjørne 4**

TODO: C3 trenger å vite om meldingen skal til Digipost eller eBoks

C3 og C4 avtaler selv protokoll seg i mellom.  De kan gjerne gjenbruke C1-C2 REST-grensesnittet, men må finne egen sikringsmekanisme (egen oauth2 autorisasjonsserver, eller bare bruke 2-vegs tls).


**6: Hjørne 4 mottek meldinga og puttar i postkassen til innnbygger**

Ved mottak av melding, må postkasse-leverandør validere ende-til-ende integritet, dvs:
a: at forretningsmelding er signert av Avsender(eller Sender) og inneholder en digest av den dokumentpakken som skal være tilhørende foretningsmeldinga
b: at dokumentpakken er signert av Avsender(eller Sender)
c: regne ut digest av dokumentpakken og kontrollere at utrekna digest stemmer med påstått verdi i forretningsmeldinga
d: validere at Avsender i forretningsmeldinger stemmer med `consumer` i `maskinporten_token` i forretningsmeldinga.
e: validere at virksomhetssertifikatet som er brukt til å signere både Dokumentpakke og DigitalPostMelding stemmer med autorisert avsender (dvs maskinporten-token)
  * lik `supplier`s orgno, dersom dette claimet finnes i maskinporten_token
  * lik `consumer`s orgno ellers
  
d: på-en-eller-annen-måte validere at privat-nøkkelen som er brukt til å signere i pkt a-c tilhører avsender.

# Kvitteringer

PK-leverandør må kunne asynkront sende kvittering tilbake i C3 -> C2 -> C1.  (dvs C3 må tilby et API eller mekanisme der PK-leverandør kan overføre kvitteringer, og dette må ta høyde for at slik kvitteringlevering skal kunne være asynkront av levereing av digitalpostmelding C3->C4.)


# 2-vegs svar

TODO
