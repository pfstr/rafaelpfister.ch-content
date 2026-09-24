---
title: "E-postloop med krypteringsgateway bakom EXO – förhindra spoofingproblem"
navTitle: "Gateway-loop"
description: "Om krypteringsgatewayen (i exemplet HIN) står bakom Exchange Online kommer varje inkommande meddelande till EOP en andra gång: med en extern avsändardomän, ogiltig DKIM-signatur och gatewayens IP-adress som källa. Resultatet blir ett spoofingutslag och skräppost. Fyra inställningar förhindrar detta utan att inaktivera filtreringen med SCL -1: PTR-post, Enhanced Filtering av, spoofingundantag för gatewayinfrastrukturen och CloudServicesMailEnabled på båda anslutningarna."
date: "2026-09-23"
kategorie: "E-postflöde och SMTP"
timeToRead: "9 min lästid"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
  - hin-gateway
  - e-mail-verschluesselung
produkte:
  - "exchange-online"
  - "hybrid-mailfluss"
  - "hin"
protokolle:
  - "mail-auth"
  - "smtp"
  - "powershell"
slug: "e-postloop-med-krypteringsgateway-bakom-exo-forhindra-spoofingproblem"
translationId: "article-b773c8d303aee87a"
aiPrompt: |
  Du bist mein Exchange-Online-Assistent. Ich betreibe ein Verschlüsselungsgateway hinter Exchange Online (Mail-Schlaufe: EXO, Gateway, EXO). Prüfe mit mir die vier Einstellungen: PTR-Record der Gateway-IP, Enhanced Filtering auf dem Inbound-Connector, Spoof-Ausnahme in der Tenant Allow/Block List für die Gateway-Infrastruktur und CloudServicesMailEnabled auf beiden Connectoren. Erkläre mir zu jeder Einstellung, warum sie nötig ist, und hilf mir, das Ergebnis anhand der Authentication-Results-Header einer Testnachricht zu verifizieren.
translationOf: verschluesselungsgateway-hinter-exchange-online
url: https://rafaelpfister.ch/sv/blog/e-postloop-med-krypteringsgateway-bakom-exo-forhindra-spoofingproblem
translationSourceHash: ac3ae7b36a2682c2913479a6d9d0356d1abd96c68459bcfb4912362884e0a708
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T08:36:29.721Z
translationReview: automatic
---

# E-postloop med krypteringsgateway bakom EXO – förhindra spoofingproblem

Många organisationer inom den schweiziska hälso- och sjukvården använder en HIN-gateway, andra använder SEPPmail eller totemomail för S/MIME och PGP. Om MX-posten pekar på Microsoft och gatewayen står bakom Exchange Online passerar varje inkommande meddelande filtreringen två gånger. Vid den andra passagen ser EOP ett meddelande med en extern avsändardomän, en ogiltig DKIM-signatur och gatewayens IP-adress som källa. Ur filtrets perspektiv är detta mönstret för avsändarförfalskning, och legitim e-post hamnar i skräppostmappen. Jag har konfigurerat denna lösning flera gånger och beskriver här de fyra inställningar som gör att loopen fungerar utan spoofingutslag, samt varför var och en av dem behövs. HIN-gatewayen används som exempel; mekanismen är identisk för alla andra gateways i samma position.

## Konfigurationen

```text
Absender > MX > Exchange Online (1. Durchlauf)
                    > HIN-Gateway: Entschlüsselung, Signaturprüfung
                            > Exchange Online (2. Durchlauf) > Postfach
```

En transportregel dirigerar inkommande meddelanden via en utgående anslutning till gatewayen. Gatewayen dekrypterar, kontrollerar signaturer och levererar meddelandet tillbaka till Exchange Online via en inkommande anslutning. Ett rubrikfält som sätts av gatewayen förhindrar att regeln tillämpas igen.

Konfigurationen har goda skäl: Microsoft filtrerar först, gatewayen får endast förkontrollerad e-post och verksamheten behöver inte exponera en egen MX-post mot internet. Priset är den andra passagen, och den kan inte stängas av utan endast konfigureras korrekt.

## Varför SCL -1 är fel svar

Den vanliga åtgärden är en transportregel på återvägen som sätter Spam Confidence Level till `-1`. Då hoppar Exchange Online helt över innehållskontrollen vid den andra passagen. Det fungerar omedelbart men har två nackdelar: den andra passagen bidrar därefter inte med något, och regeln är knuten till ett villkor (anslutning eller IP) som förändras vid varje ändring. Microsoft beskriver dessutom SCL -1 som indata för filtreringen, inte som ett slutgiltigt beslut; det stämplade värdet kan avvika. De fyra inställningarna nedan löser problemet där det uppstår: vid bedömningen av den levererande infrastrukturen.

## De fyra inställningarna

1. Publicera en PTR-post i offentlig DNS för HIN-gatewayens publika IP-adress. Utan denna post fungerar inte hela flödet.
2. Stäng av Enhanced Filtering helt på den inkommande anslutning via vilken HIN levererar till Exchange Online.
3. Skapa ett spoofingundantag för den levererande HIN-infrastrukturen i Tenant Allow/Block List.
4. Aktivera `CloudServicesMailEnabled` på den utgående anslutningen till HIN och på den inkommande anslutningen från HIN.

Kontrollera först under punkt 2 vilka anslutningar som berörs:

```powershell
Get-InboundConnector |
    Select-Object Name, Enabled, ConnectorType, EFSkipLastIP, EFSkipIPs, EFUsers, EFTestMode |
    Format-List
```

Stäng sedan av funktionen på HIN-anslutningen:

```powershell
$efAus = @{
    Identity     = "<Inbound-Connector HIN>"
    EFSkipLastIP = $false
    EFSkipIPs    = $null
    EFUsers      = $null
}
Set-InboundConnector @efAus
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Parameter | Effekt |
|---|---|
| `EFSkipLastIP = $false` | Det sista hoppet hoppas inte längre över automatiskt. Om ingen IP-adress samtidigt finns i `EFSkipIPs` är Enhanced Filtering inaktiverat på anslutningen. |
| `EFSkipIPs = $null` | Tömmer listan över IP-adresser som ska hoppas över. |
| `EFUsers = $null` | Tar bort begränsningen till enskilda mottagare; utan aktiv Enhanced Filtering saknar värdet betydelse, men förblir därmed tydligt. |
| `EFTestMode` | Endast i frågan: visar om anslutningen är i testläge. Microsoft anger parametern som intern, men den är läsbar. |

</details>

Punkt 3, spoofingundantaget:

```powershell
$spoof = @{
    Identity              = "Default"
    Action                = "Allow"
    SpoofedUser           = "*"
    SendingInfrastructure = "gateway.example.com"
    SpoofType             = "External"
}
New-TenantAllowBlockListSpoofItems @spoof
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Parameter | Effekt |
|---|---|
| `Identity = "Default"` | Själva listan; det finns bara denna enda. |
| `Action = "Allow"` | Tillåter kombinationen. `Block` klassificerar den i stället som nätfiske. |
| `SpoofedUser = "*"` | Adressen som syns i From-fältet. Platshållaren står för valfria avsändare. En platshållare är tillåten på ena sidan av paret, inte på båda. |
| `SendingInfrastructure` | Källan: domänen från PTR-posten för den levererande IP-adressen (punkt 1). Utan PTR-post accepterar listan endast `<IP>/24`. |
| `SpoofType = "External"` | Gäller externa avsändardomäner. `Internal` täcker egna accepterade domäner och kräver en andra post. |

</details>

Punkt 4, Cross-Premises-rubrikerna:

```powershell
Set-OutboundConnector -Identity "<Outbound-Connector zu HIN>" -CloudServicesMailEnabled $true
$crossPremises = @{
    Identity                 = "<Inbound-Connector HIN>"
    TreatMessagesAsInternal  = $false
    CloudServicesMailEnabled = $true
}
Set-InboundConnector @crossPremises
```

Varför `TreatMessagesAsInternal` ingår i samma kommando för den inkommande anslutningen förklarar jag nedan under punkt 4.

## Till 1: PTR-post

EOP identifierar den levererande infrastrukturen genom omvänd uppslagning av käll-IP-adressen. PTR-värdet visas i `Authentication-Results`-rubriken som sending infrastructure, och det är exakt detta värde som spoofingundantaget i punkt 3 hänvisar till. Om PTR-posten saknas bedömer Exchange Online varje meddelande på återvägen med `PTR:InfoDomainNonexistent`, och undantaget måste avse ett helt `/24` i stället för ett namn. Avsaknad av en omvänd DNS-post är dessutom sedan länge en etablerad negativ signal i alla skräppostfilter. Posten bör lösas framåt till samma IP-adress.

För HIN är den levererande IP-adressen HIN-mailgatewayens adress, genom vilken meddelandena återkommer till Exchange Online. Vem som hanterar PTR-posten för denna IP-adress beror på driftmodellen; ansvaret måste klargöras före omläggningen.

## Till 2: stäng av Enhanced Filtering

Enhanced Filtering for Connectors (Skip Listing) är byggt för den omvända konfigurationen: gateway före Microsoft 365, MX riktad mot gatewayen. Där hoppar EOP över det sista hoppet och bedömer meddelandets verkliga ursprung.

Det fungerar inte i loopkonfigurationen. Microsoft anger i dokumentationen att Enhanced Filtering inte är avsett för tjänster som behandlar e-post efter Microsoft 365, och listar icke-linjär routning (internet, Microsoft 365, externt system, Microsoft 365) som ej stödd. Som följd nämner dokumentationen exakt det symptom som uppstår i praktiken: Microsoft 365 granskar den återkommande e-posten igen, tilldelar ett `compauth`-värde och meddelandet kan klassificeras som skräppost.

Om Enhanced Filtering förblir aktivt hoppar EOP över HIN-IP-adressen och bedömer IP-adressen före den. I loopen är det antingen en Microsoft-ägd IP-adress från den första passagen eller den ursprungliga externa avsändaren. Båda är fel:

* Bedömningen görs på en IP-adress som vid den andra passagen inte längre kan säga något om meddelandet.
* Spoofingundantaget från punkt 3 träffar aldrig, eftersom det är knutet till HIN-infrastrukturen som EOP just har hoppat över.

Därför måste Enhanced Filtering vara avstängt på denna anslutning: först då blir HIN synligt och adresserbart som levererande infrastruktur.

## Till 3: spoofingundantag

En spoofingpost är alltid ett par bestående av Spoofed user (From-adressen eller dess domän) och Sending infrastructure (källan). Endast denna kombination tillåts.

`SpoofedUser = "*"` tillsammans med HIN-infrastrukturen innebär alltså: varje From-adress får leverera via HIN utan att spoofingutslaget träder i kraft. Varje annan källa som använder samma avsändare kontrolleras fortsatt. Undantaget är ingen förbikoppling av skräppostfiltret: kontroll av skräppost, innehåll och hot fortsätter oförändrat vid den andra passagen. Ett meddelande kan alltså fortfarande sorteras bort på grund av sitt innehåll, det betraktas bara inte längre som en förfalskning.

Kommandot ovan innehåller medvetet endast `SpoofType = "External"`. Egna e-postmeddelanden, alltså meddelanden med avsändare från de egna accepterade domänerna, bör inte återkomma via gatewayen. De går från interna system eller OnPrem-Exchange direkt till Exchange Online utan att passera HIN. Huruvida detta faktiskt gäller i er miljö visas av Spoof Intelligence-utvärderingen:

```powershell
Get-SpoofIntelligenceInsight |
    Select-Object SpoofedUser, SendingInfrastructure, SpoofType, MessageCount, Action |
    Sort-Object MessageCount -Descending |
    Format-Table -AutoSize |
    Out-String -Width 200
```

Om egna domäner visas där med HIN-infrastrukturen och `SpoofType Internal` krävs en andra post med `SpoofType = "Internal"`, eller ännu hellre: att orsaken till att intern e-post går via gatewayen utreds.

Spoofingposter upphör inte automatiskt. Om gatewayen tas bort eller dess infrastruktur ändras ska posten tas bort.

## Till 4: CloudServicesMailEnabled

Denna punkt är den enda som Microsoft inte uttryckligen dokumenterar för detta scenario. Den är min slutsats utifrån parameterns dokumenterade beteende: den bevarar Cross-Premises-rubrikerna genom loopen.

Parametern styr hanteringen av interna `X-MS-Exchange-Organization-*`-rubriker. På den utgående anslutningen omvandlas de till `X-MS-Exchange-CrossPremises-*` och överlever därmed sträckan via HIN. På den inkommande anslutningen skrivs de tillbaka till `X-MS-Exchange-Organization-*` och ersätter samtidigt rubriker med samma namn som redan finns i meddelandet. Om parametern är satt till `$false` tar anslutningen bort dessa rubriker.

I praktiken innebär det att det Exchange Online fastställt vid den första passagen, bland annat autentiseringsstatus och intern märkning, överlever loopen i stället för att tas bort vid återinträdet. I rubrikerna för det levererade meddelandet står då `X-CrossPremisesHeadersPromoted`, och de ursprungliga kontrollresultaten behålls under `Authentication-Results-Original`. Detta förhindrar inte i sig ett spoofingutslag (det är punkt 1 till 3 som ansvarar för det), men det bevarar bedömningen från den första passagen.

Tre saker bör beaktas:

* Om anslutningarna skapades av Hybrid Configuration Wizard (`ConnectorSource: HybridWizard`), skriver en senare HCW-körning över inställningen. HIN-anslutningarna bör därför vara separata, manuellt skapade anslutningar.
* På den inkommande anslutningen utesluter `CloudServicesMailEnabled $true` och `TreatMessagesAsInternal $true` varandra. Om `TreatMessagesAsInternal` redan är satt till `$true`, avvisar Exchange Online kommandot. Därför ska båda parametrarna ingå i samma `Set-InboundConnector`-anrop, som visas ovan.
* Microsoft rekommenderar att parametern endast sätts efter anvisning från supporten eller i produktdokumentationen. För loopen finns ingen sådan dokumentation; beslutet är ert och bör dokumenteras.

## Säkra leveransen

Eftersom Exchange Online efter punkt 4 övertar rubriker från retursträckan och efter punkt 3 accepterar varje From-adress från HIN-infrastrukturen måste den inkommande anslutningen begränsas så att endast HIN kan leverera genom den. För en anslutning av typen `Partner` är detta `RestrictDomainsToCertificate` med `TlsSenderCertificateName` eller `RestrictDomainsToIPAddresses` med `SenderIPAddresses`, vardera tillsammans med `RequireTls`. Utan denna begränsning skulle varje källa som når anslutningen kunna leverera med valfria avsändare och övertagna organisationsrubriker.

```powershell
$absicherung = @{
    Identity                     = "<Inbound-Connector HIN>"
    RequireTls                   = $true
    RestrictDomainsToCertificate = $true
    TlsSenderCertificateName     = "gateway.example.com"
}
Set-InboundConnector @absicherung
```

## Kontroll

Efter omläggningen skickar du ett testmeddelande från en extern domän med DMARC-princip via gatewayen och kontrollerar rubrikerna i det levererade meddelandet. I `Authentication-Results`-rubriken från den andra passagen ska HIN-domänen anges som sending infrastructure och `compauth` vara `pass` med en anledning inom området Tenant Allow-poster, inte längre `fail reason=001`. `X-CrossPremisesHeadersPromoted` och `Authentication-Results-Original` visar att punkt 4 fungerar. [Mail Header Analyzer](/tools/header-analyzer) på denna webbplats visar båda passagerna som ett flödesdiagram och markerar hoppen mellan Exchange Online och gatewayen.

## Källor

1.  [Enhanced filtering for connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Avgränsning mellan linjär och icke-linjär routning, information om tjänster bakom Microsoft 365, följden `compauth` och skräppostklassificering, SCL -1 som indata i stället för beslut, PowerShell-parametrarna `EFSkipLastIP`, `EFSkipIPs`, `EFUsers`.

2.  [Set-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-inboundconnector): Beteende för `CloudServicesMailEnabled` (omvandling och befordran av Cross-Premises-rubrikerna, borttagning vid `$false`), ömsesidig uteslutning med `TreatMessagesAsInternal`, `RestrictDomainsToCertificate` och `RestrictDomainsToIPAddresses` för partneranslutningar.

3.  [Set-OutboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-outboundconnector): `CloudServicesMailEnabled` på den utgående anslutningen.

4.  [Allow or block email using the Tenant Allow/Block List](https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-email-spoof-configure): Syntax för spoofingposter, regler för platshållare, fastställande av sändningsinfrastruktur via PTR-post eller `/24`, täckning av DMARC-relaterade spoofingutslag.

5.  [New-TenantAllowBlockListSpoofItems](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-tenantallowblocklistspoofitems): Parametrarna `SpoofedUser`, `SendingInfrastructure`, `SpoofType`, `Action`.

6.  [Get-SpoofIntelligenceInsight](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-spoofintelligenceinsight): Utvärdering av identifierade spoofingpar under de senaste sju dagarna.
