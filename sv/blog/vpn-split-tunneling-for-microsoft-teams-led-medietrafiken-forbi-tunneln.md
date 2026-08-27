---
title: "VPN-split tunneling för Microsoft Teams: led medietrafiken förbi tunneln"
navTitle: "Teams Split Tunneling"
description: "Teams-samtal över ett VPN drabbas av latens, jitter och omvägen via VPN-gatewayen. Artikeln visar vilka Microsoft-nät och portar som ansvarar för medietrafiken, varför IP-baserad split tunneling är bättre än appundantag och hur implementeringen ser ut i konsument-VPN, WireGuard, OpenVPN och företagsklienter."
date: "2026-08-26"
kategorie: "Microsoft Teams"
timeToRead: "8 min läsning"
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
aiPrompt: |
  Du bist mein Netzwerk-Assistent. Ich will Microsoft-Teams-Medienverkehr per Split Tunneling an meinem VPN vorbeiführen. Hilf mir Schritt für Schritt: 1. Frage mich, welchen VPN-Client ich einsetze (Consumer-VPN, WireGuard, OpenVPN, Enterprise-Client) und auf welchem Betriebssystem. 2. Nenne mir die passende Konfiguration für die drei Optimize-Netze 13.107.64.0/18, 52.112.0.0/14 und 52.122.0.0/15 (UDP 3478 bis 3481, TCP 443). 3. Erkläre mir, wie ich mit Find-NetRoute oder der Anrufintegrität in Teams prüfe, ob der Medienverkehr tatsächlich am Tunnel vorbeiläuft. 4. Weise mich auf die Sicherheitsabwägungen hin, bevor ich die Ausnahme produktiv setze.
translationOf: vpn-split-tunneling-microsoft-teams
url: https://rafaelpfister.ch/sv/blog/vpn-split-tunneling-for-microsoft-teams-led-medietrafiken-forbi-tunneln
translationSourceHash: 95e3cefa4946676022602866d6ef21ab92ef25ec8c5dd3ff4ab0219ba718a880
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:33:08.640Z
translationReview: automatic
---

Ett Teams-samtal över en VPN-anslutning låter ofta sämre än utan: rösten hackar, videon rycker och skärmdelningar laddas långsamt. Orsaken är oftast omvägen som realtidstrafiken tar genom VPN-tunneln, inte Teams i sig. Microsoft har därför i flera år rekommenderat att Teams medietrafik leds direkt till internet förbi VPN:t med hjälp av split tunneling. Detta tillvägagångssätt fungerar med i princip alla VPN-produkter, från konsumentklienter till företagsgatewayar; konfigurationen skiljer sig bara åt i detaljerna.

## Varför realtidstrafik lider i tunneln

Teams-ljud och -video använder SRTP, ett UDP-baserat protokoll som är beroende av låg latens och lågt jitter. Microsoft anger som målvärden under 100 ms tur- och returtid till närmaste Microsoft-nätverksingång och under 30 ms jitter. En VPN-tunnel försämrar båda värdena på flera sätt.

För det första förlänger tunneln vägen: I stället för att gå direkt till den geografiskt närmaste Microsoft-ingångspunkten går trafiken först till VPN-gatewayen, som kan finnas i leverantörens eller företagets datacenter, och först därefter till Microsoft. För det andra kräver det extra krypteringslagret beräkningstid och ökar overheaden per paket; medieströmmen är redan krypterad med SRTP, och VPN-krypteringen tillkommer som ett andra lager. För det tredje är VPN-gatewayen en delad flaskhals: Under belastningstoppar delar alla användare dess bandbredd och paketbuffertar, vilket skapar just det jitter som realtidstrafik är mest känslig för. För det fjärde blockerar vissa VPN-konfigurationer UDP helt eller tvingar fram TCP; Teams växlar då till TCP 443, vilket försämrar kvaliteten ytterligare eftersom TCP-omöverföringar är olämpliga för realtidsmedia.

För övrig Teams-trafik (inloggning, chatt, filåtkomst) spelar detta knappast någon roll eftersom den inte är realtidskänslig. Det räcker därför att undanta just medietrafiken.

## Relevanta nät och portar

Microsoft publicerar alla Microsoft 365-slutpunkter i maskinläsbart format och delar in dem i kategorierna Optimize, Allow och Default. För split tunneling är kategorin Optimize relevant: Den omfattar de få latenskritiska slutpunkterna med fasta IP-nät, som tillsammans står för den största delen av volymen. För Teams-media är detta slutpunkts-ID 11 och 12 i den officiella listan:

| Nät | Protokoll och portar | Syfte |
|---|---|---|
| `13.107.64.0/18` | UDP 3478 till 3481, TCP 443 | Teams-media (ljud, video, skärmdelning) |
| `52.112.0.0/14` | UDP 3478 till 3481, TCP 443 | Teams-media och transportreläer |
| `52.122.0.0/15` | UDP 3478 till 3481, TCP 443 | Teams-media och transportreläer |
| `2603:1063::/38` | UDP 3478 till 3481, TCP 443 | samma tjänster via IPv6 |

De fyra UDP-portarna motsvarar medieklasserna ljud (3478), video (3479 och 3480) och skärmdelning (3481); TCP 443 är reservvägen. Den som använder IPv6 bör även undanta IPv6-nätet, annars går en del av anslutningarna ändå genom tunneln.

Dessa nät är medvetet stabila: Microsoft meddelar ändringar i Optimize-slutpunkterna via Endpoint-webbtjänsten och håller listan kort, just för att företag ska kunna lägga in dem i routnings- och brandväggsregler. Trots det bör en regelbunden jämförelse med den officiella listan ingå i driftrutinen.

## App-baserat eller IP-baserat: två metoder med olika styrkor

Många VPN-klienter erbjuder två typer av split tunneling: undantag per applikation eller undantag per mål-IP.

Appundantag låter självklart men har två svagheter för Teams. Nya Teams är en WebView2-applikation: huvudprocessen heter `ms-teams.exe`, men en del av trafiken går via `msedgewebview2.exe`. Den som bara undantar huvudprocessen fångar inte all trafik; den som även undantar WebView2 leder också trafik från andra WebView2-applikationer, exempelvis nya Outlook, förbi tunneln. För Teams i webbläsaren hjälper appundantaget inte alls, såvida inte hela webbläsaren undantas, vilket innebär att all webbtrafik kringgår VPN:t.

IP-baserade undantag fungerar däremot på nätverksnivå och är därmed oberoende av om trafiken kommer från Teams-appen, WebView2 eller en webbläsarflik. Det undantar exakt det som är latenskritiskt och låter inloggning, chatt och övrig webbtrafik stanna i tunneln. För Teams är den IP-baserade metoden därför det bättre valet; appundantag lämpar sig som komplement när all Teams-trafik verkligen ska kringgå VPN:t.

## Implementering i vanliga VPN-produkter

Principen är densamma överallt: De tre IPv4-näten, och vid behov IPv6-nätet, undantas från tunneln så att operativsystemets rutter för dessa mål pekar på det fysiska gränssnittet.

**Konsument-VPN (Proton VPN, NordVPN, Surfshark och liknande):** Windows- och Android-klienterna erbjuder vanligtvis ett menyalternativ som ”Split Tunneling” med en undantagslista för IP-adresser eller subnät. Ange de tre näten där i CIDR-notation och återanslut VPN-anslutningen så att rutterna börjar gälla. På macOS och iOS saknas funktionen hos de flesta leverantörer eftersom system-API:erna där inte tillåter applikationsstyrd split tunneling i denna form.

**WireGuard:** WireGuard har ingen undantagslista, utan bara inställningen `AllowedIPs`, som fastställer vad som går in i tunneln. Undantag skapas genom att `0.0.0.0/0` ersätts med listan över alla nät som inte innehåller undantagsområdet. Ingen beräknar denna komplementärlista manuellt; onlinekalkylatorer som WireGuard AllowedIPs Calculator använder `0.0.0.0/0` som bas, de tre Microsoft-näten som ”Disallowed IPs” och levererar den färdiga raden för konfigurationsfilen.

**OpenVPN:** När `redirect-gateway` är aktivt har mer specifika rutter företräde. Tre ytterligare rader i klientkonfigurationen leder Microsoft-näten förbi tunneln:

```text
route 13.107.64.0 255.255.192.0 net_gateway
route 52.112.0.0 255.252.0.0 net_gateway
route 52.122.0.0 255.254.0.0 net_gateway
```

`net_gateway` avser standardgatewayen i det lokala nätet, inte VPN-gatewayen.

**Företagsklienter (Cisco Secure Client/AnyConnect, Palo Alto GlobalProtect, Fortinet FortiClient):** Här konfigurerar företaget undantagen centralt, hos Cisco som en ”Split Exclude”-lista i gruppprincipen och hos GlobalProtect som ”Exclude Access Route”. Microsoft dokumenterar uttryckligen detta tillvägagångssätt som rekommenderad modell för Microsoft 365-trafik och tillhandahåller Optimize-listan via Endpoint-webbtjänsten, så att undantagen kan hållas aktuella automatiskt. Den som som medarbetare sitter bakom ett företags-VPN kan alltså inte själv ange undantaget, utan måste begära det från nätverksteamet; Microsoft-dokumentet om detta är ett lämpligt underlag för argumentationen.

**Windows inbyggda verktyg:** I en VPN-anslutning som konfigurerats med Windows inbyggda verktyg i split-läge (`Set-VpnConnection -SplitTunneling $true`) hamnar endast de nät som lagts till med `Add-VpnConnectionRoute` i tunneln. Så länge Microsoft-näten inte finns där går de automatiskt direkt; ett uttryckligt undantag behövs då inte.

## Säkerhetsavvägning: vad som går förbi tunneln

Split tunneling är en medveten uppmjukning av principen att leda all trafik genom tunneln. Innan implementeringen bör du klargöra tre punkter.

Din offentliga IP-adress blir synlig för Microsoft, eftersom det är precis vad som avses: Medieströmmen ska ta den kortaste vägen. Den som främst använder ett VPN för att dölja sin plats ger upp detta skydd för Teams-samtal. Innehållet påverkas inte av detta eftersom SRTP krypterar medieströmmen hela vägen mellan klienten och Microsoft-infrastrukturen.

I företagsmiljöer förlorar den centrala säkerhetsgatewayen insyn i den undantagna trafiken: TLS-inspektion, IDS-signaturer och volymanalys gäller inte längre för dessa nät. Eftersom undantaget är begränsat till få, fast Microsoft-tilldelade nät med definierade portar bedömer Microsoft denna kvarvarande risk som låg; Optimize-slutpunkterna är kuraterade just för detta. Ett generellt undantag för hela applikationer eller till och med webbläsaren har däremot en betydligt större angreppsyta och bör undvikas i företagsmiljöer.

Slutligen Kill Switch: Vissa VPN-klienter tillämpar split-tunneling-undantag först efter en återanslutning, eller beter sig annorlunda när Kill Switch är aktiv. Efter varje ändring i undantagslistan bör du därför återansluta och genomföra ett kontrolltest.

## Kontroll: går medietrafiken verkligen direkt?

Om undantaget fungerar kan kontrolleras på två nivåer. På routningsnivå visar PowerShell vilket gränssnitt Windows väljer för ett mål i Microsoft-näten:

```powershell
Find-NetRoute -RemoteIPAddress 52.112.1.1 |
  Select-Object InterfaceAlias, NextHop
```

Om det fysiska gränssnittet (Ethernet eller WLAN) visas i stället för VPN-adaptern är routningen korrekt. På applikationsnivå ger Teams självt bekräftelsen: Under ett samtal visar samtalsintegriteten, under ”Fler åtgärder” i samtalsfönstret, den förhandlade anslutningstypen, tur- och returtiden och paketförlustgraden. En tur- och returtid som sjunker tydligt efter ändringen och anslutningstypen UDP i stället för TCP är de två kännetecknen på ett fungerande undantag.

Om trafiken trots korrekt rutt stannar i tunneln är det värt att kontrollera nätverksadaptrarnas ordning och klientens särdrag: Vissa VPN-klienter tvingar fram sina rutter med lägre metrik på nytt efter varje anslutning, och en föråldrad undantagslista märks först när Microsoft lägger till ett nät. Jämförelsen med den officiella slutpunktslistan bör därför ingå i samma rutin som andra återkommande nätverkskontroller.

## Källor

1.  [Microsoft: Office 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges): officiell slutpunktslista; Teams medienät finns under ID 11 och 12 i kategorin Optimize.

2.  [Microsoft: Implementing VPN split tunneling for Microsoft 365](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-vpn-implement-split-tunnel): Microsofts implementeringsguide för företags-VPN, inklusive motivering av riskbedömningen.

3.  [Microsoft: Microsoft 365 network connectivity principles](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-network-connectivity-principles): principerna bakom lokal internetutgång, inklusive latensmålvärden för realtidsmedia.

4.  [Proton VPN: How to use split tunneling](https://protonvpn.com/support/protonvpn-split-tunneling/): exempel på en konsumentklient med IP- och app-baserad split tunneling i Windows och Android.

5.  [WireGuard AllowedIPs Calculator](https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/): kalkylator för komplementärlistan när undantag måste hanteras via AllowedIPs.
