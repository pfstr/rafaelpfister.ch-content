---
title: "Eseguire il backup dell'HIN Mailgateway e ripristinarlo dopo un guasto"
navTitle: "Backup e ripristino"
description: "Un cluster protegge l'HIN Mailgateway dal guasto di un nodo, ma non sostituisce un backup. Sono determinanti la configurazione, il materiale delle chiavi, la sequenza di ripristino e le modifiche introdotte da Stargate."
date: "2026-07-08"
kategorie: "HIN-Gateway"
timeToRead: "15 min di lettura"
themen:
  - hin-gateway
slug: "eseguire-il-backup-di-hin-mailgateway-e-ripristinarlo-dopo-un-guasto"
translationOf: "hin-mailgateway-backup-disaster-recovery"
translationId: article-845fb4bd0e4c592a
translationReview: required
translationSourceHash: cd8f7c8633d8cf882725ad88feed7be5a1730ce563724481e25b4b9ee2ec84e4
translatedAt: 2026-11-06T03:22:54.214Z
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/eseguire-il-backup-di-hin-mailgateway-e-ripristinarlo-dopo-un-guasto
translationModel: gpt-5.6-terra
---

# Eseguire il backup dell'HIN Mailgateway e ripristinarlo dopo un guasto

Molti HIN Mailgateway in produzione funzionano come cluster. Se un nodo si guasta, subentra l'altro. Questa ridondanza non protegge tuttavia da una regola errata, un certificato eliminato o un'importazione danneggiata: [i dati rilevanti per il sistema vengono replicati su tutti i nodi](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html), comprese le modifiche indesiderate.

Per un ripristino affidabile è quindi necessario un backup separato. Poiché l'HIN Mailgateway si basa tecnicamente su un'appliance SEPPmail con GINA, si applicano i relativi meccanismi documentati di backup e ripristino.

## Quali dati risiedono sul gateway

Il gateway elabora le e-mail in entrata e in uscita secondo un insieme di regole centralizzato e, a seconda del destinatario, le cifra tramite S/MIME, OpenPGP o TLS; per i destinatari senza materiale crittografico proprio viene utilizzata la procedura GINA basata sul web. Per il backup è decisivo che [i contenuti dei messaggi non vengano memorizzati in modo persistente sul gateway](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): l'appliance elabora le e-mail in transito, senza archiviarle.

  

## Cosa replica il cluster

SEPPmail supporta diverse [tipologie di cluster: alta disponibilità, bilanciamento del carico e geo-cluster](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html); su tutti i nodi vengono sincronizzati parametri di sistema, dati utente e materiale delle chiavi. In un [cluster frontend/backend, il frontend non dispone di un proprio database di configurazione](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): può essere eseguito in una DMZ senza memorizzazione di dati e riceve soltanto i dati necessari per l'elaborazione corrente; il database, comprese le chiavi, risiede sul backend. Per il [Large File Transfer (LFT) vale un'eccezione](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): a ogni partner, inclusi i frontend, viene assegnato un disco della stessa dimensione e i dati LFT vengono sincronizzati su tutti i nodi.

  

## Perché la replica non è un backup

> *La replica copia lo stato attuale, compreso quello errato. Un backup conserva uno stato noto e funzionante.*

Un'importazione errata, una chiave eliminata o un dominio disattivato vengono replicati entro pochi secondi sul nodo partner. Senza un backup indipendente, non esiste più alcun punto di ripristino. Quanto siano strettamente legate disponibilità e coerenza nel cluster è emerso con i [problemi di accesso dopo l'aggiornamento alla versione 15.0.5](/blog/hin-update-issue-version-15.0.5), causati da una replica del cluster compromessa.

  

## Cosa è incluso nel backup e cosa no

Il [backup SEPPmail è volutamente snello](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): comprende esclusivamente la configurazione e il materiale crittografico delle chiavi: [nessun messaggio, nessuna coda di posta e esplicitamente nessun log](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) (i log devono pertanto essere inviati tramite Syslog a un sistema esterno). Dal firmware 14.0.0, l'appliance crea il backup [automaticamente di notte a mezzanotte come backup.tgz](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); può essere recuperato tramite `Download`, `Send Backup` (e-mail al gruppo di backup) o SCP.

| **Incluso nel backup** | **Non incluso nel backup** |
| --- | --- |
| Configurazione di sistema e insieme di regole | Contenuti e testi delle e-mail |
| Account utente e GINA | Coda di posta attuale |
| Materiale delle chiavi: S/MIME, X.509, OpenPGP | Log di sistema e della posta (proteggere esternamente tramite Syslog) |
| Configurazione TLS e certificati | Sistema operativo / immagine VM |


Ne consegue che, poiché il sistema operativo non è incluso nel backup di configurazione, una strategia DR completa richiede anche un modo per ripristinare la base dell'appliance (nuova distribuzione dall'immagine del produttore o snapshot della VM). Il backup di configurazione ripristina poi configurazione e chiavi.

  

## Gli snapshot non sono un backup del cluster

Dal firmware 14.0.0, l'appliance crea inoltre [snapshot locali, ma solo se è presente una partizione LFT con database](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). La domenica viene creato uno snapshot completo, mentre dal lunedì al sabato uno incrementale al giorno; la conservazione è di 14 giorni.

Un aspetto decisivo per la pianificazione DR: in un cluster questi snapshot vengono sì eseguiti in background, ma non viene offerto alcun ripristino da essi. Gli snapshot sono quindi uno strumento di rollback locale sui sistemi singoli, non un recovery del cluster. La protezione affidabile rimane il backup di configurazione cifrato.

  

## Configurare il backup

Il presupposto per ogni metodo di recupero è l'impostazione di una password di backup in [Administration › Backup › Change password](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); senza questa password non è possibile né scaricare né inviare né rendere disponibile il backup tramite SCP. Per impostazione predefinita, il backup notturno viene inviato via e-mail al [gruppo «backup (Backup Operator)»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html); un utente di backup dedicato necessita di un indirizzo e-mail interno valido.

-   Impostare la password di backup e [conservarla separatamente dal backup](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): il backup contiene chiavi private.
    
-   Per l'archiviazione automatizzata, [recuperare i backup tramite SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): memorizzare la chiave pubblica `SSH-RSA`\-nell'amministrazione e prelevare a mezzanotte il file `backup.tgz` reso disponibile tramite l'utente del sistema operativo `backup`.
    
-   Proteggere separatamente i log (Syslog esterno), poiché [non fanno volutamente parte del backup](https://docs.seppmail.com/de/07_mi_11_adm__administration.html).
    

  

## Strategia di backup nel funzionamento in cluster

Nel funzionamento in cluster, sono fondamentali un backup ordinato e una gestione coerente delle versioni.

-   **Ogni giorno**: recuperare il backup di configurazione cifrato [tramite SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) e archiviarlo esternamente con versionamento
    
-   **Ogni settimana**: backup completo della VM o del sistema di entrambi i nodi, sfalsato nel tempo anziché simultaneo (il sistema operativo non fa parte del backup di configurazione)
    
-   **Prima della manutenzione o di un aggiornamento**: interrompere la ricezione della posta [con Preempt](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): le e-mail in entrata vengono temporaneamente rifiutate con un codice di ritorno SMTP configurabile (predefinito `421`); l'impostazione rimane attiva anche dopo un riavvio.
    

  

Per la gestione delle versioni: nel cluster frontend/backend SEPPmail aggiorna [il frontend prima del backend](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), e negli aggiornamenti in più fasi tutti i partner devono avere la stessa versione prima di passare alla release successiva. Dopo un aggiornamento principale può essere necessario rigenerare il ruleset (messaggio *«Current ruleset created for another version»*).

  

## Ripristino e Disaster Recovery

Il caso base è semplice: [Import backup file](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), riavvio, quindi il gateway opera nuovamente con tutte le funzionalità. Va osservata la regola sulle versioni: nella versione attuale è possibile importare soltanto il backup della versione firmware immediatamente precedente (poi occorre rigenerare il ruleset); non è possibile importare il backup di un firmware più recente in una versione precedente.

Nel cluster vale una limitazione importante:

-   **Non ripristinare mai direttamente un singolo nodo**: un [ripristino di un singolo partner del cluster non è previsto](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). Rimuovere invece la macchina difettosa dal cluster, creare una nuova VM e aggiungerla nuovamente: configurazione e chiavi arrivano automaticamente dal partner integro tramite replica.
    
-   **Perdita totale su tutti i nodi**: ridistribuire l'appliance dall'immagine di base, quindi importare l'ultimo backup di configurazione noto e funzionante e riavviare.
    

Un backup è affidabile solo quanto l'ultimo test di ripristino riuscito. Un test di ripristino dovrebbe essere eseguito almeno due volte l'anno in un ambiente isolato, non sul cluster di produzione.

  

### Checklist di ripristino per l'emergenza

1.  Rimuovere il nodo difettoso dal cluster (nessun ripristino diretto di un partner).
    
2.  Creare una nuova VM oppure, in caso di perdita totale, predisporre l'appliance dall'immagine di base/snapshot della VM.
    
3.  Solo in caso di perdita totale: importare l'ultimo backup di configurazione funzionante (tenere pronta la password, rispettare la regola sulle versioni).
    
4.  Verificare il nodo in isolamento: ricezione SMTP, TLS, GINA, insieme di regole.
    
5.  Aggiungerlo al cluster e monitorare la replica; se viene visualizzato il messaggio, rigenerare il ruleset.
    
6.  Documentare l'incidente e aggiornare l'intervallo di backup e le versioni.
    

  

Due azioni di manutenzione richiedono particolare cautela e sempre un backup preventivo: [l'espansione della partizione LFT arresta l'appliance](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), e il reset di fabbrica sovrascrive il disco rigido dieci volte (la conferma di sicurezza richiede il codice scritto al contrario).

  

## Cosa cambia con «Stargate»

HIN sostituisce gradualmente il precedente Mailgateway con il [nuovo HIN Gateway](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) (progetto «Stargate», indicato presso la [Vereign AG come «Verimesh»](https://www.vereign.com/)). Non si tratta di una sostituzione uno a uno dell'appliance, bensì di un cambiamento architetturale che riguarda nel profondo backup e Disaster Recovery:

-   **Da centralizzato a decentralizzato**: i nodi comunicano direttamente tra loro; non vi è più un centro di distribuzione centrale.
    
-   **Gestione decentralizzata delle chiavi (DKMS)**: ogni organizzazione gestisce la propria identità crittografica, senza una Certificate Authority centrale.
    
-   **Crittografia end-to-end** con frammentazione dei messaggi.
    
-   **Resilienza dalla rete**: se un nodo si guasta, il mesh rimane operativo.
    
-   **Repository di deployment aperto**: il repository [Stargate Deployment](https://code.vereign.com/svdh/stargate-deployment) con i servizi di un nodo Stargate è pubblicamente consultabile sotto AGPLv3; la Vereign Client Library (vcl), invece, è disponibile solo dopo l'accesso.
    

Tempistica: l'infrastruttura decentralizzata è [in uso produttivo nel settore sanitario svizzero dall'aprile 2025](https://www.vereign.com/); per il 2026 sono previste la graduale sostituzione dei precedenti Mailgateway e un'ampia distribuzione. Le organizzazioni con domini propri HIN (`@hin.ch`, `@verband-hin.ch`) operano sull'infrastruttura HIN e sono appena interessate dalla transizione.

  

Per il manuale operativo questo significa che la classica disciplina di «esportare la configurazione e le chiavi dell'appliance e ripristinarle su un nodo sostitutivo» perde importanza. Al suo posto subentrano l'onboarding dei nodi, la custodia delle identità e delle chiavi nel mesh e la reintegrazione dei nodi nella rete.

  

## La separazione più importante

Finché l'HIN MGW opera sulla tecnologia SEPPmail, vale quanto segue: il cluster compensa i guasti hardware, ma la responsabilità dell'integrità della configurazione e delle chiavi rimane all'operatore. Il backup snello di configurazione deve essere protetto indipendentemente dal cluster (tramite SCP, con versionamento e password conservata separatamente), gli snapshot non lo sostituiscono nel cluster, le versioni restano sincronizzate e il ripristino viene regolarmente testato in isolamento. La migrazione a Stargate dovrebbe essere integrata tempestivamente nella pianificazione DR, poiché sposta resilienza e custodia delle chiavi nella rete decentralizzata.

## Cronologia delle modifiche

- **25 ottobre 2026:** Aggiunto: la Vereign Client Library è disponibile solo dopo l'accesso; il repository Stargate Deployment è ora indicato come fonte pubblicamente consultabile sotto AGPLv3.

## Fonti

1.  [Documentazione SEPPmail – «Backup / Restore»](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): contenuto del backup (solo configurazione e materiale delle chiavi), creazione notturna, ripristino automatico del cluster tramite replica.
    
2.  [Documentazione SEPPmail – «Administration»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): riferimento dettagliato: menu di backup (Download / Send Backup / Change password, `backup.tgz` a mezzanotte), snapshot LFT (14 giorni, nessun ripristino nel cluster), regole di ripristino e procedura del cluster, Preempt (codice di ritorno SMTP, predefinito 421), Device Cloning, canali di aggiornamento e ordine di aggiornamento (frontend prima del backend), Factory Reset, importazione/esportazione massiva.
    
3.  [Documentazione SEPPmail – «Creare un utente di backup»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): gruppo «backup (Backup operator)», cifratura e gestione delle password.
    
4.  [Documentazione SEPPmail – «Copiare il backup tramite SCP»](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): recupero di `backup.tgz` tramite SCP con l'utente del sistema operativo `backup` anziché invio via e-mail.
    
5.  [Documentazione SEPPmail – «Cluster / Alta disponibilità»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): tipi di cluster e dati sincronizzati su tutti i nodi (parametri di sistema, dati utente, materiale delle chiavi).
    
6.  [Documentazione SEPPmail – «Cluster frontend/backend»](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): frontend senza database di configurazione, funzionamento in DMZ, dati on-demand; backend come archivio dei dati.
    
7.  [Documentazione SEPPmail – «Archiviazione dati nel cluster (LFT)»](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): disco aggiuntivo della stessa dimensione per ciascun partner, sincronizzazione dei dati LFT su tutti i nodi.
    
8.  [HIN AG – «Dal Mailgateway al Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): comunicazione HIN sul successore Stargate: nodi decentralizzati, concetto Data Mesh, tempistica, crittografia end-to-end.
    
9.  [Vereign AG – «Verimesh»](https://vereign.com/): Verimesh (precedentemente progetto Stargate) in uso produttivo nel settore sanitario svizzero dall'aprile 2025.
    
10. [Vereign GitLab – Stargate Deployment](https://code.vereign.com/svdh/stargate-deployment): repository di deployment pubblico sotto AGPLv3 con i servizi di un nodo Stargate e rimando alla [documentazione di deployment](https://health-info-net-ag.github.io/Stargate-deployment/).
