---
title: "Ghost Sender i Exchange Online: En MX-post är ingen brandvägg"
navTitle: "Ghost Sender"
description: "Direktleverans till Exchange Online kringgår en förkopplad gateway om klientorganisationen inte uttryckligen blockerar den. Risken är verklig, orsaken är en ofullständig mailflow-konfiguration."
date: "2026-07-15"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min lästid"
themen:
  - microsoft-365-exchange
slug: "ghost-sender-i-exchange-online-en-mx-post-ar-ingen-brandvagg"
image: "../images/ghost-admin.png"
translationOf: "ghost-sender-exchange-online-nebeneingang"
url: "https://rafaelpfister.ch/sv/blog/ghost-sender-i-exchange-online-en-mx-post-ar-ingen-brandvagg"
translationId: article-d8dc8d1da6379d67
translationReview: automatic
translationSourceHash: fc228adeba2a4ea46f6b36d20946d0aeb5c30f485b32da965e52168d2806a689
translatedAt: 2026-07-29T12:29:38.960Z
---

# Ghost Sender i Exchange Online: En MX-post är ingen brandvägg

![En spökadmin håller dörren bredvid säkerhetsgrinden öppen i datacentret, medan e-postmeddelanden passerar filtret direkt till inkorgen.](../images/ghost-admin.png)

Angreppsmöjligheten som InfoGuard Labs beskriver som «Ghost Sender» är verklig: En angripare kan kringgå en förkopplad e-postgateway och leverera direkt till Exchange Online. Förutsättningen är dock att klientorganisationen fortfarande accepterar denna direkta väg. Det är ingen universell sårbarhet i Exchange Online, utan en otillräckligt säkrad mailflow-topologi.

En Mail Transfer Agent som hanterar postlådor för en domän tar i grunden emot SMTP-anslutningar från internet. MX-posten visar reguljära avsändare den önskade leveransvägen. Den är varken en brandväggsregel eller en åtkomstlista och hindrar ingen från att direkt kontakta en känd Exchange Online-slutpunkt.

## Vad «Ghost Sender» faktiskt visar

Scenariot som [InfoGuard Labs beskriver](https://labs.infoguard.ch/posts/ghost-sender/) ser ut så här:

1. En organisation driver sina postlådor i Exchange Online.
2. Den offentliga MX-posten pekar på en förkopplad Secure Email Gateway.
3. Exchange Online-slutpunkten under `*.mail.protection.outlook.com` är fortsatt direkt åtkomlig från internet.
4. Administratören har inte begränsat Exchange Online så att endast den förkopplade gatewayen får leverera dit.
5. En angripare ignorerar MX-posten och levererar sitt meddelande direkt till Exchange Online.

Den avsedda vägen är alltså:

```text
Internet -> Drittanbieter-Filter -> Exchange Online -> Postfach
```

Men denna väg har lämnats öppen:

```text
Angreifer -> Exchange Online -> Postfach
```

Det är en allvarlig felkonfiguration. Det förkopplade filtret kan kringgås på denna väg; förfalskade avsändare, nätfiske och CEO-fraud underlättas därmed avsevärt. InfoGuard förtjänar erkännande för att ha synliggjort problemet, undersökt dess utbredning och publicerat ett lättanvänt test.

Men var exakt skulle produktfelet ligga här?

Även mediernas tillspetsning hjälper föga vid bedömningen. [Heise har rubriken att Exchange Online «utan vidare släpper igenom» förfalskade e-postmeddelanden](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html), trots att endast vissa ofullständigt härdade tredjeparts- och hybridkonfigurationer berörs. [Crow in the Cloud](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) uttrycker det betydligt mer träffande: inget säkerhetshål i strikt mening, utan ett design- och konfigurationsproblem.

## «An MTA is doing MTA-Things»

Varje Exchange Online-klientorganisation har en offentlig SMTP-slutpunkt. Denna slutpunkt är ingen hemlighet och ska inte heller vara det. Microsoft förklarar själv att Exchange Online som standard tar emot meddelanden som är direkt adresserade till postlådor som finns där: [det är helt enkelt så e-post fungerar](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865).

Även [SMTP beskriver MX-posten som en mekanism för att fastställa det reguljära målsystemet](https://www.rfc-editor.org/rfc/rfc5321.html#section-5.1). Därav följer ingen skyldighet för målservern att avvisa anslutningar via varje annan nåbar värd. En angripare behöver inte följa den skyltade vägen. Om ytterligare en MTA är nåbar, känner mottagardomänen och accepterar meddelandet kommer den att testas, ungefär som spammare i årtionden har försökt kontakta sämre skyddade backup-MX-system.

Den som kopplar in ett tredjepartsfilter förändrar standardtopologin. Från «Exchange Online är min e-postgateway mot internet» blir det «endast min tredjepartsgateway får överlämna internet-e-post till Exchange Online». Denna nya `Trust-Border` uppstår inte genom en DNS-post. Den måste uttryckligen tvingas fram i det mottagande systemet.

Microsoft dokumenterar just detta: Vid extern MX ska en inkommande anslutning av typen `Partner` skapas, som för `SenderDomains *` endast accepterar certifikatet eller käll-IP-adresserna för den förkopplade tjänsten. Meddelanden som levereras direkt förbi gatewayen avvisas då. Detta står ordagrant i Microsofts guide [«Manage mail flow using a third-party cloud service with Exchange Online»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud#best-practices-for-using-a-third-party-cloud-filtering-service-with-microsoft-365-or-office-365).

Även Frank Carius beskriver denna «sidodörr» utförligt i [MSXFAQ](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm).

## SPF, DKIM och DMARC är inga dörrvakter

InfoGuard visar meddelanden där SPF, DKIM och DMARC misslyckas men som ändå hamnar i postlådan. Det ser spektakulärt ut, men är ingen kryptografisk «bypass» av dessa metoder. E-postmeddelandena lyckas just inte. De levererar `fail`. Avgörande är vilken lokal åtgärd det mottagande systemet härleder från detta resultat.

SPF kontrollerar om ett system får skicka för kuvertavsändaren. DKIM kontrollerar en signatur. DMARC kopplar dessa resultat till den synliga avsändardomänen och publicerar en önskad hantering. Även den aktuella [DMARC-standarden RFC 9989](https://www.rfc-editor.org/rfc/rfc9989.html#section-1) fastslår uttryckligen att mottagaren kan beakta denna önskade hantering, men inte är skyldig att göra det. DMARC är en viktig signal, men ingen nätverksåtkomstkontroll.

Med en förkopplad gateway tillkommer att Exchange Online först ser gatewayens IP-adress och inte den ursprungliga avsändarens. För detta finns [Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Det rekonstruerar den ursprungliga källan och förbättrar SPF-, DKIM-, DMARC-, anti-spoofing- och anti-phishing-utvärderingar. Enhanced Filtering är dock inte heller något dörrlås. Det ersätter inte den restriktiva partneranslutningen.

Felkonfigurationen blir särskilt uppenbar när en administratör försvagar eller helt kringgår EOP-kontrollen med SCL-bypass, eftersom den förkopplade produkten redan ska filtrera, men samtidigt lämnar direktleverans från internet öppen. Då har administratören inte fått en skyddsmekanism «kringgången», utan medvetet inte längre tillhandahållit något effektivt skydd för en av två ingångar.

Man kan absolut kritisera Microsoft om ett meddelande trots ett tydligt synligt autentiseringsfel hamnar i inkorgen utan varning. Man kan kritisera semantiken för anslutningstyperna, dokumentationen och avsaknaden av varningar i Configuration Analyzer. Allt detta är legitima synpunkter. Existensen av en offentligt åtkomlig SMTP-slutpunkt är dock ingen säkerhetslucka.

## «Direct Send» är inte samma sak som «direktleverans»

I diskussionen blandas två saker ihop:

- **Direct Send** betecknar hos Microsoft anonyma meddelanden vars kuvertavsändare (`5321.MailFrom`) använder en egen Accepted Domain i klientorganisationen.
- **Direktleverans till Exchange Online** betecknar generellt ett SMTP-meddelande som ignorerar den publicerade tredjeparts-MX-posten och lämnas in direkt till Exchange-slutpunkten. Avsändaren kan även använda en valfri extern domän.

Inställningen

```powershell
Set-OrganizationConfig -RejectDirectSend $true
```

är lämplig om Direct Send inte behövs. Den förhindrar intern domänspoofing via denna väg. Men den stänger inte hela sidodörren för godtyckliga externa avsändare. Microsoft beskriver den exakta omfattningen i [cmdlet-dokumentationen för `RejectDirectSend`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-organizationconfig?view=exchange-ps#-rejectdirectsend). Den som vill förhindra «Ghost Sender» helt behöver fortfarande åtkomstbegränsningen via partneranslutning eller en lämplig mailflow-regel.

## Måste Microsoft verkligen göra allt åt administratören?

Nej. Den som integrerar ett extra e-postfilter i en produktiv transportkedja tar ansvar för den transportkedjan.

Leverantören kan inte tillförlitligt gissa om skannrar, multifunktionsenheter, SaaS-tjänster, hybridservrar, partnerreläer eller andra legitima system utöver den externa MX-posten fortfarande måste kunna skicka direkt till Exchange Online. En automatisk inställning som säger «MX pekar någon annanstans, blockera alltså allt annat» skulle avbryta önskade e-postflöden i många verkliga miljöer. Därför måste administratören uttryckligen definiera den önskade förtroendegränsen.

Microsoft bör ändå göra det enklare för de ansvariga. En bra Configuration Analyzer bör upptäcka en extern MX utan restriktiv partneranslutning och varna tydligt. Konfigurationsdialogen skulle kunna förklara att en anslutning av typen «Din organisation» visserligen identifierar lämpliga anslutningar, men inte automatiskt avvisar olämpliga anslutningar. Secure-by-default-inställningar och bättre driftrapporter vore också välkomna.

Det skulle vara meningsfull produkthärdning. Men det ändrar inte den tekniska bedömningen: En osäker specialtopologi förblir en osäker konfiguration och blir inte en zero-day enbart genom sin stora spridning.

## Så stängs sidodörren

För miljöer med förkopplat filter bör minst följande punkter finnas på checklistan:

1. **Dokumentera mailflow fullständigt.** Vilka system får faktiskt leverera till Exchange Online? Hit hör även hybrid-, applikations- och reservvägar.
2. **Konfigurera en restriktiv partneranslutning.** Använd `SenderDomains *` och begränsa leveransen till ett certifikat (föredras) eller till underhållna käll-IP-intervall. En anslutning av typen `OnPremises` respektive «Din organisation» tvingar inte fram denna default-deny-effekt (se exempelvis också: [E-postrouting mellan Apache James och Exchange Online](/blog/totemomail-m365)).
3. **Konfigurera Enhanced Filtering korrekt.** Om EOP fortsatt ska filtrera måste ursprunglig IP och avsändarinformation rekonstrueras korrekt. Generella SCL-`-1`-bypasser måste granskas kritiskt.
4. **Inaktivera Direct Send om det inte används.** Kontrollera först med Message Trace respektive tillgängliga rapporter om skannrar eller applikationer är beroende av det.
5. **Växla inte blint.** Testa och övervaka därefter gatewayens IP-intervall, certifikatbyten, hybrid-mailflow samt `onmicrosoft.com`-, Teams- och andra specialvägar.

Ett förenklat exempel för den IP-baserade varianten är:

```powershell
New-InboundConnector `
  -Name "Endast från överordnad e-postgateway" `
  -ConnectorType Partner `
  -SenderDomains * `
  -RestrictDomainsToIPAddresses $true `
  -SenderIpAddresses <Gatewayens-IP-intervall> `
  -RequireTls $true
```

Där det är möjligt bör certifikatbindning föredras framför IP-allowlist. Ändringar ska först göras i ett kontrollerat test, eftersom en felaktig allowlist mycket snabbt kan förvandla den öppna sidodörren till ett fullständigt e-postavbrott.

## Det enkla självtestet

Testet som InfoGuard (och MSXFAQ) visar är användbart:

```powershell
Send-MailMessage `
  -SmtpServer <klientnamn>.mail.protection.outlook.com `
  -To admin@<klientdomän> `
  -From noreply@example.com `
  -Subject "EXO sidodörr" `
  -Body "Testmeddelande direkt till klientorganisationen"
```

Med en korrekt begränsad partneranslutning förväntas ett SMTP-avslag som `5.7.51 TenantInboundAttribution; Rejecting`. En alternativ transportregel kan först acceptera meddelandet och sedan flytta det till karantän; därför måste utöver SMTP-svaret även Message Trace, karantän och postlåda kontrolleras. `Send-MailMessage` (deprecated) används här endast för en lättförståelig illustration. Alla kontrollerade SMTP-testverktyg fyller samma syfte.

## Ett användbart test med missvisande etikett

«Ghost Sender» är ingen ny SMTP-exploit. Det är ett slagkraftigt namn för en öppen sidodörr vars säkring Microsoft länge har dokumenterat och som administratören har lämnat öppen.

Det ironiska är att InfoGuard i sitt eget bidrag beskriver problemet som «widespread and systematic misconfiguration» och avslutar med meningen «Ghost-Sender is a misconfiguration». Även Microsofts Security Response Center klassade först inte rapporten som en säkerhetslucka. Fakta finns alltså i artikeln: endast rubriken, testmeddelandet och «Vulnerability»-varumärkningen berättar tyvärr en mer dramatisk historia.

Den meningsfulla delen av publiceringen är väckarklockan: Många företag har uppenbarligen inte låst sitt mailflow ordentligt. Den problematiska delen är påståendet att Exchange Online skulle ha en universell säkerhetslucka för detta. Nej: Exchange Online beter sig här först och främst som en MTA. Det blir osäkert genom en ofullständigt konfigurerad förtroendegräns.

Måste man verkligen göra allt åt administratören? Nej. Men man måste uppenbarligen gång på gång påminna om att DNS-routing inte ersätter åtkomstkontroll.

## Källor

1.  [InfoGuard Labs: Ghost-Sender – Universal Email Spoofing against Exchange Online](https://labs.infoguard.ch/posts/ghost-sender/): Den ursprungliga undersökningen inklusive spridningsanalys och den egna slutsatsen «Ghost-Sender is a misconfiguration».

2.  [Ghost Sender: Exchange Online Mail Spoofing Tester](https://ghost-sender.com/): Onlinetestet som InfoGuard publicerat för att kontrollera den egna klientorganisationen för den öppna sidodörren.

3.  [MSXFAQ: Exchange Online som sidodörr för e-postmottagning](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm): Frank Carius bedömning: inget fel i Exchange Online, utan en felkonfiguration av administratören.

4.  [Microsoft: Direct Send vs sending directly to an Exchange Online tenant](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865): Microsoft förklarar att direkt mottagning av e-post till värdbaserade postlådor är så e-post fungerar, och avgränsar Direct Send.

5.  [Microsoft Learn: Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): Den officiella guiden med ett särskilt steg för restriktiv partneranslutning vid extern MX.

6.  [Microsoft Learn: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Rekonstruerar den ursprungliga avsändarkällan bakom en gateway; förbättrar utvärderingen men ersätter inte anslutningen.

7.  [Heise: Ghost-Sender – Exchange Online släpper utan vidare igenom förfalskade e-postmeddelanden](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html): Exempel på tillspetsad rapportering som generaliserar endast vissa felkonfigurationer.

8.  [Crow in the Cloud: Spökena som jag inte kallade på](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/): Träffande bedömning som design- och konfigurationsproblem med skyddsåtgärder.

9.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html): Beskriver MX-posten som en mekanism för att fastställa det reguljära målsystemet, inte som åtkomstkontroll.

10.  [RFC 9989: DMARC](https://www.rfc-editor.org/rfc/rfc9989.html): Fastslår att mottagaren kan beakta den publicerade DMARC-hanteringen, men inte måste.

---

## Är ditt mailflow säkert?

Osäker på om din Exchange Online-klientorganisation också har en öppen sidodörr? **adeptio** granskar hela ditt mailflow: från MX-poster, anslutningar och tredjepartsgatewayer till EOP, SPF, DKIM, DMARC och Direct Send. Praktiskt, oberoende och med konkreta rekommendationer.

Den som vill granska eller säkra sitt mailflow ordentligt kan gärna boka ett icke-bindande rådgivningssamtal:

**[Boka ett rådgivningssamtal med adeptio](https://outlook.office.com/bookwithme/user/b4d64d6bdbca4b489074d459cd30b50c@adeptio.ch/meetingtype/3Wgk7rXJfk261852Hyovkg2?anonymous&ismsaljsauthenabled&ep=mlink)**  
[adeptio.ch](https://adeptio.ch/)
