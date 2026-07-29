---
title: "Gestire correttamente le operazioni successive agli aggiornamenti di sicurezza di Exchange di luglio 2026"
navTitle: "Exchange SU 07/2026"
description: "Dopo l’installazione sono necessarie due attività di pulizia: rimuovere in modo controllato la mitigazione legacy per CVE-2026-42897 e verificare i gruppi legacy con privilegi eccessivi in Active Directory."
date: "2026-07-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min di lettura"
themen:
  - "exchange-onprem-hybrid"
  - "active-directory-entra"
slug: "gestire-correttamente-le-operazioni-successive-agli-aggiornamenti-di-sicurezza-di-exchange-di"
translationOf: "exchange-security-updates-juli-2026"
url: "https://rafaelpfister.ch/it/blog/gestire-correttamente-le-operazioni-successive-agli-aggiornamenti-di-sicurezza-di-exchange-di"
---

# Gestire correttamente le operazioni successive agli aggiornamenti di sicurezza di Exchange di luglio 2026

Con l’installazione degli aggiornamenti di sicurezza di Exchange del 14 luglio 2026, il lavoro non è ancora concluso. Successivamente, gli amministratori dovrebbero eliminare due retaggi: la mitigazione per **CVE-2026-42897** attivata a maggio e due gruppi di sicurezza Exchange storici con diritti estesi in Active Directory.

Entrambe le attività sono facili da trascurare. La mitigazione rimane intenzionalmente attiva finché non viene rimossa in modo controllato. I gruppi, invece, potrebbero essere sopravvissuti inosservati a ogni migrazione per molti anni.

## Per quali versioni di Exchange è disponibile l’aggiornamento

Le SU sono disponibili per le versioni seguenti:

- **Exchange Server Subscription Edition (SE) RTM**: come aggiornamento pubblico regolarmente disponibile.
- **Exchange Server 2019 CU14 e CU15**: solo per le organizzazioni iscritte al **programma ESU Period 2**.
- **Exchange Server 2016 CU23**: anch’esso disponibile solo tramite ESU Period 2.

Exchange 2016 e 2019 non sono più supportati. Chi non aderisce al programma ESU Period 2 (valido da maggio a ottobre 2026) non riceverà più questi aggiornamenti e non dovrebbe più rimandare il passaggio a Exchange SE. Gli ambienti Exchange Online sono già protetti; nelle configurazioni ibride, tuttavia, la SU deve essere installata su tutti i server Exchange, compresi quelli usati esclusivamente per la gestione. Come di consueto, le CVE specifiche risolte sono elencate nella Security Update Guide (filtro «Server Software» per Exchange SE o «ESU» per 2016/2019).

Nella release corrente è presente un problema noto: negli ambienti ibridi, i cosiddetti *messaggi wrapper* possono comparire nella posta in arrivo delle cassette postali condivise. I dettagli sono disponibili nel relativo articolo del supporto Microsoft.

## Rimuovere la mitigazione CVE-2026-42897 dopo l’installazione

### Breve riepilogo

CVE-2026-42897 è stata resa nota il 14 maggio 2026: una vulnerabilità di cross-site scripting (spoofing) in Outlook Web Access. Un attaccante invia un’e-mail appositamente predisposta; se la vittima la apre in OWA e sono soddisfatte determinate condizioni di interazione, è possibile eseguire JavaScript arbitrario nel contesto del browser. Erano interessati Exchange 2016, 2019 e SE a *qualsiasi* livello di patch. Microsoft ha pubblicato una mitigazione di emergenza lo stesso giorno (ID **M2.1.x**, la regola IIS specifica si chiama **M2.1.0**) e ha fornito la correzione effettiva con la SU di giugno 2026.

### Perché l’aggiornamento di luglio *non* rimuove automaticamente la mitigazione

Questo è l’aspetto che sorprende maggiormente: anche dopo l’installazione della SU di luglio, una mitigazione già applicata rimane attiva. Il motivo risiede nel meccanismo. La mitigazione è una **regola IIS URL Rewrite basata su Content Security Policy**, applicata *al di fuori* del programma di installazione MSI, tramite l’Emergency Mitigation Service (servizio EM) oppure tramite lo script EOMT. La patch MSI sostituisce i file binari, ma non gestisce queste regole IIS impostate fuori banda. Per questo motivo, la rimozione è un passaggio manuale separato.

Tra l’altro: la mitigazione non ha mai protetto i client IE e Edge in modalità IE, perché Internet Explorer non supporta CSP. Chi utilizzava tali client non è mai stato protetto dalla sola mitigazione. Questo è un ulteriore motivo per applicare le patch tempestivamente anziché affidarsi alla mitigazione.

### L’insidia: il servizio EM riapplica la mitigazione

Chi elimina la regola prematuramente avrà una sorpresa. Il servizio EM viene eseguito ogni ora e confronta lo stato effettivo con le direttive fornite dal servizio Office Config (Flighting). L’associazione «quale build richiede quale mitigazione» è gestita lato server. Solo una modifica lato server contrassegna la build di luglio 2026 come «mitigazione non più necessaria». Secondo Microsoft, questa modifica è stata distribuita completamente solo intorno al 16 luglio 2026. Fino ad allora, il servizio EM reinserisce semplicemente una regola M2.1.0 eliminata durante l’esecuzione oraria successiva.

In pratica, ciò significa che si attende fino a dopo il 16 luglio prima di rimuoverla manualmente oppure si blocca esplicitamente la mitigazione affinché non venga riattivata.

### Come rimuovere correttamente la mitigazione (percorso servizio EM)

Per prima cosa, verificare che cosa sia effettivamente applicato:

```powershell
Get-ExchangeServer -Identity <NomeServer> | Format-List Name,MitigationsApplied,MitigationsBlocked
```

Per impedire la riattivazione, l’ID della mitigazione viene aggiunto all’elenco di blocco: le voci presenti vengono ignorate dal servizio EM durante l’esecuzione oraria.

```powershell
Set-ExchangeServer -Identity <NomeServer> -MitigationsBlocked @("M2.1.0")
```

Quindi rimuovere la regola IIS vera e propria. È utile sapere, e raramente documentato, che il servizio EM crea le proprie regole URL Rewrite con il **prefisso «EEMS `<Mitigation-ID>` `<Beschreibung>`»**. In questo modo è possibile individuarle chiaramente in Gestione IIS sotto URL Rewrite (o tramite `appcmd`/PowerShell nella `applicationHost.config`) senza dover indovinare quale regola appartenga alla mitigazione. Dopo la distribuzione della modifica lato server, è possibile rimuovere nuovamente il blocco (`-MitigationsBlocked @()`), se è stato impostato solo come soluzione temporanea.

### Percorso EOMT (ambienti isolati o air-gapped)

Se la mitigazione è stata impostata mediante lo **script EOMT** scaricabile (https://aka.ms/UnifiedEOMT), il ripristino avviene tramite l’opzione di rollback:

```powershell
.\EOMT.ps1 -RollbackMitigation -CVE "CVE-2026-42897"
```

Anche qui un dettaglio poco noto: prima di ogni modifica, EOMT salva lo stato iniziale di IIS in un **file di backup JSON specifico per la CVE** in `%WINDIR%\System32\inetsrv\config\`. Il rollback legge esattamente questo file e ripristina le impostazioni originali. Importante: una mitigazione impostata con uno script legacy (EOMTv2 ecc.) deve essere rimossa anche con il relativo meccanismo di rollback: i formati di backup non sono compatibili.

### Perché vale la pena rimuoverla

La mitigazione non è «gratuita». Finché è attiva, ci si porta dietro i suoi effetti collaterali noti: la funzione OWA «Stampa calendario» non funziona, le immagini inline potrebbero non essere visualizzate correttamente nel riquadro di lettura di OWA, OWA Light (`/?layout=light`) è difettoso (e sarà comunque disattivato a breve), i calendari pubblicati restituiscono talvolta errori 500. Particolarmente insidioso per il monitoraggio: il health set **OWACalendar.Proxy** può passare a *unhealthy*, generando così falsi allarmi nel monitoraggio. Chi installa la SU ma lascia attiva la mitigazione finisce per inseguire fantasmi. Non appena l’aggiornamento è installato *e* la mitigazione è rimossa, anche questi problemi noti scompaiono.

Un caso particolare: negli ambienti misti, i server non ancora aggiornati possono mantenere la mitigazione. Occorre però sapere che l’integrazione con Office Online Server (OOS) potrebbe tornare a funzionare correttamente solo quando *tutti* i server Exchange dell’organizzazione sono aggiornati al livello di luglio.

## Health Checker: individuare gruppi di sicurezza antichissimi

Il secondo punto, indipendente dalla release della SU: l’**Exchange Health Checker** (https://aka.ms/ExchangeHealthChecker) ora verifica l’esistenza di due gruppi di sicurezza da tempo deprecati: **«Exchange Domain Servers»** e **«Exchange Enterprise Servers»**.

### Da dove provengono questi gruppi e perché rappresentano un rischio

Questi due gruppi derivano dal modello di autorizzazioni di Exchange 2000/2003 e sono deprecati da Exchange 2007. Con Exchange 2007/2010 sono stati introdotti il modello Split Permissions e RBAC e, da allora, questi gruppi non vengono più utilizzati. Il problema è che non sono scomparsi. In molte directory rimangono inosservati da circa due decenni e talvolta conservano ACL estese del vecchio modello, quindi più diritti di quanti ne avrebbe mai un moderno gruppo di sicurezza Exchange.

Questo li rende un vettore d’attacco. Un gruppo inattivo con autorizzazioni ampie e persistenti costituisce una classica catena di escalation: chi riesce ad aggiungere sé stesso, o un account controllato, a tale gruppo ne eredita i diritti nella directory. Poiché nessuno monitora attivamente il gruppo, una manipolazione di questo tipo passa difficilmente inosservata.

### Perché la maggior parte degli amministratori non li considera

Questi gruppi sono un punto cieco per vari motivi: sono inattivi da circa 20 anni, nella maggior parte dei casi esistevano già prima dell’arrivo del team attuale, sopravvivono senza problemi a ogni migrazione e finora non sono mai stati segnalati da Health Checker. Particolarmente delicato: sopravvivono persino alla dismissione *completa* di Exchange on-premises. Chi ha rimosso l’ultimo server Exchange normalmente elimina gli oggetti server, ma trascura completamente questi gruppi legacy.

### Pulizia

In futuro Health Checker segnalerà automaticamente i gruppi. Manualmente, è possibile trovarli in Active Directory, solitamente nel contenitore `Users`, oppure tramite PowerShell:

```powershell
Get-ADGroup -Filter "Name -eq 'Exchange Domain Servers' -or Name -eq 'Exchange Enterprise Servers'"
```

Procedura: verificare l’appartenenza e qualsiasi riferimento ACL personalizzato, assicurarsi che nessun elemento produttivo vi faccia riferimento e infine eliminare i gruppi. Poiché sono deprecati dal 2007, nella stragrande maggioranza degli ambienti possono essere rimossi senza rischi. Chi non gestisce più alcun Exchange on-premises dovrebbe pianificare, contestualmente, una pulizia AD più completa seguendo la guida ufficiale Microsoft.

Hayes Jupe ha pubblicato una guida dettagliata per rimuovere i gruppi nel suo post del blog [Latest Exchange health check script and deprecated groups](https://www.hayesjupe.com/latest-exchange-health-check-script-and-deprecated-groups/).

## Procedura consigliata

In breve, il flusso operativo pratico è il seguente: anzitutto inventariare l’ambiente con Health Checker (mostra CU/SU mancanti, passaggi manuali aperti *e* ora anche i gruppi legacy). Quindi installare il CU attuale e la SU di luglio, riavviare il server e verificare che tutti i servizi Exchange siano stati avviati correttamente. Successivamente, eseguire nuovamente Health Checker, rimuovere la mitigazione CVE-2026-42897 (dopo il 16 luglio oppure bloccando prima l’ID M2.1.0) e infine pulire i gruppi di sicurezza deprecati. Le SU sono cumulative: chi si trova su un CU supportato non deve installare tutte le SU intermedie, ma può installare direttamente la più recente.

## Fonti

1.  [Released: July 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-july-2026-exchange-server-security-updates/4534146): Annuncio ufficiale della release di luglio con le versioni supportate e il problema noto dei messaggi wrapper.

2.  [Addressing Exchange Server May 2026 vulnerability CVE-2026-42897 – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/addressing-exchange-server-may-2026-vulnerability-cve-2026-42897/4518498): Avviso di sicurezza originale, con la mitigazione di emergenza e gli effetti collaterali noti in OWA.

3.  [Released: June 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-june-2026-exchange-server-security-updates/4524491): Release di giugno che ha fornito la correzione effettiva per CVE-2026-42897.

4.  [Exchange Emergency Mitigation Service (Exchange EM Service) – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-emergency-mitigation-service): Funzionamento del servizio EM, che confronta le mitigazioni ogni ora e ripristina una regola eliminata prematuramente.

5.  [Set-ExchangeServer (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-exchangeserver): Parametri `MitigationsApplied` e `MitigationsBlocked` per verificare le mitigazioni e impedirne la riattivazione.

6.  [Exchange On-premises Mitigation Tool (EOMT) – Microsoft CSS-Exchange](https://microsoft.github.io/CSS-Exchange/Security/EOMT/): Lo script EOMT, inclusa l’opzione di rollback e il salvataggio JSON specifico per CVE dello stato iniziale di IIS.

7.  [CVE-2026-42897 Detail – NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-42897): Descrizione tecnica e valutazione della vulnerabilità nel National Vulnerability Database.
