---
title: "VPN Split Tunneling för Microsoft Teams: led medietrafik förbi tunneln"
navTitle: "Teams Split Tunneling"
description: "Teams-samtal via VPN drabbas av latens, jitter och omvägen via VPN-gatewayen. Artikeln visar vilka Microsoft-nät och portar som ansvarar för medietrafiken, varför IP-baserad split tunneling är bättre än appundantag och hur implementeringen ser ut i konsument-VPN, WireGuard, OpenVPN och företagsklienter."
date: "2026-08-26"
kategorie: "Microsoft Teams"
timeToRead: "8 min. läsning"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
protokolle:
  - "tcp"
hauptthema: "microsoft-teams"
slug: "vpn-split-tunneling-for-microsoft-teams-led-medietrafiken-forbi-tunneln"
translationId: "article-d15f1e7ff6af231c"
translationOf: vpn-split-tunneling-microsoft-teams
translationSourceHash: 9a96126164a12f1f411065507313733f451161a25c0cd051778cc306fd32ed4c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:26:50.259Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/vpn-split-tunneling-for-microsoft-teams-led-medietrafiken-forbi-tunneln
---

Ett Teams-samtal via en VPN-anslutning låter ofta sämre än utan: rösten hackar, videon rycker och skärmdelning byggs upp med fördröjning. Orsaken är oftast omvägen som realtidstrafiken tar genom VPN-tunneln, inte Teams självt. Microsoft har därför i flera år rekommenderat att Teams-medietrafik leds direkt ut till internet förbi VPN:et via split tunneling. Detta tillvägagångssätt fungerar med praktiskt taget alla VPN-produkter, från konsumentklienter till företagsgatewayer; konfigurationen skiljer sig bara åt i detaljer.

## Varför realtidstrafik lider i tunneln

Teams-ljud och -video använder SRTP, ett UDP-baserat protokoll som är beroende av låg latens och låg jitter. Microsoft anger som målvärden mindre än 100 ms rundresetid till närmaste Microsoft-nätingång och mindre än 30 ms jitter. En VPN-tunnel försämrar båda värdena på flera sätt.

För det första förlänger tunneln vägen: I stället för att gå direkt till närmaste Microsoft-ingångspunkt går trafiken först till VPN-gatewayen, som kan finnas i leverantörens eller företagets datacenter, och först därefter till Microsoft. För det andra kostar det extra krypteringslagret beräkningstid och ökar overheaden per paket; medieströmmen är redan krypterad med SRTP och VPN-krypteringen tillkommer som ett andra lager. För det tredje är VPN-gatewayen en delad flaskhals: Under hög belastning delar alla användare dess bandbredd och paketbuffertar, vilket skapar just den jitter som realtidstrafik är mest känslig för. För det fjärde blockerar vissa VPN-konfigurationer UDP helt eller tvingar fram TCP; Teams faller då tillbaka till TCP 443, vilket ytterligare försämrar kvaliteten eftersom TCP-omöverföringar är olämpliga för realtidsmedia.

För övrig Teams-trafik (inloggning, chatt, filåtkomst) spelar detta knappast någon roll, eftersom den inte är realtidskänslig. Det räcker därför att specifikt undanta medietrafiken.

## Relevanta nät och portar

Microsoft publicerar alla Microsoft 365-slutpunkter i maskinläsbart format och delar in dem i kategorierna Optimize, Allow och Default. För split tunneling är kategorin Optimize relevant: Den omfattar de få latenskritiska slutpunkterna med fasta IP-nät, som tillsammans står för merparten av volymen. För Teams-media är detta slutpunkts-ID 11 i den officiella listan. ID 12 omfattar samma IPv4-nät och ytterligare IPv6-nät för TCP 443 och 80 samt UDP 443, men tillhör kategorin Allow. Det tidigare också angivna nätet `13.107.64.0/18` tog Microsoft bort från båda posterna 2024 (version 2024053000 för ID 12, 2024073100 för ID 11):

| Nät | Protokoll och portar | Syfte |
|---|---|---|
| `52.112.0.0/14` | UDP 3478 till 3481 (ID 11), TCP 443 (ID 12) | Teams-media och transportreläer |
| `52.122.0.0/15` | UDP 3478 till 3481 (ID 11), TCP 443 (ID 12) | Teams-media och transportreläer |
| `2603:1063::/38` | UDP 3478 till 3481 (ID 11), TCP 443 (ID 12) | samma tjänster via IPv6 |

De fyra UDP-portarna motsvarar medieklasserna ljud (3478), video (3479 och 3480) och skärmdelning (3481); TCP 443 är reservvägen. Den som använder IPv6 bör även undanta IPv6-nätet, annars går en del av anslutningarna ändå genom tunneln.

Dessa nät är medvetet stabila: Microsoft aviserar ändringar av Optimize-slutpunkterna via Endpoint-webbtjänsten och håller listan kort, just för att företag ska kunna gjuta in dem i routing- och brandväggsregler. Trots det bör en regelbunden avstämning med den officiella listan ingå i driftrutinen.

## App-baserat eller IP-baserat: två metoder med olika styrkor

Många VPN-klienter erbjuder två typer av split tunneling: undantag per applikation eller undantag per mål-IP.

Appundantaget låter självklart, men har två svagheter för Teams. Nya Teams är en WebView2-applikation: huvudprocessen heter `ms-teams.exe`, men en del av trafiken går via `msedgewebview2.exe`. Den som bara undantar huvudprocessen fångar inte all trafik; den som även undantar WebView2 leder också trafiken från andra WebView2-applikationer (exempelvis nya Outlook) förbi tunneln. Och för Teams i webbläsaren hjälper appundantaget inte alls, såvida man inte undantar hela webbläsaren, vilket gör att all webbtrafik kringgår VPN:et.

Det IP-baserade undantaget fungerar däremot på nätverksnivå och är därmed oberoende av om trafiken kommer från Teams-appen, WebView2 eller en webbläsarflik. Det undantar exakt det som är latenskritiskt och låter inloggning, chatt och övrig webbtrafik stanna i tunneln. För Teams är därför den IP-baserade metoden det bättre valet; appundantaget lämpar sig som komplement när verkligen all Teams-trafik ska kringgå VPN:et.

## Implementering i vanliga VPN-produkter

Principen är densamma överallt: De två IPv4-näten (och vid behov IPv6-nätet) undantas från tunneln, så att operativsystemets rutter för dessa mål pekar på det fysiska gränssnittet.

**Konsument-VPN (Proton VPN, NordVPN, Surfshark och liknande):** Windows- och Android-klienterna har oftast ett menyval som ”Split Tunneling” med en undantagslista för IP-adresser eller subnät. Ange näten där i CIDR-notation och återanslut VPN-anslutningen så att rutterna börjar gälla. På macOS och iOS saknas funktionen hos de flesta leverantörer, eftersom system-API:erna där inte tillåter applikationsstyrd split tunneling i denna form.

**WireGuard:** WireGuard har ingen undantagslista utan endast inställningen `AllowedIPs`, som avgör vad som går in i tunneln. Undantag skapas genom att `0.0.0.0/0` ersätts med listan över alla nät som inte innehåller undantagsområdet. Ingen beräknar denna komplementärlista för hand; onlineverktyg som WireGuard AllowedIPs Calculator använder `0.0.0.0/0` som bas, Microsoft-näten som ”Disallowed IPs” och ger den färdiga raden för konfigurationsfilen.

**OpenVPN:** När `redirect-gateway` är aktivt har mer specifika rutter företräde. Två extra rader i klientkonfigurationen leder Microsoft-näten förbi tunneln:

```text
route 52.112.0.0 255.252.0.0 net_gateway
route 52.122.0.0 255.254.0.0 net_gateway
```

`net_gateway` står här för standardgatewayen i det lokala nätet, inte VPN-gatewayen.

**Företagsklienter (Cisco Secure Client/AnyConnect, Palo Alto GlobalProtect, Fortinet FortiClient):** Här konfigurerar företaget undantagen centralt, hos Cisco som en ”Split Exclude”-lista i grupprincipen och hos GlobalProtect som ”Exclude Access Route”. Microsoft dokumenterar uttryckligen detta tillvägagångssätt som den rekommenderade modellen för Microsoft 365-trafik och tillhandahåller Optimize-listan via Endpoint-webbtjänsten, så att undantagen kan hållas aktuella automatiskt. Den som som medarbetare sitter bakom ett företags-VPN kan alltså inte själv ange undantaget utan måste begära det från nätverksteamet; Microsoft-dokumentet om detta är ett lämpligt underlag för argumentationen.

**Inbyggda Windows-verktyg:** I en VPN-anslutning som har konfigurerats med Windows inbyggda verktyg i split-läge (`Set-VpnConnection -SplitTunneling $true`) hamnar endast de nät som har angetts med `Add-VpnConnectionRoute` i tunneln. Så länge Microsoft-näten inte finns där går de automatiskt direkt; ett uttryckligt undantag är då onödigt.

## Säkerhetsavvägning: vad som går förbi tunneln

Split tunneling är en medveten uppmjukning av principen att leda all trafik genom tunneln. Innan implementeringen bör du klargöra tre punkter.

Din egen offentliga IP-adress blir synlig för Microsoft, eftersom det är precis vad som är avsikten: Medieströmmen ska ta den kortaste vägen. Den som främst använder VPN för att dölja sin plats ger upp detta skydd för Teams-samtal. Innehållet påverkas inte av detta, eftersom SRTP krypterar medieströmmen end-to-end mellan klienten och Microsofts infrastruktur.

I företagsmiljöer förlorar den centrala säkerhetsgatewayen insyn i den undantagna trafiken: TLS-inspektion, IDS-signaturer och volymanalys gäller inte längre för dessa nät. Eftersom undantaget är begränsat till några få, fast Microsoft-tilldelade nät med definierade portar bedömer Microsoft denna kvarvarande risk som låg; Optimize-slutpunkterna är kuraterade just för detta. Ett generellt undantag för hela applikationer eller till och med webbläsaren har däremot en betydligt större angreppsyta och bör undvikas i företagsmiljöer.

Slutligen Kill Switch: Vissa VPN-klienter tillämpar split-tunneling-undantag först efter en återanslutning eller beter sig annorlunda när Kill Switch är aktiv. Efter varje ändring i undantagslistan bör därför en återanslutning och ett kontrolltest genomföras.

## Kontroll: går medietrafiken verkligen direkt?

Om undantaget fungerar kan kontrolleras på två nivåer. På routningsnivå visar PowerShell vilket gränssnitt Windows väljer för ett mål i Microsoft-näten:

```powershell
Find-NetRoute -RemoteIPAddress 52.112.1.1 |
  Select-Object InterfaceAlias, NextHop
```

Om det fysiska gränssnittet (Ethernet eller Wi-Fi) visas här i stället för VPN-adaptern är rutten korrekt. På applikationsnivå ger Teams självt bekräftelsen: Under ett samtal visar samtalskvaliteten (under ”Fler åtgärder” i samtalsfönstret) den förhandlade anslutningstypen, rundresetiden och paketförlusten. En rundresetid som sjunker märkbart efter ändringen och anslutningstypen UDP i stället för TCP är de två kännetecknen på ett fungerande undantag.

Om trafiken fortsätter att gå genom tunneln trots korrekt rutt är det värt att kontrollera nätverksadaptrarnas ordning och klientens särdrag: Vissa VPN-klienter tvingar på nytt fram sina rutter med lägre metrik efter varje anslutning, och en föråldrad undantagslista märks först när Microsoft lägger till ett nät. Avstämningen med den officiella slutpunktslistan bör därför ske i samma rytm som andra återkommande nätverkskontroller.

## Ändringshistorik

- **25 oktober 2026:** Korrigerat: Microsoft tog bort nätet 13.107.64.0/18 från slutpunktslistan 2024; det har strukits från tabellen och OpenVPN-exemplet. Även korrigerat: Endast ID 11 tillhör kategorin Optimize, ID 12 tillhör kategorin Allow.

## Källor

1.  [Microsoft: Office 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges): officiell slutpunktslista; Teams-medienäten finns under ID 11 i kategorin Optimize, medan ID 12 med samma nät finns i kategorin Allow.

2.  [Microsoft: Implementing VPN split tunneling for Microsoft 365](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-vpn-implement-split-tunnel): Microsofts implementeringsguide för företags-VPN, inklusive motivering av riskbedömningen och exempel med Teams-medienäten `52.112.0.0/14`, `52.122.0.0/15` och `2603:1063::/38`.

3.  [Microsoft: Microsoft 365 network connectivity principles](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-network-connectivity-principles): principerna bakom lokal internetutgång, inklusive latensmålvärden för realtidsmedia.

4.  [Proton VPN: How to use split tunneling](https://protonvpn.com/support/protonvpn-split-tunneling/): exempel på en konsumentklient med IP- och app-baserad split tunneling i Windows och Android.

5.  [WireGuard AllowedIPs Calculator](https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/): kalkylator för komplementärlistan när undantag måste hanteras via AllowedIPs.

6.  [Microsoft 365 IP and URL web service: changes Worldwide](https://endpoints.office.com/changes/worldwide/2024010100?clientrequestid=b10c5ed1-bad1-445f-b386-b919946339a7): ändringslogg för slutpunktslistan, inklusive borttagningen av `13.107.64.0/18` från ID 12 (version 2024053000) och ID 11 (version 2024073100).
