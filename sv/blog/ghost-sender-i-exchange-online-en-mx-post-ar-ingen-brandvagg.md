---
title: "Ghost Sender i Exchange Online: En MX-post är ingen brandvägg"
navTitle: "Ghost Sender"
description: "Direktleverans till Exchange Online kringgår en föregående gateway om klientorganisationen inte uttryckligen blockerar den. Risken är verklig, orsaken är en ofullständig e-postflödeskonfiguration."
date: "2026-07-15"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min lästid"
themen:
  - microsoft-365-exchange
slug: "ghost-sender-i-exchange-online-en-mx-post-ar-ingen-brandvagg"
image: "../images/ghost-admin.png"
translationOf: "ghost-sender-exchange-online-nebeneingang"
translationId: article-d8dc8d1da6379d67
translationReview: required
translationSourceHash: 6a500f1ed53a180322afb3c86e44376100d68659eeb55ffae35937ab434c6b61
translatedAt: 2026-09-05T07:50:23.254Z
url: https://rafaelpfister.ch/sv/blog/ghost-sender-i-exchange-online-en-mx-post-ar-ingen-brandvagg
translationModel: gpt-5.6-terra
---

# Ghost Sender i Exchange Online: En MX-post är ingen brandvägg

![En Ghost-admin håller upp dörren bredvid säkerhetsgrinden i datacentret, medan e-postmeddelanden når inkorgen direkt förbi filtret.](../images/ghost-admin.png)

Angreppsmöjligheten som InfoGuard Labs beskriver som «Ghost Sender» är verklig: En angripare kan kringgå en föregående e-postgateway och leverera direkt till Exchange Online. Förutsättningen är dock att klientorganisationen fortfarande accepterar denna direkta väg. Det är inte en universell sårbarhet i Exchange Online, utan en ofullständigt säkrad e-postflödestopologi.

En Mail Transfer Agent som hanterar postlådor för en domän tar i princip emot SMTP-anslutningar från internet. MX-posten visar reguljära avsändare den avsedda leveransvägen. Den är varken en brandväggsregel eller en åtkomstlista och hindrar ingen från att direkt kontakta en känd Exchange Online-slutpunkt.

## Vad «Ghost Sender» faktiskt visar

Scenariot som [InfoGuard Labs beskriver](https://labs.infoguard.ch/posts/ghost-sender/) ser ut så här:

1. En organisation har sina postlådor i Exchange Online.
2. Den offentliga MX-posten pekar på en föregående Secure Email Gateway.
3. Exchange Online-slutpunkten under `*.mail.protection.outlook.com` är fortfarande direkt åtkomlig från internet.
4. Administratören har inte begränsat Exchange Online så att endast den föregående gatewayen får leverera dit.
5. En angripare ignorerar MX-posten och levererar sitt meddelande direkt till Exchange Online.

Den avsedda vägen är alltså:

```text
Internet -> Drittanbieter-Filter -> Exchange Online -> Postfach
```

Men denna väg har lämnats öppen:

```text
Angreifer -> Exchange Online -> Postfach
```

Detta är en allvarlig felkonfiguration. Det föregående filtret kan kringgås via denna väg; förfalskade avsändare, nätfiske och CEO-bedrägerier underlättas därmed avsevärt. InfoGuard förtjänar erkännande för att ha synliggjort problemet, undersökt dess spridning och publicerat ett lättanvänt test.

Men var exakt skulle produktfelet vara här?

Inte heller den mediala tillspetsningen hjälper särskilt mycket vid bedömningen. [Heise har rubriken att Exchange Online släpper igenom förfalskade e-postmeddelanden «utan vidare»](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html), trots att det endast är vissa, ofullständigt härdade tredjeparts- och hybridkonfigurationer som berörs. [Crow in the Cloud](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) uttrycker det betydligt träffsäkrare: inget säkerhetshål i strikt mening, utan ett design- och konfigurationsproblem.

## «An MTA is doing MTA-Things»

Varje Exchange Online-klientorganisation har en offentlig SMTP-slutpunkt. Denna slutpunkt är ingen hemlighet och ska inte heller vara det. Microsoft förklarar självt att Exchange Online som standard tar emot meddelanden som är direkt adresserade till postlådor som finns där: [det är helt enkelt så e-post fungerar](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865).

Även [SMTP självt beskriver MX-posten som en mekanism för att fastställa det reguljära målsystemet](https://www.rfc-editor.org/rfc/rfc5321.html#section-5.1). Därav följer ingen skyldighet för målservern att avvisa anslutningar via varje annan åtkomlig värd. En angripare måste inte följa den skyltade vägen. Om ytterligare en MTA är åtkomlig, känner till mottagardomänen och accepterar meddelandet, kommer den att testas – ungefär som spammare i årtionden har försökt kontakta sämre skyddade backup-MX-system.

Den som kopplar in ett tredjepartsfilter förändrar standardtopologin. «Exchange Online är min internet-e-postgateway» blir «endast min tredjepartsgateway får överlämna internet-e-post till Exchange Online». Denna nya `Trust-Border` uppstår inte genom en DNS-post. Den måste uttryckligen framtvingas på det mottagande systemet.

Det är precis vad Microsoft dokumenterar: Vid extern MX ska en inkommande connector av typen `Partner` skapas, som för `SenderDomains *` endast accepterar certifikatet eller käll-IP-adresserna för den föregående tjänsten. Meddelanden som levereras direkt förbi gatewayen avvisas då. Detta står ordagrant i Microsofts guide [«Manage mail flow using a third-party cloud service with Exchange Online»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud#best-practices-for-using-a-third-party-cloud-filtering-service-with-microsoft-365-or-office-365).

Även Frank Carius beskriver denna «sidodörr» utförligt i [MSXFAQ](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm).

## SPF, DKIM och DMARC är inga dörrvakter

InfoGuard visar meddelanden där SPF, DKIM och DMARC misslyckas men som ändå hamnar i inkorgen. Det ser spektakulärt ut, men är ingen kryptografisk «bypass» av dessa metoder. E-postmeddelandena godkänns just inte. De levererar `fail`. Det avgörande är vilken lokal åtgärd det mottagande systemet härleder från detta resultat.

SPF kontrollerar om ett system får skicka för kuvertavsändaren. DKIM kontrollerar en signatur. DMARC kopplar samman dessa resultat med den synliga avsändardomänen och publicerar en önskad hantering. Även den aktuella [DMARC-standarden RFC 9989](https://www.rfc-editor.org/rfc/rfc9989.html#section-1) anger uttryckligen att mottagaren kan ta hänsyn till denna önskade hantering, men inte är skyldig att göra det. DMARC är en viktig signal, men ingen nätverksåtkomstkontroll.

Med en föregående gateway tillkommer att Exchange Online först ser gatewayens IP-adress, inte den ursprungliga avsändarens. För detta finns [Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): det rekonstruerar den ursprungliga källan och förbättrar SPF-, DKIM-, DMARC-, anti-spoofing- och anti-phishing-utvärderingar. Men Enhanced Filtering är inte heller ett dörrlås. Det ersätter inte den restriktiva partner-connectorn.

Felkonfigurationen blir särskilt uppenbar när en administratör försvagar EOP-kontrollen med en SCL-bypass eller helt kringgår den, eftersom den föregående produkten redan ska filtrera, men samtidigt lämnar direktleverans från internet öppen. Då har administratören inte fått en skyddsmekanism «kringgången», utan medvetet inte längre tillhandahållit något effektivt skydd för en av två ingångar.

Man kan absolut kritisera Microsoft om ett meddelande trots ett tydligt synligt autentiseringsfel hamnar i inkorgen utan varning. Man kan kritisera semantiken för connector-typerna, dokumentationen och avsaknaden av varningar i Configuration Analyzer. Allt detta är legitima synpunkter. Förekomsten av en offentligt åtkomlig SMTP-slutpunkt är dock ingen säkerhetsbrist.

## «Direct Send» är inte samma sak som «direktleverans»

I diskussionen blandas två saker ihop:

- **Direct Send** avser hos Microsoft anonyma meddelanden vars kuvertavsändare (`5321.MailFrom`) använder en egen Accepted Domain för klientorganisationen.
- **Direktleverans till Exchange Online** avser generellt ett SMTP-meddelande som ignorerar den publicerade tredjeparts-MX-posten och lämnas in direkt till Exchange-slutpunkten. Avsändaren kan då även använda en godtycklig extern domän.

För Direct Send finns en egen inställning:

```powershell
Set-OrganizationConfig -RejectDirectSend $true
```

<details class="options-details">
<summary>Alternativ förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `-RejectDirectSend $true` | Avvisar anonyma direktinlämningar vars kuvertavsändare använder en Accepted Domain för klientorganisationen |

</details>

Inställningen är meningsfull om Direct Send inte behövs. Den förhindrar spoofing av interna domäner via denna väg. Den stänger dock inte hela sidodörren för godtyckliga externa avsändare. Microsoft beskriver den exakta räckvidden i [cmdlet-dokumentationen för `RejectDirectSend`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-organizationconfig?view=exchange-ps#-rejectdirectsend). Den som vill förhindra «Ghost Sender» helt behöver fortfarande åtkomstbegränsning via partner-connector eller en lämplig e-postflödesregel.

## Måste Microsoft verkligen göra allt åt administratören?

Nej. Den som integrerar ett extra e-postfilter i en produktiv transportkedja tar ansvar för denna transportkedja.

Leverantören kan inte på ett tillförlitligt sätt gissa om skannrar, multifunktionsenheter, SaaS-tjänster, hybridservrar, partnerreläer eller andra legitima system utöver den externa MX-posten måste skicka direkt till Exchange Online. Ett automatiskt «MX pekar någon annanstans, så blockera allt annat» skulle avbryta önskade e-postflöden i många verkliga miljöer. Därför måste administratören uttryckligen definiera den önskade förtroendegränsen.

Ändå bör Microsoft göra det enklare för de ansvariga. En bra Configuration Analyzer bör identifiera en extern MX utan restriktiv partner-connector och varna tydligt. Installationsdialogen skulle kunna förklara att en connector av typen «Din organisation» visserligen identifierar lämpliga anslutningar, men inte automatiskt avvisar olämpliga anslutningar. Secure-by-default-inställningar och bättre driftrapporter vore också välkomna.

Det skulle vara meningsfull produkthärdning. Men det förändrar inte den tekniska bedömningen: En osäker specialtopologi förblir en osäker konfiguration och blir inte en zero-day enbart genom sin stora spridning.

## Så stängs sidodörren

För miljöer med föregående filter bör minst följande punkter finnas på checklistan:

1. **Dokumentera e-postflödet fullständigt.** Vilka system får faktiskt leverera till Exchange Online? Detta omfattar även hybrid-, applikations- och nödvägar.
2. **Konfigurera en restriktiv partner-connector.** Använd `SenderDomains *` och begränsa leveransen till ett certifikat (föredras) eller till underhållna käll-IP-intervall. En connector av typen `OnPremises` respektive «Din organisation» framtvingar inte denna default-deny-effekt (se till exempel även: [E-postrouting mellan Apache James och Exchange Online](/blog/totemomail-m365)).
3. **Konfigurera Enhanced Filtering korrekt.** Om EOP fortsatt ska filtrera måste ursprunglig IP och avsändarinformation rekonstrueras korrekt. Generella SCL-`-1`-bypasser måste granskas kritiskt.
4. **Inaktivera Direct Send om det inte används.** Kontrollera först med Message Trace respektive tillgängliga rapporter om skannrar eller applikationer är beroende av detta.
5. **Växla inte blint.** Testa och övervaka därefter gateway-IP-intervall, certifikatbyten, hybrid-e-postflöde samt `onmicrosoft.com`-, Teams- och andra specialvägar.

Ett förenklat exempel för den IP-baserade varianten är:

```powershell
New-InboundConnector `
  -Name "Only from upstream mail gateway" `
  -ConnectorType Partner `
  -SenderDomains * `
  -RestrictDomainsToIPAddresses $true `
  -SenderIpAddresses <IP-Bereiche-des-Gateways> `
  -RequireTls $true
```

<details class="options-details">
<summary>Alternativ förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `-Name` | Visningsnamn för den nya inkommande connectorn |
| `-ConnectorType Partner` | Connector-klass för externa partnersystem; endast denna typ tvingar fram avvisning av olämpliga anslutningar |
| `-SenderDomains *` | Connectorn gäller för e-post från alla avsändardomäner |
| `-RestrictDomainsToIPAddresses $true` | Aktiverar spärren: e-post för de angivna domänerna tas endast emot från adresserna i `-SenderIpAddresses` |
| `-SenderIpAddresses` | Tillåtna käll-IP-adresser respektive -intervall för den föregående gatewayen |
| `-RequireTls $true` | Kräver TLS-kryptering för anslutningar via denna connector |

</details>

Där det är möjligt bör certifikatbindning föredras framför IP-allowlist. Ändringar bör först göras i ett kontrollerat test, eftersom en felaktig allowlist mycket snabbt förvandlar den öppna sidodörren till ett fullständigt e-postavbrott.

## Det enkla självtestet

Testet som InfoGuard (och MSXFAQ) visar är användbart:

```powershell
Send-MailMessage `
  -SmtpServer <tenantname>.mail.protection.outlook.com `
  -To admin@<tenantdomain> `
  -From noreply@example.com `
  -Subject "EXO Nebeneingang" `
  -Body "Testmail direkt zum Tenant"
```

<details class="options-details">
<summary>Alternativ förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `-SmtpServer` | Målvärd: klientorganisationens offentliga Exchange Online-slutpunkt, medvetet förbi MX-posten |
| `-To` | Mottagaradress i klientorganisationen som ska testas |
| `-From` | Godtycklig extern avsändaradress; det är precis detta som sidodörren egentligen inte längre ska acceptera |
| `-Subject` | Ämnesrad, för att kunna hitta meddelandet i Message Trace |
| `-Body` | Meddelandetext |

</details>

Med en korrekt begränsad partner-connector kan man förvänta sig ett SMTP-avslag som `5.7.51 TenantInboundAttribution; Rejecting`. En alternativ transportregel kan först acceptera meddelandet och därefter flytta det till karantän; därför måste man kontrollera Message Trace, karantän och postlåda utöver SMTP-svaret. `Send-MailMessage` (deprecated) används här endast som en lättbegriplig illustration. Alla kontrollerade SMTP-testverktyg fyller samma syfte.

## Ett användbart test med en missvisande etikett

«Ghost Sender» är ingen ny SMTP-exploit. Det är ett slagkraftigt namn för en öppen sidodörr vars säkring Microsoft länge har dokumenterat och som administratören har lämnat öppen.

Det ironiska är att InfoGuard självt betecknar problemet i sitt eget inlägg som «widespread and systematic misconfiguration» och avslutar med meningen «Ghost-Sender is a misconfiguration». Microsofts Security Response Center klassificerade också inledningsvis rapporten som ingen säkerhetsbrist. Fakta finns alltså i artikeln: men rubriken, testmejlet och «Vulnerability»-varumärkningen antyder tyvärr en mer dramatisk tolkning.

Den meningsfulla delen av publiceringen är väckarklockan: Många företag har uppenbarligen inte låst sitt e-postflöde ordentligt. Den problematiska delen är påståendet att Exchange Online har en universell säkerhetsbrist för detta. Nej: Exchange Online beter sig här först och främst som en MTA. Det blir osäkert genom den inte färdigkonfigurerade förtroendegränsen.

Måste man verkligen göra allt åt administratören? Nej. Men man måste uppenbarligen gång på gång påminna om att DNS-routing inte ersätter åtkomstkontroll.

## Källor

1.  [InfoGuard Labs: Ghost-Sender – Universal Email Spoofing against Exchange Online](https://labs.infoguard.ch/posts/ghost-sender/): Den ursprungliga undersökningen med spridningsanalys och den egna slutsatsen «Ghost-Sender is a misconfiguration».

2.  [Ghost Sender: Exchange Online Mail Spoofing Tester](https://ghost-sender.com/): Onlinetestet som InfoGuard publicerat för att kontrollera den egna klientorganisationen för den öppna sidodörren.

3.  [MSXFAQ: Exchange Online som sidodörr för e-postmottagning](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm): Frank Carius bedömning: inget fel i Exchange Online, utan en felkonfiguration av administratören.

4.  [Microsoft: Direct Send vs sending directly to an Exchange Online tenant](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865): Microsoft förklarar att direkt mottagning av e-post till hostade postlådor är så e-post fungerar, och avgränsar Direct Send.

5.  [Microsoft Learn: Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): Den officiella guiden med ett eget steg för restriktiv partner-connector vid extern MX.

6.  [Microsoft Learn: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Rekonstruerar den ursprungliga avsändarkällan bakom en gateway; förbättrar utvärderingen men ersätter inte connectorn.

7.  [Heise: Ghost-Sender – Exchange Online släpper igenom förfalskade e-postmeddelanden utan vidare](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html): Exempel på tillspetsad rapportering som generaliserar endast vissa felkonfigurationer.

8.  [Crow in the Cloud: Spökena som jag inte kallade på](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/): Träffsäker bedömning som design- och konfigurationsproblem samt skyddsåtgärder.

9.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html): Beskriver MX-posten som en mekanism för att fastställa det reguljära målsystemet, inte som åtkomstkontroll.

10.  [RFC 9989: DMARC](https://www.rfc-editor.org/rfc/rfc9989.html): Anger att mottagaren kan ta hänsyn till den publicerade DMARC-hanteringen, men inte måste.

---

## Är ditt e-postflöde säkert?

Osäker på om din Exchange Online-klientorganisation också har en öppen sidodörr? **adeptio** granskar hela ditt e-postflöde: från MX-poster, connectors och tredjepartsgatewayar till EOP, SPF, DKIM, DMARC och Direct Send. Praktiskt, oberoende och med konkreta rekommendationer.

Den som vill granska sitt e-postflöde eller få det ordentligt säkrat kan gärna boka ett icke-bindande rådgivningssamtal:

**[Boka ett rådgivningssamtal med adeptio](https://outlook.office.com/bookwithme/user/b4d64d6bdbca4b489074d459cd30b50c@adeptio.ch/meetingtype/3Wgk7rXJfk261852Hyovkg2?anonymous&ismsaljsauthenabled&ep=mlink)**  
[adeptio.ch](https://adeptio.ch/)
