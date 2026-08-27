---
title: "VPN Split Tunneling per Microsoft Teams: instradare il traffico multimediale fuori dal tunnel"
navTitle: "Teams Split Tunneling"
description: "Le chiamate Teams tramite VPN soffrono di latenza, jitter e del percorso indiretto attraverso il gateway VPN. L’articolo mostra quali reti e porte Microsoft sono responsabili del traffico multimediale, perché lo Split Tunneling basato su IP è superiore all’esclusione per app e come implementarlo in VPN consumer, WireGuard, OpenVPN e client enterprise."
date: "2026-08-26"
kategorie: "Microsoft Teams"
timeToRead: "8 min di lettura"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
protokolle:
  - "tcp"
hauptthema: "microsoft-teams"
slug: "vpn-split-tunneling-per-microsoft-teams-instradare-il-traffico-multimediale-fuori-dal-tunnel"
translationId: "article-d15f1e7ff6af231c"
aiPrompt: |
  Du bist mein Netzwerk-Assistent. Ich will Microsoft-Teams-Medienverkehr per Split Tunneling an meinem VPN vorbeiführen. Hilf mir Schritt für Schritt: 1. Frage mich, welchen VPN-Client ich einsetze (Consumer-VPN, WireGuard, OpenVPN, Enterprise-Client) und auf welchem Betriebssystem. 2. Nenne mir die passende Konfiguration für die drei Optimize-Netze 13.107.64.0/18, 52.112.0.0/14 und 52.122.0.0/15 (UDP 3478 bis 3481, TCP 443). 3. Erkläre mir, wie ich mit Find-NetRoute oder der Anrufintegrität in Teams prüfe, ob der Medienverkehr tatsächlich am Tunnel vorbeiläuft. 4. Weise mich auf die Sicherheitsabwägungen hin, bevor ich die Ausnahme produktiv setze.
translationOf: vpn-split-tunneling-microsoft-teams
url: https://rafaelpfister.ch/it/blog/vpn-split-tunneling-per-microsoft-teams-instradare-il-traffico-multimediale-fuori-dal-tunnel
translationSourceHash: 95e3cefa4946676022602866d6ef21ab92ef25ec8c5dd3ff4ab0219ba718a880
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:32:07.961Z
translationReview: automatic
---

Una chiamata Teams tramite connessione VPN spesso ha una qualità peggiore rispetto a una senza VPN: l’audio si interrompe, il video scatta e le condivisioni dello schermo si caricano in ritardo. La causa risiede generalmente nel percorso indiretto che il traffico in tempo reale compie attraverso il tunnel VPN, non in Teams stesso. Per questo Microsoft raccomanda da anni di instradare direttamente in Internet il traffico multimediale di Teams, aggirando la VPN tramite Split Tunneling. Questo approccio funziona con praticamente qualsiasi prodotto VPN, dal client consumer al gateway enterprise; la configurazione differisce solo nei dettagli.

## Perché il traffico in tempo reale soffre nel tunnel

L’audio e il video di Teams utilizzano SRTP, un protocollo basato su UDP che dipende da una bassa latenza e da poco jitter. Microsoft indica come valori obiettivo un tempo di andata e ritorno inferiore a 100 ms verso il punto di ingresso di rete Microsoft più vicino e un jitter inferiore a 30 ms. Un tunnel VPN peggiora entrambi i valori in più modi.

In primo luogo, il tunnel allunga il percorso: invece di raggiungere direttamente il punto di ingresso Microsoft geograficamente più vicino, il traffico passa prima dal gateway VPN, che può trovarsi nel data center del provider o dell’azienda, e solo da lì raggiunge Microsoft. In secondo luogo, il livello di crittografia aggiuntivo richiede tempo di elaborazione e aumenta l’overhead per pacchetto; il flusso multimediale è già crittografato con SRTP, mentre la crittografia VPN si aggiunge come secondo livello. In terzo luogo, il gateway VPN è un collo di bottiglia condiviso: nelle ore di punta tutti gli utenti condividono la sua larghezza di banda e i buffer dei pacchetti, generando proprio il jitter al quale il traffico in tempo reale è più sensibile. In quarto luogo, alcune configurazioni VPN bloccano completamente UDP o impongono TCP; Teams ricorre allora a TCP 443, peggiorando ulteriormente la qualità, poiché le ritrasmissioni TCP non sono adatte ai contenuti multimediali in tempo reale.

Per il resto del traffico Teams (accesso, chat, accesso ai file), tutto questo ha poca importanza, perché non è sensibile al tempo reale. È quindi sufficiente escludere selettivamente il traffico multimediale.

## Le reti e le porte rilevanti

Microsoft pubblica tutti gli endpoint Microsoft 365 in formato leggibile dalle macchine e li suddivide nelle categorie Optimize, Allow e Default. Per lo Split Tunneling è rilevante la categoria Optimize: comprende i pochi endpoint critici per la latenza con reti IP fisse, che insieme costituiscono la maggior parte del volume. Per i contenuti multimediali Teams si tratta degli ID endpoint 11 e 12 dell’elenco ufficiale:

| Rete | Protocollo e porte | Scopo |
|---|---|---|
| `13.107.64.0/18` | UDP da 3478 a 3481, TCP 443 | Contenuti multimediali Teams (audio, video, condivisione schermo) |
| `52.112.0.0/14` | UDP da 3478 a 3481, TCP 443 | Contenuti multimediali Teams e relay di trasporto |
| `52.122.0.0/15` | UDP da 3478 a 3481, TCP 443 | Contenuti multimediali Teams e relay di trasporto |
| `2603:1063::/38` | UDP da 3478 a 3481, TCP 443 | Gli stessi servizi tramite IPv6 |

Le quattro porte UDP corrispondono alle classi multimediali audio (3478), video (3479 e 3480) e condivisione schermo (3481); TCP 443 è il percorso di fallback. Chi utilizza IPv6 dovrebbe escludere anche la rete IPv6, altrimenti una parte delle connessioni tornerà a passare nel tunnel.

Queste reti sono intenzionalmente stabili: Microsoft annuncia le modifiche agli endpoint Optimize tramite l’Endpoint Web Service e mantiene breve l’elenco, proprio affinché le aziende possano inserirlo in regole di routing e firewall. Tuttavia, un confronto periodico con l’elenco ufficiale dovrebbe far parte delle procedure operative.

## Basato su app o su IP: due approcci con punti di forza diversi

Molti client VPN offrono due tipi di Split Tunneling: eccezioni per applicazione oppure per IP di destinazione.

L’esclusione dell’app sembra ovvia, ma con Teams presenta due debolezze. Il nuovo Teams è un’applicazione WebView2: il processo principale si chiama `ms-teams.exe`, ma una parte del traffico passa attraverso `msedgewebview2.exe`. Chi esclude solo il processo principale non intercetta tutto il traffico; chi esclude anche WebView2 instrada fuori dal tunnel anche il traffico di altre applicazioni WebView2, come il nuovo Outlook. Inoltre, l’esclusione dell’app non aiuta affatto con Teams nel browser, a meno di escludere l’intero browser, facendo così aggirare la VPN a tutto il traffico web.

L’esclusione basata su IP, invece, agisce a livello di rete ed è quindi indipendente dal fatto che il traffico provenga dall’app Teams, da WebView2 o da una scheda del browser. Esclude esattamente ciò che è critico per la latenza e lascia nel tunnel l’accesso, la chat e il restante traffico web. Per Teams, l’approccio basato su IP è quindi la scelta migliore; l’esclusione dell’app è utile come integrazione se si desidera davvero che l’intero traffico Teams aggiri la VPN.

## Implementazione nei prodotti VPN più comuni

Il principio è ovunque lo stesso: le tre reti IPv4 (e, se necessario, la rete IPv6) vengono escluse dal tunnel, in modo che le route del sistema operativo per queste destinazioni puntino all’interfaccia fisica.

**VPN consumer (Proton VPN, NordVPN, Surfshark e simili):** I client per Windows e Android offrono solitamente una voce di menu come “Split Tunneling” con un elenco di esclusione per indirizzi IP o subnet. Inserire qui le tre reti in notazione CIDR e ristabilire la connessione VPN affinché le route abbiano effetto. Su macOS e iOS questa funzione manca nella maggior parte dei provider, perché le API di sistema non consentono lo Split Tunneling controllato dalle applicazioni in questa forma.

**WireGuard:** WireGuard non conosce un elenco di esclusione, ma solo l’impostazione `AllowedIPs`, che definisce cosa entra nel tunnel. Le eccezioni si ottengono sostituendo `0.0.0.0/0` con l’elenco di tutte le reti che non contengono l’intervallo da escludere. Nessuno calcola a mano questo elenco complementare; calcolatori online come WireGuard AllowedIPs Calculator usano `0.0.0.0/0` come base, le tre reti Microsoft come “Disallowed IPs” e forniscono la riga completa per il file di configurazione.

**OpenVPN:** Con `redirect-gateway` attivo, le route più specifiche hanno la precedenza. Tre righe aggiuntive nella configurazione client instradano le reti Microsoft fuori dal tunnel:

```text
route 13.107.64.0 255.255.192.0 net_gateway
route 52.112.0.0 255.252.0.0 net_gateway
route 52.122.0.0 255.254.0.0 net_gateway
```

`net_gateway` indica il gateway predefinito della rete locale, non il gateway VPN.

**Client enterprise (Cisco Secure Client/AnyConnect, Palo Alto GlobalProtect, Fortinet FortiClient):** In questo caso l’azienda configura centralmente le eccezioni, in Cisco come elenco “Split Exclude” nella policy di gruppo, in GlobalProtect come “Exclude Access Route”. Microsoft documenta espressamente questa procedura come modello raccomandato per il traffico Microsoft 365 e fornisce l’elenco Optimize tramite l’Endpoint Web Service, così che le eccezioni possano essere mantenute aggiornate automaticamente. Chi, in qualità di dipendente, si trova dietro una VPN aziendale non può quindi impostare da sé l’eccezione, ma deve richiederla al team di rete; il documento Microsoft in merito costituisce una base adeguata per motivare la richiesta.

**Strumenti integrati di Windows:** In una connessione VPN configurata con gli strumenti integrati di Windows in modalità split (`Set-VpnConnection -SplitTunneling $true`), nel tunnel finiscono solo le reti aggiunte tramite `Add-VpnConnectionRoute`. Finché le reti Microsoft non compaiono lì, vengono instradate automaticamente in modo diretto; non è quindi necessaria un’esclusione esplicita.

## Valutazione della sicurezza: cosa passa fuori dal tunnel

Lo Split Tunneling è un allentamento consapevole del principio di instradare tutto il traffico attraverso il tunnel. Prima dell’implementazione occorre chiarire tre punti.

Il proprio indirizzo IP pubblico diventa visibile a Microsoft, poiché è proprio questo l’obiettivo: il flusso multimediale deve seguire il percorso più breve. Chi utilizza una VPN principalmente per nascondere la propria posizione rinuncia a questa protezione per le chiamate Teams. I contenuti rimangono inalterati, perché SRTP crittografa il flusso multimediale end-to-end tra il client e l’infrastruttura Microsoft.

In ambito aziendale, il gateway di sicurezza centrale perde visibilità sul traffico escluso: l’ispezione TLS, le firme IDS e l’analisi dei volumi non si applicano più a queste reti. Poiché l’eccezione è limitata a poche reti fisse assegnate a Microsoft e a porte definite, Microsoft considera basso questo rischio residuo; gli endpoint Optimize sono selezionati appositamente per questo scopo. Un’eccezione generalizzata per intere applicazioni o addirittura per il browser presenta invece una superficie di attacco notevolmente maggiore e dovrebbe essere evitata in ambito aziendale.

Infine, il Kill Switch: alcuni client VPN applicano le eccezioni di Split Tunneling solo dopo aver ristabilito la connessione oppure si comportano diversamente quando il Kill Switch è attivo. Dopo ogni modifica all’elenco di esclusione è quindi necessario ristabilire la connessione ed effettuare un test di controllo.

## Verifica: il traffico multimediale passa davvero direttamente?

L’efficacia dell’eccezione può essere verificata su due livelli. A livello di routing, PowerShell mostra quale interfaccia Windows sceglie per una destinazione nelle reti Microsoft:

```powershell
Find-NetRoute -RemoteIPAddress 52.112.1.1 |
  Select-Object InterfaceAlias, NextHop
```

Se qui compare l’interfaccia fisica (Ethernet o WLAN) invece dell’adattatore VPN, la route è corretta. A livello dell’applicazione, è Teams stesso a fornire la conferma: durante una chiamata, l’integrità della chiamata (in “Altre azioni” nella finestra della chiamata) mostra il tipo di connessione negoziato, il tempo di andata e ritorno e il tasso di perdita dei pacchetti. Un tempo di andata e ritorno che diminuisce sensibilmente dopo la modifica e il tipo di connessione UDP anziché TCP sono i due segnali di un’eccezione funzionante.

Se il traffico continua a passare nel tunnel nonostante la route corretta, vale la pena controllare l’ordine degli adattatori di rete e le peculiarità del client: alcuni client VPN impongono nuovamente le proprie route con una metrica più bassa dopo ogni connessione e un elenco di esclusione obsoleto diventa evidente solo quando Microsoft aggiunge una rete. Il confronto con l’elenco ufficiale degli endpoint dovrebbe quindi seguire lo stesso ritmo delle altre verifiche di rete periodiche.

## Fonti

1.  [Microsoft: Office 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges): elenco ufficiale degli endpoint; le reti multimediali Teams sono indicate dagli ID 11 e 12 nella categoria Optimize.

2.  [Microsoft: Implementing VPN split tunneling for Microsoft 365](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-vpn-implement-split-tunnel): guida di implementazione Microsoft per VPN enterprise, inclusa la motivazione della valutazione del rischio.

3.  [Microsoft: Microsoft 365 network connectivity principles](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-network-connectivity-principles): i principi alla base dell’uscita Internet locale, inclusi i valori obiettivo di latenza per i contenuti multimediali in tempo reale.

4.  [Proton VPN: How to use split tunneling](https://protonvpn.com/support/protonvpn-split-tunneling/): esempio di client consumer con Split Tunneling basato su IP e app in Windows e Android.

5.  [WireGuard AllowedIPs Calculator](https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/): calcolatore per l’elenco complementare quando le eccezioni devono essere configurate tramite AllowedIPs.
