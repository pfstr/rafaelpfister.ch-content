---
title: "Vanliga orsaker till e-postloopar och hur de åtgärdas"
navTitle: "Åtgärda e-postloopar"
description: "Så här kan SMTP-e-postloopar i Exchange Online, hybridmiljöer och framförliggande e-postgateways systematiskt identifieras och åtgärdas med hjälp av NDR, rubriker, Message Trace, mottagarobjekt och anslutningar."
date: "2026-08-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "12 min lästid"
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
translationSourceHash: 6bd8a9a875aa7795711db421084241c47d7b2c1f24abb9ba64bc1b18128d86b7
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:38:40.710Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/vanliga-orsaker-till-e-postloopar-och-hur-de-atgardas
---

# Vanliga orsaker till e-postloopar och hur de åtgärdas

En e-postloop uppstår när minst två transportsystem gång på gång överlämnar samma meddelande till varandra. Inget av systemen identifierar sig som det slutliga målet, men båda känner till ett förmodat lämpligt nästa hopp. Loopen upphör först när en server konstaterar att det tillåtna antalet transportstationer har överskridits och skapar en NDR.

I Exchange är två meddelanden särskilt informativa:

- `554 5.4.6 Hop count exceeded - possible mail loop` skapas vanligtvis av den lokala Exchange-servern.
- `554 5.4.14 Hop count exceeded - possible mail loop ATTR34` skapas av Exchange Online.

Hoppgränsen är inte orsaken utan skyddet mot en oändlig upprepning. Att höja den löser därför ingenting. Målet är att hitta den punkt där meddelandet, i strid med målarkitekturen, återlämnas till ett system som redan har passerats.

## Identifiera loopmönstret i rubriken

NDR:en och de fullständiga ursprungliga meddelandehuvudena bör sparas innan några ändringar görs. `Received`-rader läses nedifrån och upp: den nedersta raden är det tidigast dokumenterade hoppet, den översta det senaste.

En loop visar sig oftast som en återkommande sekvens:

```text
Internet
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → ...
```

Inte varje Microsoft-värdnamn som förekommer flera gånger är redan en loop. Exchange Online behandlar meddelanden internt via flera transportroller. Det som är anmärkningsvärt är den upprepade återkomsten mellan samma administrativa gränser, exempelvis mellan Exchange Online och en lokal gateway. Tidsstämplar, sändande IP-adress, mottagande värd och `Message-ID` hjälper till att entydigt identifiera varvet.

För den första analysen besvaras följande frågor:

1. Vilket system skapade NDR:en?
2. Vilka två eller tre hopp upprepas?
3. Vilket system borde ha levererat meddelandet slutgiltigt?
4. På grund av vilket domän-, mottagar-, anslutnings- eller regelbeslut vidarebefordrades det?
5. Vilken ändring har senast påverkat e-postflödet?

## Diagnostik i Exchange Online

Med `Get-MessageTraceV2` kan behandlingen under de senaste 90 dagarna undersökas; högst tio dagar är tillåtna per fråga. Ett snävt tidsfönster och den konkreta mottagaradressen ger de mest användbara resultaten:

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
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-RecipientAddress` | Filtrerar spårningen till den angivna mottagaradressen |
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
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-MessageTraceId` | Entydigt spårnings-ID från resultatet av `Get-MessageTraceV2` |
| `-RecipientAddress` | Mottagaradress för träffen; krävs tillsammans med spårnings-ID:t för detaljfrågan |
| `Format-Table … -AutoSize` | Anpassar kolumnbredderna till innehållet så att händelsedetaljerna förblir läsbara |

</details>

Därefter granskas domän, mottagare och anslutningar tillsammans:

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
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity $recipient` | Väljer mottagarobjektet via adress, alias eller namn |
| `-IncludeTestModeConnectors` | Tar även med anslutningar i testläge i utdata |
| `Format-Table … -AutoSize` | Tabellvy med kolumnbredder anpassade efter innehållet |
| `Format-List …` | Listvy över de angivna egenskaperna, lämplig för långa värden som adresslistor |

</details>

Det avgörande är inte om ett enskilt objekt ser rimligt ut. Domäntyp, faktisk mottagartyp och tillämplig anslutning måste tillsammans beskriva samma måldestination.

## Diagnostik i lokala Exchange

I en hybridmiljö kontrolleras samma mottagare även lokalt. Frågorna skiljer mellan en verklig lokal brevlåda, en RemoteMailbox och en MailUser:

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
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity $recipient` | Väljer objektet via adress, alias eller namn |
| `-ErrorAction SilentlyContinue` | Undertrycker felmeddelandet om objektet inte finns i respektive typ; frågan ger då helt enkelt inget resultat |

</details>

För transportvägen behövs sänd- och mottagningsanslutningar samt spårningsloggar:

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
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `Where-Object { … }` | Begränsar serverlistan till Mailbox- och Hub Transport-servrar, alltså rollerna med spårningsloggar |
| `-Start` / `-End` | Tidsfönster för logsökningen |
| `-Recipients $recipient` | Filtrerar till spårningshändelser med denna mottagaradress |
| `-ResultSize Unlimited` | Tar bort standardgränsen på 1 000 returnerade poster |
| `Select-Object …` | Begränsar utdata till de fält som är relevanta för väganalysen |
| `Sort-Object Timestamp` | Sorterar händelserna från alla servrar kronologiskt |

</details>

Ett `SEND` till Exchange Online, följt av ett nytt `RECEIVE` av samma meddelande från Exchange Online, synliggör återlämningen. Med `MessageId` och `NetworkMessageId` kan man undvika att förväxla olika testmeddelanden med varandra.

## De vanligaste orsakerna i korthet

| Mönster | Typisk orsak | Åtgärd |
| --- | --- | --- |
| Okända mottagare pendlar mellan två system | Accepted Domain är inställd på `InternalRelay`, men båda sidor vidarebefordrar okända mottagare | Definiera ett tydligt ansvar; använd `Authoritative` vid fullständig leverans i EXO eller fastställ ett enda slutligt hopp för en split domain |
| EXO skickar till lokal Exchange, som omedelbart skickar tillbaka till EXO | Hybrid-anslutning eller Centralized Mail Transport stämmer inte längre överens med brevlådeplatsen | Kontrollera HCW-konfigurationen och `RouteAllMessagesViaOnPremises`; inaktivera föråldrad central routning eller korrigera lokal mottagarupplösning |
| Meddelandet pendlar mellan EXO och en gateway för säkerhet, signering eller kryptering | Återkommande meddelanden uppfyller utgående regel på nytt | Använd en header som gatewayen sätter eller en dokumenterad mekanism för loop prevention som undantag; autentisera in- och utgående anslutningar entydigt |
| Endast en mottagare påverkas | Föråldrat eller felaktigt `targetAddress`, fel RemoteMailbox-typ eller motstridiga proxyadresser | Fastställ Source of Authority, korrigera mottagarobjektet där och synkronisera |
| Endast vidarebefordrade meddelanden loopar | Transportregel, vidarebefordran av brevlåda eller inkorgsregel adresserar den ursprungliga vägen på nytt | Inaktivera regeln, korrigera målet och definiera ett robust undantag |
| Endast en underdomän eller applikation påverkas | Överordnad domän täcker inte underdomänen korrekt i den förväntade anslutningsvägen | Konfigurera underdomänen uttryckligen som Accepted Domain och i rätt Send Connector |
| Alla meddelanden loopar efter en gateway- eller DNS-ändring | Smart Host eller MX pekar på det sändande systemets ingång | Korrigera nästa hopp och kontrollera DNS-, NAT- och load-balancer-mål separat |

## Orsak 1: Fel typ av Accepted Domain

En authoritative domain innebär att alla giltiga mottagare i denna domän är kända i Exchange-organisationen; okända mottagare avvisas. En Internal Relay-domän innebär att en del av mottagarna finns i ett annat system och måste vidarebefordras via en Send Connector eller Outbound Connector.

Den problematiska konstellationen uppstår när Exchange Online skickar okända mottagare till ett lokalt system och detta system inte heller behandlar samma domän slutgiltigt, utan skickar tillbaka den till Exchange Online via MX eller Smart Host.

```powershell
Get-AcceptedDomain -Identity contoso.com |
    Format-List DomainName,DomainType,MatchSubDomains
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity contoso.com` | Väljer den Accepted Domain som ska kontrolleras |
| `Format-List …` | Visar domännamn, domäntyp och underdomäntäckning som lista |

</details>

Om alla mottagare finns i Exchange Online efter slutförd migrering är `Authoritative` oftast rätt måltillstånd:

```powershell
# Kör endast efter fullständig kontroll av mottagare och routning.
Set-AcceptedDomain -Identity contoso.com -DomainType Authoritative
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity contoso.com` | Den Accepted Domain som ska ändras |
| `-DomainType Authoritative` | Ställer in domänen som authoritative: okända mottagare avvisas i stället för att vidarebefordras |

</details>

I en verklig split domain kan `InternalRelay` vara korrekt. Då krävs dock en tydlig anslutning till systemet som känner till de återstående mottagarna. Detta mål får inte skicka tillbaka okända adresser till utgångspunkten.

## Orsak 2: Överlappande hybridanslutningar och Centralized Mail Transport

Centralized Mail Transport leder medvetet utgående Exchange Online-meddelanden via lokal Exchange. Detta är meningsfullt för vissa efterlevnadskrav, men skapar ytterligare transportvägar. Om alternativet förblir aktivt efter en migrering, trots att det lokala systemet skickar meddelanden tillbaka till Exchange Online via sin egen MX, kan en cirkel uppstå.

```powershell
Get-OutboundConnector -IncludeTestModeConnectors |
    Format-Table Name,Enabled,ConnectorType,RouteAllMessagesViaOnPremises,
        RecipientDomains,SmartHosts,UseMXRecord -AutoSize
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-IncludeTestModeConnectors` | Tar även med anslutningar i testläge i utdata |
| `Format-Table … -AutoSize` | Tabellvy över routningsegenskaper med kolumnbredder anpassade efter innehållet |

</details>

Flera anslutningar med överlappande omfattning bör också kontrolleras. Microsoft rekommenderar en dedikerad lokal anslutning för hybrid-e-postflöde; en reparation med Hybrid Configuration Wizard är ofta säkrare än isolerade enskilda ändringar.

Om Centralized Mail Transport bevisligen inte längre behövs kan inställningen inaktiveras specifikt:

```powershell
# Endast efter kontroll av efterlevnads- och gatewaykraven.
Set-OutboundConnector `
    -Identity "Outbound to On-Premises" `
    -RouteAllMessagesViaOnPremises:$false
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity "Outbound to On-Premises"` | Outbound Connector som ska ändras |
| `-RouteAllMessagesViaOnPremises:$false` | Inaktiverar Centralized Mail Transport: utgående meddelanden från Exchange Online går inte längre via lokal Exchange |

</details>

## Orsak 3: En gateway behandlar sina returer på nytt

I ett in-and-out-scenario skickar Exchange Online ett meddelande till en tilläggstjänst för signering, kryptering eller arkivering. Tjänsten skickar sedan tillbaka det till Exchange Online. Den utgående regeln måste känna igen returen; annars skickas den åter till tjänsten.

Kontrollen börjar med alla regler som väljer anslutningar, omdirigerar mottagare eller utvärderar headers:

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Format-List Name,State,Mode,Priority,FromScope,SentToScope,
        RedirectMessageTo,RouteMessageOutboundConnector,
        SetHeaderName,SetHeaderValue,ExceptIfHeaderContainsMessageHeader,
        ExceptIfHeaderContainsWords
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `Sort-Object Priority` | Sorterar reglerna i deras utvärderingsordning |
| `Format-List …` | Visar de egenskaper som väljer anslutningar, omdirigerar mottagare eller sätter headers respektive utvärderar dem som undantag |

</details>

Det konkreta undantaget måste följa dokumentationen från gateway-tillverkaren. Vanligt är en header som sätts av tjänsten och som inte på ett tillförlitligt sätt kan förfalskas från internet. Dessutom bör inkommande anslutningar identifiera tjänsten via certifikat eller fast avsändar-IP. Ett generellt undantag för alla meddelanden som förefaller vara ”interna” är för brett.

## Orsak 4: Mottagarobjektet och den faktiska brevlådan finns inte på samma plats

Ett objekt kan visas i Exchange Online som `MailUser` trots att den aktiva brevlådan finns lokalt. I en synkroniserad hybridmiljö är detta inte automatiskt ett dubblettobjekt. Inte heller bevisar en `ExternalEmailAddress`, som motsvarar den primära SMTP-adressen, i sig själv en felkonfiguration.

Avgörande är kombinationen av alla frågor:

- `Get-Mailbox` lokalt ger ett resultat: Den aktiva brevlådan finns lokalt.
- `Get-RemoteMailbox` lokalt ger ett resultat: Det hanterade målet finns i Exchange Online.
- `Get-EXOMailbox` ger ett resultat: Det finns en verklig brevlåda i molnet.
- `Get-EXORecipient` ger endast en MailUser: Objektet är ett routningsmål, inte en molnbrevlåda.

Problematiska är föråldrade objekt efter en migrering, felaktiga fjärrroutningsdomäner eller manuellt angivna `targetAddress`-värden vars domän leder tillbaka via samma transportväg. Ändringar görs hos Source of Authority: i synkroniserade miljöer alltså med Exchange-hanteringsverktyg lokalt och inte genom direkt redigering av enskilda attribut i Exchange Online.

## Orsak 5: Vidarebefordringar och transportregler bildar en cirkel

En regel kan omdirigera från adress A till B, medan B skickar tillbaka till A via en andra regel, vidarebefordran av brevlåda eller ett externt system. Sådana loopar berör ofta endast enskilda mottagare eller meddelandetyper.

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
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `Sort-Object Priority` | Sorterar transportreglerna i deras utvärderingsordning |
| `-ResultSize Unlimited` | Tar bort standardgränsen på 1 000 returnerade brevlådor |
| `-Mailbox user01@contoso.com` | Brevlådan vars inkorgsregler efterfrågas |
| `Select-Object …` | Begränsar utdata till vidarebefordrings- och omdirigeringsmålen |

</details>

Åtgärden består inte bara i att tillfälligt stänga av en regel. Hela kedjan måste lösas upp, och regler för externa tjänster behöver ett undantag som på ett säkert sätt identifierar redan behandlade meddelanden.

## Orsak 6: MX, Smart Host eller underdomän pekar tillbaka

En gateway kan behöva ett annat nästa hopp internt än externa avsändare. Om den helt enkelt använder den offentliga MX-posten för vidarebefordran kan denna i vissa fall peka tillbaka på själva gatewayen. Samma problem uppstår om en Smart Host genom NAT eller lastbalansering leder tillbaka till sin egen lyssnare.

```powershell
Resolve-DnsName -Type MX contoso.com
Resolve-DnsName -Type MX app.contoso.com

Get-SendConnector |
    Format-List Name,AddressSpaces,DNSRoutingEnabled,SmartHosts
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Type MX` | Frågar efter MX-poster i stället för standard-A-poster |
| `contoso.com` / `app.contoso.com` | Domän som ska frågas efter som positionsargument (parametern `-Name`) |
| `Format-List …` | Visar adressutrymmen, routningsläge och Smart Hosts per Send Connector |

</details>

Underdomäner förtjänar en egen kontroll. Microsoft har i en numera tillbakadragen supportartikel dokumenterat fall där en applikationsunderdomän uttryckligen måste skapas som Internal Relay-domän och synkroniseras till Edge-systemen:

```powershell
New-AcceptedDomain `
    -Name "app.contoso.com" `
    -DomainName app.contoso.com `
    -DomainType InternalRelay

Start-EdgeSynchronization
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Name "app.contoso.com"` | Visningsnamn för det nya Accepted Domain-objektet |
| `-DomainName app.contoso.com` | SMTP-domänen som Exchange accepterar meddelanden för |
| `-DomainType InternalRelay` | En del av mottagarna finns utanför organisationen; okända mottagare vidarebefordras via en Send Connector i stället för att avvisas |

</details>

Dessa kommandon är ingen universell lösning. De passar endast om `app.contoso.com` faktiskt levereras utanför Exchange-organisationen och Send Connector har ett entydigt nästa hopp.

## Säkert tillvägagångssätt vid en aktiv loop

Under störningen bör multipliceringen först stoppas. Beroende på arkitekturen inaktiveras den utlösande transportregeln eller den specifika anslutningen kontrollerat, eller så håller gatewayen tillbaka den berörda kön. Konfiguration och meddelandeexempel exporteras dessförinnan.

Därefter följer ett test med exakt en avsändare, en mottagare och en tydligt igenkännbar ämnesrad. Meddelandet följs utan luckor via headers, Message Trace och lokala spårningsloggar. Först när det avslutas vid det avsedda målet öppnas e-postflödet stegvis igen.

Följande rekommenderas inte:

- höja hoppgränser
- ändra flera anslutningar samtidigt
- växla Accepted Domains på måfå mellan `Authoritative` och `InternalRelay`
- återupprepat mata in en problematisk kö utan kontroll
- korrigera synkroniserade Exchange-attribut direkt i AD eller Exchange Online
- inaktivera TLS-, IP- eller certifikatkontroller som en förmodad snabb lösning

## Slutkontroll

Efter korrigeringen bör dokumentationen för varje relevant domän innehålla exakt ett uttalande: Vilket system känner till mottagaren, vilken anslutning är tillämplig och vilken värd är det slutliga nästa hoppet?

Det tekniska godkännandet omfattar minst:

- externt och internt testmeddelande
- okänd mottagare i samma domän
- mottagare på vardera sidan av en verklig split domain
- utgående meddelande med aktiv gateway eller Centralized Mail Transport
- headers utan återkommande hoppsekvens
- Message Trace med `Delivered` respektive förväntad överlämning
- lokal spårning utan nytt `RECEIVE` efter ett `SEND` till samma mål
- anslutningsvalidering för alla anslutningar som fortfarande behövs

En åtgärdad e-postloop är först avslutad när inte bara testmeddelandet kommer fram, utan även okända mottagare och alternativa e-postflödesvägar avslutas på ett definierat sätt. Det är just där de flesta återfallen sker.

## Ändringshistorik

- **25 oktober 2026:** Microsoft har dragit tillbaka supportartikeln om scenariot för underdomän och EdgeSync; källan hänvisar nu till den arkiverade versionen från 19 april 2026.

## Källor

1. [Microsoft Learn – Fix NDR error 5.4.6 or 5.4.14 in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/fix-error-code-5-4-6-through-5-4-20-in-exchange-online): Betydelsen av Exchange-NDR:er och typiska orsaker i Accepted Domains och hybridanslutningar.

2. [Microsoft Learn – Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains): Skillnader mellan `Authoritative` och `InternalRelay`.

3. [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains): Ansvar, relädomäner och Recipient Lookup i lokala Exchange.

4. [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing): Förväntade transportvägar med och utan Centralized Mail Transport.

5. [Microsoft Learn – Validate connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/validate-connectors): Anslutningsvalidering och information om flera anslutningar som matchar samtidigt.

6. [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): E-postflödesmönster som stöds med framförliggande tredjepartstjänster.

7. [Microsoft Learn – Mail flow rules in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/mail-flow-rules): Behandling, prioritet, åtgärder och undantag för transportregler.

8. [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): Sökning efter meddelanden i Exchange Online-transporten.

9. [Microsoft Learn – Search message tracking logs](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/search-message-tracking-logs): Lokal meddelandespårning över alla Exchange-servrar.

10. [Microsoft Learn – Hop count exceeded for an on-premises application subdomain (arkiverad version från 19 april 2026)](https://web.archive.org/web/20260419203247/https://learn.microsoft.com/en-us/troubleshoot/exchange/mailflow/hop-count-exceeded-possible-mail-loop): Dokumenterat scenario för underdomän/EdgeSync med uttrycklig Internal Relay-domän; Microsoft har sedan dess dragit tillbaka artikeln.
