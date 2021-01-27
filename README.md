# Bakgrunn

 Den proprietære transportinfrastrukturen for Digital Postkasse til innbyggere skal endrast til å bruke ein standard-infrastruktur for meldingsutvekling i det offentlege.

- Meldingsformidler forsvinn
- 4-hjørnes-modell med CEF eDelivery/PEPPOL vert ny transport-infrastruktur.
    - Avsender (og evt. avsender sin leverandør/databehandler) er hjørne 1.
    - Digdir gjennomfører ein felles anskaffelse av hjørne-2 tenester
    - Postkasse-leverandørane blir i hjørne 4, og vil truleg operere sin eigen aksesspunkt i hjørne 3.

Det treng etablerast ein ny transport-protokoll mellom hjørne 1 og hjørne 2.  I prinsippet er dette opp til aksesspunktleverandøren å bestemme, men for å sikre rask overgang til ny transportinfrastruktur, vil Digdir føreslå ein protokoll.


# Overordna skildring

Dette dokumentet er eit framlegg til hjørne1-til-hjørne2-protokoll basert på eit proprietært REST-api sikret med Maskinporten.

Framlegget nyttar standardmekanismar i Maskinporten, sjå:  https://docs.digdir.no/maskinporten_auth_server-to-server-oauth2.html

Avsender sitt fagsystem (subsidiært Avsender sitt integrasjonspunkt i eFormidling) i Hjørne 1 er ein oauth2-klient registeret i Maskinporten

Aksesspunkt-leverandør i Hjørne 2 tilbyr enkle REST-endepunkt mot Avsender for å sende post og hente kvitteringar.  

# 1: Oppsett

Digdir gjev Avsender tilgang i Maskinporten til oauth2-scopet `dpi:send`  når bruksvilkår for Digital Postkasse er inngått.

Alt 1: Avsender som skal setje opp sitt eige system, må få tilgang til Samarbeidsportalen (i Prod) og registere ein maskinporten-integrasjon med det aktuelle scopet.  Fagsystemet/oauth2-klieten må konfigurerast med den genererte `client_id` og Avsender sitt virksomheitssertifikat.

Alt 2: Avsender som nyttar systemleverandør/databehandlar, må istaden logge inn i Altinn og delegere tilgangen til å sende DPI, vidare til systemleverandør.   Systemleverandør må få tilgang til Samarbeidsportalen og lage Maskinporten-integrasjon på same måte som sjølvstendige avsendere. Merk at systemleverandør autentiserer seg mot Maskinporten med sitt eige virksomheitsertifikat, og treng ikkje ha Avsender sitt.

Avsender (evt. systemleverandør) inngår so ein avtale med ein aksesspunkt-leverandør.  Fagsystemet blir konfigureret og integerert mot aksesspunktleverandør.

# Grensesnittsdefinisjon REST-api

Aksesspunktleverandør skal tilby følgjande endepunkt:

- POST /send/digitalpostmelding/{meldingsid}
- POST /send/fysiskpostmelding/{meldingsid}
- GET /kvittering/{id}

Payload i sende-operasjoner er ein multipart med to delar
1. Forretningsmelding, TODO:  forenkla til JSON istadenfor XML.
2. Dokumentpakke (ingen endringer i ASiC-pakke)

Autorisert avsender-informasjon kjem frå Maskinporten-tokenet direkte, og ikkje lenger frå signerte ebMS-meldingar.


TODO: trengs serializering  av JSON/XML ?
TODO: kva med PUT og DELETE på tidlegare innsendt melding
TODO: kva med store filer /mange vedlegg?
TODO: kva med POST /flytt/digitalpost/


## Eksempel på å sende melding

**1:  Avsender henter token fra maskinporten**

Fagsystem signerer eit JWT grant ihht https://docs.digdir.no/maskinporten_protocol_jwtgrant.html, sender dette til Maskinporten og får  eit access_token i retur.

- Maskinporten sjekkar at Avsender har lov til å bruka DPI.
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

Dersom avsender nyttar ein systemleverandør, so vil tokenet ogso ha eit `supplier`-claim som inneheld systemleverandørens organisasjonsnummer.


Avsender konstruerer en forretningsmelding, og putter access_token inni denne



TODO: Avsender signerer dokument-pakke.   Må/bør dette vere samme sertiifkat som opp mot maskinporten ?   Og kva viss det er Databehandler som signerer ?  Korleis kan PK-leverandør kontrollere denne signaturen?  

`meldingsid` og `ConversationId` må ha tilstrekkeleg entropi til at den vil vere unik over alle meldinger frå alle avsendere over tid.

**2:   Avsender sender post-melding**



```
POST /dpi/send/digitalpost/{meldingsid}
Host: api.aksesspunktleveradandør.no
Content-Type: application/jose
Authorization: Bearer <maskinporten_token m/ dpi:send scope>

{
    "typ":"JWT",
    "alg":"HS256"
}
.
{
    "ConversationId": "37efbd4c-413d-4e2c-bbc5-257ef4a65a45",
    "maskinporten_token": eyJA....
    "dokumentpakkefingeravtrykk": {
        "DigestMethod": "http://www.w3.org/2001/04/xmlenc#sha256",
        "DigestValue": "xxxxxxx"
    }

    // forretningsmelding under her:
    "digital": {
        "sikkerhetsnivaa": "",
        "hoveddokument": "",
        "avsenderId": "", // valgfri
        "tittel": "",
        "spraak": "NO",
        "digitalPostInfo": {
            "virkningsdato": "",
            "aapningskvittering": "false"
        },
        "varsler": { // valgfri
            "epostTekst": "Varseltekst",
            "smsTekst": "Varseltekst"
        },
        "metadataFiler": { // valgfri
            "test.pdf": "test-metadata.xml"
        }
    }
}
.
{
  < signatur-verdi>
}

--DPI-separator
Content-Disposition: form-data; name="dokumentpakke"
Content-Transfer-Encoding: chunked

<ASiC dokumentpakke; stor klump med alle dokument og vedlegg>

```

TODO:  trengs payload signerast?
TODO: finst det betre handtering av store dok og/eller mange vedlegg ?


**3: Aksesspunkt-leverandør mottek melding**

Aksesspunktleverandør (APL) må gjennomføre ei teknisk validering av Maskinporten-tokenet (ustedet av Maskinporten, gyldig signatur, ikkje utløpt).  
- Utgått token medfører 401-respons
- Gylidg token men ikkje "dpi:send"-scope eller manglande avtale med APL -> 403 repons

Aksesspunktleverandør må validere at `consumer`-claimet i token stemmer med DigitalPostMelding/Avsender i forretningsmelding.

Aksesspunkt-leverandørar *bør* bruke [audience-begrensa](https://docs.digdir.no/maskinporten_func_audience_restricted_tokens.html) token, slik at ikkje avsender sitt fagsystem ved uhell eller forsett sender post til ein annan aksesspunkt-leverandør enn dei har avtale med.

Aksesspunktleverandør må validere at meldingsid ikkje er forsøkt brukt tidlegare.


**4: Aksesspunkt-leverandør sender meldinga vidare**


TODO:  må APL "pakke om" meldinga før den sendes vidare i PEPPOL ?, td:
- Aksesspunktleverandør kopierer Avsenders orgno frå `consumer`-claim i token, inn i forretningsmeldinga som `avsender`
- Aksesspunktleverandør kopierer Databehandlers orgno frå `supplier`-claim i token, inn i forretningsmelidnga som `databehandler`

```
{
    "ConversationId": "37efbd4c-413d-4e2c-bbc5-257ef4a65a45",
    "digital": {
        "sikkerhetsnivaa": "",
        "hoveddokument": "",
        "avsenderId": "", // valgfri
        "tittel": "",
        "spraak": "NO",
        "avsender": {
          "Identifier": {
            "Authority": "iso6523-actorid-upis",
            "ID": "0192:991825827"
          }
```


TODO: Kvar kjem adressa til aktuell postkasse inn i bildet ?

**5: Hjørne 3 mottek meldinga, og sender vidare til hjørne 4**

TODO: er vi sårbare for at hjørne 3 kan modifisere relle meldingar,  eller injisere falske meldingar?

**6: Hjørne 4 mottek meldinga og puttar i postkassen til innnbygger**

Ved mottak av melding, må postkasse-leverandør validere ende-til-ende integritet, dvs:
a: at forretningsmelding er signert og inneholder en digest av den dokumentpakken som skal være tilhørende foretningsmeldinga
b: at dokumentpakken er signert
c: regne ut digest av dokumentpakken og kontrollere at utrekna digest stemmer med påstått verdi i forretningsmeldinga
d: validere at Avsender i forretningsmeldinger stemmer med `consumer` i `maskinporten_token` i forretningsmeldinga.

d: på-en-eller-annen-måte validere at privat-nøkkelen som er brukt til å signere i pkt a-c tilhører avsender.

# Kvitteringer

PK-leverandør må kunne asynkront sende kvittering tilbake i C3 -> C2 -> C1.  (dvs C3 må tilby et API eller mekanisme der PK-leverandør kan overføre kvitteringer, og dette må ta høyde for at slik kvitteringlevering skal kunne være asynkront av levereing av digitalpostmelding C3->C4.)


# 2-vegs svar

TODO
