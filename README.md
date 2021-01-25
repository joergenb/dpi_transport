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

Avsender som skal setje opp sitt eige system, må få tilgang til Samarbeidsportalen (i Prod) og registere ein maskinporten-integrasjon med det aktuelle scopet.  Fagsystemet/oauth2-klieten må konfigurerast med den genererte `client_id` og Avsender sitt virksomheitssertifikat.

Avsender som nyttar systemleverandør/databehandlar, må istaden logge inn i Altinn og delegere tilgangen til å sende DPI, vidare til systemleverandør.   Systemleverandør må få tilgang til Samarbeidsportalen og lage Maskinporten-integrasjon på same måte som sjølvstendige avsendere. Merk at systemleverandør autentiserer seg mot Maskinporten med sitt eige virksomheitsertifikat, og treng ikkje ha Avsender sitt.

Avsender (evt. systemleverandør) inngår so ein avtale med ein aksesspunkt-leverandør.  Fagsystemet blir konfigureret og integerert mot aksesspunktleverandør.

# Grensesnittsdefinisjon REST-api

Aksesspunktleverandør skal tilby følgjande endepunkt:

- POST /send/digitalpostmelding/{meldingsid}
- POST /send/fysiskpostmelding/{meldingsid}
- GET /kvittering/{id}

evt:
- POST /flytt/digitalpost/


Payload i sende-operasjoner er ein multipart, i to delar
1. Forretningsmelding, TODO:  forenkla til JSON istadenfor XML.
2. Dokumentpakke (ingen endringer i ASiC-pakke)

Avsender-informasjon kjem frå tokenet direkte.

TODO: trengs serializering  av JSON/XML ?

## Eksempel på å sende melding

**1:  Avsender henter token fra maskinporten**

Fagsystem signerer eit JWT grant ihht https://docs.digdir.no/maskinporten_protocol_jwtgrant.html, sender dette til Maskinporten og får  eit access_token i retur.

- Maskinporten sjekkar at Avsender har lov til å bruka DPI.
- Viss Avsender har databehandler, sjekkar Maskinporten mot Altinn Autorisasjon at aktuell autentisert Databehandlar har lov til å opptre på vegne av Avsender.


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



TODO: Avsender signerer dokument-pakke.   Må/bør dette vere samme sertiifkat som opp mot maskinporten ?   Og kva viss det er Databehandler som signerer ?  Korleis kan PK-leverandør kontrollere denne signaturen?  



**2:   Avsender sender post-melding**



```
POST /dpi/send/digitalpost/{meldingsid}
Host: api.aksesspunktleveradandør.no
Content-Type: multipart/form-data; boundary=DPIseparator
Authorization: Bearer <maskinporten_token m/ dpi:send scope>

--DPI-separator
Content-Disposition: form-data; name="forretningsmelding"
Content-Type: application/json

{
    "ConversationId": "37efbd4c-413d-4e2c-bbc5-257ef4a65a45",
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
    "dokumentpakkefingeravtrykk": {
        "DigestMethod": "http://www.w3.org/2001/04/xmlenc#sha256",
        "DigestValue": "xxxxxxx"
    }
}

--DPI-separator
Content-Disposition: form-data; name="dokumentpakke"
Content-Transfer-Encoding: chunked

xxxxxx

```

TODO:  trengs payload signerast?

**3: Aksesspunkt-leverandør mottek melding**

Aksesspunktleverandør (APL) må gjennomføre ei teknisk validering av Maskinporten-tokenet (ustedet av Maskinporten, gyldig signatur, ikkje utløpt).

Aksesspunktleverandør må validere at `consumer`-claimet i token stemmer med DigitalPostMelding/Avsender i forretningsmelding.

Aksesspunkt-leverandørar *bør* bruke [audience-begrensa](https://docs.digdir.no/maskinporten_func_audience_restricted_tokens.html) token, slik at ikkje avsender sitt fagsystem ved uhell eller forsett sender post til ein annan aksesspunkt-leverandør enn dei har avtale med.


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

postkasse treng ikkje lenger validere at Manifest/Avsender
