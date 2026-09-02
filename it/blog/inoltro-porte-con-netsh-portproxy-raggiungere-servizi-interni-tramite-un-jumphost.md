---
title: "Inoltro porte con netsh portproxy: raggiungere servizi interni tramite un jumphost"
navTitle: "netsh portproxy"
description: "Windows include un inoltro di porte TCP integrato con netsh interface portproxy. In combinazione con una VPN come Tailscale, consente di raggiungere dall'esterno un servizio interno, ad esempio l'interfaccia di un NAS, senza esporlo pubblicamente. Come configurare, proteggere e rimuovere l'inoltro e quali sono i suoi limiti: niente UDP, nessuna crittografia aggiuntiva, insidie legate a certificati e reindirizzamenti."
date: "2026-09-02"
kategorie: "Windows e rete"
timeToRead: "9 min di lettura"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "tcp"
  - "haertung"
slug: "inoltro-porte-con-netsh-portproxy-raggiungere-servizi-interni-tramite-un-jumphost"
translationId: "article-236adcb4ae982572"
aiPrompt: |
  Du bist mein Windows-Administrationsassistent. Hilf mir, mit netsh interface portproxy eine TCP-Portweiterleitung über einen Windows-Jumphost einzurichten, um einen internen Dienst (z. B. eine NAS-Weboberfläche) über ein VPN zu erreichen: Weiterleitung anlegen, Firewall auf den VPN-Bereich beschränken, prüfen, wieder entfernen, und die Grenzen (kein UDP, keine Verschlüsselung, Zertifikats- und Redirect-Probleme) einordnen.
translationOf: windows-portproxy-portweiterleitung
url: https://rafaelpfister.ch/it/blog/inoltro-porte-con-netsh-portproxy-raggiungere-servizi-interni-tramite-un-jumphost
translationSourceHash: a4888a85b953fbf7b2248232b7b7361e752300872cdb570d6fd15b1cb806ef89
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:03:26.181Z
translationReview: automatic
---

# Inoltro porte con netsh portproxy: raggiungere servizi interni tramite un jumphost

Un servizio interno spesso è in ascolto solo sulla rete locale: l'interfaccia web di un NAS, il pannello di una stampante, una pagina di amministrazione. Se volete accedervi dall'esterno senza esporre il servizio a Internet, vi serve un percorso attraverso un computer che veda entrambi i lati. Windows include uno strumento integrato: `netsh interface portproxy` inoltra le connessioni TCP in entrata a un'altra destinazione. In combinazione con una VPN come Tailscale o WireGuard, un computer nella rete di destinazione diventa un jumphost attraverso il quale raggiungere il servizio interno.

Un esempio concreto: un NAS con l'interfaccia web su `10.0.0.245:5000` è raggiungibile solo nella rete locale. Nella stessa rete si trova un PC Windows, raggiungibile anche tramite VPN. Configurate su questo PC un inoltro di porte dal suo indirizzo VPN al NAS e aprite poi l'interfaccia del NAS nel browser tramite l'indirizzo VPN del PC. Il servizio rimane nella rete interna; solo il jumphost è raggiungibile tramite VPN.

## Come funziona portproxy

`portproxy` è un componente del servizio IP Helper (`iphlpsvc`). Il servizio accetta connessioni su una porta locale e le inoltra a una destinazione. È un puro relay TCP a livello applicativo: non una regola NAT del firewall, ma un processo che copia byte tra due connessioni. Se `iphlpsvc` non è in esecuzione, nessun inoltro funziona. Il servizio è presente per impostazione predefinita; il suo tipo di avvio dovrebbe essere impostato su automatico affinché l'inoltro sopravviva a un riavvio.

## Configurazione

Un inoltro richiede due passaggi: la regola portproxy e una regola firewall che consenta l'accesso al listener. Eseguite entrambi in un prompt dei comandi o in PowerShell con privilegi di amministratore.

Per prima cosa l'inoltro. Si associa a un indirizzo e a una porta locali e punta all'IP e alla porta di destinazione:

```powershell
netsh interface portproxy add v4tov4 listenaddress=100.100.10.10 listenport=5000 connectaddress=10.0.0.245 connectport=5000
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `v4tov4` | IPv4 ascolta, IPv4 si connette; sono possibili anche: `v4tov6`, `v6tov4`, `v6tov6` |
| `listenaddress` | Indirizzo locale su cui ascoltare; qui l'indirizzo VPN del jumphost, così le connessioni in entrata arrivano solo tramite VPN |
| `listenport` | Porta locale su cui ascoltare |
| `connectaddress` | IP di destinazione a cui inoltrare (il servizio interno) |
| `connectport` | Porta di destinazione del servizio interno |

</details>

L'associazione all'indirizzo VPN anziché a `0.0.0.0` è la prima misura di sicurezza: il listener compare solo sull'interfaccia VPN, non su tutte le schede di rete del jumphost. La seconda misura di sicurezza è il firewall. Aprite la porta del listener esclusivamente per l'intervallo di indirizzi della vostra VPN, non per tutti gli indirizzi:

```powershell
New-NetFirewallRule -DisplayName "NAS-Proxy (VPN)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5000 -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Direction Inbound` | Regola per il traffico in entrata |
| `-Protocol TCP` | portproxy inoltra solo TCP, quindi TCP |
| `-LocalPort 5000` | La porta del listener della regola portproxy |
| `-RemoteAddress 100.64.0.0/10` | Sono consentite solo fonti provenienti da questo intervallo; qui l'intervallo Tailscale, altrimenti il blocco CIDR della vostra VPN |

</details>

## Verifica e utilizzo

Verificate anzitutto sul jumphost stesso se il servizio interno è effettivamente raggiungibile e visualizzate l'inoltro attivo:

```powershell
Test-NetConnection -ComputerName 10.0.0.245 -Port 5000
netsh interface portproxy show v4tov4
```

Se la destinazione risponde e la regola compare nell'elenco, testate dal dispositivo remoto. Il servizio è ora raggiungibile tramite indirizzo e porta del jumphost:

```powershell
Test-NetConnection -ComputerName 100.100.10.10 -Port 5000
```

Nel browser aprite quindi `http://100.100.10.10:5000`. Se vi servono più porte dello stesso servizio, ad esempio 5000 e 5001 per http e https, create una regola portproxy separata e la relativa autorizzazione firewall per ogni porta.

## Panoramica in stile manpage

I principali sottocomandi di `netsh interface portproxy`:

<details class="options-details">
<summary>Panoramica delle opzioni</summary>

| Comando | Scopo |
|---|---|
| `add v4tov4 …` | Creare un inoltro (listenaddress/listenport → connectaddress/connectport) |
| `show v4tov4` | Visualizzare gli inoltri IPv4 attivi |
| `show all` | Visualizzare tutti gli inoltri di tutte le varianti di protocollo |
| `delete v4tov4 listenaddress=… listenport=…` | Rimuovere un inoltro |
| `reset` | Eliminare tutte le regole portproxy |

</details>

Le regole si trovano nel registro in `HKLM\SYSTEM\CurrentControlSet\Services\PortProxy` e persistono dopo un riavvio. Sono visibili solo tramite `netsh` o direttamente nel registro, non nell'interfaccia grafica del firewall.

## Alternative

`portproxy` è pratico quando il jumphost è già Windows e non volete installare nulla. Due alternative risolvono lo stesso problema con caratteristiche diverse.

Un tunnel SSH con inoltro locale (`ssh -L 5000:10.0.0.245:5000 benutzer@jumphost`) cifra il percorso fino al jumphost stesso e funziona su più piattaforme. Richiede un server SSH sul jumphost ed esiste solo finché la sessione SSH rimane attiva.

Un subnet router Tailscale (`tailscale up --advertise-routes=10.0.0.0/24`) rende raggiungibile l'intera sottorete interna per i vostri dispositivi VPN. Potete quindi indirizzare il servizio interno direttamente al suo IP reale, senza inoltro per singola porta. È il modo più diretto se volete raggiungere più dispositivi interni, ma richiede l'approvazione della rotta nella gestione di Tailscale.

## Limiti

Un inoltro di porte con portproxy risolve l'accesso, ma presenta limiti chiari che dovreste conoscere prima dell'uso:

- **Solo TCP.** `portproxy` inoltra esclusivamente TCP. I servizi che richiedono UDP (DNS, molti protocolli VPN e di gioco, alcune trasmissioni video) non possono essere implementati in questo modo.
- **Nessuna crittografia aggiuntiva.** L'inoltro copia i byte senza modificarli. La riservatezza del percorso è fornita esclusivamente dalla VPN attraverso la quale raggiungete il jumphost. Su una rete di trasporto non cifrata, il traffico sarebbe privo di protezione.
- **Avviso del certificato con HTTPS tramite IP.** Se inoltrate un servizio HTTPS e lo richiamate tramite l'IP del jumphost, il certificato della destinazione non corrisponde all'indirizzo richiamato. Il browser avvisa. Per un breve test è accettabile, ma non per un utilizzo continuativo.
- **Reindirizzamenti e indirizzi assoluti.** Alcune interfacce web reindirizzano automaticamente al proprio nome host o a un'altra porta, oppure creano link assoluti con il proprio indirizzo interno. In tal caso l'accesso tramite il jumphost non funziona, anche se l'inoltro è configurato. Questi servizi richiedono un vero reverse proxy anziché un semplice relay di porte.
- **Associazione a un indirizzo che deve esistere all'avvio.** Se la regola si associa a un determinato `listenaddress`, questo indirizzo deve essere presente all'avvio del servizio. Se l'interfaccia VPN si attiva solo in seguito, l'associazione può fallire finché il servizio o la regola non vengono reimpostati.
- **Un percorso aggiuntivo nella rete interna.** Ogni inoltro è un percorso dall'esterno a un servizio interno. Limitate strettamente il firewall all'intervallo VPN, associatevi all'indirizzo VPN e rimuovete l'inoltro non appena non vi serve più.

## Rimozione

Al termine del lavoro, eliminate l'inoltro e la regola firewall:

```powershell
netsh interface portproxy delete v4tov4 listenaddress=100.100.10.10 listenport=5000
Remove-NetFirewallRule -DisplayName "NAS-Proxy (VPN)"
```

Un inoltro di porte è uno strumento per un accesso mirato e temporaneo, non per un canale permanentemente aperto. Per il funzionamento continuativo di un servizio interno tramite Internet, un reverse proxy con certificato valido o una VPN con subnet routing è la soluzione più pulita.

## Fonti

1.  [netsh interface portproxy (Microsoft Learn)](https://learn.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-interface-portproxy): riferimento per sottocomandi, varianti di protocollo e dipendenza dal servizio IP Helper.

2.  [New-NetFirewallRule (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule): parametri della regola firewall, inclusa la limitazione agli intervalli di indirizzi tramite RemoteAddress.

3.  [Tailscale: Subnet routers](https://tailscale.com/kb/1019/subnets): rendere raggiungibile tramite VPN un'intera sottorete, come alternativa all'inoltro per singola porta.
