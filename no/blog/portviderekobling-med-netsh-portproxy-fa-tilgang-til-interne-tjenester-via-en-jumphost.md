---
title: "Portviderekobling med netsh portproxy: få tilgang til interne tjenester via en jumphost"
navTitle: "netsh portproxy"
description: "Windows leveres med innebygd TCP-portviderekobling gjennom netsh interface portproxy. Sammen med en VPN-løsning som Tailscale kan du dermed nå en intern tjeneste, for eksempel et NAS-grensesnitt, utenfra uten å eksponere den offentlig. Slik setter du opp, sikrer og fjerner viderekoblingen, og dette er begrensningene: ingen UDP, ingen ekstra kryptering samt fallgruver knyttet til sertifikater og omdirigeringer."
date: "2026-09-02"
kategorie: "Windows og nettverk"
timeToRead: "9 min lesetid"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "tcp"
  - "haertung"
slug: "portviderekobling-med-netsh-portproxy-fa-tilgang-til-interne-tjenester-via-en-jumphost"
translationId: "article-236adcb4ae982572"
aiPrompt: |
  Du bist mein Windows-Administrationsassistent. Hilf mir, mit netsh interface portproxy eine TCP-Portweiterleitung über einen Windows-Jumphost einzurichten, um einen internen Dienst (z. B. eine NAS-Weboberfläche) über ein VPN zu erreichen: Weiterleitung anlegen, Firewall auf den VPN-Bereich beschränken, prüfen, wieder entfernen, und die Grenzen (kein UDP, keine Verschlüsselung, Zertifikats- und Redirect-Probleme) einordnen.
translationOf: windows-portproxy-portweiterleitung
url: https://rafaelpfister.ch/no/blog/portviderekobling-med-netsh-portproxy-fa-tilgang-til-interne-tjenester-via-en-jumphost
translationSourceHash: a4888a85b953fbf7b2248232b7b7361e752300872cdb570d6fd15b1cb806ef89
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:04:51.950Z
translationReview: automatic
---

# Portviderekobling med netsh portproxy: få tilgang til interne tjenester via en jumphost

En intern tjeneste lytter ofte bare i det lokale nettverket: webgrensesnittet til en NAS, et skriverpanel eller en administrasjonsside. Hvis du vil ha tilgang til den utenfra uten å legge tjenesten ut på internett, trenger du en vei via en maskin som ser begge sider. Windows har et innebygd verktøy for dette: `netsh interface portproxy` videresender innkommende TCP-forbindelser til et annet mål. I kombinasjon med en VPN-løsning som Tailscale eller WireGuard blir en datamaskin i målnettverket til en jumphost som gir deg tilgang til den interne tjenesten.

Et konkret eksempel: En NAS med webgrensesnitt på `10.0.0.245:5000` er bare tilgjengelig i det lokale nettverket. I samme nettverk står en Windows-PC som i tillegg er tilgjengelig via VPN. Hvis du setter opp en portviderekobling på denne PC-en fra VPN-adressen til NAS-en, kan du deretter åpne NAS-grensesnittet i nettleseren via PC-ens VPN-adresse. Tjenesten forblir i det interne nettverket, og bare jumphosten er tilgjengelig via VPN-et.

## Slik fungerer portproxy

`portproxy` er en del av tjenesten IP Helper (`iphlpsvc`). Tjenesten tar imot forbindelser på en lokal port og sender dem videre til et mål. Det er et rent TCP-relé på applikasjonsnivå: ingen NAT-regel i brannmuren, men en prosess som kopierer byte mellom to forbindelser. Hvis `iphlpsvc` ikke kjører, fungerer ingen viderekobling. Tjenesten finnes som standard; oppstartstypen bør være satt til automatisk hvis viderekoblingen skal overleve en omstart.

## Oppsett

En viderekobling krever to trinn: portproxy-regelen og en brannmurregel som tillater tilgang til lytteren. Kjør begge i en ledetekst eller PowerShell med administratorrettigheter.

Først viderekoblingen. Den binder til en lokal adresse og port og peker til mål-IP og målport:

```powershell
netsh interface portproxy add v4tov4 listenaddress=100.100.10.10 listenport=5000 connectaddress=10.0.0.245 connectport=5000
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `v4tov4` | IPv4 lytter, IPv4 kobler til; også mulig: `v4tov6`, `v6tov4`, `v6tov6` |
| `listenaddress` | Lokal adresse det lyttes på; her VPN-adressen til jumphosten, slik at forbindelser bare kommer inn via VPN-et |
| `listenport` | Lokal port det lyttes på |
| `connectaddress` | Mål-IP-en som trafikken videresendes til (den interne tjenesten) |
| `connectport` | Målport på den interne tjenesten |

</details>

Bindingen til VPN-adressen i stedet for `0.0.0.0` er den første sikringen: Lytteren vises bare på VPN-grensesnittet, ikke på alle nettverkskortene til jumphosten. Den andre sikringen er brannmuren. Åpne lytterporten utelukkende for adresseområdet til VPN-et ditt, ikke for alle adresser:

```powershell
New-NetFirewallRule -DisplayName "NAS-Proxy (VPN)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5000 -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-Direction Inbound` | Regel for innkommende trafikk |
| `-Protocol TCP` | portproxy videresender bare TCP, derfor TCP |
| `-LocalPort 5000` | Lytterporten fra portproxy-regelen |
| `-RemoteAddress 100.64.0.0/10` | Bare kilder fra dette området tillates; her Tailscale-området, ellers CIDR-blokken til VPN-et ditt |

</details>

## Kontroller og bruk

Kontroller først på selve jumphosten om den interne tjenesten i det hele tatt er tilgjengelig, og vis den aktive viderekoblingen:

```powershell
Test-NetConnection -ComputerName 10.0.0.245 -Port 5000
netsh interface portproxy show v4tov4
```

Hvis målet svarer og regelen står i listen, tester du fra den eksterne enheten. Tjenesten er nå tilgjengelig via adressen og porten til jumphosten:

```powershell
Test-NetConnection -ComputerName 100.100.10.10 -Port 5000
```

I nettleseren åpner du deretter `http://100.100.10.10:5000`. Hvis du trenger flere porter for samme tjeneste, for eksempel 5000 og 5001 for http og https, oppretter du en egen portproxy-regel og riktig brannmuråpning for hver port.

## Oversikt i manpage-stil

De viktigste underkommandoene i `netsh interface portproxy`:

<details class="options-details">
<summary>Oversikt over alternativer</summary>

| Kommando | Formål |
|---|---|
| `add v4tov4 …` | Opprette viderekobling (listenaddress/listenport → connectaddress/connectport) |
| `show v4tov4` | Vise aktive IPv4-viderekoblinger |
| `show all` | Vise alle viderekoblinger for alle protokollvarianter |
| `delete v4tov4 listenaddress=… listenport=…` | Fjerne en viderekobling |
| `reset` | Slette alle portproxy-regler |

</details>

Reglene ligger i registeret under `HKLM\SYSTEM\CurrentControlSet\Services\PortProxy` og overlever en omstart. De er bare synlige via `netsh` eller direkte i registeret, ikke i det grafiske brannmurgrensesnittet.

## Alternativer

`portproxy` er praktisk når jumphosten allerede kjører Windows og du ikke vil installere noe ekstra. To alternativer løser samme problem med andre egenskaper.

En SSH-tunnel med lokal viderekobling (`ssh -L 5000:10.0.0.245:5000 benutzer@jumphost`) krypterer strekningen til selve jumphosten og fungerer på tvers av plattformer. Den krever en SSH-server på jumphosten og eksisterer bare så lenge SSH-økten kjører.

En Tailscale-subnett-ruter (`tailscale up --advertise-routes=10.0.0.0/24`) gjør hele det interne subnettet tilgjengelig for VPN-enhetene dine. Da adresserer du den interne tjenesten direkte på dens faktiske IP, uten viderekobling per port. Dette er den mest direkte løsningen hvis du vil nå flere interne enheter, men krever at ruten godkjennes i Tailscale-administrasjonen.

## Begrensninger

En portviderekobling med portproxy løser tilgangen, men har klare begrensninger du bør kjenne til før bruk:

- **Kun TCP.** `portproxy` videresender utelukkende TCP. Tjenester som trenger UDP (DNS, mange VPN- og spillprotokoller, enkelte videooverføringer), kan ikke realiseres med dette.
- **Ingen ekstra kryptering.** Viderekoblingen kopierer byte uendret. Fortroligheten for forbindelsen leveres kun av VPN-et du bruker for å nå jumphosten. Over et ukryptert transportnett ville trafikken vært ubeskyttet.
- **Sertifikatadvarsel ved HTTPS over IP.** Hvis du videresender en HTTPS-tjeneste og åpner den via IP-adressen til jumphosten, samsvarer ikke målets sertifikat med den forespurte adressen. Nettleseren varsler. Dette er akseptabelt for en kort test, men ikke for varig drift.
- **Omdirigeringer og absolutte adresser.** Enkelte webgrensesnitt omdirigerer selv til vertsnavnet sitt eller en annen port, eller bygger absolutte lenker med den interne adressen sin. Da bryter tilgangen via jumphosten, selv om viderekoblingen er på plass. Slike tjenester trenger en ekte reverse-proxy i stedet for et rent portrelé.
- **Binding til en adresse som må eksistere ved oppstart.** Hvis regelen binder til en bestemt `listenaddress`, må denne adressen være tilgjengelig når tjenesten starter. Hvis VPN-grensesnittet kommer opp senere, kan bindingen mislykkes til tjenesten eller regelen settes opp på nytt.
- **En ekstra vei inn i det interne nettverket.** Hver viderekobling er en sti utenfra til en intern tjeneste. Begrens brannmuren strengt til VPN-området, bind til VPN-adressen, og fjern viderekoblingen så snart du ikke lenger trenger den.

## Fjern igjen

Slett viderekoblingen og brannmurregelen når arbeidet er gjort:

```powershell
netsh interface portproxy delete v4tov4 listenaddress=100.100.10.10 listenport=5000
Remove-NetFirewallRule -DisplayName "NAS-Proxy (VPN)"
```

En portviderekobling er et verktøy for målrettet, tidsbegrenset tilgang, ikke for en permanent åpen kanal. For varig drift av en intern tjeneste over internett er en reverse-proxy med gyldig sertifikat eller et VPN med subnett-ruting den ryddigere løsningen.

## Kilder

1.  [netsh interface portproxy (Microsoft Learn)](https://learn.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-interface-portproxy): Referanse for underkommandoene, protokollvariantene og avhengigheten av IP Helper-tjenesten.

2.  [New-NetFirewallRule (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule): Parametere for brannmurregelen, inkludert begrensning til adresseområder via RemoteAddress.

3.  [Tailscale: Subnet routers](https://tailscale.com/kb/1019/subnets): Gjøre et helt subnett tilgjengelig via VPN-et, som alternativ til viderekobling per port.
