---
title: "Vem levererar egentligen till din tenant? Aggregera avsändande IP-adresser"
navTitle: "Avsändande IP-adresser"
description: "En enda analys visar vilka system som faktiskt levererar e-post till din tenant: bortglömda anslutningar, program som skickar direkt och tjänsteleverantörer som ingen har dokumenterat, inklusive de typiska analysfelen kring sidlogik och tolkning."
date: "2026-08-11"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "12 min lästid"
themen:
  - microsoft-365-exchange
  - smtp-mailflow
  - exchange-onprem-hybrid
hauptthema: "microsoft-365-exchange"
produkte:
  - "exchange-online"
  - "exchange-hybrid"
protokolle:
  - "smtp"
  - "powershell"
  - "haertung"
related:
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - ghost-sender-exchange-online-nebeneingang
  - mailflow-fehlersuche-kontrollierte-experimente
slug: "vem-levererar-egentligen-till-din-tenant-aggregera-avsandande-ip-adresser"
translationId: "article-5879cc0eb17ed951"
draft: false
translationOf: einliefernde-ip-adressen-aggregieren
translationSourceHash: 9209720819061360cb72bfa01ab6261e6af80e547a398c25f6802edfbe49bb6c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:06:56.374Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/vem-levererar-egentligen-till-din-tenant-aggregera-avsandande-ip-adresser
---

# Vem levererar egentligen till din tenant? Aggregera avsändande IP-adresser

Knappast någon e-postmiljö vet längre fullt ut vem som levererar till den. Med åren samlas anslutningar från migreringar, program som skickar direkt, tjänsteleverantörer vars avtal sedan länge har löpt ut och testkonfigurationer som aldrig har avvecklats. Så länge e-posten flödar märker ingen det.

En enda analys skapar klarhet: gruppering av alla inkommande meddelanden efter deras käll-IP-adress. Den är klar på två minuter, och resultatlistan är regelbundet överraskande. Den här artikeln visar frågan, förklarar hur du får den **fullständig** och framför allt hur du tolkar siffrorna rätt. Tolkningen är nämligen den svårare delen.

## Varför det är värt det

Listan besvarar fyra frågor som annars är mödosamma att reda ut var för sig. Vilka system skickar över huvud taget till din tenant? Går allt via de vägar som du har dokumenterat, eller finns det en andra ingång? Används en anslutning som du vill avveckla fortfarande? Och: Skickar ett program direkt till tjänsten förbi din gateway och kringgår därmed filtreringen?

Särskilt den sista frågan är säkerhetsrelevant. Den som levererar direkt kringgår inte bara filtreringen utan ofta även loggningen som du vill kunna förlita dig på vid en incident.

## Frågan

I tenanten grupperar du meddelandespårningen efter `FromIP`:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) `
    -EndDate (Get-Date) `
    -ResultSize 5000 |
  Group-Object FromIP |
  Sort-Object Count -Descending |
  Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-StartDate (Get-Date).AddHours(-2)` | Start för frågefönstret, här för två timmar sedan |
| `-EndDate (Get-Date)` | Slut för frågefönstret, den aktuella tidpunkten |
| `-ResultSize 5000` | Maximalt antal rader per anrop; 5000 är samtidigt högsta värdet |
| `Group-Object FromIP` | Grupperar meddelandena efter den levererande IP-adressen |
| `Sort-Object Count -Descending` | Sorterar grupperna fallande efter antal meddelanden |
| `Format-Table Count, Name -AutoSize` | Tvåkolumnsutdata (antal, IP-adress) med automatisk kolumnbredd |

</details>

Ett typiskt resultat:

```text
Count Name
----- ----
 1771 255.255.255.255
 1649 10.0.20.23
  260 10.0.20.21
   49 2603:10a6:150:1f3::17
   46 165.225.94.87
   36 136.226.192.164
   35 147.161.246.105
   12 198.51.100.77
    3 203.0.113.9
```

Innan du drar slutsatser av det måste två saker stämma: listan måste vara fullständig och du måste veta vad posterna betyder.

## Felkälla 1: Listan är nästan alltid ofullständig

`Get-MessageTraceV2` returnerar resultat sidvis, högst 5000 rader per anrop. Vid hög volym täcker en sida bara en bråkdel av ditt tidsfönster. Du grupperar då ett urval och betraktar resultatet som helheten.

Det känns igen på denna varning:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**Om denna varning visas är din analys värdelös.** I synnerhet får en saknad post då inte tolkas som frånvaro. En adress med tre meddelanden per dag visas ändå inte i ett urval.

Det finns två vägar ut. Den enkla: Minska tidsfönstret tills varningen uteblir. Vid 5000 meddelanden per timme blir det 55 minuter och inte sju dagar. För frågan ”vilka system skickar över huvud taget” räcker ett fullständigt kort fönster oftast helt och hållet.

Den grundliga vägen bläddrar igenom alla sidor och samlar in dem:

```powershell
$start = (Get-Date).AddHours(-24)
$ende  = Get-Date
$alle  = @()
$naechster = $null

do {
    $seite = if ($naechster) {
        Get-MessageTraceV2 -StartDate $start -EndDate $ende `
            -StartingRecipientAddress $naechster -ResultSize 5000
    } else {
        Get-MessageTraceV2 -StartDate $start -EndDate $ende -ResultSize 5000
    }

    $alle += $seite
    $naechster = if ($seite.Count -eq 5000) { $seite[-1].RecipientAddress } else { $null }
    Write-Host "Gesammelt: $($alle.Count)"
} while ($naechster)

$alle | Group-Object FromIP | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-StartDate` / `-EndDate` | Frågefönster, här de senaste 24 timmarna |
| `-StartingRecipientAddress` | Fortsättningspunkt för sidlogiken: mottagaradressen där nästa sida börjar |
| `-ResultSize 5000` | Sidstorlek; en full sida signalerar att fler resultat följer |
| `Group-Object FromIP` | Grupperar hela beståndet efter den levererande IP-adressen |
| `Sort-Object Count -Descending` | Sorterar grupperna fallande efter antal meddelanden |
| `Format-Table Count, Name -AutoSize` | Visar antal per adress med automatisk kolumnbredd |

</details>

Slingan hämtar ytterligare sidor så länge en sida med exakt 5000 rader returneras och fortsätter varje gång med den sista mottagaradressen från föregående sida; först därefter grupperas hela beståndet.

Räkna med några minuters körtid för 24 timmar i en medelstor miljö. För en engångsinventering är det väl investerat.

## Felkälla 2: Siffrorna betyder inte vad de verkar betyda

Resultatlistan innehåller fyra helt olika typer av poster, och den som blandar ihop dem drar fel slutsatser.

**`255.255.255.255` står inte för ett system.** Detta värde visas när det inte fanns någon inkommande SMTP-anslutning utifrån för meddelandet. Det gäller meddelanden som skapats i själva tjänsten: journalrapporter, leveransfelmeddelanden, frånvaromeddelanden och meddelanden mellan postlådor i samma tenant. I nästan varje miljö är detta den största posten, och den är helt odramatisk.

**Privata adresser enligt RFC 1918** kommer från ditt eget nätverk. I hybridmiljöer ser du här de lokala transportservrarna, eftersom deras interna adress bevaras vid överlämningen till tjänsten. Det är de stora siffrorna i listan och i regel den förväntade huvudvägen.

**Adresser från säkerhets- och filtreringstjänster** identifierar du genom operatören, inte genom siffervärdet. Molnproxyservrar, framförliggande e-postgatewayer och webbsäkerhetstjänster visas med många angränsande adresser och medelstora tal. De hör oftast hemma där, men bör stå i driftmanualen.

**Enskilda offentliga adresser med små tal** är de intressanta. Det är just där bortglömda program, gamla tjänsteleverantörer och system som ingen längre känner till döljer sig.

## Upplösningen: från adresser till namn

För allt du inte direkt kan identifiera hjälper omvänd namnuppslagning. Den är inte alltid konfigurerad och inte alltid tillförlitlig, men i de flesta fall ger den den avgörande ledtråden:

```powershell
$unbekannt = '198.51.100.77','203.0.113.9'

$unbekannt | ForEach-Object {
    $name = try { (Resolve-DnsName $_ -Type PTR -ErrorAction Stop).NameHost } catch { '(kein PTR)' }
    [pscustomobject]@{ IP = $_; Name = $name }
} | Format-Table -AutoSize
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Resolve-DnsName $_ -Type PTR` | Frågar efter den omvända posten (PTR) för respektive IP-adress |
| `-ErrorAction Stop` | Gör en saknad post till ett fel som kan fångas upp för blocket `try`/`catch` |
| `[pscustomobject]@{ … }` | Skapar ett objekt per adress med IP och upplöst namn för tabellutdata |
| `Format-Table -AutoSize` | Utdata med automatisk kolumnbredd |

</details>

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

En saknad PTR är i sig inget tecken på ett problem, men det är en god anledning att titta närmare. Granska de tillhörande meddelandena för sådana adresser:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.FromIP -eq '203.0.113.9' } |
  Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-StartDate` / `-EndDate` / `-ResultSize` | Frågefönster och sidstorlek som i huvudfrågan |
| `Where-Object { $_.FromIP -eq '203.0.113.9' }` | Filtrerar på klientsidan efter den aktuella källadressen |
| `Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize` | Visar mottagningstid, avsändare, mottagare, ämne och leveransstatus per meddelande |

</details>

Avsändare och ämne talar i regel omedelbart om vilket program som ligger bakom.

## Jämförelsen: vilken adress hör till vilken anslutning?

Jämför din resultatlista med de konfigurerade anslutningarna:

```powershell
Get-InboundConnector |
    Format-List Name, Enabled, ConnectorType, ConnectorSource,
        @{n='SenderIPAddresses'; e={$_.SenderIPAddresses -join ', '}},
        @{n='SenderDomains';     e={$_.SenderDomains -join ', '}},
        RequireTls, TlsSenderCertificateName,
        RestrictDomainsToCertificate, CloudServicesMailEnabled, TreatMessagesAsInternal
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Get-InboundConnector` | Listar alla inkommande anslutningar i tenanten; här medvetet utan begränsande parametrar |
| `Format-List <Eigenschaften>` | Utdata som en lista över de angivna egenskaperna, en per rad |
| `@{n='…'; e={…}}` | Beräknad egenskap med namn (`n`) och uttryck (`e`) |
| `-join ', '` | Gör arrayen med adresser respektive domäner till en läsbar, kommaseparerad rad |

</details>

Tre konstellationer är talande.

**En adress levererar, men nämns inte i någon anslutning.** Då kommer e-posten in som vanlig internet-e-post. Det är tillåtet, men innebär att programmet inte får någon särbehandling och att dess meddelanden omfattas av fullständig filtrering. Om någon påstår att en anslutning är konfigurerad för detta system stämmer det uppenbarligen inte längre.

**En anslutning anger adresser från vilka inget kommer.** En kandidat för avveckling. Kontrollera innan du tar bort den om det rör sig om säsongsbetonade eller sällan använda system, och inaktivera först i stället för att ta bort direkt.

**En anslutning sätter `TreatMessagesAsInternal` eller `CloudServicesMailEnabled` till sant.** Här är det värt att titta noga. Båda inställningarna gör att meddelanden via denna väg behandlas som interna för organisationen. Om e-post från internet kommer in den vägen kringgår den därmed kontroller som är avsedda för externa meddelanden, bland annat skyddet mot förfalskade avsändare från den egna domänen. För en ren hybridanslutning är det korrekt; för en anslutning via vilken godtyckliga system levererar är det ett fynd.

## Vad du vanligtvis hittar

Från praktiken, utan anspråk på fullständighet: en testanslutning från en migrering som har varit aktiv i åratal. Ett verksamhetssystem som skickar direkt till tjänsten trots att alla tror att det går via gatewayen. En nyhetsbrevsleverantör vars avtal har löpt ut men som fortfarande får leverera. Och regelbundet en anslutning med mycket öppna villkor, som någon en gång skapade för att lösa ett akut problem.

Inget av dessa fynd är dramatiskt i sig. Tillsammans ger de bilden av en miljö som ingen längre har full överblick över, och det är just den egentliga risken.

## Metodens begränsningar

Det finns tre begränsningar du bör känna till.

Meddelandespårningen via cmdleten sträcker sig bara omkring tio dagar tillbaka. För längre tidsperioder behöver du den historiska sökningen, som körs asynkront och täcker upp till 90 dagar. Annars missar du sällsynta system som skickar månadsvis.

`FromIP` betyder inte samma sak överallt. För e-post från internet är det adressen till den levererande servern. För hybrid-e-post är det adressen till din lokala transportserver, inte den ursprungliga avsändaren. Analysen visar alltså **sista stationen före tjänsten**, inte ursprunget.

Och kopplingen till en anslutning är inte direkt synlig i tenanten. Du drar slutsatsen utifrån adress, certifikat och avsändardomän. För ett tillförlitligt uttalande om användningen av en enskild anslutning är anslutningsrapporten i Exchange Admin Center under Rapporter och e-postflöde en bättre källa, eftersom den aggregerar på serversidan över längre tidsperioder.

## Som återkommande kontroll

Denna analys lämpar sig väl som en kvartalsrutin. Spara resultatet och jämför vid nästa genomgång. Nya adresser i listan är antingen dokumenterade ändringar eller något du vill känna till.

Om du ändå granskar e-postkonfigurationen för dina domäner: [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) visar SPF, DKIM, DMARC och övriga e-poststandarder för valfria domäner direkt i webbläsaren, även för sido- och marknadsföringsdomäner som erfarenhetsmässigt glöms bort vid sådana inventeringar. Och för själva frågorna ger [Befehls-Generator](https://rafaelpfister.ch/tools/command-builder) färdiga byggblock för PowerShell och Unix-skal.

Hur du följer upp enskilda avvikande meddelanden beskrivs i [Analysera Exchange-e-postflöde: meddelandespårning, SMTP-loggar och Receive Connectors](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Källor

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): Fältlista inklusive FromIP och ToIP samt sidlogiken med StartingRecipientAddress.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): asynkron meddelandespårning över upp till 90 dagar för äldre tidsperioder.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector): parametrar för inkommande anslutningar, bland annat SenderIPAddresses och TreatMessagesAsInternal.

4.  [Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): samspelet mellan anslutningstyperna och när respektive typ används.

5.  [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918): definierar de privata adressintervallen som du måste skilja från offentliga adresser i analysen.
