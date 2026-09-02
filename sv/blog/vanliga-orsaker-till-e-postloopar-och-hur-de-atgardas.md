---
title: "Vanliga orsaker till e-postloopar och hur de åtgärdas"
navTitle: "Åtgärda e-postloopar"
description: "Så här kan SMTP-e-postloopar i Exchange Online, hybridmiljöer och framförliggande e-postgatewayer systematiskt identifieras och åtgärdas med hjälp av NDR, headers, Message Trace, mottagarobjekt och anslutningar."
date: "2026-08-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "12 min. läsning"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
related:
  - exchange-mailboxen-ohne-remote-move-cloud-neu-erstellen
  - totemomail-m365
  - ghost-sender-exchange-online-nebeneingang
slug: "vanliga-orsaker-till-e-postloopar-och-hur-de-atgardas"
translationId: "article-4c91e7b2a8605fd3"
draft: false
translationOf: typische-ursachen-fuer-mail-loops-und-deren-behebung
translationSourceHash: c71063cb6e7d05a1f311a5269e4d6805d8b219e8d0fb103485738925ef99f990
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T09:03:57.687Z
translationReview: required
url: https://rafaelpfister.ch/sv/blog/vanliga-orsaker-till-e-postloopar-och-hur-de-atgardas
---

# Vanliga orsaker till e-postloopar och hur de åtgärdas

En e-postloop uppstår när minst två transportsystem upprepade gånger överlämnar samma meddelande till varandra. Inget av systemen identifierar sig som det slutliga målet, men båda känner till ett förmodat lämpligt nästa hopp. Loopen avslutas först när en server konstaterar att det tillåtna antalet transporthopp har överskridits och skapar en NDR.

I Exchange är två meddelanden särskilt informativa:

- `554 5.4.6 Hop count exceeded - possible mail loop` skapas vanligtvis av den lokala Exchange-servern.
- `554 5.4.14 Hop count exceeded - possible mail loop ATTR34` skapas av Exchange Online.

Hoppgränsen är inte orsaken, utan skyddet mot en oändlig upprepning. Att höja den åtgärdar därför ingenting. Det som måste hittas är punkten där meddelandet, i strid med målarkitekturen, returneras till ett system som redan har passerats.

## Identifiera loopmönstret i headern

NDR:en och de fullständiga ursprungliga meddelandehuvudena bör sparas innan några ändringar görs. `Received`-rader läses nedifrån och upp: den nedersta raden är det tidigast dokumenterade hoppet, den översta är det senaste.

En loop visar sig vanligtvis som en återkommande sekvens:

```text
Internet
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → ...
```

Alla Microsoft-värdnamn som förekommer flera gånger innebär inte redan en loop. Exchange Online behandlar meddelanden internt via flera transportroller. Det som är iögonfallande är den upprepade återkomsten mellan samma administrativa gränser, exempelvis mellan Exchange Online och en lokal gateway. Tidsstämplar, sändande IP-adress, mottagande värd och `Message-ID` hjälper till att tydligt identifiera varvet.

För den första analysen besvaras följande frågor:

1. Vilket system skapade NDR:en?
2. Vilka två eller tre hopp upprepas?
3. Vilket system skulle ha levererat meddelandet slutgiltigt?
4. På grundval av vilket domän-, mottagar-, connector- eller regelbeslut vidarebefordrades det?
5. Vilken ändring har senast påverkat e-postflödet?

## Diagnostik i Exchange Online

Med `Get-MessageTraceV2` kan bearbetningen under de senaste 90 dagarna undersökas; högst tio dagar är tillåtna per fråga. Ett snävt tidsfönster och den specifika mottagaradressen ger de mest användbara resultaten:

```powershell
$start = (Get-Date).AddHours(-2)
$end = Get-Date
$recipient = "user01@contoso.com"

$trace = Get-MessageTraceV2 `
    -RecipientAddress $recipient `
    -StartDate $start `
    -EndDate $end `
    -ResultSize 5000

$trace |
    Select-Object Received,SenderAddress,RecipientAddress,Subject,
        Status,FromIP,ToIP,MessageTraceId,MessageId |
    Sort-Object Received
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-RecipientAddress` | Filtrerar spårningen på den angivna mottagaradressen |
| `-StartDate` / `-EndDate` | Tidsfönster för frågan; högst tio dagar är tillåtna per fråga |
| `-ResultSize 5000` | Maximalt antal returnerade poster |
| `Select-Object …` | Begränsar utdata till de fält som är relevanta för loopanalysen |
| `Sort-Object Received` | Sorterar träffarna kronologiskt efter mottagningstidpunkt |

</details>

Detaljerna för en träff visar enskilda transporthändelser:

```powershell
$trace | ForEach-Object {
    Get-MessageTraceDetailV2 `
        -MessageTraceId $_.MessageTraceId `
        -RecipientAddress $_.RecipientAddress
} | Format-Table Date,Event,Action,Detail -AutoSize
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-MessageTraceId` | Unikt spårnings-ID från resultatet av `Get-MessageTraceV2` |
| `-RecipientAddress` | Mottagaradress för träffen; krävs tillsammans med spårnings-ID:t för detaljfrågan |
| `Format-Table … -AutoSize` | Anpassar kolumnbredderna till innehållet så att händelsedetaljerna förblir läsbara |

</details>

Därefter granskas domän, mottagare och connectors tillsammans:

```powershell
Get-AcceptedDomain |
    Format-Table Name,DomainName,DomainType,MatchSubDomains -AutoSize

Get-EXORecipient -Identity $recipient |
    Format-List DisplayName,RecipientTypeDetails,PrimarySmtpAddress,
        ExternalEmailAddress,EmailAddresses

Get-OutboundConnector -IncludeTestModeConnectors |
    Format-List Name,Enabled,ConnectorType,RecipientDomains,SmartHosts,
        UseMXRecord,RouteAllMessagesViaOnPremises,TlsSettings

Get-InboundConnector |
    Format-List Name,Enabled,ConnectorType,SenderDomains,SenderIPAddresses,
        TlsSenderCertificateName,RequireTls,RestrictDomainsToIPAddresses,
        RestrictDomainsToCertificate
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity $recipient` | Väljer mottagarobjektet via adress, alias eller namn |
| `-IncludeTestModeConnectors` | Tar även med connectors i testläge i utdata |
| `Format-Table … -AutoSize` | Tabellvy med kolumnbredder baserade på innehållet |
| `Format-List …` | Listvy över angivna egenskaper, lämplig för långa värden som adresslistor |

</details>

Det avgörande är inte om ett enskilt objekt ser rimligt ut. Domäntyp, faktisk mottagartyp och tillämplig connector måste tillsammans beskriva samma målplats.

## Diagnostik i lokal Exchange

I en hybridmiljö kontrolleras samma mottagare även lokalt. Frågorna skiljer mellan en verklig lokal postlåda, en RemoteMailbox och en MailUser:

```powershell
Get-Recipient -Identity $recipient |
    Format-List DisplayName,RecipientType,RecipientTypeDetails,
        PrimarySmtpAddress,EmailAddresses

Get-Mailbox -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,ServerName,Database,PrimarySmtpAddress

Get-RemoteMailbox -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress

Get-MailUser -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExternalEmailAddress
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity $recipient` | Väljer objektet via adress, alias eller namn |
| `-ErrorAction SilentlyContinue` | Undertrycker felmeddelandet om objektet inte finns i respektive typ; frågan returnerar då helt enkelt inget resultat |

</details>

För transportvägen krävs Send- och Receive-connectors samt trackingloggar:

```powershell
Get-SendConnector |
    Format-List Name,Enabled,AddressSpaces,DNSRoutingEnabled,SmartHosts,
        SourceTransportServers,CloudServicesMailEnabled,TlsDomain

Get-ReceiveConnector |
    Format-List Identity,Enabled,Bindings,RemoteIPRanges,PermissionGroups

$servers = Get-ExchangeServer |
    Where-Object { $_.IsMailboxServer -or $_.IsHubTransportServer }

$servers |
    Get-MessageTrackingLog `
        -Start $start `
        -End $end `
        -Recipients $recipient `
        -ResultSize Unlimited |
    Select-Object Timestamp,ServerHostname,ClientHostname,Source,EventId,
        ConnectorId,Sender,Recipients,MessageId,NetworkMessageId |
    Sort-Object Timestamp
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Where-Object { … }` | Begränsar serverlistan till Mailbox- och Hub Transport-servrar, alltså rollerna med trackingloggar |
| `-Start` / `-End` | Tidsfönster för logsökningen |
| `-Recipients $recipient` | Filtrerar på trackinghändelser med denna mottagaradress |
| `-ResultSize Unlimited` | Tar bort standardgränsen på 1 000 returnerade poster |
| `Select-Object …` | Begränsar utdata till de fält som är relevanta för sökvägsanalysen |
| `Sort-Object Timestamp` | Sorterar händelserna från alla servrar kronologiskt |

</details>

Ett `SEND` till Exchange Online, följt av ett nytt `RECEIVE` för samma meddelande från Exchange Online, synliggör returen. Med `MessageId` och `NetworkMessageId` undviker du att förväxla olika testmeddelanden med varandra.

## De vanligaste orsakerna i korthet

| Mönster | Typisk orsak | Åtgärd |
| --- | --- | --- |
| Okända mottagare pendlar mellan två system | Accepted Domain är inställd på `InternalRelay`, men båda sidor vidarebefordrar okända mottagare | Definiera ett tydligt ansvar; använd `Authoritative` för fullständig EXO-leverans eller fastställ ett enda avslutande hopp för split domain |
| EXO skickar till lokal Exchange, som genast skickar tillbaka till EXO | Hybrid-connector eller Centralized Mail Transport stämmer inte längre överens med postlådeplatsen | Kontrollera HCW-konfigurationen och `RouteAllMessagesViaOnPremises`; inaktivera föråldrad central routing eller korrigera lokal mottagarupplösning |
| Meddelandet pendlar mellan EXO och en gateway för säkerhet, signatur eller kryptering | Returnerade meddelanden uppfyller utgående regel igen | Använd en header som gatewayen har angett eller en dokumenterad loopprevention-mekanism som undantag; autentisera in- och utgående connectors entydigt |
| Endast en mottagare påverkas | Föråldrat eller felaktigt `targetAddress`, fel RemoteMailbox-typ eller motstridiga proxyadresser | Fastställ Source of Authority, korrigera mottagarobjektet där och synkronisera |
| Endast vidarebefordrade meddelanden loopar | Transportregel, postlådevidarebefordran eller Inbox-regel adresserar den ursprungliga vägen på nytt | Inaktivera regeln, korrigera målet och definiera ett robust undantag |
| Endast en underdomän eller applikation påverkas | Överordnad domän täcker inte underdomänen korrekt i den förväntade connectorvägen | Konfigurera underdomänen uttryckligen som Accepted Domain och i lämplig Send Connector |
| Alla meddelanden loopar efter en gateway- eller DNS-ändring | Smart Host eller MX pekar mot ingången på det sändande systemet | Korrigera nästa hopp och kontrollera DNS-, NAT- och lastbalanseringsmål separat |

## Orsak 1: Fel typ för Accepted Domain

En authoritative domain innebär: Alla giltiga mottagare för denna domän är kända i Exchange-organisationen; okända mottagare avvisas. En Internal Relay-domän innebär: En del av mottagarna finns i ett annat system och måste vidarebefordras via en Send- eller Outbound-connector.

Den problematiska konfigurationen uppstår när Exchange Online skickar okända mottagare till ett lokalt system och detta system inte heller behandlar samma domän slutgiltigt, utan skickar tillbaka den till Exchange Online via MX eller Smart Host.

```powershell
Get-AcceptedDomain -Identity contoso.com |
    Format-List DomainName,DomainType,MatchSubDomains
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity contoso.com` | Väljer den Accepted Domain som ska kontrolleras |
| `Format-List …` | Visar domännamn, domäntyp och underdomänstäckning som en lista |

</details>

Om alla mottagare finns i Exchange Online efter en slutförd migrering är `Authoritative` vanligtvis önskat slutläge:

```powershell
# Kör först efter fullständig kontroll av mottagare och routing.
Set-AcceptedDomain -Identity contoso.com -DomainType Authoritative
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity contoso.com` | Den Accepted Domain som ska ändras |
| `-DomainType Authoritative` | Ställer in domänen som authoritative: okända mottagare avvisas i stället för att vidarebefordras |

</details>

För en verklig split domain kan `InternalRelay` vara korrekt. Då krävs dock en tydlig connector till systemet som känner till de återstående mottagarna. Detta mål får inte skicka tillbaka okända adresser till utgångspunkten.

## Orsak 2: Överlappande hybrid-connectors och Centralized Mail Transport

Centralized Mail Transport dirigerar medvetet utgående Exchange Online-meddelanden via den lokala Exchange-miljön. Det är meningsfullt för vissa compliancekrav, men skapar ytterligare transportvägar. Om alternativet förblir aktivt efter en migrering, trots att det lokala systemet skickar meddelanden tillbaka till Exchange Online via sin egen MX, kan en cirkel uppstå.

```powershell
Get-OutboundConnector -IncludeTestModeConnectors |
    Format-Table Name,Enabled,ConnectorType,RouteAllMessagesViaOnPremises,
        RecipientDomains,SmartHosts,UseMXRecord -AutoSize
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-IncludeTestModeConnectors` | Tar även med connectors i testläge i utdata |
| `Format-Table … -AutoSize` | Tabellvy över routingegenskaper med kolumnbredder baserade på innehållet |

</details>

Flera connectors med överlappande omfattning bör också kontrolleras. Microsoft rekommenderar en dedikerad lokal connector för hybrid-e-postflöde; en reparation med Hybrid Configuration Wizard är ofta säkrare än enskilda, isolerade ändringar.

Om Centralized Mail Transport bevisligen inte längre behövs kan inställningen inaktiveras specifikt:

```powershell
# Endast efter kontroll av compliance- och gatewaykrav.
Set-OutboundConnector `
    -Identity "Outbound to On-Premises" `
    -RouteAllMessagesViaOnPremises:$false
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity "Outbound to On-Premises"` | Den Outbound Connector som ska ändras |
| `-RouteAllMessagesViaOnPremises:$false` | Inaktiverar Centralized Mail Transport: utgående meddelanden från Exchange Online går inte längre via lokal Exchange |

</details>

## Orsak 3: En gateway behandlar sina returer på nytt

I ett in-and-out-scenario skickar Exchange Online ett meddelande till en tilläggstjänst för signering, kryptering eller arkivering. Tjänsten skickar sedan tillbaka det till Exchange Online. Den utgående regeln måste känna igen returen, annars skickas den åter till tjänsten.

Kontrollen börjar med alla regler som väljer connectors, omdirigerar mottagare eller utvärderar headers:

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Format-List Name,State,Mode,Priority,FromScope,SentToScope,
        RedirectMessageTo,RouteMessageOutboundConnector,
        SetHeaderName,SetHeaderValue,ExceptIfHeaderContainsMessageHeader,
        ExceptIfHeaderContainsWords
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Sort-Object Priority` | Sorterar reglerna i deras utvärderingsordning |
| `Format-List …` | Visar egenskaperna som väljer connectors, omdirigerar mottagare eller anger headers respektive utvärderar dem som undantag |

</details>

Det specifika undantaget måste följa gatewaytillverkarens dokumentation. Vanligt är en header som anges av tjänsten och som inte kan förfalskas på ett tillförlitligt sätt från internet. Dessutom bör inbound-connectors identifiera tjänsten med certifikat eller fast avsändar-IP. Ett generellt undantag för alla meddelanden som verkar vara ”interna” är för brett.

## Orsak 4: Mottagarobjektet och den faktiska postlådan finns inte på samma plats

Ett objekt kan visas i Exchange Online som `MailUser`, trots att den aktiva postlådan finns lokalt. I en synkroniserad hybridmiljö är detta inte automatiskt ett dubblettobjekt. Inte heller en `ExternalEmailAddress`, som motsvarar den primära SMTP-adressen, bevisar i sig en felkonfiguration.

Avgörande är kombinationen av alla frågor:

- `Get-Mailbox` lokalt ger ett resultat: Den aktiva postlådan finns lokalt.
- `Get-RemoteMailbox` lokalt ger ett resultat: Det hanterade målet finns i Exchange Online.
- `Get-EXOMailbox` ger ett resultat: Det finns en verklig postlåda i molnet.
- `Get-EXORecipient` ger endast en MailUser: Objektet är ett routingmål, inte en molnpostlåda.

Problematiska är föråldrade objekt efter en migrering, felaktiga fjärrroutingdomäner eller manuellt angivna `targetAddress`-värden vars domän leder tillbaka via samma transportväg. Ändringar görs vid Source of Authority: i synkroniserade miljöer alltså med Exchange-hanteringsverktyg lokalt och inte genom direkt redigering av enskilda attribut i Exchange Online.

## Orsak 5: Vidarebefordringar och transportregler bildar en cirkel

En regel kan omdirigera från adress A till B, medan B skickar tillbaka till A via en andra regel, en postlådevidarebefordran eller ett externt system. Sådana loopar påverkar ofta bara enskilda mottagare eller meddelandetyper.

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Select-Object Name,State,Mode,Priority,RedirectMessageTo,
        BlindCopyTo,AddToRecipients,RouteMessageOutboundConnector

Get-Mailbox -ResultSize Unlimited |
    Select-Object DisplayName,PrimarySmtpAddress,
        ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward

Get-InboxRule -Mailbox user01@contoso.com |
    Select-Object Name,Enabled,Priority,ForwardTo,RedirectTo,ForwardAsAttachmentTo
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Sort-Object Priority` | Sorterar transportreglerna i deras utvärderingsordning |
| `-ResultSize Unlimited` | Tar bort standardgränsen på 1 000 returnerade postlådor |
| `-Mailbox user01@contoso.com` | Postlåda vars Inbox-regler frågas efter |
| `Select-Object …` | Begränsar utdata till vidarebefordrings- och omdirigeringsmålen |

</details>

Åtgärden består inte bara i att tillfälligt stänga av en regel. Hela kedjan måste lösas upp, och regler för externa tjänster behöver ett undantag som säkert känner igen redan behandlade meddelanden.

## Orsak 6: MX, Smart Host eller underdomän pekar tillbaka

En gateway kan behöva ett annat nästa hopp internt än externa avsändare. Om den helt enkelt använder den offentliga MX-posten för vidarebefordran kan denna i vissa fall peka tillbaka på gatewayen själv. Samma problem uppstår om en Smart Host via NAT eller lastbalansering leder tillbaka till sin egen listener.

```powershell
Resolve-DnsName -Type MX contoso.com
Resolve-DnsName -Type MX app.contoso.com

Get-SendConnector |
    Format-List Name,AddressSpaces,DNSRoutingEnabled,SmartHosts
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Type MX` | Frågar efter MX-poster i stället för standard-A-poster |
| `contoso.com` / `app.contoso.com` | Domän som ska frågas efter som positionsargument (parametern `-Name`) |
| `Format-List …` | Visar adressutrymmen, routingläge och Smart Hosts för varje Send Connector |

</details>

Underdomäner förtjänar en egen kontroll. Microsoft dokumenterar fall där en applikationsunderdomän uttryckligen måste skapas som Internal Relay-domän och synkroniseras till Edge-systemen:

```powershell
New-AcceptedDomain `
    -Name "app.contoso.com" `
    -DomainName app.contoso.com `
    -DomainType InternalRelay

Start-EdgeSynchronization
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Name "app.contoso.com"` | Visningsnamn för det nya Accepted Domain-objektet |
| `-DomainName app.contoso.com` | SMTP-domänen som Exchange accepterar meddelanden för |
| `-DomainType InternalRelay` | En del av mottagarna finns utanför organisationen; okända mottagare vidarebefordras via en Send Connector i stället för att avvisas |

</details>

Dessa kommandon är ingen universell lösning. De passar bara om `app.contoso.com` faktiskt levereras utanför Exchange-organisationen och Send Connector har ett entydigt nästa hopp.

## Säkert tillvägagångssätt vid en aktiv loop

Under störningen bör först mångfaldigandet stoppas. Beroende på arkitekturen inaktiveras den utlösande transportregeln eller den specifika connectorn på ett kontrollerat sätt, eller så håller gatewayen tillbaka den berörda kön. Konfiguration och meddelandeexempel exporteras först.

Därefter följer ett test med exakt en avsändare, en mottagare och en tydligt igenkännbar ämnesrad. Meddelandet följs utan avbrott via headers, Message Trace och lokala trackingloggar. Först när det avslutas vid det avsedda målet öppnas e-postflödet stegvis igen.

Följande rekommenderas inte:

- höja hoppgränser
- ändra flera connectors samtidigt
- växla Accepted Domains på måfå mellan `Authoritative` och `InternalRelay`
- mata in en problematisk kö upprepade gånger utan kontroll
- korrigera synkroniserade Exchange-attribut direkt i AD eller Exchange Online
- stänga av TLS-, IP- eller certifikatkontroller som en förment snabb lösning

## Slutkontroll

Efter korrigeringen bör dokumentationen innehålla exakt ett påstående för varje relevant domän: Vilket system känner till mottagaren, vilken connector är tillämplig och vilken värd är det slutliga nästa hoppet?

Den tekniska acceptansen omfattar minst:

- extern och intern testmeddelande
- okänd mottagare i samma domän
- mottagare på vardera sidan av en verklig split domain
- utgående meddelande med aktiverad gateway eller Centralized Mail Transport
- headers utan återkommande hoppsekvens
- Message Trace med `Delivered` respektive förväntad överlämning
- lokal tracking utan nytt `RECEIVE` efter ett `SEND` till samma mål
- connectorvalidering för alla connectors som fortfarande behövs

En åtgärdad e-postloop är först avslutad när inte bara testmeddelandet kommer fram, utan även okända mottagare och alternativa e-postflödesvägar avslutas på definierat sätt. Det är just där de flesta återfallen sker.

## Källor

1. [Microsoft Learn – Fix NDR error 5.4.6 or 5.4.14 in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/fix-error-code-5-4-6-through-5-4-20-in-exchange-online): Betydelsen av Exchange-NDR:er och typiska orsaker i Accepted Domains och hybrid-connectors.

2. [Microsoft Learn – Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains): Skillnader mellan `Authoritative` och `InternalRelay`.

3. [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains): Ansvar, relay-domäner och Recipient Lookup i lokal Exchange.

4. [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing): Förväntade transportvägar med och utan Centralized Mail Transport.

5. [Microsoft Learn – Validate connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/validate-connectors): Connectorvalidering och information om flera samtidigt matchande connectors.

6. [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): E-postflödesmönster som stöds med framförliggande tredjepartstjänster.

7. [Microsoft Learn – Mail flow rules in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/mail-flow-rules): Bearbetning, prioritet, åtgärder och undantag för transportregler.

8. [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): Sökning efter meddelanden i Exchange Online-transporten.

9. [Microsoft Learn – Search message tracking logs](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/search-message-tracking-logs): Lokal meddelandespårning på alla Exchange-servrar.

10. [Microsoft Learn – Hop count exceeded for an on-premises application subdomain](https://learn.microsoft.com/en-us/troubleshoot/exchange/mailflow/hop-count-exceeded-possible-mail-loop): Dokumenterat scenario med underdomän/EdgeSync och uttrycklig Internal Relay-domän.
