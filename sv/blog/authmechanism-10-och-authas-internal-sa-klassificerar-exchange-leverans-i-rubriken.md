---
title: "AuthMechanism 10 och AuthAs Internal: Så klassificerar Exchange leverans i rubriken"
navTitle: "AuthMechanism 10"
description: "Rubriken X-MS-Exchange-Organization-AuthMechanism dokumenterar hur en levererande server har autentiserats. Värdet 10 står för en Receive Connector med Externally Secured och klassificerar externa e-postmeddelanden som interna, med konsekvenser för skräppostfilter, e-postflödesregler och skydd mot spoofing."
date: "2026-08-26"
featured: "2026-08-27"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "8 min. läsning"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-hybrid"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-hybrid-header-intern-extern
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
slug: "authmechanism-10-och-authas-internal-sa-klassificerar-exchange-leverans-i-rubriken"
translationId: "article-0df383d5c49016da"
translationOf: exchange-authmechanism-10-authas-internal
url: https://rafaelpfister.ch/sv/blog/authmechanism-10-och-authas-internal-sa-klassificerar-exchange-leverans-i-rubriken
translationSourceHash: 5a9335a90afc9bf7df78b908f71b679f64c29f3b9e96bd7f25bcc916123b82df
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:18:19.447Z
translationReview: automatic
---

# AuthMechanism 10 och AuthAs Internal: Så klassificerar Exchange leverans i rubriken

Vid analys av skräppost-, spoofing- och e-postflödesfall i Exchange-miljöer är tre rubriker som Exchange stämplar vid mottagning avgörande:

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthSource: EX01.example.com
X-MS-Exchange-Organization-AuthMechanism: 10
```

`AuthAs` fastställer hur avsändaren framträdde för transporten. `AuthSource` anger servern som gjorde bedömningen. `AuthMechanism` dokumenterar genom vilken mekanism autentiseringen skedde. Tillsammans avgör de om Exchange behandlar ett meddelande som internt eller externt, och denna klassificering får betydande konsekvenser.

## Varför klassificeringen är viktig

`AuthAs` har i praktiken två värden: `Internal` och `Anonymous`. Ett meddelande som klassificeras som `Internal` behandlas annorlunda än extern e-post:

- E-postflödesregler med villkoret ”avsändaren finns utanför organisationen” tillämpas inte.
- Meddelandet får levereras till distributionsgrupper och postlådor som kräver autentiserade avsändare (`RequireSenderAuthenticationEnabled`).
- Antispam- och antispoofing-kontroller blir mindre strikta eller uteblir; i hybridmiljöer läggs den externa disclaimern inte till och Outlook visar ingen ”Extern”-markering.
- Visningsnamnet löses upp från adressboken och e-postmeddelandet ser ut som intern e-post för mottagarna.

Just därför bör frågan ”AuthAs Internal eller Anonymous?” ställas i början av varje rubrikanalys: Då går det att klarlägga varför ett uppenbart spoofing-meddelande passerade skräppostfiltret eller varför en e-postflödesregel aldrig utlöstes.

## AuthMechanism-värdena

Microsoft dokumenterar inte kodningen av `AuthMechanism` fullständigt offentligt. Två värden är relevanta för felsökning och väl dokumenterade:

| Värde | Betydelse |
|---|---|
| `04` | Autentiserad Exchange-trafik: postlåda till postlåda inom organisationen samt hybridtrafik via de connectorer som konfigurerats av Hybrid Configuration Wizard. |
| `10` | Receive Connector med autentiseringsalternativet `ExternalAuthoritative` (”Externally secured”): Anslutningen anses vara säkrad utanför Exchange, och allt som levereras via den behandlas som internt. |

Fler värden förekommer i rubriker, men saknar officiell referens. I praktiken räcker distinktionen: `04` innebär verklig Exchange-autentisering, `10` innebär förtroende genom connector-konfiguration.

## Vad Externally Secured verkligen innebär

Alternativet `ExternalAuthoritative` på en Receive Connector säger till Exchange: Någon annan ansvarar för att säkra den här anslutningen, exempelvis en brandvägg, ett dedikerat nätverkssegment eller IPsec. Exchange kontrollerar då inte längre något utan behandlar varje leverans via denna connector som autentiserad och intern, inklusive rätten att använda godtyckliga interna avsändaradresser.

Detta är avsett för ett fåtal scenarier, till exempel en helt betrodd applikationsserver i det egna datacentret. Det blir problematiskt om connectorn pekar på en framförliggande e-postgateway eller ett skräppostfilter i DMZ:en, via vilken även internetpost tas emot. Då får varje externt e-postmeddelande efter leveransen:

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthMechanism: 10
```

Konsekvenserna är att externa e-postmeddelanden betraktas som interna, e-postflödesregler för externa avsändare inte tillämpas, spoofing-skyddet för den egna domänen blir verkningslöst och att alla som når gatewayen kan leverera med interna avsändaradresser till mottagare som egentligen kräver autentiserade avsändare.

## Hitta berörda connectorer

Exchange Management Shell visar vilka Receive Connectorer som är konfigurerade med `ExternalAuthoritative`:

```powershell
Get-ReceiveConnector | Where-Object {
  $_.AuthMechanism -match "ExternalAuthoritative"
} | Format-Table Identity, RemoteIPRanges, AuthMechanism, PermissionGroups
```

Kontrollera för varje träff vilka `RemoteIPRanges` som är angivna och om systemen bakom verkligen behöver detta förtroende. En gateway som endast ska vidarebefordra e-post behöver det inte.

## Alternativet för reläscenarier

Om ett system endast ska reläa anonymt via Exchange (skrivare, applikationer, övervakning) är en anonym relay-connector den renare lösningen: anonym leverans plus rätten att leverera till godtyckliga mottagare, men utan Internal-klassificering.

```powershell
New-ReceiveConnector -Name "Anonymous Relay" -TransportRole FrontendTransport `
  -RemoteIPRanges 192.0.2.10 -Bindings 0.0.0.0:25 -Usage Custom -PermissionGroups AnonymousUsers

Get-ReceiveConnector "EX01\Anonymous Relay" | Add-ADPermission `
  -User "NT AUTHORITY\ANONYMOUS LOGON" -ExtendedRights "ms-Exch-SMTP-Accept-Any-Recipient"
```

E-post via denna connector förblir `AuthAs: Anonymous`, genomgår normala kontroller och kan inte utge sig för att ha interna avsändare. `ExternalAuthoritative` bör reserveras för de system som du medvetet vill ge rätten att använda interna avsändaradresser.

## Läs rubriken i sitt sammanhang

Huruvida ett specifikt meddelande har klassificerats som internt eller externt och vilken väg det tog framgår snabbast av den fullständiga rubriken: `AuthAs`, `AuthMechanism` och `AuthSource` tillsammans med `Received`-kedjan. [Mail Header Analyzer](/tools/header-analyzer) på denna webbplats analyserar dessa fält direkt i webbläsaren och markerar hybridklassificeringen i leveransvägen; rubriken lämnar inte webbläsaren.

Hur klassificeringen bevaras mellan Exchange Online och OnPrem i hybridmiljöer och hur en felaktig tilldelning kan identifieras behandlas i artikeln [Intern eller extern? Klassificera Exchange-hybridmeddelanden i rubriken](/blog/exchange-hybrid-header-intern-extern).

## Källor

1.  [Microsoft Q&A: Exchange 2016 mail flow rule, which header is checked for "outside the organization"?](https://learn.microsoft.com/en-us/answers/questions/54418/exchange-2016-mail-flow-rule-which-header-is-check): Koppling av AuthAs och AuthMechanism 10 till Externally Secured-konfigurationen samt dess effekt på e-postflödesregler.

2.  [Microsoft Tech Community: Demystifying and troubleshooting hybrid mail flow: when is a message internal?](https://techcommunity.microsoft.com/blog/exchange/demystifying-and-troubleshooting-hybrid-mail-flow-when-is-a-message-internal/1420838): Officiell beskrivning av Internal-klassificeringen och dess konsekvenser i hybrid-e-postflödet.

3.  [msxfaq: X-MS-Exchange-Organization-AuthAs](https://www.msxfaq.de/cloud/exchangeonline/transport/x-ms-exchange-organization-authas.htm): Observerade AuthAs-, AuthSource- och AuthMechanism-värden i olika leveransscenarier.

4.  [Microsoft Learn: Allow anonymous relay on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/allow-anonymous-relay): Konfiguration av den anonyma relay-connectorn som alternativ till Externally Secured.
