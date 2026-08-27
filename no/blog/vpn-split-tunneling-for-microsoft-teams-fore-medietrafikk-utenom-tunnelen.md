---
title: "VPN-split tunneling for Microsoft Teams: føre medietrafikk utenom tunnelen"
navTitle: "Teams Split Tunneling"
description: "Teams-samtaler over VPN lider under forsinkelse, jitter og omveien via VPN-gatewayen. Artikkelen viser hvilke Microsoft-nettverk og porter som brukes til medietrafikk, hvorfor IP-basert split tunneling er bedre enn app-unntak, og hvordan det implementeres i forbruker-VPN-er, WireGuard, OpenVPN og enterprise-klienter."
date: "2026-08-26"
kategorie: "Microsoft Teams"
timeToRead: "8 min lesetid"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
protokolle:
  - "tcp"
hauptthema: "microsoft-teams"
slug: "vpn-split-tunneling-for-microsoft-teams-fore-medietrafikk-utenom-tunnelen"
translationId: "article-d15f1e7ff6af231c"
aiPrompt: |
  Du bist mein Netzwerk-Assistent. Ich will Microsoft-Teams-Medienverkehr per Split Tunneling an meinem VPN vorbeiführen. Hilf mir Schritt für Schritt: 1. Frage mich, welchen VPN-Client ich einsetze (Consumer-VPN, WireGuard, OpenVPN, Enterprise-Client) und auf welchem Betriebssystem. 2. Nenne mir die passende Konfiguration für die drei Optimize-Netze 13.107.64.0/18, 52.112.0.0/14 und 52.122.0.0/15 (UDP 3478 bis 3481, TCP 443). 3. Erkläre mir, wie ich mit Find-NetRoute oder der Anrufintegrität in Teams prüfe, ob der Medienverkehr tatsächlich am Tunnel vorbeiläuft. 4. Weise mich auf die Sicherheitsabwägungen hin, bevor ich die Ausnahme produktiv setze.
translationOf: vpn-split-tunneling-microsoft-teams
url: https://rafaelpfister.ch/no/blog/vpn-split-tunneling-for-microsoft-teams-fore-medietrafikk-utenom-tunnelen
translationSourceHash: 95e3cefa4946676022602866d6ef21ab92ef25ec8c5dd3ff4ab0219ba718a880
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:33:40.420Z
translationReview: automatic
---

En Teams-samtale over en VPN-forbindelse høres ofte dårligere ut enn uten: Stemmen faller ut, video hakker, og skjermdeling bygges opp med forsinkelse. Årsaken er som regel omveien sanntidstrafikken tar gjennom VPN-tunnelen, ikke Teams selv. Microsoft har derfor i mange år anbefalt å føre Teams-medietrafikk direkte til internett utenom VPN-et ved hjelp av split tunneling. Denne tilnærmingen fungerer med praktisk talt alle VPN-produkter, fra forbrukerklienter til enterprise-gatewayer; konfigurasjonen varierer bare i detaljene.

## Hvorfor sanntidstrafikk lider i tunnelen

Teams-lyd og -video bruker SRTP, en UDP-basert protokoll som er avhengig av lav forsinkelse og lite jitter. Microsoft oppgir som målverdier under 100 ms rundturstid til nærmeste Microsoft-nettverksinngang og under 30 ms jitter. En VPN-tunnel forverrer begge verdiene på flere måter.

For det første forlenger tunnelen veien: I stedet for å gå direkte til det geografisk nærmeste Microsoft-tilgangspunktet, går trafikken først til VPN-gatewayen, som kan være plassert i leverandørens eller bedriftens datasenter, og derfra videre til Microsoft. For det andre krever det ekstra krypteringslaget prosesseringstid og øker overheaden per pakke; mediestrømmen er allerede kryptert med SRTP, og VPN-krypteringen kommer i tillegg som et andre lag. For det tredje er VPN-gatewayen en delt flaskehals: I rushtider deler alle brukere båndbredden og pakkebufferne, noe som skaper nettopp det jitteret sanntidstrafikk er mest følsom for. For det fjerde blokkerer enkelte VPN-konfigurasjoner UDP helt eller tvinger frem TCP; Teams faller da tilbake til TCP 443, noe som forringer kvaliteten ytterligere fordi TCP-gjenoverføringer er uegnet for sanntidsmedier.

For den øvrige Teams-trafikken (pålogging, chat, filtilgang) spiller dette knapt noen rolle, fordi den ikke er sanntidssensitiv. Det er derfor tilstrekkelig å unnta medietrafikken målrettet.

## De relevante nettverkene og portene

Microsoft publiserer alle Microsoft 365-endepunkter i maskinlesbar form og deler dem inn i kategoriene Optimize, Allow og Default. Kategorien Optimize er relevant for split tunneling: Den omfatter de få forsinkelseskritiske endepunktene med faste IP-nettverk, som til sammen utgjør størstedelen av volumet. For Teams-medier er dette endepunkt-ID-ene 11 og 12 i den offisielle listen:

| Nettverk | Protokoll og porter | Formål |
|---|---|---|
| `13.107.64.0/18` | UDP 3478 til 3481, TCP 443 | Teams-medier (lyd, video, skjermdeling) |
| `52.112.0.0/14` | UDP 3478 til 3481, TCP 443 | Teams-medier og transportreléer |
| `52.122.0.0/15` | UDP 3478 til 3481, TCP 443 | Teams-medier og transportreléer |
| `2603:1063::/38` | UDP 3478 til 3481, TCP 443 | de samme tjenestene via IPv6 |

De fire UDP-portene står for medieklassene lyd (3478), video (3479 og 3480) og skjermdeling (3481); TCP 443 er reserveveien. De som bruker IPv6, bør også unnta IPv6-nettverket, ellers vil en del av forbindelsene likevel gå gjennom tunnelen.

Disse nettverkene er bevisst stabile: Microsoft varsler endringer i Optimize-endepunktene via Endpoint-webtjenesten og holder listen kort, nettopp slik at bedrifter kan legge dem inn i ruting- og brannmurregler. Likevel bør det inngå i driftsrutinene å sammenligne med den offisielle listen med jevne mellomrom.

## App-basert eller IP-basert: to tilnærminger med ulike styrker

Mange VPN-klienter tilbyr to typer split tunneling: unntak per applikasjon eller unntak per mål-IP.

App-unntaket virker nærliggende, men har to svakheter for Teams. Nye Teams er en WebView2-applikasjon: Hovedprosessen heter `ms-teams.exe`, men en del av trafikken går via `msedgewebview2.exe`. Den som bare unntar hovedprosessen, får ikke med all trafikken; den som også unntar WebView2, fører også trafikk fra andre WebView2-applikasjoner (for eksempel nye Outlook) utenom tunnelen. Og for Teams i nettleseren hjelper ikke app-unntaket i det hele tatt, med mindre hele nettleseren unntas, slik at all nettrafikk omgår VPN-et.

Det IP-baserte unntaket virker derimot på nettverksnivå og er dermed uavhengig av om trafikken kommer fra Teams-appen, WebView2 eller en nettleserfane. Det unntar nøyaktig det som er forsinkelseskritisk, og lar pålogging, chat og resten av nettrafikken bli i tunnelen. For Teams er den IP-baserte tilnærmingen derfor det beste valget; app-unntaket egner seg som et supplement når virkelig all Teams-trafikk skal omgå VPN-et.

## Implementering i vanlige VPN-produkter

Prinsippet er det samme overalt: De tre IPv4-nettverkene (og ved behov IPv6-nettverket) unntas fra tunnelen, slik at operativsystemets ruter for disse målene peker mot det fysiske grensesnittet.

**Forbruker-VPN-er (Proton VPN, NordVPN, Surfshark og lignende):** Windows- og Android-klientene har som regel et menypunkt som «Split Tunneling» med en unntaksliste for IP-adresser eller delnett. Legg inn de tre nettverkene i CIDR-notasjon der, og opprett VPN-forbindelsen på nytt slik at rutene tas i bruk. På macOS og iOS mangler funksjonen hos de fleste leverandører, fordi system-API-ene der ikke tillater applikasjonsstyrt split tunneling i denne formen.

**WireGuard:** WireGuard har ingen unntaksliste, men bare `AllowedIPs`-angivelsen, som fastsetter hva som går inn i tunnelen. Unntak oppstår ved å erstatte `0.0.0.0/0` med listen over alle nettverk som ikke inneholder unntaksområdet. Ingen beregner denne komplementærlisten for hånd; nettbaserte kalkulatorer som WireGuard AllowedIPs Calculator bruker `0.0.0.0/0` som grunnlag, de tre Microsoft-nettverkene som «Disallowed IPs» og gir den ferdige linjen for konfigurasjonsfilen.

**OpenVPN:** Når `redirect-gateway` er aktivt, vinner mer spesifikke ruter. Tre ekstra linjer i klientkonfigurasjonen fører Microsoft-nettverkene utenom tunnelen:

```text
route 13.107.64.0 255.255.192.0 net_gateway
route 52.112.0.0 255.252.0.0 net_gateway
route 52.122.0.0 255.254.0.0 net_gateway
```

`net_gateway` står her for standardgatewayen i det lokale nettverket, ikke VPN-gatewayen.

**Enterprise-klienter (Cisco Secure Client/AnyConnect, Palo Alto GlobalProtect, Fortinet FortiClient):** Her konfigurerer bedriften unntakene sentralt, hos Cisco som en «Split Exclude»-liste i gruppepolicyen og hos GlobalProtect som «Exclude Access Route». Microsoft dokumenterer uttrykkelig denne fremgangsmåten som den anbefalte modellen for Microsoft 365-trafikk og leverer Optimize-listen via Endpoint-webtjenesten, slik at unntakene kan holdes oppdatert automatisk. Ansatte bak en bedrifts-VPN kan altså ikke sette unntaket selv, men må be nettverksteamet om det; Microsoft-dokumentet om dette er et passende grunnlag for argumentasjonen.

**Windows innebygde verktøy:** I en VPN-forbindelse satt opp med Windows innebygde verktøy i split-modus (`Set-VpnConnection -SplitTunneling $true`) havner bare nettverkene som er lagt inn med `Add-VpnConnectionRoute` i tunnelen. Så lenge Microsoft-nettverkene ikke finnes der, går de automatisk direkte; et eksplisitt unntak er da unødvendig.

## Sikkerhetsvurdering: hva som går utenom tunnelen

Split tunneling er en bevisst oppmyking av prinsippet om å føre all trafikk gjennom tunnelen. Før implementering bør du avklare tre punkter.

Din egen offentlige IP-adresse blir synlig for Microsoft, for det er nettopp hensikten: Mediestrømmen skal ta den korteste veien. De som primært bruker VPN for å skjule egen plassering, gir opp denne beskyttelsen for Teams-samtaler. Innholdet påvirkes ikke av dette, fordi SRTP krypterer mediestrømmen ende-til-ende mellom klienten og Microsoft-infrastrukturen.

I bedriftsmiljøer mister den sentrale sikkerhetsgatewayen innsyn i den unntatte trafikken: TLS-inspeksjon, IDS-signaturer og volumanalyse gjelder ikke lenger for disse nettverkene. Siden unntaket er begrenset til noen få, fast Microsoft-tilordnede nettverk med definerte porter, vurderer Microsoft denne restrisikoen som lav; Optimize-endepunktene er kuratert nettopp for dette. Et generelt unntak for hele applikasjoner eller til og med nettleseren har derimot en betydelig større angrepsflate og bør unngås i bedriftsmiljøer.

Til slutt, Kill Switch: Enkelte VPN-klienter bruker split-tunneling-unntak først etter at forbindelsen er opprettet på nytt, eller oppfører seg annerledes når Kill Switch er aktiv. Etter hver endring i unntakslisten bør du derfor gjenopprette forbindelsen og utføre en kontrolltest.

## Kontroll: går medietrafikken virkelig direkte?

Om unntaket fungerer, kan kontrolleres på to nivåer. På rutingnivå viser PowerShell hvilket grensesnitt Windows velger for et mål i Microsoft-nettverkene:

```powershell
Find-NetRoute -RemoteIPAddress 52.112.1.1 |
  Select-Object InterfaceAlias, NextHop
```

Hvis det fysiske grensesnittet (Ethernet eller WLAN) vises her i stedet for VPN-adapteren, er ruten riktig. På applikasjonsnivå gir Teams selv bekreftelsen: Under en samtale viser samtalekvaliteten (under «Flere handlinger» i samtalevinduet) den forhandlede tilkoblingstypen, rundturstiden og pakketapsraten. En rundturstid som synker betydelig etter omleggingen, og tilkoblingstypen UDP i stedet for TCP, er de to kjennetegnene på et fungerende unntak.

Hvis trafikken fortsatt går gjennom tunnelen til tross for korrekt rute, bør du se på rekkefølgen på nettverksadapterne og klientsæregenheter: Enkelte VPN-klienter tvinger gjennom rutene sine på nytt med lavere metrikk etter hver tilkobling, og en utdatert unntaksliste blir først synlig når Microsoft legger til et nettverk. Sammenligning med den offisielle endepunktlisten bør derfor inngå i samme rytme som andre gjentakende nettverkskontroller.

## Kilder

1.  [Microsoft: Office 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges): offisiell endepunktliste; Teams-medienettverkene finnes under ID-ene 11 og 12 i kategorien Optimize.

2.  [Microsoft: Implementing VPN split tunneling for Microsoft 365](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-vpn-implement-split-tunnel): Microsofts implementeringsveiledning for enterprise-VPN-er, inkludert begrunnelsen for risikovurderingen.

3.  [Microsoft: Microsoft 365 network connectivity principles](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-network-connectivity-principles): prinsippene bak lokal internettutgang, inkludert målverdiene for forsinkelse for sanntidsmedier.

4.  [Proton VPN: How to use split tunneling](https://protonvpn.com/support/protonvpn-split-tunneling/): eksempel på en forbrukerklient med IP- og app-basert split tunneling i Windows og Android.

5.  [WireGuard AllowedIPs Calculator](https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/): kalkulator for komplementærlisten når unntak må angis via AllowedIPs.
