# dpi_transport
 forslag ny transportinfrastruktur DPI


 ## Bakgrunn

 Transportinfrastrukturen for Digital Postkasse til innbyggere skal endrast.  Meldingsformidler forsvinner og da bør ein annen protokoll enn AS4/ebms brukes ut frå Avsender.

 Denne sida er eit framlegg til alternativ basert på eit proprietært REST-api sikret med Maskinporten.

 ##

 For å kunne sende, må avsender ha fått tildelt oauth2-scopet `dpi:send`.  Digdir tildeler scopet til


 ##

 ```
 POST /dpi/send/{}
 host:
 Authorization: Bearer <maskinporten_token m/ dpi:send scope>

 ```
