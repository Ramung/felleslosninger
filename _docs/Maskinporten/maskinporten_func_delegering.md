---
title: Delegering i Maskinporten
description: Delegering i Maskinporten
summary: "En API-konsument kan bruke Altinn til å delegere en tildelt API-tilgang videre til en leverandør."
permalink: maskinporten_func_delegering.html
sidebar: maskinporten_sidebar
product: Maskinporten
---

## Bakgrunn

Maskinporten har i samarbeid med Altinn Autorisasjon støtte for at API-konsumenter kan delegere en API-tilgang til en leverandør.

Denne funksjonaliteten inngår i nasjonal referansearkitetur for eOppslag. Se [beskrivelse av eOppslag i nasjonalt arkitekturbibliotek](https://doc.difi.no/nasjonal-arkitektur/nab_referanse_arkitekturer_eoppslag/) for ytterligere informasjon.  Se også [Altinn sin dokumentasjon av tjenesten](https://altinn.github.io/docs/utviklingsguider/sikkerhet-i-eoppslag/).

<div class="mermaid">
graph LR

  k[API-konsument]
  a[API-tilbyder]
  l[Leverandør]

  k---| har avtale om tilgang til data med | a
  k---| har databehandleravtale med | l
  l---| henter data på vegne av konsument fra | a
</div>



### Initielt oppsett

For å klaregjøre for bruk av delegering, må API-tilbyder registrere nødvendige opplysninger om APIet som følger:  

<div class="mermaid">
sequenceDiagram
  participant api as API-tilbyder
  participant m as Maskinporten
  participant a as Altinn
  participant k as API-katalogen

  api ->> m: opprette oauth2 scopes  
  api ->> a:  opprette delegeringsoppsett
  api ->> k: registrere APIet i API-katalogen

</div>


Scope i Maskinporten [opprettes ved bruk av selvbetjening](/maskinporten_guide_apitilbyder.html#administrasjon-av-api),  men du må passe på å registrere en **delegeringskilde**.  Liste over gyldige delegeringskilder finnes nederst på denne siden, du skal bruke `issuer`-verdien som `delegation_source` på scopet ditt:

```
POST /scopes HTTP/1.1
Host: integrasjon-ver2.difi.no
Content-Type: application/json
Authorization: Bearer <et OIDC-token med 'idporten:scopes.write' >
{
  "prefix": "difitest",
  "subscope": "test2",
  "description": "scope som støtter ekstern delegeringskilde",
  "delegation_source": "https://tt02.altinn.no"
}
```

Du må så [opprette et delegeringsoppett i Altinn](https://altinn.github.io/docs/utviklingsguider/sikkerhet-i-eoppslag/api-eier/#registrering-av-delegerbar-ressurs-i-altinn), som inneholder scopet du nettopp registrerte i Maskinporten:

```
POST /maskinporten-api/delegationSchemes HTTP/1.1
Host: tt02.altinn.no
Content-Type: application/json
Authorization: Bearer  <et maskinporten-token med 'altinn:maskinporten/delegationsSchemes.write'>

{
    "owner_org": "991825827",
    "scopes": [
        "difitest:test2"
    ],
    "title": [
        {
            "code": "nb_NO",
            "value": "Difi tester delegering"
        }
    ],
    "description": [
        {
            "code": "nb_NO",
            "value": "Bla bla bla bla bla og enno meir blah"
        }
    ],
    "default_language": "nb_NO"
}
```
Merk at det kan ta noen minutter før delegeringsoppsettet blir synlig i Altinn-portalen.


Til slutt anbefaler vi at du registrerer API'et ditt i API-katalogen, slik at andre potensielle konsumenter kan finne det.

### Gi tilgang til konsumenter

API-tilbyder bruker [selvbetjening for å gi tilgang til konsumenter](https://difi.github.io/felleslosninger/maskinporten_guide_apitilbyder.html#2b-tilgangsstyring---oauth2-selvbetjeningsklient) på ordinær måte.  

### Lage en leverandør-integrasjon

Leverandør-integrasjoner som skal bruke ekstern delegering, er litt forskjellige fra andre integrasjoner i ID-porten/Maskinporten, og det er derfor viktig å få de registrert korrekt.


ID-porten/Maskinporten har allerede [to eksisterende interne delegeringsmekanismer](oidc_api_admin.html#eierskap-til-integrasjoner), og det er viktig at disse ikke blandes:

* For leverandører som bruker selvbetjeningsløsningen på samarbeidsportalen, er det viktig å merke seg at du må opprette integrasjonen som tilhørende deg selv, og ikke velge "på vegne av en kunde" eller "på vegne av flere kunder".
* For leverandører som bruke selvbetjenings-API, må de **ikke** bruke tokens med `idporten:dcr.supplier`-scopet, men derimot `idporten:dcr.write`: leverandøren skal altså ikke sette `client_orgno` i registrerings-kallet.


**Merk: i første utgave av denne funksjonaliteten er det ikke mulig for leverandør å registrere det aktuelle oauth2-scopet tilhørende API-tilbyder på klienten. Dette vil bli endret ila. våren 2020.**


### Delegering i Altinn

Bemyndiget ansatt hos API-konsument logger inn i Altinn, velger å representere foretaket, søker opp og delegerer API-tilgangen videre til leverandøren i portal-løsningen.  [Dette er nærmere dokumentert hos Altinn](https://altinn.github.io/docs/utviklingsguider/sikkerhet-i-eoppslag/tilgangsstyrer/).

Merk at det er mulig for konsument å utføre en delegering i Altinn, selv om  API-tilbyder ennå ikke har gitt konsument tilgang til scopet i Maskinporten.


### Hente tokens

Når overstående punkter er utført, kan leverandøren [forespørre token fra Maskinporten](https://difi.github.io/felleslosninger/maskinporten_protocol_token.html).

Leverandøren **må** inkludere konsumentens organisasjonsnummer i `consumer_org`-claimet i JWT-grantet.



## Miljøer og testdata

Delegering kan kun testes i VER2-miljøet, dette er koblet mot Altinns TT02-miljø.

Endepunkt for støttede delegeringsoppsett:

|Miljø| Endepunkt|
|-|-|
| VER2 | https://integrasjon-ver2.difi.no/delegationsources |
| PROD | https://integrasjon.difi.no/delegationsources |


For å teste som konsument, må du be Altinn om å få en testbruker tilknyttet en  organisasjon i TT02-miljøet.

For API-tilbydere som ønsker å teste delegering, er det enklest å bruke eget orgno som leverandør siden leverandøren (=oauth2-klienten) trenger test-virksomhetssertifikat.
