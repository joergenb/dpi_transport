Dette dokumentet er eit framlegg til nytt, proprietært REST-api sikret med Maskinporten, for å sende Digital Post til Innbygger.


# Bakgrunn

Den proprietære transportinfrastrukturen for Digital Postkasse til innbyggere skal erstattes med en standard-infrastruktur for meldingsutvekling i det offentlege, dvs 4-hjørnes-modell med CEF eDelivery/PEPPOL:
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


Det treng etablerast ein ny transport-protokoll mellom hjørne 1 og hjørne 2.  I tradisjonell 4-hjørnes modell er dette noe som markedsmekanismene selv skal ta fram.   prinsippet er dette opp til aksesspunktleverandøren å bestemme, men for å sikre rask overgang til ny transportinfrastruktur, vil Digdir føreslå ein protokoll.


# Revidert meldingsformat i DPI

For å tilpasse meldingsformatet noe til REST, men samtidig unngå for store endringer i eksisterende format, foreslår vi at meldingsformatat for DPI endres noe.  Nytt format består av to deler:

1. *Forretningsmelding* blir forenklet til en signert JSON-struktur
2. *Dokumentpakke*, dvs ASiC-pakken beholdes uendret



# Grensesnittsdefinisjon REST-api

Aksesspunkt-leverandør i Hjørne 2 skal tilby et enkelt REST-endepunkt som Avsender bruker å sende post og hente kvitteringar.

REST-grensesnittet skal sikres med Bearer tokens fra Maskinporten, se:  https://docs.digdir.no/maskinporten_auth_server-to-server-oauth2.html

Følgenede endepunkt skal tilbys:

- POST /send/digitalpostmelding/{meldingsid}
- POST /send/fysiskpostmelding/{meldingsid}
- GET /kvittering/{ConversationId}

TODO:  treng vi skille på digital og fysisk dersom vi heller mapper dette som to ulike processiD i PEPPOL ?

Grensesnitt er dokumentert i OpenAPI-format her:




Autorisert avsender-informasjon kjem frå Maskinporten-tokenet direkte, og ikkje lenger frå signerte ebMS-meldingar.


TODO: trengs serializering  av JSON/XML ?
TODO: kva med PUT og DELETE på tidlegare innsendt melding
TODO: kva med store filer /mange vedlegg?
TODO: kva med POST /flytt/digitalpost/





# 1: System-oppsett

Digdir oppretter maskinporten-scopet `dpi:send`. Tilgang til dette scopet betyr at Avsender har inngår bruksvilkår for Digital Postkasse til Innbygger.  Digdir settes som eier, som betyr at det er Digdir som administrerer hvem som får tilgang. Faktura for konsumentene (= alle avsendere) går til Digdir selv og ikke PK-leverandørene.

Det opprettes `procesid` i PEPPOL  (TOOD:  hvor ?BCP/BCL ?) for de dokumenttyper som trengs støttes.
- digitalpost
- fysiskpost
- dpi-kvittering


TODO: trengs det opprettes noe mer ?


# 2: Oppsett av PK-leverandør

PK-leverandørene registreres som mottakere av aktuelle processid i TODO??





# 3: Oppsett av ny Avsender

Digdir gjev Avsender tilgang i Maskinporten til oauth2-scopet `dpi:send`  når bruksvilkår for Digital Postkasse er inngått.

Alt 1: Avsender som skal setje opp sitt eige system, må få tilgang til Samarbeidsportalen (i Prod) og registere ein maskinporten-integrasjon med det aktuelle scopet.  Fagsystemet/oauth2-klieten må konfigurerast med den genererte `client_id` og Avsender sitt virksomheitssertifikat.

Alt 2: Avsender som nyttar systemleverandør/databehandlar, må istaden logge inn i Altinn og delegere tilgangen til å sende DPI, vidare til systemleverandør.   Systemleverandør må få tilgang til Samarbeidsportalen og lage Maskinporten-integrasjon på same måte som sjølvstendige avsendere. Merk at systemleverandør autentiserer seg mot Maskinporten med sitt eige virksomheitsertifikat, og treng ikkje ha Avsender sitt.

Avsender (evt. systemleverandør) inngår so ein avtale med ein aksesspunkt-leverandør.  Fagsystemet blir konfigureret og integerert mot aksesspunktleverandør.

TODO: noko om oppsett av Integrasjonspunktet?

TODO: Avsender (eller Databehandler???) blir satt opp i ELMA?   som mottaker av DPI-kvitteringsmeldinger ?




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

Dersom avsender nyttar ein Databehandler, so vil tokenet ogso ha eit `supplier`-claim som inneheld systemleverandørens organisasjonsnummer.


Avsender konstruerer en **forretningsmelding**
* token mottatt fra Maskinporten puttes i `maskinporten_token`.
* Avsender må sjekke KRR for å finne hvilken postkasse avsender benytter, og sette `postkasseorg` lik postkassens orgno.
* Avsneder må sette `processID` til korrekt verdi, alt etter om det er fysisk post (TODO: verdi) eller digital post (TODO: verdi)
* Avsender regner ut en digest av dokumentpakke, og inkluderer som `dokumentpakkefingeravtrykk`
* `meldingsid` og `ConversationId` må ha tilstrekkeleg entropi til at den vil vere unik over alle meldinger frå alle avsendere over tid.


Avsender signerer forretningsmelding slik at den blir en JWS.



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
    "transportinfo": {
        "sender": {
            "Identifier": {
                "Authority": "iso6523-actorid-upis",
                "ID": "0192:999888777"
            }
        },
        "receiver": {
            "Identifier": {
                "Authority": "iso6523-actorid-upis",
                "ID": "0192:999111222"
            }
        },
        "processid": "digitalpost",
        "ConversationId": "37efbd4c-413d-4e2c-bbc5-257ef4a65a45"
    },
    "digitalpostmelding": {
        "avsender": {
            "Identifier": {
                "Authority": "iso6523-actorid-upis",
                "ID": "0192:999888999"
            }
        },
        "mottaker": {
            "Identifier": {
                "Authority": "norsk_personidentifikator",
                "ID": "17050411111"
            }
        },
        "dokumentpakkefingeravtrykk": {
            "DigestMethod": "http://www.w3.org/2001/04/xmlenc#sha256",
            "DigestValue": "xxxxxxx"
        },
        "maskinporten_token": "eyJAxxx",
        "digitalpostinfo": {
            "sikkerhetsnivaa": 4,
            "virkningsdato": "2020-12-31",
            "aapningskvittering": "false",
            "ikkesensitivtittel": "",
            "epostvarsel": {
                "epostTekst": "Varseltekst",
                "epostAdresse": "dpi@digdir.no",
                "dagerEtter": [
                    0,
                    7
                ]
            },
            "smsvarsel": {
                "smsTekst": "Varseltekst",
                "mobilnummer": "004799999999",
                "dagerEtter": [
                    0,
                    7
                ]
            }
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




**4: Aksesspunkt-leverandør sender meldinga vidare**

APL ser på forretningsmelding og lager en PEPPOL-SBDH basert på denne:
* SBDH `Receiever` settes lik `postkasseorg`
* SBDH `processid` settes lik `processid`
* SBDH `Sender` settes lik
  *

TODO:  må APL "pakke om" meldinga før den sendes vidare i PEPPOL ?, td:
- Aksesspunktleverandør kopierer Avsenders orgno frå `consumer`-claim i token, inn i forretningsmeldinga som `avsender`
- Aksesspunktleverandør kopierer Databehandlers orgno frå `supplier`-claim i token, inn i forretningsmelidnga som `databehandler`

```

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
