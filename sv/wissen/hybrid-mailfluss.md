---
title: "Hybridmailflöde: routning mellan Exchange Online och lokalt"
blatt: "hybrid-mailfluss"
description: "Hybridmailflöde steg för steg: direkt och central leverans, connectors, TLS-certifikat, fjärrdomäner, gemensamma SMTP-domäner, e-postgatewayer, Message Trace och Message Tracking."
fakten:
  - label: Uppgift
    wert: SMTP-routning mellan Exchange Online och lokal Exchange-organisation
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Konfiguration
    wert: Hybrid Configuration Wizard skapar och underhåller transportkonfigurationen
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
  - label: Transport
    wert: SMTP över TCP 25 med TLS
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Kontroll av motpart
    wert: Certifikatnamn och connectorvillkor
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow
  - label: Cloudconnectors
    wert: Inbound- och Outbound Connectors
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail
  - label: Lokala connectors
    wert: Receive- och Send Connectors
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors
  - label: Mottagarroutning
    wert: Remote Mailbox, Target Address och samexistensdomän
    href: https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox
  - label: Standardroutning
    wert: Cloud- och lokal organisation kan vardera skicka internetpost direkt
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Centralized Mail Transport
    wert: Internetpost från Exchange Online går via den lokala organisationen
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Externa gatewayer
    wert: Ytterligare connector- och filtreringskedja före eller efter Microsoft 365
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud
  - label: Clouddiagnostik
    wert: Message Trace och connectorvalidering
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Lokal diagnostik
    wert: Message Tracking, köer och protokollloggar
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - smtp-mailflow
translationSourceHash: 442ec532c64e034403d3449ffe1b46d4b6de05a2337d3c3210da2d83cb5f618f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T11:04:37.269Z
translationReview: automatic
---

# Hybridmailflöde: routning mellan Exchange Online och lokalt

**Hybridmailflöde** är SMTP-vägen mellan en lokal Exchange-organisation och Exchange Online. Det gör att postlådor på båda sidor kan använda samma SMTP-domän och att meddelanden ändå hamnar på rätt plats. Hybrid Configuration Wizard konfigurerar connectors och TLS-parametrar för detta ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing), [Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

E-postflödet är bara en del av Exchange Hybrid. Katalogsynkronisering, ledig/upptagen, OAuth och flytt av postlådor använder andra vägar. Den här artikeln fokuserar därför medvetet på en enda fråga: **Vilka SMTP-hopp genomgår ett specifikt meddelande, och vilket beslut fattas vid varje hopp?**

**Protokollstacken** är överskådlig: DNS namnger de offentligt tillgängliga målen, SMTP över TCP 25 transporterar meddelandet, TLS skyddar och identifierar anslutningen, och Exchange-connectors anger vilken motpart som används för vilken domän. Mottagarobjekt tillhandahåller routningsadressen; Message Trace och lokala trackingloggar visar sedan vad varje organisation har gjort med meddelandet ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing), [Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

## Grundmodellen: två Exchange-organisationer, ett adressutrymme

En hybridmiljö har minst två transportorganisationer. Den lokala Exchange-organisationen känner till lokala postlådor och Remote Mailbox-objekt. Exchange Online känner till molnpostlådor och synkroniserade representationer av lokala mottagare. Båda sidor kan använda samma primära domän som `example.com` ([Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

För att ett meddelande inte ska hamna på fel plats behöver varje sida en uppgift om mottagarens faktiska plats. För en molnpostlåda innehåller det lokala Remote Mailbox-objektet en fjärrroutningsadress i samexistensdomänen, vanligtvis `tenant.mail.onmicrosoft.com`. Omvänt känner Exchange Online till synkroniserade lokala mottagare som e-postaktiverade objekt ([Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)).

Det normala flödet är därmed enkelt:

1. Den första Exchange-organisationen tar emot meddelandet.
2. Den löser upp mottagaren i sin katalog.
3. Mottagarobjektet visar om postlådan finns lokalt eller på den andra sidan.
4. Rätt hybridconnector skickar via SMTP/TLS till den andra organisationen.
5. Där löses mottagaren upp på nytt och meddelandet levereras.

Experter kontrollerar dessutom om transportregler ändrar vägen, om en gateway är infogad och vilken domän- eller connectorprioritet som förklarar valt nästa hopp.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-hybrid-mailfluss.svg?v=20260813" title="Interaktive Infografik: direkter und zentraler Hybrid-Mailfluss zwischen Internet, Exchange Online, Exchange On-Premises und Mail-Gateway" loading="lazy">
  <a href="/images/kb-interaktiv-hybrid-mailfluss.svg?v=20260813">Öppna interaktiv grafik för hybridmailflöde direkt</a>.
</iframe>

## Standardvägen utan central internettransport

I den vanliga decentraliserade modellen skickar varje sida sin egen internetpost. En lokal postlåda använder den lokala Exchange-transportorganisationen för utgående internetpost. En molnpostlåda skickar via Exchange Online Protection. Endast meddelanden mellan lokala postlådor och molnpostlådor passerar hybridconnectors ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)).

Inkommande internetpost följer den publicerade MX-posten. Om MX-posten pekar på Exchange Online tar EOP först emot meddelandet. För en molnpostlåda levererar Exchange Online lokalt; för en synkroniserad lokal mottagare använder det Hybrid-Outbound-Connector. Om MX-posten i stället pekar på den lokala miljön eller en föregående gateway sker det första mottagarbeslutet där.

Den här modellen håller internetvägarna korta, men ger flera möjliga egress-IP-adresser och filtreringsplatser. SPF, DKIM, DMARC, allowlisting och partnerregler måste ta hänsyn till båda utgående vägarna. Det är inte ett fel i hybridmodellen utan en följd av distribuerad leverans.

## Sätt Centralized Mail Transport i rätt sammanhang

**Centralized Mail Transport**, CMT, ändrar just denna utgående väg. Meddelanden från postlådor i Exchange Online till internet skickas först till den lokala Exchange-organisationen. Först där lämnar de organisationen. Den lokala sidan kan därmed fortsätta använda centrala transportregler, appliances eller fasta egress-IP-adresser ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)).

Fördelen är gemensam kontroll av utgående trafik. Priset är ytterligare hopp och beroenden. Om den lokala transporten eller dess internetanslutning fallerar påverkar det nu även utgående molnpost. Latens, köplats, egress-IP och platsen för den sista filtreringen förändras.

För avancerade administratörer är beslutet därför inte ”CMT på eller av”, utan: Vilken konkret policy kräver det lokala hoppet, vilken kapacitet måste det bära och hur routas trafiken vid ett avbrott? Experter dokumenterar dessutom loopschutz, TLS-krav, connectorprioritet och bevis på att varje avsett meddelande verkligen tar den centrala vägen.

Därmed är ämnet CMT avslutat. Autentisering av Outlook-klienter eller Hybrid Modern Authentication hör inte hemma här, eftersom den inte väljer något SMTP-hopp.

## Hur connectors identifierar motparten

När vägen har valts måste varje sida kunna lita på motparten. Hybrid Configuration Wizard skapar lokala Send-/Receive-Connectors och lämpliga Inbound-/Outbound-Connectors i Exchange Online. Transporten sker över SMTP på TCP 25 och använder TLS ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites), [Hybrid mail flow](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow)).

Den lokala Send Connector bestämmer mål och TLS-krav för samexistensdomänen. Cloud-Outbound-Connector beskriver den lokala organisationen som mål. I motsatt riktning accepterar Receive respektive Inbound Connector trafiken baserat på dokumenterade villkor, däribland certifikatidentitet och ursprung.

Certifikatet fyller en specifik uppgift: det identifierar SMTP-slutpunkten i TLS-handskakningen. Subject respektive Subject Alternative Name, connectorparametrar, presenterad certifikatkedja och faktiskt värdnamn måste stämma överens. Ett giltigt certifikat i certifikatarkivet räcker inte om transporttjänsten presenterar ett annat.

Experter granskar därför båda riktningarna separat. Riktning A kan fungera, medan riktning B misslyckas på grund av en annan connector, ett annat DNS-mål eller ett annat certifikatnamn.

## Mottagarroutning och delade domäner

En fungerande connector anger ännu inte vilka meddelanden som ska använda den. Det beslutet börjar med mottagarobjektet. En lokal Remote Mailbox hänvisar till molnet. Ett synkroniserat lokalt postlådeobjekt i Exchange Online hänvisar tillbaka till den lokala organisationen.

Accepted Domains avgör dessutom om en organisation ansvarar för alla mottagare i en domän eller får vidarebefordra okända mottagare. I delade domäner är en Internal-Relay-konfiguration endast säker om nästa hopp känner till eller avvisar okända mottagare korrekt ([Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains), [Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)).

Ett inaktuellt Remote Mailbox-objekt kan därför leda till felroutning trots fungerande TLS. Omvänt hjälper en korrekt Target Address inte om cloudconnectorn är inaktiverad. Diagnosen kopplar samman objekt och transport i stället för att bara betrakta en sida.

För experter blir vidarebefordringar, e-postkontakter, distributionsexpansion och transportregler viktiga. De kan ändra den ursprungliga mottagaradressen eller skapa ytterligare mottagare. Varje resulterande meddelande får sitt eget routningsbeslut.

## E-postgatewayer före eller efter Exchange Online

Många organisationer kompletterar hybrid med en Secure Email Gateway eller en molnfiltreringsplattform. Det innebär minst ytterligare ett SMTP-hopp. Vägen måste ritas separat för inkommande och utgående meddelanden ([Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)).

Om gatewayen står före Exchange Online pekar MX-posten på gatewayen. EOP ser då först dess käll-IP. Enhanced Filtering for Connectors kan inkludera den ursprungliga avsändarinformationen i Microsofts filterbedömning om connectorn och överhoppade IP-adresser är korrekt konfigurerade ([Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

För utgående trafik måste det vara klart om Exchange Online skickar direkt, via gatewayen eller, vid CMT, först lokalt och därefter via gatewayen. Flera tillåtna vägar kan kringgå policyer och skapa olika DKIM-signaturer, egress-IP-adresser och loggar.

Expertkontrollen är en graf över tillåtna vägar: Varje pil anger initiativtagare, mål, port, TLS-kontroll, tillåtna domäner, filtreringsuppgift, köägare och loggkälla. Ett gatewaynamn utan dessa uppgifter är ännu ingen arkitektur.

## Följ ett meddelande från början till slut

Felsökningen börjar med ett testmeddelande vars avsändare, mottagare och tidpunkt är kända. Först granskas den offentliga vägen, sedan händelserna på varje berörd Exchange-sida.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS- und SMTP-Test">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.com
Test-NetConnection mail.example.com -Port 25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig MX example.com
nc -vz mail.example.com 25
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) visar det publicerade MX-målet. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) kontrollerar om TCP 25 är nåbar från mätpunkten. Det bevisar ännu inte att TLS- eller connectorvalideringen lyckas.

Därefter följer SMTP-handskakningen. OpenSSL kan användas på båda administrationsplattformarna.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-STARTTLS-Test">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
openssl s_client -starttls smtp -connect mail.example.com:25 `
  -servername mail.example.com -showcerts
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
openssl s_client -starttls smtp -connect mail.example.com:25 \
  -servername mail.example.com -showcerts
```

  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) visar certifikatkedja, namn och TLS-förhandling. För en fullständig auktoriserad testdialog lämpar sig [`swaks`](https://jetmore.org/john/code/swaks/). Produktionsmeddelanden testas inte med godtyckligt påhittade avsändare; testidentitet och förväntad väg fastställs i förväg.

I Exchange Online ger [`Get-MessageTraceV2`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2) molnhändelserna. Lokalt visar [`Get-MessageTrackingLog`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog) bearbetningen på Exchange-servrar, och [`Get-Queue`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue) visar väntande nästa hopp. Tidsstämplar omvandlas till en gemensam tidszon; Internet Message ID och Network Message ID hjälper till att koppla ihop avsnitten ([Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq), [Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)).

## Vanliga felbilder utan ämnessprång

Ett **TLS-fel** behandlas först som ett transportproblem: Vilken värd anslöts, vilket certifikat presenterade den och vilket connectorvillkor förväntade motparten? Mottagarsynkronisering blir relevant först när meddelandet efter lyckad mottagning routas fel.

En **NDR för okänd mottagare** leder i stället först till mottagarobjektet och typen av Accepted Domain. Först när objektet är korrekt kontrolleras om den valda vägen transporterar det till rätt sida.

En **väntande kö** kräver nästa hopp, återförsökstid och `LastError`. Den öppna porten på målvärden hjälper bara som nästa test. En **loop** syns i upprepade `Received`-headers, hopp eller trackinghändelser och uppstår vanligtvis när båda sidor vidarebefordrar okända mottagare till varandra ([RFC 5321: Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3)).

En **anslutning som bara fungerar i en riktning** är ingen motsägelse. Motriktningarna använder olika avsändare, connectors och certifikatkontroller. De registreras och testas separat.

## Säkerhet, drift och ändringar

Hybrid-SMTP öppnar en medvetet tillåten transportväg. Denna väg bör begränsas till dokumenterade käll- och målsystem, certifikat och domäner. Öppna reläer, alltför breda IP-intervall eller connectors som klassar varje meddelande som betrott strider mot denna modell.

Certifikatbyten planeras som routningsändringar. Före utgångsdatumet kontrolleras det nya certifikatet, tjänsttilldelningen, den presenterade kedjan och connectorförväntningen på båda sidor. Därefter följer testmeddelanden i båda riktningarna och en kontrollerad återställningsplan.

För löpande drift övervakas minst hybridconnectors, certifikatutgång, köökning, Message Trace-fel, DNS-mål och gatewayhälsa. Vid Centralized Mail Transport tillkommer kapaciteten för den lokala utgående vägen.

## Säkerhetskopiering och återuppbyggnad av e-postvägen

SMTP-meddelanden ligger under ett avbrott i köer på de system som ansvarar för dem. En säkerhetskopia av konfigurationen återställer inte dessa väntande meddelanden. Konfiguration och transporttillstånd betraktas därför separat.

Vid återuppbyggnad ingår connectorparametrar på båda sidor, Accepted och Remote Domains, transportregler, offentliga DNS-poster, certifikat med privata nycklar, gatewaykonfiguration och HCW-val. Hemligheter lagras skyddat; läsbara exporter dokumenterar struktur och beroenden.

Efter en återuppbyggnad testas inte bara en port. Ett markerat meddelande passerar den förväntade vägen i varje riktning. Message Trace, lokala trackingloggar, gatewayloggar och målpostlådan bekräftar varje hopp. Först denna end-to-end-acceptans visar att routning och filtrering fungerar korrekt igen.

## Teknisk utveckling och avvägningar

Hybrid Configuration Wizard automatiserade under flera Exchange-generationer anslutningen till Exchange Online. Transporten förblev SMTP/TLS, medan cloudconnectors, certifikat som stöds och routningsalternativ vidareutvecklades ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

Direkt internettransport håller vägarna korta och använder respektive plattform där postlådan finns. Centralized Mail Transport samlar kontrollen men gör molnpost beroende av den lokala utgående vägen. En tredjepartsgateway tillför specialiserad filtrering eller kryptering, men ökar antalet hopp och loggkällor. Rätt val följer av ett verifierbart krav, inte av önskemålet att allt i diagrammet ska passera samma ruta.

## Källor

- [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)
- [Microsoft Learn – Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)
- [Microsoft Learn – Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)
- [Microsoft Learn – Hybrid mail flow](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow)
- [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [Microsoft Learn – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)
- [Microsoft Learn – Queues in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)
- [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2)
- [Microsoft Learn – Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog)
- [Microsoft Learn – Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc(1)](https://man.openbsd.org/nc)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [RFC 5321 – Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3)
