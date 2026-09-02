---
title: "Portvidarebefordran med netsh portproxy: nå interna tjänster via en jumphost"
navTitle: "netsh portproxy"
description: "Windows har inbyggd TCP-portvidarebefordran med netsh interface portproxy. Tillsammans med ett VPN som Tailscale kan du nå en intern tjänst, till exempel ett NAS-gränssnitt, utifrån utan att exponera den offentligt. Så här konfigurerar, säkrar och tar du bort vidarebefordran – och här är dess begränsningar: ingen UDP, ingen extra kryptering samt fallgropar med certifikat och omdirigeringar."
date: "2026-09-02"
kategorie: "Windows och nätverk"
timeToRead: "9 min. läsning"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "tcp"
  - "haertung"
slug: "portvidarebefordran-med-netsh-portproxy-na-interna-tjanster-via-en-jumphost"
translationId: "article-236adcb4ae982572"
aiPrompt: |
  Du bist mein Windows-Administrationsassistent. Hilf mir, mit netsh interface portproxy eine TCP-Portweiterleitung über einen Windows-Jumphost einzurichten, um einen internen Dienst (z. B. eine NAS-Weboberfläche) über ein VPN zu erreichen: Weiterleitung anlegen, Firewall auf den VPN-Bereich beschränken, prüfen, wieder entfernen, und die Grenzen (kein UDP, keine Verschlüsselung, Zertifikats- und Redirect-Probleme) einordnen.
translationOf: windows-portproxy-portweiterleitung
url: https://rafaelpfister.ch/sv/blog/portvidarebefordran-med-netsh-portproxy-na-interna-tjanster-via-en-jumphost
translationSourceHash: a4888a85b953fbf7b2248232b7b7361e752300872cdb570d6fd15b1cb806ef89
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:04:20.502Z
translationReview: automatic
---

# Portvidarebefordran med netsh portproxy: nå interna tjänster via en jumphost

En intern tjänst lyssnar ofta bara på det lokala nätverket: webbgränssnittet för en NAS, en skrivares kontrollpanel eller en administrationssida. Om du vill komma åt den utifrån utan att lägga ut tjänsten på internet behöver du en väg via en dator som kan se båda sidorna. Windows har ett inbyggt verktyg för detta: `netsh interface portproxy` vidarebefordrar inkommande TCP-anslutningar till ett annat mål. I kombination med ett VPN som Tailscale eller WireGuard blir en dator i målnätet en jumphost via vilken du når den interna tjänsten.

Ett konkret exempel: En NAS med webbgränssnittet på `10.0.0.245:5000` är endast åtkomlig i det lokala nätverket. I samma nätverk finns en Windows-dator som dessutom är åtkomlig via VPN. Konfigurera portvidarebefordran på den datorn från dess VPN-adress till NAS:en och öppna sedan NAS-gränssnittet i webbläsaren via datorns VPN-adress. Tjänsten stannar i det interna nätverket; endast jumphosten är åtkomlig via VPN.

## Så fungerar portproxy

`portproxy` är en del av tjänsten IP Helper (`iphlpsvc`). Tjänsten tar emot anslutningar på en lokal port och skickar dem vidare till ett mål. Det är en ren TCP-reläfunktion på applikationsnivå: ingen NAT-regel i brandväggen, utan en process som kopierar byte mellan två anslutningar. Om `iphlpsvc` inte körs fungerar ingen vidarebefordran. Tjänsten finns normalt redan; starttypen bör vara inställd på automatisk om vidarebefordran ska överleva en omstart.

## Konfigurera

En vidarebefordran kräver två steg: portproxy-regeln och en brandväggsregel som tillåter åtkomst till lyssnaren. Kör båda i Kommandotolken eller PowerShell med administratörsbehörighet.

Först vidarebefordran. Den binder till en lokal adress och port och pekar på mål-IP och målport:

```powershell
netsh interface portproxy add v4tov4 listenaddress=100.100.10.10 listenport=5000 connectaddress=10.0.0.245 connectport=5000
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `v4tov4` | IPv4 lyssnar, IPv4 ansluter; även möjligt: `v4tov6`, `v6tov4`, `v6tov6` |
| `listenaddress` | Lokal adress som tjänsten lyssnar på; här jumphostens VPN-adress, så att anslutningar endast tas emot via VPN |
| `listenport` | Lokal port som tjänsten lyssnar på |
| `connectaddress` | Mål-IP som anslutningar vidarebefordras till (den interna tjänsten) |
| `connectport` | Målport på den interna tjänsten |

</details>

Att binda till VPN-adressen i stället för `0.0.0.0` är den första säkerhetsåtgärden: lyssnaren visas endast på VPN-gränssnittet, inte på jumphostens alla nätverkskort. Den andra säkerhetsåtgärden är brandväggen. Öppna lyssnarporten uteslutande för VPN:ets adressintervall, inte för alla adresser:

```powershell
New-NetFirewallRule -DisplayName "NAS-Proxy (VPN)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5000 -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Direction Inbound` | Regel för inkommande trafik |
| `-Protocol TCP` | portproxy vidarebefordrar endast TCP, alltså TCP |
| `-LocalPort 5000` | Lyssnarporten från portproxy-regeln |
| `-RemoteAddress 100.64.0.0/10` | Endast källor från detta intervall tillåts; här Tailscale-intervallet, annars CIDR-blocket för ditt VPN |

</details>

## Kontrollera och använd

Kontrollera först på själva jumphosten om den interna tjänsten över huvud taget är åtkomlig, och visa sedan den aktiva vidarebefordran:

```powershell
Test-NetConnection -ComputerName 10.0.0.245 -Port 5000
netsh interface portproxy show v4tov4
```

Om målet svarar och regeln finns i listan testar du från din fjärrenhet. Tjänsten är nu åtkomlig via jumphostens adress och port:

```powershell
Test-NetConnection -ComputerName 100.100.10.10 -Port 5000
```

I webbläsaren öppnar du sedan `http://100.100.10.10:5000`. Behöver du flera portar för samma tjänst, till exempel 5000 och 5001 för http och https, skapar du en egen portproxy-regel och lämplig brandväggsöppning för varje port.

## Översikt i manpage-stil

De viktigaste underkommandona för `netsh interface portproxy`:

<details class="options-details">
<summary>Översikt över alternativ</summary>

| Kommando | Syfte |
|---|---|
| `add v4tov4 …` | Skapa vidarebefordran (listenaddress/listenport → connectaddress/connectport) |
| `show v4tov4` | Visa aktiva IPv4-vidarebefordringar |
| `show all` | Visa alla vidarebefordringar för samtliga protokollvarianter |
| `delete v4tov4 listenaddress=… listenport=…` | Ta bort en vidarebefordran |
| `reset` | Ta bort alla portproxy-regler |

</details>

Reglerna lagras i registret under `HKLM\SYSTEM\CurrentControlSet\Services\PortProxy` och överlever en omstart. De syns endast via `netsh` eller direkt i registret, inte i det grafiska brandväggsgränssnittet.

## Alternativ

`portproxy` är praktiskt om jumphosten redan kör Windows och du inte vill installera något extra. Två alternativ löser samma problem med andra egenskaper.

En SSH-tunnel med lokal vidarebefordran (`ssh -L 5000:10.0.0.245:5000 benutzer@jumphost`) krypterar sträckan till själva jumphosten och fungerar plattformsoberoende. Den kräver en SSH-server på jumphosten och finns bara så länge SSH-sessionen är aktiv.

En Tailscale-subnätsrouter (`tailscale up --advertise-routes=10.0.0.0/24`) gör hela det interna subnätet åtkomligt för dina VPN-enheter. Då adresserar du den interna tjänsten direkt via dess verkliga IP, utan vidarebefordran per port. Det är den rakaste vägen om du vill nå flera interna enheter, men kräver att rutten godkänns i Tailscale-administrationen.

## Begränsningar

Portvidarebefordran med portproxy löser åtkomsten, men den har tydliga begränsningar som du bör känna till före användning:

- **Endast TCP.** `portproxy` vidarebefordrar uteslutande TCP. Tjänster som behöver UDP (DNS, många VPN- och spelprotokoll samt viss videoöverföring) kan inte hanteras på detta sätt.
- **Ingen extra kryptering.** Vidarebefordran kopierar byte oförändrat. Det är endast VPN:et som du använder för att nå jumphosten som ger sträckan konfidentialitet. Över ett okrypterat transportnät skulle trafiken vara oskyddad.
- **Certifikatvarning för HTTPS via IP-adressen.** Om du vidarebefordrar en HTTPS-tjänst och öppnar den via jumphostens IP-adress matchar målets certifikat inte den anropade adressen. Webbläsaren varnar. Det kan godtas för ett kort test, men inte för kontinuerlig drift.
- **Omdirigeringar och absoluta adresser.** Vissa webbgränssnitt omdirigerar själva till sitt värdnamn eller en annan port, eller skapar absoluta länkar med sin interna adress. Då fungerar inte åtkomst via jumphosten trots att vidarebefordran är konfigurerad. Sådana tjänster behöver en riktig reverse proxy i stället för ett rent portrelä.
- **Bindning till en adress som måste finnas vid start.** Om regeln binder till en specifik `listenaddress` måste den adressen finnas när tjänsten startar. Om VPN-gränssnittet kommer upp först senare kan bindningen misslyckas tills tjänsten eller regeln sätts om.
- **En ytterligare väg in i det interna nätet.** Varje vidarebefordran är en väg utifrån till en intern tjänst. Begränsa brandväggen strikt till VPN-intervallet, bind till VPN-adressen och ta bort vidarebefordran så snart du inte längre behöver den.

## Ta bort igen

Ta bort vidarebefordran och brandväggsregeln när arbetet är klart:

```powershell
netsh interface portproxy delete v4tov4 listenaddress=100.100.10.10 listenport=5000
Remove-NetFirewallRule -DisplayName "NAS-Proxy (VPN)"
```

Portvidarebefordran är ett verktyg för riktad, tidsbegränsad åtkomst, inte för en permanent öppen kanal. För kontinuerlig drift av en intern tjänst över internet är en reverse proxy med ett giltigt certifikat eller ett VPN med subnätsrouting en mer robust lösning.

## Källor

1.  [netsh interface portproxy (Microsoft Learn)](https://learn.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-interface-portproxy): Referens för underkommandon, protokollvarianter och beroendet av IP Helper-tjänsten.

2.  [New-NetFirewallRule (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule): Parametrar för brandväggsregeln, inklusive begränsning till adressintervall via RemoteAddress.

3.  [Tailscale: Subnet routers](https://tailscale.com/kb/1019/subnets): Gör ett helt subnät åtkomligt via VPN som ett alternativ till vidarebefordran per port.
