---
title: "Vanliga orsaker till e-postloopar och hur de åtgärdas"
navTitle: "Åtgärda e-postloopar"
description: "Så här kan SMTP-e-postloopar i Exchange Online, hybridmiljöer och framförliggande e-postgateways systematiskt identifieras och åtgärdas med hjälp av NDR, headers, Message Trace, mottagarobjekt och connectors."
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
url: https://rafaelpfister.ch/sv/blog/vanliga-orsaker-till-e-postloopar-och-hur-de-atgardas
translationSourceHash: 5353684681217adafc789a3b28ec218fa927e18d801c82c437ae281e1e1017bd
translationModel: gpt-5.6-terra
translatedAt: 2026-08-08T14:01:11.596Z
translationReview: automatic
---

# Vanliga orsaker till e-postloopar och hur de åtgärdas

En e-postloop uppstår när minst två transportsystem gång på gång överlämnar samma meddelande till varandra. Inget av systemen känner igen sig självt som slutligt mål, men båda känner till ett till synes lämpligt nästa hopp. Loopen avslutas först när en server bedömer att det tillåtna antalet transportstationer har överskridits och skapar ett NDR.

I Exchange är två meddelanden särskilt informativa:

- `554 5.4.6 Hop count exceeded - possible mail loop` skapas vanligtvis av den lokala Exchange-servern.
- `554 5.4.14 Hop count exceeded - possible mail loop ATTR34` skapas av Exchange Online.

Hoppgränsen är inte orsaken utan skyddet mot en oändlig upprepning. Att höja den löser därför ingenting. Det som ska hittas är punkten där meddelandet, i strid med målarkitekturen, skickas tillbaka till ett system som redan har passerats.

## Identifiera loopmönstret i headern

NDR:et och de fullständiga ursprungliga meddelandehuvudena bör sparas innan några ändringar görs. `Received`-rader läses nerifrån och upp: den nedersta raden är det tidigast dokumenterade hoppet och den översta är det senaste.

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

Alla Microsoft-värdnamn som förekommer flera gånger innebär inte automatiskt en loop. Exchange Online behandlar meddelanden internt via flera transportroller. Misstänkt är en upprepad återkomst mellan samma administrativa gränser, till exempel mellan Exchange Online och en lokal gateway. Tidsstämplar, sändande IP-adress, mottagande värd och `Message-ID` hjälper till att tydligt identifiera rundan.

För den första analysen besvaras följande frågor:

1. Vilket system skapade NDR:et?
2. Vilka två eller tre hopp upprepas?
3. Vilket system skulle ha levererat meddelandet slutgiltigt?
4. På grund av vilket domän-, mottagar-, connector- eller regelbeslut vidarebefordrades det?
5. Vilken ändring har senast påverkat e-postflödet?

## Diagnos i Exchange Online

Med `Get-MessageTraceV2` kan behandlingen under de senaste 90 dagarna undersökas; högst tio dagar tillåts per fråga. Ett snävt tidsfönster och den specifika mottagaradressen ger de mest användbara resultaten:

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

Detaljerna för en träff visar enskilda transporthändelser:

```powershell
$trace | ForEach-Object {
    Get-MessageTraceDetailV2 `
        -MessageTraceId $_.MessageTraceId `
        -RecipientAddress $_.RecipientAddress
} | Format-Table Date,Event,Action,Detail -AutoSize
```

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

Det avgörande är inte om ett enskilt objekt verkar rimligt. Domäntyp, faktisk mottagartyp och tillämplig connector måste tillsammans beskriva samma målplats.

## Diagnos i lokal Exchange

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

Ett `SEND` till Exchange Online, följt av ett nytt `RECEIVE` för samma meddelande från Exchange Online, synliggör returen. Med `MessageId` och `NetworkMessageId` går det att undvika att olika testmeddelanden förväxlas.

## De vanligaste orsakerna i översikt

| Mönster | Typisk orsak | Åtgärd |
| --- | --- | --- |
| Okända mottagare pendlar mellan två system | Accepted Domain är inställd på `InternalRelay`, men båda sidor vidarebefordrar okända mottagare | Definiera ett tydligt ansvar; använd `Authoritative` vid fullständig EXO-leverans eller fastställ ett enda avslutande hopp för split-domain |
| EXO skickar till lokal Exchange, som omedelbart skickar tillbaka till EXO | Hybrid-connector eller Centralized Mail Transport passar inte längre brevlådans placering | Kontrollera HCW-konfigurationen och `RouteAllMessagesViaOnPremises`; inaktivera föråldrad central rutt eller korrigera lokal mottagarupplösning |
| Meddelandet pendlar mellan EXO och en gateway för säkerhet, signering eller kryptering | Returnerade meddelanden uppfyller utgående regel igen | Använd en header som gatewayen har angett eller en dokumenterad mekanism för loop prevention som undantag; autentisera inkommande och utgående connectors tydligt |
| Endast en mottagare påverkas | Föråldrad eller felaktig `targetAddress`, fel RemoteMailbox-typ eller motstridiga proxyadresser | Fastställ Source of Authority, korrigera mottagarobjektet där och synkronisera |
| Endast vidarebefordrade meddelanden loopar | Transportregel, brevlådevidarebefordran eller Inbox-regel adresserar den ursprungliga vägen igen | Inaktivera regeln, korrigera målet och definiera ett robust undantag |
| Endast en subdomän eller applikation påverkas | Överordnad domän täcker inte subdomänen korrekt i den förväntade connectorvägen | Konfigurera subdomänen explicit som Accepted Domain och i rätt Send Connector |
| Alla meddelanden loopar efter en gateway- eller DNS-ändring | Smart Host eller MX pekar på det sändande systemets ingång | Korrigera nästa hopp och kontrollera DNS-, NAT- och load balancer-mål separat |

## Orsak 1: Fel typ av Accepted Domain

En authoritative-domän innebär att alla giltiga mottagare för den domänen är kända i Exchange-organisationen; okända mottagare avvisas. En Internal-Relay-domän innebär att en del av mottagarna finns i ett annat system och måste vidarebefordras via en Send- eller Outbound-connector.

Den problematiska konfigurationen uppstår när Exchange Online skickar okända mottagare till ett lokalt system och detta system inte heller behandlar samma domän slutgiltigt, utan skickar tillbaka den till Exchange Online via MX eller Smart Host.

```powershell
Get-AcceptedDomain -Identity contoso.com |
    Format-List DomainName,DomainType,MatchSubDomains
```

När alla mottagare finns i Exchange Online efter en slutförd migrering är `Authoritative` oftast rätt måltillstånd:

```powershell
# Kör endast efter en fullständig kontroll av mottagare och routning.
Set-AcceptedDomain -Identity contoso.com -DomainType Authoritative
```

För en verklig split-domain kan `InternalRelay` vara korrekt. Då krävs dock en tydlig connector till systemet som känner till de återstående mottagarna. Detta mål får inte skicka okända adresser tillbaka till utgångspunkten.

## Orsak 2: Överlappande hybrid-connectors och Centralized Mail Transport

Centralized Mail Transport dirigerar medvetet utgående Exchange Online-meddelanden via lokal Exchange. Det är meningsfullt för vissa compliancekrav, men skapar ytterligare transportvägar. Om alternativet förblir aktivt efter en migrering, trots att det lokala systemet skickar meddelanden tillbaka till Exchange Online via sin egen MX, kan en cirkel uppstå.

```powershell
Get-OutboundConnector -IncludeTestModeConnectors |
    Format-Table Name,Enabled,ConnectorType,RouteAllMessagesViaOnPremises,
        RecipientDomains,SmartHosts,UseMXRecord -AutoSize
```

Flera connectors med överlappande omfattning är också misstänkta. Microsoft rekommenderar en dedikerad lokal connector för hybrid-e-postflöde; en reparation med Hybrid Configuration Wizard är ofta säkrare än enskilda isolerade ändringar.

Om Centralized Mail Transport bevisligen inte längre behövs kan inställningen inaktiveras specifikt:

```powershell
# Endast efter kontroll av compliance- och gatewaykrav.
Set-OutboundConnector `
    -Identity "Outbound to On-Premises" `
    -RouteAllMessagesViaOnPremises:$false
```

## Orsak 3: En gateway behandlar sina returer på nytt

I ett in-and-out-scenario skickar Exchange Online ett meddelande till en tilläggstjänst för signering, kryptering eller arkivering. Tjänsten skickar sedan tillbaka det till Exchange Online. Den utgående regeln måste känna igen returen; annars skickas den åter till tjänsten.

Kontrollen börjar med alla regler som väljer connectors, omdirigerar mottagare eller utvärderar headers:

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Format-List Name,State,Mode,Priority,FromScope,SentToScope,
        RedirectMessageTo,RouteMessageOutboundConnector,
        SetHeaderName,SetHeaderValue,ExceptIfHeaderContainsMessageHeader,
        ExceptIfHeaderContainsWords
```

Det specifika undantaget måste följa gatewayleverantörens dokumentation. Vanligt är en header som sätts av tjänsten och som inte kan förfalskas trovärdigt av internet. Dessutom bör inbound-connectors identifiera tjänsten via certifikat eller fast avsändar-IP. Ett generellt undantag för alla meddelanden som verkar vara ”interna” är för brett.

## Orsak 4: Mottagarobjekt och faktisk brevlåda finns inte på samma plats

Ett objekt kan visas i Exchange Online som `MailUser`, trots att den aktiva brevlådan finns lokalt. I en synkroniserad hybridmiljö är detta inte automatiskt ett dubblettobjekt. Inte heller en `ExternalEmailAddress`, som motsvarar den primära SMTP-adressen, bevisar i sig en felkonfiguration.

Avgörande är kombinationen av alla frågor:

- `Get-Mailbox` lokalt ger ett resultat: Den aktiva brevlådan finns lokalt.
- `Get-RemoteMailbox` lokalt ger ett resultat: Det hanterade målet finns i Exchange Online.
- `Get-EXOMailbox` ger ett resultat: Det finns en riktig brevlåda i molnet.
- `Get-EXORecipient` ger endast en MailUser: Objektet är ett routningsmål, inte en molnbrevlåda.

Problematiska är föråldrade objekt efter en migrering, felaktiga remote routing-domäner eller manuellt satta `targetAddress`-värden vars domän leder tillbaka via samma transportväg. Ändringar görs vid Source of Authority: i synkroniserade miljöer alltså med Exchange-hanteringsverktyg lokalt och inte genom direkt redigering av enskilda attribut i Exchange Online.

## Orsak 5: Vidarebefordringar och transportregler bildar en cirkel

En regel kan omdirigera från adress A till B, medan B skickar tillbaka till A via en andra regel, brevlådevidarebefordran eller ett externt system. Sådana loopar påverkar ofta endast enskilda mottagare eller meddelandetyper.

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

Åtgärden består inte bara i att tillfälligt stänga av en regel. Hela kedjan måste lösas upp, och regler för externa tjänster behöver ett undantag som säkert identifierar redan behandlade meddelanden.

## Orsak 6: MX, Smart Host eller subdomän pekar tillbaka

En gateway kan internt behöva ett annat nästa hopp än externa avsändare. Om den helt enkelt använder den offentliga MX-posten för vidarebefordran kan denna i sin tur peka tillbaka på gatewayen själv. Samma problem uppstår när en Smart Host leder tillbaka till dess egen listener genom NAT eller load balancing.

```powershell
Resolve-DnsName -Type MX contoso.com
Resolve-DnsName -Type MX app.contoso.com

Get-SendConnector |
    Format-List Name,AddressSpaces,DNSRoutingEnabled,SmartHosts
```

Subdomäner förtjänar en egen kontroll. Microsoft dokumenterar fall där en applikationssubdomän uttryckligen måste skapas som en Internal-Relay-domän och synkroniseras till Edge-systemen:

```powershell
New-AcceptedDomain `
    -Name "app.contoso.com" `
    -DomainName app.contoso.com `
    -DomainType InternalRelay

Start-EdgeSynchronization
```

Dessa kommandon är ingen universell lösning. De passar bara om `app.contoso.com` faktiskt levereras utanför Exchange-organisationen och Send Connector har ett entydigt nästa hopp.

## Säkert tillvägagångssätt vid en aktiv loop

Under störningen bör först mångfaldigandet stoppas. Beroende på arkitekturen inaktiveras den utlösande transportregeln eller den specifika connectorn på ett kontrollerat sätt, eller så håller gatewayen tillbaka den berörda kön. Innan dess exporteras konfiguration och meddelandeexempel.

Därefter följer ett test med exakt en avsändare, en mottagare och en tydligt identifierbar ämnesrad. Meddelandet följs utan luckor via headers, Message Trace och lokala trackingloggar. Först när det avslutas vid avsett mål öppnas e-postflödet stegvis igen.

Rekommenderas inte:

- höja hoppgränser
- ändra flera connectors samtidigt
- växla Accepted Domains på måfå mellan `Authoritative` och `InternalRelay`
- mata in en problematisk kö upprepade gånger utan kontroll
- korrigera synkroniserade Exchange-attribut direkt i AD eller Exchange Online
- stänga av TLS-, IP- eller certifikatkontroller som en förment snabbfix

## Slutkontroll

Efter korrigeringen bör dokumentationen innehålla exakt ett påstående för varje relevant domän: Vilket system känner till mottagaren, vilken connector är tillämplig och vilken värd är det slutliga nästa hoppet?

Den tekniska acceptansen omfattar minst:

- externt och internt testmeddelande
- okänd mottagare i samma domän
- mottagare på vardera sidan av en verklig split-domain
- utgående meddelande med aktiverad gateway eller Centralized Mail Transport
- headers utan återkommande hoppsekvens
- Message Trace med `Delivered` respektive förväntad överlämning
- lokal spårning utan ett nytt `RECEIVE` efter ett `SEND` till samma mål
- connector-validering för alla connectors som fortfarande behövs

En åtgärdad e-postloop är inte avslutad förrän inte bara testmejlet kommer fram, utan även okända mottagare och alternativa e-postflödesvägar avslutas enligt definitionen. Det är just där de flesta återfallen finns.

## Källor

1. [Microsoft Learn – Fix NDR error 5.4.6 or 5.4.14 in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/fix-error-code-5-4-6-through-5-4-20-in-exchange-online): Betydelsen av Exchange-NDR:er och typiska orsaker i Accepted Domains och hybrid-connectors.

2. [Microsoft Learn – Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains): Skillnader mellan `Authoritative` och `InternalRelay`.

3. [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains): Ansvar, relay-domäner och Recipient Lookup i lokal Exchange.

4. [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing): Förväntade transportvägar med och utan Centralized Mail Transport.

5. [Microsoft Learn – Validate connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/validate-connectors): Connector-validering och information om flera connectors som matchar samtidigt.

6. [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): E-postflödesmönster som stöds med framförliggande tredjepartstjänster.

7. [Microsoft Learn – Mail flow rules in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/mail-flow-rules): Bearbetning, prioritet, åtgärder och undantag för transportregler.

8. [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): Sökning efter meddelanden i Exchange Online-transporten.

9. [Microsoft Learn – Search message tracking logs](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/search-message-tracking-logs): Lokal meddelandespårning över alla Exchange-servrar.

10. [Microsoft Learn – Hop count exceeded for an on-premises application subdomain](https://learn.microsoft.com/en-us/troubleshoot/exchange/mailflow/hop-count-exceeded-possible-mail-loop): Dokumenterat subdomän-/EdgeSync-scenario med explicit Internal-Relay-domän.
