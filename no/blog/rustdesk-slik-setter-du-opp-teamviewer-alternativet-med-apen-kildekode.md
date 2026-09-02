---
title: "RustDesk: Slik setter du opp TeamViewer-alternativet med åpen kildekode"
navTitle: "Sett opp RustDesk"
description: "RustDesk er programvare for fjernsupport med åpen kildekode under AGPL, gratis og mulig å drifte selv. Slik installerer du klienten på Windows (også uovervåket via MSI), hvordan forbindelsen opprettes via den offentlige formidlingsserveren, en egen server eller en direkteforbindelse, hvilke funksjoner som trengs i den daglige supporten, og hvor grensene for gratis bruk går."
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
aiPrompt: |
  Du bist mein IT-Support-Assistent. Hilf mir, RustDesk als quelloffene TeamViewer-Alternative einzurichten: Client installieren, Verbindungsart wählen (öffentlicher Vermittlungsserver, eigener Server oder Direktverbindung über ein privates Netz), unbeaufsichtigten Zugriff absichern und die Grenzen der kostenlosen Nutzung einordnen.
translationOf: rustdesk-teamviewer-alternative
url: https://rafaelpfister.ch/no/blog/rustdesk-slik-setter-du-opp-teamviewer-alternativet-med-apen-kildekode
translationSourceHash: f812fc4b04abe0aa92cca47b285a30a18f5cd1e99ab328593b224ee26051a7f3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:50:27.491Z
translationReview: automatic
---

# RustDesk: Slik setter du opp TeamViewer-alternativet med åpen kildekode

TeamViewer og AnyDesk dekker fjernsupport på en pålitelig måte, men krever lisens for kommersiell bruk, og prisene øker med antallet enheter som administreres. RustDesk er et alternativ under lisensen AGPL-3.0: med åpen kildekode, gratis og uten lisenskrav. Klienten kjører på Windows, macOS, Linux, Android og iOS samt i nettleseren. Den er skrevet i Rust, med grensesnitt i Flutter.

Den avgjørende forskjellen fra de kommersielle produktene ligger i formidlingen: RustDesk skiller klienten fra serverinfrastrukturen. Du kan bruke den gratis offentlige formidlingsserveren, drifte din egen server eller opprette en direkteforbindelse helt uten formidlingsserver. Dermed kan RustDesk brukes fra én enkelt arbeidsstasjon til en selvhostet supportplattform, uten at forbindelsesdata må gå via en leverandør.

## De tre tilkoblingstypene

Før du installerer, bør du fastsette tilkoblingstypen, siden konfigurasjonen og nødvendige åpne porter avhenger av den.

| Tilkoblingstype | Slik fungerer den | Når den er hensiktsmessig |
|---|---|---|
| Offentlig formidlingsserver | To klienter finner hverandre via ID-en (nisifret nummer) på RustDesk-serveren, forbindelsen går direkte eller via et relé | Rask oppstart, test, privat sporadisk support |
| Egen server (self-hosted) | Du drifter serverkomponentene `hbbs` (formidling) og `hbbr` (relé) selv, og alle klienter angir adressen deres | Kommersiell bruk, mange enheter, full kontroll over data |
| Direkteforbindelse (Direct IP Access) | Klienten kobler seg direkte til IP-adressen til motparten uten formidlingsserver | Begge enhetene er tilgjengelige i samme nettverk eller via VPN |

Den offentlige serveren er uttrykkelig ment for tester og privat bruk. For produktiv, kommersiell drift anbefaler prosjektet en egen server, blant annet fordi den offentlige tjenesten er strupet og ikke har noen tilgjengelighetsgaranti.

## Installasjon på Windows

Du laster ned installasjonsprogrammet fra den offisielle kilden, prosjektets GitHub-utgivelser (`github.com/rustdesk/rustdesk`). For Windows finnes det en kjørbar fil og en MSI-pakke. For interaktiv installasjon holder det å dobbeltklikke. Hvis du vil distribuere RustDesk på flere datamaskiner eller i bakgrunnen, bruker du MSI-filen med en stille installasjon:

```powershell
msiexec /i rustdesk-1.4.9-x86_64.msi /qn /norestart
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `/i <paket>` | Installerer den angitte MSI-pakken |
| `/qn` | Ingen brukerflate, ingen dialogbokser (stille) |
| `/norestart` | Forhindrer automatisk omstart etter installasjonen |

</details>

Den stille installasjonen oppretter tjenesten `RustDesk`, som kjører ved systemoppstart og muliggjør uovervåket tilgang. Etter installasjonen kan du hente enhetens ID via kommandolinjen uten å åpne brukerflaten:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --get-id
```

Du kan også angi et fast passord for uovervåket tilgang via kommandolinjen. Velg et eget, tilstrekkelig langt passord, ikke brukerens påloggingspassord:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --password "IhrLangesEinmalpasswort"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `--get-id` | Viser enhetens nisifrede RustDesk-ID |
| `--password <wert>` | Angir det faste passordet for uovervåket tilgang |
| `--silent-install` | Installerer den kjørbare utgaven (`.exe`) uten brukerflate som tjeneste |

</details>

## Legg inn egen server

Hvis du drifter din egen formidlingsserver, angir klientene adressen og den offentlige nøkkelen. I brukerflaten finner du dette under nettverksinnstillingene som ID-server, reléserver og nøkkel. For massedistribusjon kan konfigurasjonen også angis som fil eller via miljøvariabler, slik at hver klient starter forhåndskonfigurert.

En egen server trenger de to komponentene `hbbs` og `hbbr`, som vanligvis kjører som Docker-containere. Begge krever åpne porter for at klienter skal kunne registrere seg og bruke et relé.

| Port | Protokoll | Komponent og formål |
|---|---|---|
| 21114 | TCP | Nettgrensesnitt for Pro-utgaven (kun der) |
| 21115 | TCP | `hbbs`, testing av NAT-type |
| 21116 | TCP og UDP | `hbbs`, registrering (UDP) og tilkoblingsopprettelse (TCP) |
| 21117 | TCP | `hbbr`, relétrafikk |
| 21118, 21119 | TCP | Støtte for nettklienter |

Åpne bare portene som tilkoblingstypen din faktisk trenger, og begrens tilgangen i brannmuren til nettverkene som supporten ytes fra.

## Direkteforbindelse uten formidlingsserver

Hvis begge enhetene er tilgjengelige i samme nettverk eller via VPN, fungerer RustDesk helt uten formidlingsserver. Aktiver direkte tilgang på målenheten (i brukerflaten under sikkerhet som «Aktiver direkte IP-tilgang», internt bryteren `direct-server`). Klienten lytter da på standardporten 21118 (TCP). I tilkoblingsvinduet angir du IP-adressen til motparten i stedet for ID-en.

Begrens direkte tilgang i brannmuren til nettverket du kobler til fra. Hvis tilgangen går via VPN, åpner du porten bare for VPN-adresseområdet, ikke for hele Internett.

## Funksjoner i den daglige supporten

RustDesk dekker funksjonene som trengs for fjernsupport i hverdagen:

- Skjermdeling og fjernstyring av tastatur og mus, med valg av skjerm ved flere skjermer.
- Filoverføring i begge retninger via et todelt vindu.
- Tekstchat under økten.
- Uovervåket tilgang med fast passord, for enheter uten en tilstedeværende bruker.
- Øktopptak som videofil, eventuelt automatisk.
- TCP-tunnel og videresending for å nå enkeltjenester hos motparten lokalt.
- Adressebok og flere lagrede enheter, lokalt i gratisutgaven og delt på serversiden i Pro-utgaven.

For veiledet support er dette viktig: Som standard spør RustDesk på den andre siden om forbindelsen skal godtas, og viser under økten at det er aktiv tilgang. Personen ved enheten vet dermed om det. Først et fast passord for uovervåket tilgang fjerner forespørselen. Bruk bare uovervåket tilgang på enheter der brukerne vet at programvaren er installert og hva den brukes til.

## Begrensninger og grenser

RustDesk erstatter TeamViewer i mange tilfeller, men har begrensninger du bør kjenne til før bruk:

- Den offentlige formidlingsserveren er strupet, uten tilgjengelighetsgaranti og ikke ment for kontinuerlig kommersiell drift. Den som vil arbeide pålitelig, hoster selv.
- En egen server innebærer driftsarbeid: Containere, åpne porter, sertifikater og oppdateringer er ditt ansvar.
- En delt adressebok på serversiden, sentral brukeradministrasjon og nettgrensesnittet for administrasjon hører til Pro-utgaven, som er betalingspliktig fra et bestemt antall enheter. Klienten selv og grunnleggende drift forblir gratis.
- Uten fast passord er uovervåket tilgang ikke mulig, noe som er riktig for veiledet support, men som hindrer spontan tilgang til en ubemannet enhet.
- Funksjonsomfanget og stabiliteten på enkelte plattformer, særlig på mobile enheter, når ikke opp til de kommersielle produktene i alle detaljer. Kontroller funksjonene som er viktige for deg før du bytter.
- Enkelte sikkerhetsprogrammer varsler om programvare for fjernsupport som potensielt uønsket. Legg ved behov inn et unntak og dokumenter hvorfor programvaren er installert.

For privat bruk og support av enkeltstående enheter er gratisutgaven med den offentlige serveren eller en direkteforbindelse tilstrekkelig. Så snart du administrerer mange enheter, arbeider kommersielt eller trenger full kontroll over data, trenger du din egen server, med tilsvarende driftsarbeid som motytelse for uavhengigheten.

## Kilder

1.  [RustDesk på GitHub](https://github.com/rustdesk/rustdesk): Kildekode, utgivelser med installasjonsprogrammene og lisensen AGPL-3.0.

2.  [RustDesk-dokumentasjon](https://rustdesk.com/docs/): Installasjon, egen server, porter og konfigurasjon av klientene.

3.  [rustdesk-server på GitHub](https://github.com/rustdesk/rustdesk-server): Serverkomponentene `hbbs` og `hbbr` inkludert portoversikten for egen drift.
