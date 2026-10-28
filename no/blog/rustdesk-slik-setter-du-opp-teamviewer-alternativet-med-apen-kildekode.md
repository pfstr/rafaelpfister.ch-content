---
title: "RustDesk: konfigurere det åpne TeamViewer-alternativet"
navTitle: "Konfigurere RustDesk"
description: "RustDesk er programvare for fjernsupport med åpen kildekode under AGPL, gratis og mulig å selvhoste. Slik installerer du klienten på Windows (også uovervåket via MSI), hvordan tilkoblingen opprettes via den offentlige formidlingsserveren, en egen server eller en direkte tilkobling, hvilke funksjoner hverdagen med support krever, og hvor grensene for gratis bruk går."
date: "2026-09-01"
kategorie: "Fjernsupport og brukerstøtte"
timeToRead: "9 min lesetid"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "haertung"
slug: "rustdesk-slik-setter-du-opp-teamviewer-alternativet-med-apen-kildekode"
translationId: "article-425ae4b8d562ae41"
translationOf: rustdesk-teamviewer-alternative
translationSourceHash: 3c44a4d5fc03c777c33f77525bcaebe75596ddde249f697041599b54e77cae2b
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T10:00:31.374Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/rustdesk-slik-setter-du-opp-teamviewer-alternativet-med-apen-kildekode
---

# RustDesk: konfigurere det åpne TeamViewer-alternativet

TeamViewer og AnyDesk dekker fjernsupport pålitelig, men krever lisens for kommersiell bruk, og prisene øker med antallet administrerte enheter. RustDesk er et alternativ under AGPL-3.0-lisensen: åpen kildekode, gratis og uten lisenskrav. Klienten kjører på Windows, macOS, Linux, Android og iOS, samt i nettleseren. Den er skrevet i Rust, med grensesnitt i Flutter.

Den avgjørende forskjellen fra kommersielle produkter ligger i formidlingen: RustDesk skiller klienten fra serverinfrastrukturen. Du kan bruke den gratis offentlige formidlingsserveren, drifte din egen server eller opprette en direkte tilkobling helt uten formidlingsserver. Dermed kan RustDesk brukes fra enkeltmaskiner til en selvhostet supportplattform, uten at tilkoblingsdata må gå via en leverandør.

## De tre tilkoblingsmåtene

Før du installerer, bør du velge tilkoblingsmåte, ettersom konfigurasjonen og åpne porter avhenger av dette.

| Tilkoblingsmåte | Slik fungerer den | Når den er hensiktsmessig |
|---|---|---|
| Offentlig formidlingsserver | To klienter finner hverandre via ID-en (nisifret nummer) på RustDesk-serveren, og tilkoblingen går direkte eller via en relay | Rask oppstart, testing, privat sporadisk support |
| Egen server (self-hosted) | Du drifter serverkomponentene `hbbs` (formidling) og `hbbr` (relay) selv, og alle klienter angir adressen deres | Kommersiell bruk, mange enheter, full kontroll over data |
| Direkte tilkobling (Direct IP Access) | Klienten kobler direkte til IP-adressen til motparten uten formidlingsserver | Begge enhetene kan nå hverandre i samme nettverk eller via VPN |

Den offentlige serveren er uttrykkelig ment for testing og privat bruk. For produktiv, kommersiell drift anbefaler prosjektet en egen server, også fordi den offentlige tjenesten er begrenset og ikke har noen tilgjengelighetsgaranti.

## Installasjon på Windows

Du laster ned installasjonsprogrammet fra den offisielle kilden, prosjektets GitHub-utgivelser (`github.com/rustdesk/rustdesk`). For Windows finnes det en kjørbar fil og en MSI-pakke. For interaktiv installasjon holder det å dobbeltklikke. Hvis du vil distribuere RustDesk på flere datamaskiner eller i bakgrunnen, bruker du MSI med en stille installasjon. Filnavnet inneholder versjonsnummeret; gjeldende versjon er 1.5.0 siden 30. september 2026:

```powershell
msiexec /i rustdesk-1.5.0-x86_64.msi /qn /norestart
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `/i <paket>` | Installerer den angitte MSI-pakken |
| `/qn` | Ingen grensesnitt, ingen dialogbokser (stille) |
| `/norestart` | Hindrer automatisk omstart etter installasjonen |

</details>

Den stille installasjonen oppretter tjenesten `RustDesk`, som kjører ved systemoppstart og muliggjør uovervåket tilgang. Etter installasjonen kan du lese enhetens ID via kommandolinjen uten å åpne grensesnittet:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --get-id
```

Du kan også angi et fast passord for uovervåket tilgang via kommandolinjen. Velg et separat og tilstrekkelig langt passord, ikke brukerens innloggingspassord:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --password "IhrLangesEinmalpasswort"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `--get-id` | Skriver ut enhetens nisifrede RustDesk-ID |
| `--password <wert>` | Angir det faste passordet for uovervåket tilgang |
| `--silent-install` | Installerer den kjørbare versjonen (`.exe`) uten grensesnitt som en tjeneste |

</details>

## Legge inn egen server

Hvis du drifter en egen formidlingsserver, angir klientene adressen og den offentlige nøkkelen. I grensesnittet finner du dette under nettverksinnstillingene som ID-server, relay-server og nøkkel. For massedistribusjon kan konfigurasjonen også angis som fil eller via miljøvariabler, slik at hver klient starter forhåndskonfigurert.

En egen server krever de to komponentene `hbbs` og `hbbr`, som vanligvis kjører som Docker-containere. Begge krever åpne porter slik at klienter kan registrere seg og bruke en relay.

| Port | Protokoll | Komponent og formål |
|---|---|---|
| 21114 | TCP | Nettgrensesnitt for Pro-versjonen (kun der) |
| 21115 | TCP | `hbbs`, testing av NAT-type |
| 21116 | TCP og UDP | `hbbs`, registrering (UDP) og tilkoblingsopprettelse (TCP) |
| 21117 | TCP | `hbbr`, relay-trafikk |
| 21118, 21119 | TCP | Støtte for nettklienter |

Åpne bare portene som tilkoblingsmåten din faktisk trenger, og begrens tilgangen via brannmuren til nettverkene supporten utføres fra.

## Direkte tilkobling uten formidlingsserver

Hvis begge enhetene kan nå hverandre i samme nettverk eller via VPN, fungerer RustDesk helt uten formidlingsserver. Aktiver da direktetilgang på målenheten (i grensesnittet under sikkerhet som «Aktiver direkte IP-tilgang», internt bryteren `direct-server`). Klienten lytter deretter på standardporten 21118 (TCP). I tilkoblingsvinduet skriver du inn motpartens IP-adresse i stedet for ID-en.

Begrens direktetilgang via brannmuren til nettverket du kobler til fra. Hvis tilgangen går via VPN, åpner du porten bare for VPN-adresseområdet, ikke for hele internett.

## Funksjoner i daglig support

RustDesk dekker funksjonene som trengs for fjernsupport i hverdagen:

- Skjermdeling og fjernstyring av tastatur og mus, med valg av skjerm ved flere skjermer.
- Filoverføring i begge retninger via et delt vindu.
- Tekstchat under økten.
- Uovervåket tilgang med fast passord, for enheter uten en tilstedeværende bruker.
- Øktopptak som videofil, om ønskelig automatisk.
- TCP-tunneler og videresending for å nå enkelte tjenester hos motparten lokalt.
- Adressebok og flere lagrede enheter, lokalt i gratisversjonen og delt på serversiden i Pro-versjonen.

For betjent support er dette viktig: Som standard spør RustDesk på motpartens side om tilkoblingen skal godtas, og viser under økten at tilgang pågår. Personen ved enheten er altså informert. Først et fast passord for uovervåket tilgang fjerner forespørselen. Bruk bare uovervåket tilgang på enheter der brukerne vet at programvaren er installert og hva den brukes til.

## Begrensninger og grenser

RustDesk erstatter TeamViewer i mange tilfeller, men har begrensninger du bør kjenne til før bruk:

- Den offentlige formidlingsserveren er begrenset, uten tilgjengelighetsgaranti og ikke beregnet for kontinuerlig kommersiell drift. De som vil arbeide pålitelig, hoster selv.
- En egen server medfører driftsarbeid: Containere, åpne porter, sertifikater og oppdateringer er ditt ansvar.
- En delt adressebok på serversiden, sentral brukeradministrasjon og nettgrensesnittet for administrasjon tilhører Pro-versjonen, som er betalingspliktig fra et bestemt antall enheter. Klienten selv og grunnleggende drift forblir gratis.
- Uten fast passord er uovervåket tilgang ikke mulig. Det er riktig for betjent support, men forhindrer spontan tilgang til en ubemannet enhet.
- Funksjonsomfanget og stabiliteten på enkelte plattformer, særlig mobile enheter, når ikke de kommersielle produktene på alle punkter. Kontroller funksjonene som er viktige for deg før du bytter.
- Noen sikkerhetsprogrammer varsler om programvare for fjernsupport som potensielt uønsket. Legg ved behov inn et unntak og dokumenter hvorfor programvaren er installert.

For privat bruk og support av enkeltstående enheter holder gratisversjonen med den offentlige serveren eller en direkte tilkobling. Så snart du administrerer mange enheter, arbeider kommersielt eller trenger full kontroll over data, trenger du en egen server, med tilsvarende driftsarbeid som motytelse for uavhengigheten.

## Endringslogg

- **25. oktober 2026:** Installasjonseksempelet oppdatert til RustDesk 1.5.0 fra 30. september 2026.

## Kilder

1.  [RustDesk på GitHub](https://github.com/rustdesk/rustdesk): Kildekode, utgivelser med installasjonsprogrammene og AGPL-3.0-lisensen.

2.  [RustDesk-dokumentasjon](https://rustdesk.com/docs/): Installasjon, egen server, porter og konfigurasjon av klientene.

3.  [rustdesk-server på GitHub](https://github.com/rustdesk/rustdesk-server): Serverkomponentene `hbbs` og `hbbr`, inkludert portoversikten for egen drift.
