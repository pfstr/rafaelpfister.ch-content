---
title: "Gestire correttamente le operazioni successive agli aggiornamenti di sicurezza Exchange di luglio 2026"
navTitle: "Exchange SU 07/2026"
description: "Dopo l'installazione sono necessarie due operazioni di pulizia: rimuovere in modo controllato la vecchia mitigazione CVE-2026-42897 e verificare i gruppi legacy con privilegi eccessivi in Active Directory."
date: "2026-07-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min di lettura"
themen:
  - exchange-updates
  - active-directory-entra
slug: "gestire-correttamente-le-operazioni-successive-agli-aggiornamenti-di-sicurezza-di-exchange-di"
translationOf: "exchange-security-updates-juli-2026"
translationId: article-731b5b840aee096c
translationReview: automatic
translationSourceHash: e5d9295515965d3e7801752cd605f6d2a78cacfc9fb965e0f63d645658b39e9b
translatedAt: 2026-09-05T07:52:13.087Z
url: https://rafaelpfister.ch/it/blog/gestire-correttamente-le-operazioni-successive-agli-aggiornamenti-di-sicurezza-di-exchange-di
translationModel: gpt-5.6-terra
---

# Gestire correttamente le operazioni successive agli aggiornamenti di sicurezza Exchange di luglio 2026

Con l'installazione degli aggiornamenti di sicurezza Exchange del 14 luglio 2026, il lavoro non è ancora concluso. Successivamente, gli amministratori dovrebbero eliminare due retaggi: la mitigazione per **CVE-2026-42897** attivata a maggio e due gruppi di sicurezza Exchange storici con autorizzazioni estese in Active Directory.

Entrambe le attività sono facili da trascurare. La mitigazione rimane intenzionalmente attiva finché non viene rimossa in modo controllato. I gruppi, invece, potrebbero essere sopravvissuti inosservati a ogni migrazione per molti anni.

## Per quali versioni di Exchange è disponibile l'aggiornamento

Le SU sono disponibili per le seguenti versioni:

- **Exchange Server Subscription Edition (SE) RTM**: come aggiornamento pubblico disponibile regolarmente.
- **Exchange Server 2019 CU14 e CU15**: solo per organizzazioni iscritte al **programma ESU Period 2**.
- **Exchange Server 2016 CU23**: anch'esso disponibile solo tramite ESU Period 2.

Exchange 2016 e 2019 non sono più supportati. Chi non partecipa al programma ESU Period 2 (valido da maggio a ottobre 2026) non riceverà più questi aggiornamenti e non dovrebbe rimandare ulteriormente il passaggio a Exchange SE. Gli ambienti Exchange Online sono già protetti; nelle configurazioni ibride, tuttavia, la SU deve comunque essere installata su tutti i server Exchange, inclusi i server di sola gestione. Le CVE specifiche risolte sono indicate, come di consueto, nella Security Update Guide (filtro «Server Software» per Exchange SE o «ESU» per 2016/2019).

La release corrente presenta un problema noto: negli ambienti ibridi, i cosiddetti *messaggi wrapper* possono comparire nella posta in arrivo delle cassette postali condivise. I dettagli sono disponibili nel corrispondente articolo di supporto Microsoft.

## Rimuovere la mitigazione CVE-2026-42897 dopo l'installazione

### Breve riepilogo

CVE-2026-42897 è stata annunciata il 14 maggio 2026: una vulnerabilità Cross-Site Scripting (spoofing) in Outlook Web Access. Un attaccante invia un'e-mail appositamente predisposta; se la vittima la apre in OWA e sono soddisfatte determinate condizioni di interazione, è possibile eseguire JavaScript arbitrario nel contesto del browser. Erano interessati Exchange 2016, 2019 e SE a *qualsiasi* livello di patch. Microsoft ha pubblicato lo stesso giorno una mitigazione di emergenza (ID **M2.1.x**, mentre la regola IIS specifica si chiama **M2.1.0**) e ha fornito la correzione effettiva con la SU di giugno 2026.

### Perché l'aggiornamento di luglio *non* rimuove automaticamente la mitigazione

Questo è il punto che sorprende maggiormente: anche dopo l'installazione della SU di luglio, una mitigazione già applicata rimane attiva. Il motivo risiede nel meccanismo. La mitigazione è una **regola IIS URL Rewrite basata su Content Security Policy**, applicata *al di fuori* dell'installer MSI, tramite Emergency Mitigation Service (servizio EM) oppure tramite lo script EOMT. La patch MSI sostituisce i file binari, ma non gestisce queste regole IIS impostate out-of-band. La rimozione è quindi un passaggio manuale separato.

A margine: la mitigazione non ha mai protetto i client IE ed Edge in modalità IE, poiché Internet Explorer non supporta CSP. Chi utilizzava tali client non è mai stato protetto dalla sola mitigazione. Questo è un ulteriore motivo per applicare le patch tempestivamente invece di fare affidamento sulla mitigazione.

### Il punto delicato: il servizio EM applica nuovamente la mitigazione

Una regola eliminata prematuramente non rimane rimossa in modo permanente. Il servizio EM viene eseguito ogni ora e confronta lo stato effettivo con le direttive fornite da Office Config Service (Flighting). L'associazione fra build e mitigazione necessaria risiede lato server. Solo una modifica lato server contrassegna la build di luglio 2026 come «mitigazione non più necessaria». Secondo Microsoft, questa modifica è stata distribuita completamente solo attorno al 16 luglio 2026. Fino ad allora, il servizio EM reinserisce semplicemente una regola M2.1.0 eliminata alla successiva esecuzione oraria.

In pratica, ciò significa: attendere il 16 luglio prima di rimuoverla manualmente, oppure bloccare esplicitamente la mitigazione affinché non venga riattivata.

### Come rimuovere correttamente la mitigazione (percorso servizio EM)

Innanzitutto, verificare cosa è stato effettivamente applicato:

```powershell
Get-ExchangeServer -Identity <Servername> | Format-List Name,MitigationsApplied,MitigationsBlocked
```

Per impedire la riattivazione, l'ID della mitigazione viene aggiunto alla lista di blocco: le voci presenti vengono ignorate dal servizio EM durante l'esecuzione oraria.

```powershell
Set-ExchangeServer -Identity <Servername> -MitigationsBlocked @("M2.1.0")
```

Rimuovere quindi la regola IIS vera e propria. Un dettaglio utile e raramente documentato: il servizio EM crea le proprie regole URL Rewrite con il **prefisso «EEMS `<Mitigation-ID>` `<Beschreibung>`»**. Ciò consente di individuarle senza ambiguità in IIS Manager, in URL Rewrite (oppure tramite `appcmd`/PowerShell nell'`applicationHost.config`), senza dover indovinare quale regola appartenga alla mitigazione. Dopo la distribuzione della modifica lato server, è possibile rimuovere nuovamente il blocco (`-MitigationsBlocked @()`), se era stato impostato solo come soluzione temporanea.

### Percorso EOMT (ambienti isolati o air-gapped)

Se la mitigazione è stata applicata tramite lo **script EOMT** scaricabile (https://aka.ms/UnifiedEOMT), il ripristino avviene mediante l'opzione di rollback:

```powershell
.\EOMT.ps1 -RollbackMitigation -CVE "CVE-2026-42897"
```

Anche qui un dettaglio poco noto: prima di ogni modifica, EOMT salva lo stato iniziale di IIS in un **file di backup JSON specifico per CVE** in `%WINDIR%\System32\inetsrv\config\`. Il rollback legge esattamente questo file e ripristina le impostazioni originarie. Importante: una mitigazione applicata con uno script legacy (EOMTv2 ecc.) deve essere rimossa anche tramite il relativo meccanismo di rollback: i formati di backup non sono compatibili.

### Perché vale la pena rimuoverla

La mitigazione non è «gratuita». Finché rimane attiva, comporta i suoi effetti collaterali noti: la funzione OWA «Stampa calendario» non funziona, le immagini inline potrebbero non essere visualizzate correttamente nel riquadro di lettura OWA, OWA Light (`/?layout=light`) è difettoso (e verrà comunque disattivato prossimamente), mentre i calendari pubblicati restituiscono talvolta errori 500. Particolarmente insidioso per il monitoraggio: il health set **OWACalendar.Proxy** può passare a *unhealthy*, generando così falsi allarmi nel monitoraggio. Chi installa la SU ma lascia attiva la mitigazione finirà per cercare errori che non esistono. Non appena l'aggiornamento è installato *e* la mitigazione è rimossa, scompaiono anche questi problemi noti.

Un caso particolare: in ambienti misti, i server non ancora aggiornati possono mantenere la mitigazione. Occorre tuttavia sapere che l'integrazione Office Online Server (OOS) potrebbe tornare a funzionare correttamente solo quando *tutti* i server Exchange dell'organizzazione saranno aggiornati alla versione di luglio.

## Health Checker: individuare gruppi di sicurezza antichissimi

Il secondo punto, indipendente dalla release della SU: **Exchange Health Checker** (https://aka.ms/ExchangeHealthChecker) verifica ora l'esistenza di due gruppi di sicurezza da tempo deprecati: **«Exchange Domain Servers»** e **«Exchange Enterprise Servers»**.

### Da dove provengono questi gruppi e perché rappresentano un rischio

Questi due gruppi derivano dal modello di autorizzazioni di Exchange 2000/2003 e sono deprecati da Exchange 2007. Con Exchange 2007/2010 sono stati introdotti il modello Split Permissions e RBAC e, da allora, questi gruppi non vengono più utilizzati. Il problema è che non sono scomparsi. In molte directory giacciono inosservati da circa due decenni e talvolta conservano ancora ACL estese del vecchio modello, cioè più autorizzazioni di quante ne avrebbe mai un moderno gruppo di sicurezza Exchange.

È proprio questo a renderli un vettore di attacco. Un gruppo inattivo con autorizzazioni ampie permanenti costituisce una classica catena di escalation: chi riesce ad aggiungere sé stesso, o un account controllato, a uno di questi gruppi eredita i relativi diritti nella directory. Poiché nessuno monitora attivamente il gruppo, una simile manipolazione passa difficilmente inosservata.

### Perché la maggior parte degli amministratori non li conosce

Questi gruppi sono un punto cieco per diversi motivi: sono inattivi da circa 20 anni, nella maggior parte dei casi esistevano già prima dell'arrivo dell'attuale team, sopravvivono senza problemi a ogni migrazione e finora non venivano segnalati da Health Checker. Aspetto particolarmente delicato: sopravvivono persino alla *completa* dismissione di Exchange on-premises. Chi rimuove l'ultimo server Exchange solitamente elimina gli oggetti server, ma trascura completamente questi gruppi legacy.

### Pulizia

In futuro Health Checker segnalerà automaticamente i gruppi. Manualmente, possono essere individuati in Active Directory (solitamente nel contenitore `Users`) oppure tramite PowerShell:

```powershell
Get-ADGroup -Filter "Name -eq 'Exchange Domain Servers' -or Name -eq 'Exchange Enterprise Servers'"
```

Procedura: verificare l'appartenenza e gli eventuali riferimenti ACL personalizzati, accertarsi che nulla in produzione vi faccia riferimento e infine eliminare i gruppi. Poiché sono deprecati dal 2007, nella grande maggioranza degli ambienti possono essere rimossi senza rischi. Chi non utilizza più alcun Exchange on-premises dovrebbe pianificare contestualmente una pulizia AD più completa secondo la guida ufficiale Microsoft.

Hayes Jupe ha pubblicato nel suo post del blog [Latest Exchange health check script and deprecated groups](https://www.hayesjupe.com/latest-exchange-health-check-script-and-deprecated-groups/) una guida dettagliata per la rimozione dei gruppi.

## Procedura consigliata

In breve, la sequenza pratica è la seguente: innanzitutto inventariare l'ambiente con Health Checker (che mostra CU/SU mancanti, passaggi manuali aperti *e* ora anche i gruppi legacy). Quindi installare la CU corrente e la SU di luglio, riavviare il server e verificare che tutti i servizi Exchange siano stati avviati correttamente. Eseguire poi nuovamente Health Checker, rimuovere la mitigazione CVE-2026-42897 (dopo il 16 luglio oppure bloccando prima l'ID M2.1.0) e infine ripulire i gruppi di sicurezza deprecati. Le SU sono cumulative: chi utilizza una CU supportata non deve installare ogni SU intermedia, ma installa direttamente la più recente.

## Fonti

1.  [Released: July 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-july-2026-exchange-server-security-updates/4534146): Annuncio ufficiale della release di luglio con le versioni supportate e il problema noto dei messaggi wrapper.

2.  [Addressing Exchange Server May 2026 vulnerability CVE-2026-42897 – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/addressing-exchange-server-may-2026-vulnerability-cve-2026-42897/4518498): Avviso di sicurezza originale, con mitigazione di emergenza ed effetti collaterali noti in OWA.

3.  [Released: June 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-june-2026-exchange-server-security-updates/4524491): La release di giugno che ha fornito la correzione effettiva per CVE-2026-42897.

4.  [Exchange Emergency Mitigation Service (Exchange EM Service) – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-emergency-mitigation-service): Funzionamento del servizio EM, che confronta le mitigazioni ogni ora e reinserisce una regola eliminata prematuramente.

5.  [Set-ExchangeServer (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-exchangeserver): Parametri `MitigationsApplied` e `MitigationsBlocked` per verificare le mitigazioni e impedirne la riattivazione.

6.  [Exchange On-premises Mitigation Tool (EOMT) – Microsoft CSS-Exchange](https://microsoft.github.io/CSS-Exchange/Security/EOMT/): Lo script EOMT, incluse l'opzione di rollback e la copia di sicurezza JSON specifica per CVE dello stato iniziale di IIS.

7.  [CVE-2026-42897 Detail – NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-42897): Descrizione tecnica e valutazione della vulnerabilità nel National Vulnerability Database.
