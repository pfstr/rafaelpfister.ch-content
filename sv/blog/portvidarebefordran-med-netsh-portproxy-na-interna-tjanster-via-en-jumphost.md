---
title: "Portvidarebefordran med netsh portproxy: nå interna tjänster via en jumphost"
navTitle: "netsh portproxy"
description: "Windows har inbyggd TCP-portvidarebefordran med netsh interface portproxy. Tillsammans med ett VPN som Tailscale kan du nå en intern tjänst, till exempel ett NAS-gränssnitt, utifrån utan att exponera den offentligt. Så här konfigurerar, säkrar och tar du bort vidarebefordran, samt dess begränsningar: ingen UDP, ingen extra kryptering och fallgropar med certifikat och omdirigeringar."
date: "2026-09-02"
kategorie: "Windows och nätverk"
timeToRead: "9 min läsning"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "tcp"
  - "haertung"
slug: "portvidarebefordran-med-netsh-portproxy-na-interna-tjanster-via-en-jumphost"
translationId: "article-236adcb4ae982572"
translationOf: windows-portproxy-portweiterleitung
translationSourceHash: 152d2f6d20774179128718ab4e218fcc0cd76cbcf0706060df7d60f30e012900
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:55:59.451Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/portvidarebefordran-med-netsh-portproxy-na-interna-tjanster-via-en-jumphost
---

# Portvidarebefordran med netsh portproxy: nå interna tjänster via en jumphost

En intern tjänst lyssnar ofta bara i det lokala nätverket: webbgränssnittet för en NAS, en skrivares kontrollpanel eller en administrationssida. Om du vill komma åt den utifrån utan att lägga ut tjänsten på internet behöver du en väg via en dator som ser båda sidorna. Windows har ett inbyggt verktyg för detta: `netsh interface portproxy` vidarebefordrar inkommande TCP-anslutningar till ett annat mål. I kombination med ett VPN som Tailscale eller WireGuard blir en dator i målnätet en jumphost genom vilken du når den interna tjänsten.

Ett konkret exempel: En NAS med webbgränssnittet på `10.0.0.245:5000` kan bara nås i det lokala nätverket. I samma nätverk finns en Windows-dator som dessutom kan nås via VPN. Konfigurera en portvidarebefordran på den här datorn från dess VPN-adress till NAS:en och öppna sedan NAS-gränssnittet i webbläsaren via datorns VPN-adress. Tjänsten stannar i det interna nätverket; endast jumphosten kan nås via VPN.

## Så fungerar portproxy

`portproxy` är en del av tjänsten IP Helper (`iphlpsvc`). Tjänsten tar emot anslutningar på en lokal port och skickar dem vidare till ett mål. Det är en ren TCP-reläfunktion på applikationsnivå: ingen brandväggs-NAT-regel, utan en process som kopierar byte mellan två anslutningar. Om `iphlpsvc` inte körs fungerar ingen vidarebefordran. Tjänsten finns som standard; starttypen bör vara automatisk om vidarebefordran ska överleva en omstart.

## Konfigurera

En vidarebefordran kräver två steg: portproxy-regeln och en brandväggsregel som tillåter åtkomst till lyssnaren. Kör båda i Kommandotolken eller PowerShell med administratörsbehörighet.

Först vidarebefordran. Den binder till en lokal adress och en port och hänvisar till mål-IP och målport:

```powershell
netsh interface portproxy add v4tov4 listenaddress=100.100.10.10 listenport=5000 connectaddress=10.0.0.245 connectport=5000
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `v4tov4` | IPv4 lyssnar, IPv4 ansluter; även möjligt: `v4tov6`, `v6tov4`, `v6tov6` |
| `listenaddress` | Lokal adress som lyssnar på; här jumphostens VPN-adress, så att inkommande trafik endast sker via VPN |
| `listenport` | Lokal port som lyssnar på |
| `connectaddress` | Mål-IP som trafiken vidarebefordras till (den interna tjänsten) |
| `connectport` | Målport på den interna tjänsten |

</details>

Att binda till VPN-adressen i stället för `0.0.0.0` är den första säkerhetsåtgärden: lyssnaren visas bara på VPN-gränssnittet, inte på jumphostens alla nätverkskort. Den andra säkerhetsåtgärden är brandväggen. Öppna lyssnarporten enbart för ditt VPN:s adressområde, inte för alla adresser:

```powershell
New-NetFirewallRule -DisplayName "NAS-Proxy (VPN)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5000 -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-Direction Inbound` | Regel för inkommande trafik |
| `-Protocol TCP` | portproxy vidarebefordrar endast TCP, alltså TCP |
| `-LocalPort 5000` | Lyssnarporten från portproxy-regeln |
| `-RemoteAddress 100.64.0.0/10` | Endast källor från detta område tillåts; här Tailscale-området, annars CIDR-blocket för ditt VPN |

</details>

## Kontrollera och använd

Kontrollera först på själva jumphosten om den interna tjänsten över huvud taget går att nå, och visa den aktiva vidarebefordran:

```powershell
Test-NetConnection -ComputerName 10.0.0.245 -Port 5000
netsh interface portproxy show v4tov4
```

Om målet svarar och regeln finns i listan testar du från din fjärrenhet. Tjänsten kan nu nås via jumphostens adress och port:

```powershell
Test-NetConnection -ComputerName 100.100.10.10 -Port 5000
```

I webbläsaren öppnar du sedan `http://100.100.10.10:5000`. Behöver du flera portar för samma tjänst, till exempel 5000 och 5001 för http och https, skapar du en egen portproxy-regel och lämplig brandväggsregel för varje port.

## Översikt i manpage-stil

De viktigaste underkommandona för `netsh interface portproxy`:

<details class="options-details">
<summary>Översikt över alternativ</summary>

| Kommando | Syfte |
|---|---|
| `add v4tov4 …` | Skapa vidarebefordran (listenaddress/listenport → connectaddress/connectport) |
| `show v4tov4` | Visa aktiva IPv4-vidarebefordringar |
| `show all` | Visa alla vidarebefordringar för alla protokollvarianter |
| `delete v4tov4 listenaddress=… listenport=…` | Ta bort en vidarebefordran |
| `reset` | Ta bort alla portproxy-regler |

</details>

Reglerna lagras i registret under `HKLM\SYSTEM\CurrentControlSet\Services\PortProxy` och överlever en omstart. De syns bara via `netsh` eller direkt i registret, inte i det grafiska brandväggsgränssnittet.

## Alternativ

`portproxy` är praktiskt när jumphosten ändå kör Windows och du inte vill installera något ytterligare. Två alternativ löser samma problem med andra egenskaper.

En SSH-tunnel med lokal vidarebefordran (`ssh -L 5000:10.0.0.245:5000 benutzer@jumphost`) krypterar sträckan fram till själva jumphosten och fungerar plattformsoberoende. Den kräver en SSH-server på jumphosten och finns bara så länge SSH-sessionen är aktiv.

En Tailscale-subnätsrouter (`tailscale set --advertise-routes=10.0.0.0/24`) gör hela det interna subnätet tillgängligt för dina VPN-enheter. Då adresserar du den interna tjänsten direkt med dess verkliga IP, utan vidarebefordran per port. Det är den rakaste vägen om du vill nå flera interna enheter, men kräver att rutten godkänns i Tailscale-administrationen.

## Begränsningar

En portvidarebefordran med portproxy löser åtkomsten, men har tydliga begränsningar som du bör känna till före användning:

- **Endast TCP.** `portproxy` vidarebefordrar uteslutande TCP. Tjänster som behöver UDP (DNS, många VPN- och spelprotokoll samt viss videoöverföring) kan inte hanteras med detta.
- **Ingen extra kryptering.** Vidarebefordran kopierar byte oförändrade. Sekretessen för anslutningen tillhandahålls enbart av VPN:et som du använder för att nå jumphosten. Över ett okrypterat transportnät skulle trafiken vara oskyddad.
- **Certifikatvarning för HTTPS via IP-adressen.** Om du vidarebefordrar en HTTPS-tjänst och öppnar den via jumphostens IP-adress matchar målets certifikat inte den anropade adressen. Webbläsaren varnar. Det är acceptabelt för ett kort test, men inte för permanent drift.
- **Omdirigeringar och absoluta adresser.** Vissa webbgränssnitt omdirigerar själva till sitt värdnamn eller en annan port, eller bygger absoluta länkar med sin interna adress. Då fungerar åtkomsten via jumphosten inte, trots att vidarebefordran finns. Sådana tjänster behöver en riktig omvänd proxy i stället för ett rent portrelä.
- **Bindning till en adress som måste finnas vid start.** Om regeln binder till en viss `listenaddress` måste adressen finnas när tjänsten startar. Om VPN-gränssnittet kommer upp först senare kan bindningen misslyckas tills tjänsten eller regeln återställs.
- **En ytterligare väg in i det interna nätverket.** Varje vidarebefordran är en väg från utsidan till en intern tjänst. Begränsa brandväggen strikt till VPN-området, bind till VPN-adressen och ta bort vidarebefordran så snart du inte längre behöver den.

## Ta bort igen

Ta bort vidarebefordran och brandväggsregeln när arbetet är klart:

```powershell
netsh interface portproxy delete v4tov4 listenaddress=100.100.10.10 listenport=5000
Remove-NetFirewallRule -DisplayName "NAS-Proxy (VPN)"
```

En portvidarebefordran är ett verktyg för riktad, tidsbegränsad åtkomst, inte för en permanent öppen kanal. För permanent drift av en intern tjänst över internet är en omvänd proxy med ett giltigt certifikat eller ett VPN med subnätsroutning den mer ändamålsenliga lösningen.

## Ändringshistorik

- **25 oktober 2026:** Korrigerat: Subnätsrutter annonseras med tailscale set --advertise-routes i stället för tailscale up. Källänkar uppdaterade till de nya adresserna hos Microsoft Learn och Tailscale.

## Källor

1.  [netsh interface (Microsoft Learn)](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netsh-interface): Kommandoreferens för `netsh interface` med syntax för portproxy-underkommandon och protokollvarianter.

2.  [New-NetFirewallRule (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule?view=windowsserver2025-ps): Parametrar för brandväggsregeln, inklusive begränsning till adressområden via RemoteAddress.

3.  [Tailscale: Subnet routers](https://tailscale.com/docs/features/subnet-routers): Gör ett helt subnät tillgängligt via VPN som alternativ till vidarebefordran per port; rutter annonseras med `tailscale set --advertise-routes`.
