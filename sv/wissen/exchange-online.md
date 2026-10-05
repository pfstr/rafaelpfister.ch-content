---
title: "Exchange Online: arkitektur, e-postflöde och drift"
blatt: "exchange-online"
description: "Exchange Online för meddelandeadministratörer: klientorganisation- och mottagarmodell, EOP-transport, anslutningar, klientåtkomst, PowerShell och Graph, Message Trace, kvarhållning, säkerhet och återställning."
fakten:
  - label: Produktroll
    wert: Molnbaserad tjänst för e-post, kalender och katalog
    href: https://learn.microsoft.com/en-us/exchange/exchange-online
  - label: Plattform
    wert: Microsoft 365
    href: https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description
  - label: E-postmottagning
    wert: Exchange Online Protection och SMTP
    href: https://learn.microsoft.com/en-us/defender-office-365/eop-about
  - label: Mottagare
    wert: Postlådor, grupper, kontakter, e-postanvändare och resurser
    href: https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online
  - label: Domäner
    wert: Authoritative eller Internal Relay
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains
  - label: Dirigering
    wert: Inbound- och Outbound Connectors, regler och MX
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail
  - label: Klientåtkomst
    wert: Outlook, Outlook på webben, ActiveSync och specialfall för IMAP/POP
    href: https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online
  - label: Identitet
    wert: Microsoft Entra ID och modern autentisering
    href: https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online
  - label: Administration
    wert: Exchange Admin Center och Exchange Online PowerShell
    href: https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell
  - label: API
    wert: Microsoft Graph för e-post-, kalender- och administrationsfunktioner
    href: https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview
  - label: Diagnostik
    wert: Message Trace, rapporter och Service Health
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Kvarhållning
    wert: Recoverable Items, Retention och Holds
    href: https://learn.microsoft.com/en-us/purview/retention-policies-exchange
werbung:
  - tools
  - newsletter
ctaThemen:
  - microsoft-365-exchange
  - smtp-mailflow
translationSourceHash: 5965bf4a9447505ffbe8b9a5d00c6f1abf629dfbde070d9c7700d9fdc9e4b603
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:30:20.843Z
translationReview: automatic
---

# Exchange Online: arkitektur, e-postflöde och drift

**Exchange Online** är den Exchange-tjänst som Microsoft driver i Microsoft 365. Den tillhandahåller postlådor, kalendrar, kontakter, grupper, SMTP-transport och administrationsfunktioner. Klientorganisationsadministratören beslutar om mottagare, domäner, anslutningar, regler, behörigheter och kvarhållning. Microsoft driver däremot postlådeservrarna, databaskopiorna, interna köer, korrigeringar och redundansväxlingar ([Exchange Online service description](https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description), [Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

Därmed liknar Exchange Online funktionellt ett eget Exchange-system, men inte driftmässigt. En lokal administratör kan undersöka en köfil eller aktivera en databaskopia. I Exchange Online ser han eller hon i stället de händelser, tillstånd och konfigurationsobjekt som tjänsten tillhandahåller. Den viktigaste förmågan är därför att mappa ett användarklagomål till en tydlig väg: identitet, klientåtkomst, mottagarobjekt, transport, filtrering, leverans eller kvarhållning.

## Från klientorganisation till postlåda

Klientorganisationen utgör den organisatoriska ramen. I den hanterar Exchange Online e-postaktiverade mottagare: användar- och delade postlådor, rums- och utrustningspostlådor, distributionslistor, Microsoft 365-grupper, kontakter och e-postanvändare. Mottagartypen avgör om data lagras, hur leverans sker och vilka behörigheter som är tillgängliga ([Recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)).

Ett användarkonto i Microsoft Entra ID och en Exchange-postlåda hänger ihop, men är inte samma objekt. Licensiering kan utlösa etableringen av en postlåda. Exchange lägger då till e-postrelaterade attribut och tjänster. Om en administratör tar bort en licens eller raderar ett konto gäller olika kvarhållnings- och raderingsperioder. För drift och offboarding måste därför identitetslivscykeln, postlådelivscykeln och compliance-kvarhållningen planeras tillsammans ([Delete or restore user mailboxes](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/delete-or-restore-mailboxes), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

För experter blir attributens ursprung viktigt. I en ren molnklientorganisation hanteras Exchange-egenskaper online. För synkroniserade identiteter kan den lokala miljön fortfarande vara den auktoritativa källan för vissa mottagarattribut. Då visas ett värde visserligen i Exchange Online, men måste ändras lokalt och synkroniseras på nytt. Denna modell hör hemma i artikeln [Exchange Hybrid](/kb/exchange-hybrid), eftersom den inte existerar utan katalogsynkronisering.

## Hur ett inkommande meddelande når postlådan

När mottagaren har förståtts går det att följa e-postvägen. En domäns publika MX pekar normalt på Exchange Online Protection, EOP. EOP accepterar SMTP-anslutningen, bedömer avsändaren och meddelandet, tillämpar skydds- och transportregler och överlämnar ett tillåtet meddelande till Exchange Online. För lokala mottagare följer sedan leveransen till postlådan ([Exchange Online Protection overview](https://learn.microsoft.com/en-us/defender-office-365/eop-about), [Mail flow in EOP](https://learn.microsoft.com/en-us/defender-office-365/eop-mail-flow)).

**Accepted Domain** fastställer hur Exchange Online behandlar mottagardomänen. Vid `Authoritative` förväntar sig tjänsten att alla giltiga mottagare finns i den egna organisationen och avvisar okända adresser. `Internal Relay` tillåter att okända mottagare vidarebefordras till ett annat system. Denna inställning är bara meningsfull när nästa hopp och mottagarupplösningen är tillförlitligt planerade; annars uppstår leveransfel eller slingor ([Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)).

Ett internt meddelande stannar inte automatiskt ”på samma server”. Exchange Online löser upp avsändare och mottagare, kontrollerar regler och skyddsprinciper samt registrerar transporthändelser. För administratören är denna händelsekedja avgörande: `Delivered` betyder att tjänsten har levererat till sitt mål; `Filtered`, `Failed`, `Pending` eller `Expanded` beskriver andra steg. Message Trace synliggör dessa steg, men ersätter inte kontrollen av målpostlådan eller en efterföljande regel ([Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message), [Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)).

## Utgående meddelanden och anslutningar

För utgående meddelanden avgörs först om Exchange Online skickar direkt till målsystemet eller använder en konfigurerad Outbound Connector. En anslutning kan styra meddelanden till den egna infrastrukturen, en partner eller en e-postgateway. Valet baseras bland annat på mottagardomän, anslutningsvillkor och transportregler ([Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)).

Inbound Connectors beskriver omvänt under vilka förhållanden Exchange Online litar på ett sändande system. Typiska kriterier är käll-IP eller ett TLS-certifikat. Dessa uppgifter är säkerhetsrelevanta: ett för stort IP-intervall eller ett otydligt kontrollerat certifikat kan få extern trafik att framstå som intern partnertrafik.

Om en extern e-postgateway står framför EOP ser Microsoft först gatewayens IP-adress. **Enhanced Filtering for Connectors** kan inkludera information om det ursprungliga hoppet i filterbedömningen. Funktionen är inte en allmän ”spamfilteromkopplare”, utan måste passa den faktiska vägen, anslutningarna och de överhoppade IP-adresserna ([Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

Expertfrågan här är: Vilken motpart har faktiskt accepterat ett meddelande, vilken identitet kontrollerades för anslutningen och vid vilket hopp skedde den senaste innehållsfiltreringen? Dessa tre svar hör hemma i varje diagram över e-postflödet.

## Teknisk uppbyggnad ur administratörens perspektiv

Exchange Online publicerar ingen serverlista som en klientorganisationsadministratör hanterar som en lokal farm. Tjänsten har ändå tydligt identifierbara tekniska byggstenar. De syns via protokoll och administrationsgränssnitt.

| Byggsten | Uppgift | Vad klientorganisationsadministratören ser |
|---|---|---|
| Exchange Online Protection | SMTP-mottagning, skydd mot skadlig kod, skräppostskydd och transportbearbetning | Karantän, principer, rapporter och Message Trace |
| Exchange-transport | Mottagarupplösning, regler, dirigering och leverans | Anslutningar, Accepted Domains, regler och händelser |
| Postlådetjänst | Lagring av e-post, kalender, kontakter och mappar | Postlådeobjekt, kvoter, behörigheter och klientåtkomst |
| Microsoft Entra ID | Identiteter för användare, grupper, program och inloggning | Konton, roller, Conditional Access och appregistreringar |
| Exchange Online PowerShell | Exchange-specifik administration | Cmdlets, RBAC och granskningsbara ändringar |
| Microsoft Graph | REST-API för program och automatisering | OAuth-behörigheter, resurser och begränsning |

Teknikstacken i periferin består därmed främst av SMTP och TLS för e-posttransport samt HTTPS, OAuth, PowerShell och REST för klient- och administrationsåtkomst. De interna implementeringsdetaljerna är bara relevanta för kunden i den utsträckning Microsoft dokumenterar dem som tjänstebeteende, begränsning eller diagnostikgränssnitt ([About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2), [Microsoft Graph mail API](https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1056" src="/images/kb-interaktiv-exchange-online.svg?v=20260813" title="Interaktive Infografik: Exchange-Online-Pfad von DNS und EOP über Transport und Postfach bis Entra, PowerShell, Graph und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-online.svg?v=20260813">Öppna den interaktiva Exchange Online-grafiken direkt</a>.
</iframe>

## Klientåtkomst och modern autentisering

E-posttransporten slutar i postlådan; användare ansluter därefter via klientprotokoll. Outlook, Outlook på webben, mobila klienter och program använder HTTPS-baserade slutpunkter. Autodiscover hjälper klienter att hitta rätt tjänst. Inloggningen sker via Microsoft Entra ID, medan Exchange kontrollerar behörigheten för postlådan ([Clients and mobile in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online), [Modern authentication in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online)).

Detta skiljer mellan två fel som ofta blandas ihop. Om inloggningen misslyckas i Entra når klienten ofta inte Exchange alls. Om tokenet är giltigt kan Exchange ändå neka åtkomst på grund av en saknad roll, postlådebehörighet, klientprincip eller fel målpostlåda. Inloggningsloggen och Exchange-diagnostiken måste därför granskas tillsammans tidsmässigt.

Program ansluter helst via Microsoft Graph eller Exchange-gränssnitt som stöds. En Graph-Application-Permission kan gälla brett; Exchange RBAC for Applications kan ange ett snävare tillgängligt postlådeområde. Ett giltigt OAuth-token är alltså bara det första steget. Därefter kontrollerar resurstitjänsten vilken åtgärd som är tillåten för vilken postlåda ([Role Based Access Control for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)).

## Spåra behörigheter och ändringar

Exchange Online har egna administrativa roller. Entra-roller kan möjliggöra åtkomst till Exchange-administrationen, men de faktiska Exchange-cmdlets och deras omfattning bestäms av Exchange-RBAC ([Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)).

Därtill finns postlådebehörigheter som Full Access, Send As och Send on Behalf. De styr olika åtgärder och bör inte inventeras som en gemensam ”delegeringsbehörighet”. För program tillkommer OAuth- och Exchange-programroller ([Manage permissions for recipients](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)).

För experter är ändringens ursprung lika viktigt som slutläget. Granskningsloggar, Entra-inloggningsloggar och konfigurationsexporter svarar på vem som har ändrat en regel, anslutning eller behörighet. En export nattetid av centrala e-postflödesobjekt underlättar jämförelser, men ersätter inte en skyddad granskningskälla.

## Diagnostik: först DNS, sedan transporthändelser

En analys av e-postflödet börjar utanför klientorganisationen. MX visar vilket system som tar emot internetpost. Därefter används Message Trace för att kontrollera om Exchange Online har sett det specifika meddelandet och hur det har bearbetat det.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für MX- und Autodiscover-Abfrage">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.com
Resolve-DnsName autodiscover.example.com
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig MX example.com
dig autodiscover.example.com
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) visar publicering och upplösning. De säger ännu inget om huruvida EOP har accepterat meddelandet eller om en postlåda har tagit emot det.

För nästa steg väljs ett snävt tidsintervall med avsändare och mottagare. Samma Exchange Online PowerShell körs i Windows och med `pwsh` på Unix-system som stöds.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Exchange Online PowerShell">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Connect-ExchangeOnline
Get-MessageTraceV2 -SenderAddress sender@example.net `
  -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date)
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```powershell
pwsh
Connect-ExchangeOnline
Get-MessageTraceV2 -SenderAddress sender@example.net `
  -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date)
```

  </div>
</div>

[`Connect-ExchangeOnline`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/connect-exchangeonline) upprättar den autentiserade administrationssessionen. [`Get-MessageTraceV2`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2) söker efter transporthändelser; [`Get-Date`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-date) begränsar tidsfönstret. För trender tillkommer rapporter, för Microsoft-störningar Service Health. En enskild grön indikator besvarar inte alla tre frågorna ([Exchange Online monitoring](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-exchange-monitoring?view=o365-worldwide)).

## Kvarhållning, radering och återställning

Microsoft skyddar den löpande tjänsten med flera databaskopior, Shadow Redundancy och Safety Net. Dessa mekanismer tjänar tjänstens tillgänglighet och dataintegritet. De är inte användargränssnittet för att återställa ett meddelande som raderats av misstag ([Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

För användar- och compliancefall används andra funktioner: Deleted Item Retention, Recoverable Items, Single Item Recovery, Retention Policies och Holds. Deras effekter överlappar, men de har olika syften. En kvarhållningsregel kan skydda innehåll mot permanent radering; den utgör inte automatiskt en separat säkerhetskopia oberoende av klientorganisationen med fritt valbar återställningspunkt ([Recoverable Items folder](https://learn.microsoft.com/en-us/exchange/security-and-compliance/recoverable-items-folder/recoverable-items-folder), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

Ett robust återställningskoncept fastställer därför vilka händelser Microsofts tjänsteresiliens täcker, vilket innehåll som kan återhämtas via Exchange- eller Purview-kvarhållning och för vilka krav en oberoende kopia behövs. Återställningstester bör använda konkreta fall: enskilt meddelande, mapp, postlåda efter användarradering, juridiskt kvarhållet objekt och störning i hela klientorganisationen.

## Säkerhet och typiska begränsningar

Exchange Online sammanför flera säkerhetsområden: internetpost, EOP, klientorganisationskonfiguration, Entra-inloggning, postlåderättigheter och program. Skyddseffekten beror på att den verkliga meddelande- och inloggningsvägen överensstämmer med konfigurationen.

För e-postflödet innebär det att MX, anslutningsidentitet, Enhanced Filtering, SPF/DKIM/DMARC och transportregler måste kontrolleras som en kedja. För klientåtkomst är modern autentisering, Conditional Access, Exchange-RBAC och postlådebehörigheter separata kontroller. För program tillkommer OAuth-consent och det tillåtna postlådeområdet.

Den djupare administratörsfrågan är alltid densamma: Vilket system fattade beslutet, vilka indata såg det vid tillfället och var loggas resultatet? Utan dessa tre uppgifter är även en formellt korrekt princip svår att kontrollera.

## Teknisk utveckling och medvetna avvägningar

Exchange Online utvecklades ur Microsofts värdbaserade Exchange-erbjudanden och övertog många koncept från serverprodukten: mottagare, postlådedatabaser, transport, DAG:er, Shadow Redundancy och Safety Net. Tjänsten automatiserar driften av denna infrastruktur och ger klientorganisationsadministratörer en högre administrationsnivå ([Exchange Team: 20 years ago](https://techcommunity.microsoft.com/blog/exchange/20-years-ago-in-a-galaxy-far-away8230/604456), [Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

Vinsten ligger i outsourcad plattformsdrift, global tjänsteintegration och standardiserade administrationsgränssnitt. Priset är mindre åtkomst till enskilda servrar, köer och databaskopior samt ett större beroende av publicerade funktioner för diagnostik, export och återställning. För experter handlar uppgiften därför inte om att gissa den osynliga interna topologin, utan om att fullt ut använda de dokumenterade klientorganisationskontrollerna och tjänstsignalerna.

## Källor

- [Microsoft Learn – Exchange Online](https://learn.microsoft.com/en-us/exchange/exchange-online)
- [Microsoft – Exchange Online service description](https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description)
- [Microsoft Defender – Exchange Online Protection overview](https://learn.microsoft.com/en-us/defender-office-365/eop-about)
- [Microsoft Defender – Mail flow in EOP](https://learn.microsoft.com/en-us/defender-office-365/eop-mail-flow)
- [Microsoft Learn – Recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)
- [Microsoft Learn – Delete or restore user mailboxes](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/delete-or-restore-mailboxes)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Clients and mobile in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online)
- [Microsoft Learn – Modern authentication in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online)
- [Microsoft Learn – Exchange Online PowerShell](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell)
- [Microsoft Learn – About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2)
- [Microsoft Graph – Mail API overview](https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview)
- [Microsoft Learn – RBAC for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)
- [Microsoft Learn – Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)
- [Microsoft Learn – Manage permissions for recipients](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)
- [Microsoft Service Assurance – Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)
- [Microsoft Learn – Recoverable Items folder](https://learn.microsoft.com/en-us/exchange/security-and-compliance/recoverable-items-folder/recoverable-items-folder)
- [Microsoft Purview – Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)
- [Microsoft Learn – Exchange Online monitoring](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-exchange-monitoring?view=o365-worldwide)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [Microsoft Learn – Connect-ExchangeOnline](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/connect-exchangeonline)
- [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2)
- [Microsoft Learn – Get-Date](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-date)
- [Exchange Team – 20 years ago](https://techcommunity.microsoft.com/blog/exchange/20-years-ago-in-a-galaxy-far-away8230/604456)
