---
title: "Eseguire il backup di HIN Mailgateway e ripristinarlo dopo un guasto"
navTitle: "Backup e ripristino"
description: "Un cluster protegge HIN Mailgateway dal guasto di un nodo, ma non sostituisce un backup. Sono determinanti la configurazione, il materiale delle chiavi, l'ordine di ripristino e le modifiche introdotte da Stargate."
date: "2026-07-08"
kategorie: "HIN-Gateway"
timeToRead: "15 min di lettura"
themen:
  - "hin-gateway"
slug: "eseguire-il-backup-di-hin-mailgateway-e-ripristinarlo-dopo-un-guasto"
translationOf: "hin-mailgateway-backup-disaster-recovery"
url: "https://rafaelpfister.ch/it/blog/eseguire-il-backup-di-hin-mailgateway-e-ripristinarlo-dopo-un-guasto"
---

# Eseguire il backup di HIN Mailgateway e ripristinarlo dopo un guasto

Molti HIN Mailgateway in produzione operano come cluster. Se un nodo si guasta, subentra l'altro. Tuttavia, questa ridondanza non protegge da una regola errata, un certificato eliminato o un'importazione danneggiata: [i dati rilevanti per il sistema vengono replicati su tutti i nodi](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html), incluse le modifiche indesiderate.

Per un ripristino affidabile è quindi necessario un backup separato. Poiché HIN Mailgateway si basa tecnicamente su un'appliance SEPPmail con GINA, si applicano i relativi meccanismi documentati di backup e ripristino.

## Quali dati risiedono sul gateway

Il gateway elabora le e-mail in entrata e in uscita in base a un set di regole centrale e le cifra, a seconda del destinatario, tramite S/MIME, OpenPGP o TLS; per i destinatari senza proprio materiale di chiavi viene utilizzata la procedura web GINA. Per il backup è fondamentale che [i contenuti dei messaggi non vengano memorizzati in modo persistente sul gateway](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): l'appliance elabora le e-mail in transito, senza archiviarle.

  

## Cosa replica il cluster

SEPPmail supporta diverse [configurazioni di cluster: alta disponibilità, bilanciamento del carico e geo-cluster](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html); su tutti i nodi vengono sincronizzati parametri di sistema, dati utente e materiale delle chiavi. Nel [cluster frontend/backend il frontend non dispone di un proprio database di configurazione](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): può operare in una DMZ senza archiviazione dei dati e riceve solo i dati necessari all'elaborazione corrente; il database con le chiavi risiede sul backend. Per il [Large File Transfer (LFT) si applica un'eccezione](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): a ogni partner, inclusi i frontend, viene assegnato un disco delle stesse dimensioni e i dati LFT vengono sincronizzati su tutti i nodi.

  

## Perché la replica non è un backup

> *La replica copia lo stato attuale, incluso quello errato. Un backup conserva uno stato noto e funzionante.*

Un'importazione errata, una chiave eliminata o un dominio disattivato vengono replicati sul nodo partner entro pochi secondi. Senza un backup indipendente, non esiste più alcun punto di ripristino. Quanto strettamente disponibilità e coerenza siano correlate nel cluster è emerso con i [problemi di accesso dopo l'aggiornamento alla versione 15.0.5](/blog/hin-update-issue-version-15.0.5), causati da una replica del cluster compromessa.

  

## Cosa è incluso nel backup e cosa no

Il [backup SEPPmail è volutamente snello](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): comprende esclusivamente configurazione e materiale di chiavi crittografiche: [nessun messaggio, nessuna coda di posta e categoricamente nessun log](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) (i log devono quindi essere inviati tramite Syslog a un sistema esterno). Dal firmware 14.0.0 l'appliance crea il backup [automaticamente ogni notte a mezzanotte come backup.tgz](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); può essere recuperato tramite `Download`, `Send Backup` (e-mail al gruppo di backup) o SCP.

| **Incluso nel backup** | **Non incluso nel backup** |
| --- | --- |
| Configurazione di sistema e set di regole | Contenuti e testi delle e-mail |
| Account utente e GINA | Coda di posta attuale |
| Materiale delle chiavi: S/MIME, X.509, OpenPGP | Log di sistema e di posta (salvare esternamente tramite Syslog) |
| Configurazione TLS e certificati | Sistema operativo / immagine VM |


Ne consegue che, poiché il sistema operativo non è incluso nel backup di configurazione, una strategia DR completa necessita anche di un metodo per ripristinare la base dell'appliance, mediante nuova distribuzione dall'immagine del produttore o snapshot della VM. Il backup di configurazione ripristina quindi configurazione e chiavi.

  

## Gli snapshot non sono un backup del cluster

Dal firmware 14.0.0 l'appliance crea inoltre [snapshot locali, ma solo se è presente una partizione LFT con database](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). La domenica viene creato uno snapshot completo, dal lunedì al sabato uno snapshot incrementale al giorno; la conservazione è di 14 giorni.

Per la pianificazione DR è decisivo quanto segue: nel funzionamento in cluster questi snapshot vengono eseguiti in background, ma non è disponibile alcun ripristino da essi. Gli snapshot sono quindi uno strumento di rollback locale su sistemi singoli, non un recovery del cluster. Il backup affidabile rimane il backup di configurazione cifrato.

  

## Configurare il backup

Per ogni modalità di recupero è necessario impostare una password di backup in [Amministrazione › Backup › Modifica password](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); senza questa password non è possibile né scaricare né inviare il backup né renderlo disponibile via SCP. Per impostazione predefinita, il backup notturno viene inviato via e-mail al [gruppo «backup (Backup Operator)»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html); un utente di backup dedicato necessita di un indirizzo e-mail interno valido.

-   Impostare la password di backup e [conservarla separatamente dal backup](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): il backup contiene chiavi private.
    
-   Per l'archiviazione automatizzata, [recuperare i backup tramite SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): memorizzare le chiavi pubbliche `SSH-RSA`\- nell'amministrazione e recuperare il file `backup.tgz` reso disponibile a mezzanotte tramite l'utente OS `backup`.
    
-   Salvare separatamente i log tramite Syslog esterno, poiché [non fanno intenzionalmente parte del backup](https://docs.seppmail.com/de/07_mi_11_adm__administration.html).
    

  

## Strategia di backup nel funzionamento in cluster

Nel funzionamento in cluster, sono essenziali un backup ordinato e una gestione coerente delle versioni.

-   **Ogni giorno**: recuperare il backup di configurazione cifrato [tramite SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) e archiviarlo esternamente con versionamento
    
-   **Ogni settimana**: backup completo della VM o del sistema di entrambi i nodi, sfalsato nel tempo anziché simultaneo (il sistema operativo non fa parte del backup di configurazione)
    
-   **Prima di manutenzione o aggiornamento**: interrompere l'accettazione delle e-mail tramite [Preempt](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): le e-mail in entrata vengono temporaneamente rifiutate con un codice di ritorno SMTP configurabile (predefinito `421`); l'impostazione resta attiva anche dopo un riavvio.
    

  

Per quanto riguarda la gestione delle versioni: nel cluster frontend/backend SEPPmail aggiorna [il frontend prima del backend](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), e negli aggiornamenti a più livelli tutti i partner devono avere la stessa versione prima di passare alla release successiva. Dopo un major update può essere necessaria la rigenerazione del set di regole (messaggio *«Current ruleset created for another version»*).

  

## Ripristino e Disaster Recovery

Il caso base è semplice: [Import backup file](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), riavvio, quindi il gateway torna a operare con tutte le funzionalità. Occorre osservare la regola delle versioni: nella versione attuale può essere importato solo il backup della versione firmware immediatamente precedente, dopo di che occorre rigenerare il set di regole; non è possibile importare il backup di un firmware più recente in una versione precedente.

Nel cluster si applica un'importante limitazione:

-   **Non ripristinare mai direttamente un singolo nodo**: il [ripristino di un singolo partner del cluster non è previsto](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). Rimuovere invece la macchina difettosa dal cluster, configurare una nuova VM e aggiungerla nuovamente: configurazione e chiavi arrivano automaticamente tramite replica dal partner integro.
    
-   **Perdita totale su tutti i nodi**: ridistribuire l'appliance dall'immagine di base, quindi importare l'ultimo backup di configurazione noto e funzionante e riavviare.
    

Un backup è affidabile quanto l'ultimo test di ripristino riuscito. Un ripristino di prova dovrebbe essere eseguito almeno due volte all'anno in un ambiente isolato, non sul cluster di produzione.

  

### Checklist di ripristino per le emergenze

1.  Rimuovere il nodo difettoso dal cluster, senza ripristinare direttamente un partner.
    
2.  Configurare una nuova VM oppure, in caso di perdita totale, predisporre l'appliance dall'immagine di base/snapshot VM.
    
3.  Solo in caso di perdita totale: importare l'ultimo backup di configurazione funzionante (tenere pronta la password e rispettare la regola delle versioni).
    
4.  Verificare il nodo in isolamento: accettazione SMTP, TLS, GINA, set di regole.
    
5.  Aggiungerlo al cluster e monitorare la replica; se richiesto, rigenerare il set di regole.
    
6.  Documentare l'incidente e aggiornare l'intervallo di backup e le versioni.
    

  

Due operazioni di manutenzione richiedono particolare cautela e sempre un backup preventivo: [l'ampliamento della partizione LFT arresta l'appliance](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), e il Factory Reset sovrascrive il disco rigido dieci volte, mentre la richiesta di conferma impone il codice scritto al contrario.

  

## Cosa cambia con «Stargate»

HIN sta sostituendo gradualmente il precedente Mailgateway con il [nuovo HIN Gateway](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) (progetto «Stargate», denominato «Verimesh» presso la zugese [Vereign AG](https://www.vereign.com/)). Non si tratta di una sostituzione 1:1 dell'appliance, bensì di un cambiamento architetturale che incide direttamente su backup e Disaster Recovery:

-   **Da centralizzato a decentralizzato**: i nodi comunicano direttamente tra loro; non è più necessario un centro di distribuzione centrale.
    
-   **Gestione decentralizzata delle chiavi (DKMS)**: ogni organizzazione gestisce la propria identità crittografica, senza una Certificate Authority centrale.
    
-   **Crittografia end-to-end** con frammentazione dei messaggi.
    
-   **Resilienza dalla rete**: se un nodo si guasta, la mesh rimane operativa.
    
-   **Implementazione di riferimento aperta**: la [Vereign Client Library (vcl)](https://code.vereign.com/code/vcl/-/tree/0.4-rc1) è disponibile come codice sorgente aperto sotto AGPLv3.
    

Tempistica: l'infrastruttura decentralizzata è [in uso produttivo nel settore sanitario svizzero dall'aprile 2025](https://www.vereign.com/); per il 2026 sono previste la graduale sostituzione dei precedenti Mailgateway e un'ampia distribuzione. Le organizzazioni con domini propri HIN (`@hin.ch`, `@verband-hin.ch`) operano sull'infrastruttura HIN e sono appena interessate dalla transizione.

  

Per il manuale operativo ciò significa che la disciplina classica di «esportare la configurazione e le chiavi dell'appliance e ripristinarle su un nodo sostitutivo» perde importanza. Al suo posto subentrano l'onboarding dei nodi, la custodia di identità e chiavi nella mesh e la riammissione dei nodi nella rete.

  

## La distinzione più importante

Finché HIN MGW opera su tecnologia SEPPmail, vale quanto segue: il cluster compensa i guasti hardware, ma la responsabilità dell'integrità della configurazione e delle chiavi rimane all'operatore. Il backup snello di configurazione deve essere protetto indipendentemente dal cluster, tramite SCP, con versionamento e password conservata separatamente; gli snapshot non lo sostituiscono nel cluster, le versioni rimangono sincronizzate e il ripristino viene testato regolarmente in isolamento. Il passaggio a Stargate dovrebbe essere integrato tempestivamente nella pianificazione DR, poiché trasferisce resilienza e custodia delle chiavi nella rete decentralizzata.

## Fonti

1.  [Documentazione SEPPmail – «Backup / Restore»](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): contenuto del backup, solo configurazione e materiale delle chiavi, creazione notturna, ripristino automatico del cluster tramite replica.
    
2.  [Documentazione SEPPmail – «Administration»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): riferimento dettagliato: menu Backup (Download / Send Backup / Change password, `backup.tgz` a mezzanotte), snapshot LFT (14 giorni, nessun ripristino nel cluster), regole di ripristino e procedura per cluster, Preempt (codice di ritorno SMTP, predefinito 421), clonazione del dispositivo, canali e sequenza di aggiornamento (frontend prima del backend), Factory Reset, importazione/esportazione in blocco.
    
3.  [Documentazione SEPPmail – «Creare utente di backup»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): gruppo «backup (Backup operator)», cifratura e gestione delle password.
    
4.  [Documentazione SEPPmail – «Copiare il backup tramite SCP»](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): recupero del file `backup.tgz` tramite SCP attraverso l'utente OS `backup` anziché invio via e-mail.
    
5.  [Documentazione SEPPmail – «Cluster / Alta disponibilità»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): tipi di cluster e dati sincronizzati su tutti i nodi, parametri di sistema, dati utente e materiale delle chiavi.
    
6.  [Documentazione SEPPmail – «Cluster frontend/backend»](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): frontend senza database di configurazione, operatività DMZ, dati on demand; backend come detentore dei dati.
    
7.  [Documentazione SEPPmail – «Archiviazione dati nel cluster (LFT)»](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): disco aggiuntivo delle stesse dimensioni per ogni partner, sincronizzazione dei dati LFT su tutti i nodi.
    
8.  [HIN AG – «Dal Mailgateway al Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): comunicazione HIN sul successore Stargate: nodi decentralizzati, concetto Data Mesh, tempistica, crittografia end-to-end.
    
9.  [Vereign AG – «Verimesh» / Vereign Client Library (vcl, tag 0.4-rc1)](https://code.vereign.com/code/vcl/-/tree/0.4-rc1): base tecnica di Stargate: gestione decentralizzata delle chiavi (DKMS), crittografia end-to-end con frammentazione dei messaggi, codice sorgente aperto sotto AGPLv3.
