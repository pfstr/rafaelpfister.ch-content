---
title: "VPN-splittunnellering for Microsoft Teams: led medietrafikk utenom tunnelen"
navTitle: "Teams-splittunnellering"
description: "Teams-samtaler over VPN lider under forsinkelse, jitter og omveien via VPN-gatewayen. Artikkelen viser hvilke Microsoft-nettverk og porter som brukes til medietrafikk, hvorfor IP-basert splittunnellering er bedre enn app-unntak, og hvordan dette implementeres i forbruker-VPN-er, WireGuard, OpenVPN og bedriftsklienter."
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
translationOf: vpn-split-tunneling-microsoft-teams
translationSourceHash: 9a96126164a12f1f411065507313733f451161a25c0cd051778cc306fd32ed4c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:27:19.007Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/vpn-split-tunneling-for-microsoft-teams-fore-medietrafikk-utenom-tunnelen
---

En Teams-samtale over en VPN-forbindelse høres ofte dårligere ut enn uten: Stemmen faller ut, video hakker, og skjermdeling bygges opp med forsinkelse. Årsaken ligger som regel i omveien sanntidstrafikken tar gjennom VPN-tunnelen, ikke i Teams selv. Microsoft har derfor i årevis anbefalt å føre Teams-medietrafikk direkte til internett utenom VPN-et ved hjelp av splittunnellering. Denne tilnærmingen fungerer med praktisk talt alle VPN-produkter, fra forbrukerklienter til bedriftsgatewayer; konfigurasjonen varierer bare i detaljene.

## Hvorfor sanntidstrafikk lider i tunnelen

Teams-lyd og -video kjører over SRTP, en UDP-basert protokoll som er avhengig av lav forsinkelse og lav jitter. Microsoft oppgir målverdier på under 100 ms tur-retur-tid til nærmeste Microsoft-nettverksinngang og under 30 ms jitter. En VPN-tunnel forverrer begge verdiene på flere måter.

For det første forlenger tunnelen ruten: I stedet for å gå direkte til nærmeste geografiske Microsoft-innstegspunkt, går trafikken først til VPN-gatewayen, som kan stå i leverandørens eller bedriftens datasenter, og først derfra til Microsoft. For det andre krever det ekstra krypteringslaget prosesseringstid og øker overhenget per pakke; mediestrømmen er allerede kryptert med SRTP, og VPN-krypteringen kommer i tillegg som et nytt lag. For det tredje er VPN-gatewayen en delt flaskehals: I perioder med høy belastning deler alle brukere båndbredden og pakkebufferne, noe som skaper nettopp den jitteren sanntidstrafikk er mest følsom for. For det fjerde blokkerer enkelte VPN-konfigurasjoner UDP helt eller tvinger frem TCP; Teams faller da tilbake til TCP 443, noe som ytterligere forringer kvaliteten fordi TCP-gjenoverføringer er uegnet for sanntidsmedier.

For resten av Teams-trafikken (pålogging, chat, filtilgang) spiller dette knapt noen rolle, fordi den ikke er sanntidssensitiv. Det er derfor nok å unnta medietrafikken målrettet.

## Relevante nettverk og porter

Microsoft publiserer alle Microsoft 365-endepunkter i maskinlesbart format og deler dem inn i kategoriene Optimize, Allow og Default. Kategorien Optimize er relevant for splittunnellering: Den omfatter de få forsinkelseskritiske endepunktene med faste IP-nettverk som samlet står for størstedelen av volumet. For Teams-medier er dette endepunkt-ID 11 i den offisielle listen. ID 12 omfatter de samme IPv4-nettverkene og flere IPv6-nettverk for TCP 443 og 80 samt UDP 443, men ligger i kategorien Allow. Nettverket `13.107.64.0/18` som tidligere også var oppført, fjernet Microsoft fra begge oppføringene i 2024 (versjon 2024053000 for ID 12, 2024073100 for ID 11):

| Nettverk | Protokoll og porter | Formål |
|---|---|---|
| `52.112.0.0/14` | UDP 3478 til 3481 (ID 11), TCP 443 (ID 12) | Teams-medier og transportreléer |
| `52.122.0.0/15` | UDP 3478 til 3481 (ID 11), TCP 443 (ID 12) | Teams-medier og transportreléer |
| `2603:1063::/38` | UDP 3478 til 3481 (ID 11), TCP 443 (ID 12) | de samme tjenestene over IPv6 |

De fire UDP-portene representerer medieklassene lyd (3478), video (3479 og 3480) og skjermdeling (3481); TCP 443 er reserveveien. De som bruker IPv6, bør også unnta IPv6-nettverket, ellers går en del av forbindelsene likevel gjennom tunnelen.

Disse nettverkene er bevisst stabile: Microsoft varsler endringer i Optimize-endepunktene gjennom Endpoint-webtjenesten og holder listen kort, nettopp for at bedrifter skal kunne legge dem inn i rute- og brannmurregler. Likevel bør regelmessig sammenligning med den offisielle listen inngå i driftsrutinene.

## App-basert eller IP-basert: to tilnærminger med ulike styrker

Mange VPN-klienter tilbyr to former for splittunnellering: unntak per applikasjon eller unntak per mål-IP.

App-unntak høres nærliggende ut, men har to svakheter med Teams. Nye Teams er en WebView2-applikasjon: Hovedprosessen heter `ms-teams.exe`, men en del av trafikken går via `msedgewebview2.exe`. Den som bare unntar hovedprosessen, får ikke med all trafikken; den som også unntar WebView2, leder også trafikken fra andre WebView2-applikasjoner (som nye Outlook) utenom tunnelen. For Teams i nettleseren hjelper app-unntaket ikke i det hele tatt, med mindre hele nettleseren unntas, slik at all nettrafikk omgår VPN-et.

Det IP-baserte unntaket virker derimot på nettverksnivå og er dermed uavhengig av om trafikken kommer fra Teams-appen, WebView2 eller en nettleserfane. Det unntar nøyaktig det som er forsinkelseskritisk, og lar pålogging, chat og resten av nettrafikken bli i tunnelen. For Teams er den IP-baserte tilnærmingen derfor det bedre valget; app-unntaket egner seg som et supplement dersom all Teams-trafikk faktisk skal omgå VPN-et.

## Implementering i vanlige VPN-produkter

Prinsippet er det samme overalt: De to IPv4-nettverkene (og IPv6-nettverket ved behov) unntas fra tunnelen, slik at operativsystemets ruter for disse målene peker mot det fysiske grensesnittet.

**Forbruker-VPN-er (Proton VPN, NordVPN, Surfshark og lignende):** Windows- og Android-klientene har som regel et menypunkt som «Split Tunneling» med en unntaksliste for IP-adresser eller subnett. Legg inn nettverkene i CIDR-notasjon der, og opprett VPN-forbindelsen på nytt for at rutene skal tre i kraft. På macOS og iOS mangler funksjonen hos de fleste leverandører, fordi system-API-ene der ikke tillater applikasjonsstyrt splittunnellering i denne formen.

**WireGuard:** WireGuard har ingen unntaksliste, bare `AllowedIPs`-innstillingen som angir hva som går inn i tunnelen. Unntak opprettes ved å erstatte `0.0.0.0/0` med listen over alle nettverk som ikke inneholder unntaksområdet. Ingen beregner denne komplementærlisten manuelt; nettbaserte kalkulatorer som WireGuard AllowedIPs Calculator bruker `0.0.0.0/0` som grunnlag, Microsoft-nettverkene som «Disallowed IPs» og leverer den ferdige linjen til konfigurasjonsfilen.

**OpenVPN:** Når `redirect-gateway` er aktivert, har mer spesifikke ruter prioritet. To ekstra linjer i klientkonfigurasjonen leder Microsoft-nettverkene utenom tunnelen:

```text
route 52.112.0.0 255.252.0.0 net_gateway
route 52.122.0.0 255.254.0.0 net_gateway
```

`net_gateway` står her for standardgatewayen i det lokale nettverket, ikke VPN-gatewayen.

**Bedriftsklienter (Cisco Secure Client/AnyConnect, Palo Alto GlobalProtect, Fortinet FortiClient):** Her konfigurerer bedriften unntakene sentralt, hos Cisco som en «Split Exclude»-liste i gruppepolicyen og hos GlobalProtect som «Exclude Access Route». Microsoft dokumenterer uttrykkelig denne fremgangsmåten som anbefalt modell for Microsoft 365-trafikk og leverer Optimize-listen gjennom Endpoint-webtjenesten, slik at unntakene kan holdes oppdatert automatisk. Ansatte bak et bedrifts-VPN kan altså ikke angi unntaket selv, men må be nettverksteamet om det; Microsoft-dokumentet om dette gir et godt grunnlag for begrunnelsen.

**Innebygde Windows-verktøy:** I en VPN-forbindelse satt opp med Windows' innebygde verktøy i splittmodus (`Set-VpnConnection -SplitTunneling $true`) havner bare nettverkene som er lagt inn med `Add-VpnConnectionRoute` i tunnelen. Så lenge Microsoft-nettverkene ikke er oppført der, går de automatisk direkte; et eksplisitt unntak er da unødvendig.

## Sikkerhetsvurdering: hva som går utenom tunnelen

Splittunnellering er en bevisst oppmyking av prinsippet om å føre all trafikk gjennom tunnelen. Før implementering bør du avklare tre punkter.

Din egen offentlige IP-adresse blir synlig for Microsoft, for det er nettopp hensikten: Mediestrømmen skal ta den korteste veien. De som primært bruker VPN for å skjule sin egen plassering, mister denne beskyttelsen for Teams-samtaler. Innholdet påvirkes ikke av dette, fordi SRTP krypterer mediestrømmen ende-til-ende mellom klienten og Microsoft-infrastrukturen.

I bedriftsmiljøer mister den sentrale sikkerhetsgatewayen innsyn i den unntatte trafikken: TLS-inspeksjon, IDS-signaturer og volumanalyse gjelder ikke lenger for disse nettverkene. Fordi unntaket er begrenset til få, fast Microsoft-tilordnede nettverk med definerte porter, vurderer Microsoft denne gjenværende risikoen som lav; Optimize-endepunktene er kuratert nettopp for dette formålet. Et generelt unntak for hele applikasjoner eller til og med nettleseren har derimot en betydelig større angrepsflate og bør unngås i bedriftsmiljøer.

Til slutt Kill Switch: Enkelte VPN-klienter bruker først splittunnelleringsunntak etter at forbindelsen er opprettet på nytt, eller oppfører seg annerledes når Kill Switch er aktiv. Etter hver endring i unntakslisten bør forbindelsen derfor opprettes på nytt og testes.

## Kontroll: går medietrafikken virkelig direkte?

Om unntaket virker, kan kontrolleres på to nivåer. På rutenivå viser PowerShell hvilket grensesnitt Windows velger for et mål i Microsoft-nettverkene:

```powershell
Find-NetRoute -RemoteIPAddress 52.112.1.1 |
  Select-Object InterfaceAlias, NextHop
```

Hvis det fysiske grensesnittet (Ethernet eller WLAN) vises her i stedet for VPN-adapteren, er ruten riktig. På applikasjonsnivå gir Teams selv bekreftelsen: Under en samtale viser samtalekvaliteten (under «Flere handlinger» i samtalevinduet) den forhandlede forbindelsestypen, tur-retur-tiden og pakketapsraten. En tur-retur-tid som synker betydelig etter endringen, og forbindelsestypen UDP i stedet for TCP, er de to kjennetegnene på et fungerende unntak.

Hvis trafikken fortsatt går i tunnelen til tross for korrekt rute, er det verdt å se på rekkefølgen til nettverksadapterne og klientspesifikke forhold: Enkelte VPN-klienter tvinger rutene sine med lavere metrikk på nytt etter hver opprettelse av forbindelse, og en utdatert unntaksliste blir først synlig når Microsoft legger til et nettverk. Sammenligning med den offisielle endepunktlisten bør derfor inngå i samme rytme som andre regelmessige nettverkskontroller.

## Endringslogg

- **25. oktober 2026:** Korrigert: Microsoft fjernet nettverket 13.107.64.0/18 fra endepunktlisten i 2024; det er tatt ut av tabellen og OpenVPN-eksemplet. Også korrigert: Bare ID 11 tilhører kategorien Optimize, mens ID 12 tilhører kategorien Allow.

## Kilder

1.  [Microsoft: Office 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges): offisiell endepunktliste; Teams-medienettverkene står under ID 11 i kategorien Optimize, og ID 12 med de samme nettverkene i kategorien Allow.

2.  [Microsoft: Implementing VPN split tunneling for Microsoft 365](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-vpn-implement-split-tunnel): Microsofts implementeringsveiledning for bedrifts-VPN-er, inkludert begrunnelsen for risikovurderingen og et eksempel med Teams-medienettverkene `52.112.0.0/14`, `52.122.0.0/15` og `2603:1063::/38`.

3.  [Microsoft: Microsoft 365 network connectivity principles](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-network-connectivity-principles): prinsippene bak lokalt internettuttak, inkludert målene for forsinkelse i sanntidsmedier.

4.  [Proton VPN: How to use split tunneling](https://protonvpn.com/support/protonvpn-split-tunneling/): eksempel på en forbrukerklient med IP- og app-basert splittunnellering i Windows og Android.

5.  [WireGuard AllowedIPs Calculator](https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/): kalkulator for komplementærlisten når unntak må representeres gjennom AllowedIPs.

6.  [Microsoft 365 IP and URL web service: Änderungen Worldwide](https://endpoints.office.com/changes/worldwide/2024010100?clientrequestid=b10c5ed1-bad1-445f-b386-b919946339a7): endringslogg for endepunktlisten, blant annet fjerningen av `13.107.64.0/18` fra ID-ene 12 (versjon 2024053000) og 11 (versjon 2024073100).
