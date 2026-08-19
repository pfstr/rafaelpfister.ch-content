---
title: "Vem levererar egentligen till din tenant? Aggregera avsändande IP-adresser"
navTitle: "Avsändande IP-adresser"
description: "En enda analys visar vilka system som faktiskt levererar e-post till din tenant: bortglömda anslutningar, applikationer som skickar direkt och tjänsteleverantörer som ingen har dokumenterat. Inklusive fallgropar kring sidlogik och tolkning."
date: "2026-08-11"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "12 min läsning"
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
url: https://rafaelpfister.ch/sv/blog/vem-levererar-egentligen-till-din-tenant-aggregera-avsandande-ip-adresser
translationSourceHash: 9dc48329a06945f705380eb3db428efb548f0c36a1fe3c4f2fb7de1185fee879
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:13:35.023Z
translationReview: required
---

# Vem levererar egentligen till din tenant? Aggregera avsändande IP-adresser

Nästan ingen e-postmiljö vet längre fullt ut vem som levererar e-post till den. Med åren samlas anslutningar från migreringar, applikationer som skickar direkt, tjänsteleverantörer vars avtal sedan länge har löpt ut och testkonfigurationer som aldrig avvecklades. Så länge e-posten flödar märker ingen det.

En enda analys skapar klarhet: gruppering av alla inkommande meddelanden efter deras käll-IP-adress. Den är klar på två minuter och resultatlistan är regelbundet överraskande. Den här artikeln visar frågan, förklarar hur du får den **fullständig** och framför allt hur du läser siffrorna rätt. För tolkningen är den svårare delen.

## Varför det lönar sig

Listan besvarar fyra frågor som annars är besvärliga att reda ut var för sig. Vilka system skickar över huvud taget till din tenant? Går allt via de vägar du har dokumenterat, eller finns det en andra ingång? Används fortfarande en anslutning som du vill avveckla? Och: Skickar en applikation direkt till tjänsten förbi din gateway och kringgår därmed filtreringen?

Särskilt den sista frågan är säkerhetsrelevant. Den som levererar direkt kringgår inte bara filtreringen, utan ofta även loggningen som du vill kunna lita på i ett allvarligt läge.

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

En typisk utdata:

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

Innan du drar slutsatser måste två saker stämma: Listan måste vara fullständig och du måste veta vad posterna betyder.

## Fallgrop 1: Listan är nästan alltid ofullständig

`Get-MessageTraceV2` returnerar resultat sida för sida, högst 5 000 rader per anrop. Vid hög volym täcker en sida bara en bråkdel av ditt tidsfönster. Då grupperar du över ett urval och betraktar resultatet som helheten.

Det märks på den här varningen:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**Om den här varningen visas är din analys värdelös.** Framför allt får en saknad post då inte tolkas som frånvaro. En adress med tre meddelanden per dag syns ändå inte i ett urval.

Det finns två vägar ut. Den enkla: Minska tidsfönstret tills varningen inte längre visas. Vid 5 000 meddelanden per timme blir det just 55 minuter och inte sju dagar. För frågan ”vilka system skickar över huvud taget” räcker ett fullständigt kort tidsfönster oftast helt och hållet.

Den grundliga vägen bläddrar igenom alla sidor och samlar in resultaten:

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

Räkna med några minuters körtid för 24 timmar i en medelstor miljö. För en engångsinventering är det väl investerad tid.

## Fallgrop 2: Siffrorna betyder inte vad de verkar betyda

Resultatlistan innehåller fyra helt olika typer av poster, och den som blandar ihop dem drar fel slutsatser.

**`255.255.255.255` står inte för ett system.** Detta värde visas när det inte fanns någon inkommande SMTP-anslutning utifrån för meddelandet. Det gäller meddelanden som skapas i själva tjänsten: journalrapporter, leveransfelmeddelanden, frånvaromeddelanden och meddelanden mellan postlådor i samma tenant. I nästan varje miljö är detta den största posten och den är helt normal. Bli inte orolig.

**Privata adresser enligt RFC 1918** kommer från ditt eget nätverk. I hybridmiljöer ser du här de lokala transportservrarna, eftersom deras interna adress behålls vid överlämningen till tjänsten. Det är de stora siffrorna i listan och i regel den förväntade huvudvägen.

**Adresser från säkerhets- och filtreringstjänster** känner du igen på operatören, inte på siffervärdet. Molnproxier, framförliggande e-postgateways och webbsäkerhetstjänster visas med många närliggande adresser och medelstora siffror. De hör oftast hemma där, men bör finnas dokumenterade i driftmanualen.

**Enskilda offentliga adresser med små siffror** är de intressanta. Det är just där bortglömda applikationer, gamla tjänsteleverantörer och system som ingen längre känner till gömmer sig.

## Upplösningen: från adresser till namn

För allt du inte omedelbart kan identifiera hjälper omvänd uppslagning. Den är inte alltid konfigurerad och inte alltid sanningsenlig, men i de flesta fall ger den den avgörande ledtråden:

```powershell
$unbekannt = '198.51.100.77','203.0.113.9'

$unbekannt | ForEach-Object {
    $name = try { (Resolve-DnsName $_ -Type PTR -ErrorAction Stop).NameHost } catch { '(kein PTR)' }
    [pscustomobject]@{ IP = $_; Name = $name }
} | Format-Table -AutoSize
```

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

En saknad PTR är inget bevis på något skadligt, men det är en god anledning att titta närmare. Undersök de tillhörande meddelandena för sådana adresser:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.FromIP -eq '203.0.113.9' } |
  Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize
```

Avsändare och ämnesrad säger i regel genast vilken applikation som ligger bakom.

## Avstämningen: vilken adress hör till vilken anslutning?

Nu kommer den egentliga insikten. Jämför resultatlistan med de konfigurerade anslutningarna:

```powershell
Get-InboundConnector |
    Format-List Name, Enabled, ConnectorType, ConnectorSource,
        @{n='SenderIPAddresses'; e={$_.SenderIPAddresses -join ', '}},
        @{n='SenderDomains';     e={$_.SenderDomains -join ', '}},
        RequireTls, TlsSenderCertificateName,
        RestrictDomainsToCertificate, CloudServicesMailEnabled, TreatMessagesAsInternal
```

Tre konstellationer är informativa.

**En adress levererar, men nämns i ingen anslutning.** Då kommer e-posten in som vanlig internet-e-post. Det är tillåtet, men innebär att denna applikation inte får någon särbehandling och att dess meddelanden omfattas av fullständig filtrering. Om någon påstår att en anslutning är konfigurerad för detta system stämmer det uppenbarligen inte längre.

**En anslutning nämner adresser från vilka inget kommer.** Kandidaten för avveckling. Kontrollera innan du tar bort den om det handlar om säsongsbundna eller sällan använda system, och inaktivera först i stället för att ta bort direkt.

**En anslutning sätter `TreatMessagesAsInternal` eller `CloudServicesMailEnabled` till sant.** Här lönar det sig att titta noga. Båda inställningarna gör att meddelanden via denna väg behandlas som interna för organisationen. Om e-post från internet kommer in den vägen kringgår den därmed kontroller som är avsedda för externa meddelanden, bland annat skyddet mot förfalskade avsändare från den egna domänen. För en ren hybridanslutning är det rätt; för en anslutning via vilken godtyckliga system levererar är det ett fynd.

## Vad du vanligtvis hittar

Från praktiken, utan anspråk på fullständighet: en testanslutning från en migrering som har varit aktiv i åratal. En verksamhetsapplikation som skickar direkt till tjänsten, trots att alla tror att den går via gatewayen. En nyhetsbrevstjänsteleverantör vars avtal har löpt ut, men som fortfarande får leverera. Och regelbundet en anslutning med vidöppna villkor som någon en gång skapade för att lösa ett akut problem.

Inget av dessa fynd är dramatiskt i sig. Tillsammans ger de bilden av en miljö som ingen längre har full överblick över, och det är just den verkliga risken.

## Metodens begränsningar

Det finns tre begränsningar du bör känna till.

Meddelandespårningen via cmdleten sträcker sig bara ungefär tio dagar tillbaka. För längre perioder behöver du den historiska sökningen, som körs asynkront och täcker upp till 90 dagar. Annars missar du sällsynta system som skickar månadsvis.

`FromIP` betyder inte samma sak överallt. För e-post från internet är det adressen till den levererande servern. För hybride-post är det adressen till din lokala transportserver, inte den ursprungliga avsändarens. Analysen visar alltså den **sista stationen före tjänsten**, inte ursprunget.

Och kopplingen till en anslutning syns inte direkt i tenanten. Du drar slutsatsen utifrån adress, certifikat och avsändardomän. För ett tillförlitligt uttalande om användningen av en enskild anslutning är anslutningsrapporten i Exchange Admin Center under Rapporter och E-postflöde den bättre källan, eftersom den aggregeras på serversidan över längre tidsperioder.

## Som återkommande kontroll

Den här analysen lämpar sig väl som kvartalsrutin. Spara resultatet och jämför vid nästa genomgång. Nya adresser i listan är antingen dokumenterade ändringar eller något du vill känna till.

Om du ändå granskar e-postkonfigurationen för dina domäner: [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) visar SPF, DKIM, DMARC och övriga e-poststandarder för valfria domäner direkt i webbläsaren, även för sido- och marknadsföringsdomäner som enligt erfarenhet glöms bort vid sådana inventeringar. Och för själva frågorna erbjuder [Kommandogeneratorn](https://rafaelpfister.ch/tools/command-builder) färdiga byggblock för PowerShell och Unix-skal.

Hur du följer upp enskilda iögonfallande meddelanden beskrivs i [Analysera e-postflödet i Exchange: Message Tracking, SMTP-protokoll och Receive-anslutningar](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Källor

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): Fältlista inklusive FromIP och ToIP samt sidlogiken med StartingRecipientAddress.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): asynkron meddelandespårning över upp till 90 dagar för äldre tidsperioder.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector): parametrar för inkommande anslutningar, bland annat SenderIPAddresses och TreatMessagesAsInternal.

4.  [Konfigurera e-postflöde med anslutningar i Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): samspelet mellan anslutningstyperna och när respektive typ används.

5.  [RFC 1918: Adresstilldelning för privata internet](https://www.rfc-editor.org/rfc/rfc1918): definierar de privata adressområden som du måste skilja från offentliga adresser i analysen.
