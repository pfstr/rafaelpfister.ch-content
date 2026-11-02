---
title: "totemomail 9.5.1: risolte l'acquisizione di account nel Web Portal e SSRF tramite e-mail in entrata"
navTitle: "totemomail 9.5.1"
description: "totemomail 9.5.1 risolve otto vulnerabilità di sicurezza e porta totemomail al livello a cui fanno riferimento gli advisory Kiteworks pubblicati a fine settembre per l'Email Protection Gateway. Due vulnerabilità sono sfruttabili senza autenticazione: un errore nel reset della password del Web Portal che consente l'acquisizione di account amministratore e richieste lato server di URL presenti nei messaggi in entrata. Si aggiungono correzioni di path traversal nell'Admin API e nella Mail API, nonché librerie Tomcat e Batik aggiornate."
date: "2026-10-31"
kategorie: "Totemomail"
timeToRead: "5 min di lettura"
themen:
  - totemomail
  - sicherheitsluecken
produkte:
  - "totemomail"
  - "sicherheitsluecken"
protokolle:
  - "releases"
  - "haertung"
slug: "totemomail-9-5-1-risolte-l-acquisizione-di-account-nel-web-portal-e-ssrf-tramite-e-mail-in"
translationId: "article-2bd15f63632ab728"
draft: false
translationOf: totemomail-9-5-1-sicherheitsupdate
url: https://rafaelpfister.ch/it/blog/totemomail-9-5-1-risolte-l-acquisizione-di-account-nel-web-portal-e-ssrf-tramite-e-mail-in
translationSourceHash: 5d441bdc7ec39d7fcef423e826bf14772f11bca2aa3dbdf4e1e3c83297131635
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T19:46:40.016Z
translationReview: automatic
---

# totemomail 9.5.1: risolte l'acquisizione di account nel Web Portal e SSRF tramite e-mail in entrata

totemo ha rilasciato totemomail 9.5.1 (Build 101). La patch contiene otto correzioni di sicurezza, un miglioramento e tre correzioni di errori. Due vulnerabilità sono sfruttabili senza autenticazione: un errore nel reset della password del Web Portal, che consente anche l'acquisizione di account amministratore, e richieste lato server di URL presenti nei messaggi in entrata. Entrambe riguardano componenti che, nella maggior parte delle installazioni, sono raggiungibili da Internet. Pianificate quindi tempestivamente l'aggiornamento.

## Da 9.2.1 direttamente a 9.5.1

Le release note elencano la versione 9.5.1 immediatamente dopo la 9.2.1; le versioni 9.3 e 9.4 non compaiono nella cronologia di totemomail. Tuttavia, una voce nella 9.5.1 fa riferimento a un «security hardening introduced in 9.3.1». Le versioni intermedie provengono dal Kiteworks Email Protection Gateway (EPG), che si basa sullo stesso codice: gli avvisi di sicurezza di Kiteworks indicano come versioni corrette EPG 9.3.1, 9.4.0, 9.4.1, 9.5.0 e 9.5.1. Per le installazioni totemomail con la 9.2.1, la 9.5.1 è il prossimo aggiornamento; le release note non indicano altre patch intermedie.

## Vulnerabilità senza autenticazione

**Server-Side Request Forgery tramite il contenuto dei messaggi.** totemomail recuperava lato server URL da messaggi elaborati senza limitare sufficientemente la destinazione. Un mittente non autenticato poteva predisporre un messaggio in modo che il gateway inviasse richieste a servizi interni o endpoint di metadati di istanze cloud e restituisse le risposte. A tale scopo è sufficiente un'e-mail elaborata dal gateway; con un gateway nel flusso di posta in entrata, può farlo qualsiasi mittente esterno. Il caso è particolarmente delicato per le installazioni in Azure, AWS o altri cloud, dove l'endpoint di metadati `169.254.169.254` fornisce dati di configurazione e, a seconda della piattaforma, token di accesso dell'istanza. Le release note non indicano un numero CVE per questa vulnerabilità. Per l'EPG, Kiteworks elenca cinque advisory sull'accesso a risorse di rete interne (CVE-2026-102095, CVE-2026-102102 fino a CVE-2026-102105, tutti con CVSS 9.1, risolti in EPG 9.5.0); dai testi sintetici degli advisory non risulta quale di essi corrisponda alla voce di totemomail.

**Reset della password nel Web Portal (CVE-2026-85066).** Un errore nella procedura di reimpostazione della password consentiva di saltare i passaggi di verifica e impostare una nuova password per un account esistente. Le release note includono esplicitamente gli account amministratore. Kiteworks valuta la vulnerabilità nell'advisory associato GHSA-rwpq-5xfv-54pv con CVSS 9.8 (critico) e indica EPG 9.5.0 come versione corretta. Il Web Portal è destinato ai destinatari esterni di messaggi crittografati e pertanto è generalmente raggiungibile pubblicamente.

## Vulnerabilità con diritti di amministratore

Altre tre correzioni richiedono l'autenticazione con diritti amministrativi. In combinazione con l'acquisizione di account nel Web Portal, tali requisiti perdono tuttavia importanza.

- **Admin API, gestione della coda (CVE-2026-34843):** tramite upload di file protetti in modo insufficiente era possibile scrivere file in posizioni arbitrarie del file system. Secondo le release note, ciò consentiva la Remote Code Execution. L'advisory GHSA-mhc9-4pph-2r9x valuta la vulnerabilità con CVSS 7.2; nell'EPG è risolta dalla versione 9.4.0.
- **Rule Designer:** durante l'importazione di una configurazione di regole, la pipeline di elaborazione poteva caricare ed eseguire codice al di fuori dei componenti autorizzati per l'elaborazione della posta. Sono necessari diritti per configurare le regole di posta.
- **Mail API:** durante l'estrazione di file ZIP caricati era presente una vulnerabilità di path traversal. Ora i file vengono estratti solo nella directory temporanea.

La quarta correzione riguarda la proprietà `security.login.requestingKeysAvailable`. Finora, nel Web Portal nascondeva solo il collegamento per richiedere chiavi pubbliche; la funzione stessa restava raggiungibile tramite una richiesta diretta. Dalla 9.5.1, la proprietà disattiva la funzione. Se avete impostato la proprietà per impedire richieste di chiavi, avrà l'effetto previsto solo con questo aggiornamento.

## Cosa elenca in aggiunta il registro Kiteworks

Il 30 settembre 2026, Kiteworks ha pubblicato collettivamente 125 avvisi di sicurezza nel proprio registro di advisory su GitHub, di cui 28 relativi all'Email Protection Gateway. Le release note di totemomail 9.5.1 indicano molti meno punti. Il registro non distingue tra EPG e totemomail; non è quindi possibile stabilire se ogni vulnerabilità EPG riguardi anche totemomail. Le voci critiche dell'EPG sono:

| CVE | CVSS | Impatto secondo l'advisory | Risolto da EPG |
|---|---|---|---|
| CVE-2026-54154 | 10.0 | Esecuzione di codice con diritti root senza autenticazione | 9.4.1 |
| CVE-2026-85066 | 9.8 | Acquisizione di account utente (reset della password) | 9.5.0 |
| CVE-2026-85065 | 9.8 | Acquisizione di account utente | 9.5.0 |
| CVE-2026-102149 | 9.4 | Acquisizione di account utente | 9.5.1 |
| CVE-2026-102106 | 9.1 | Modifica della configurazione di amministrazione | 9.5.0 |
| CVE-2026-102095, CVE-2026-102102 fino a 102105 | 9.1 | Accesso a risorse di rete interne | 9.5.0 |

Si aggiungono 13 voci classificate come alte, per la maggior parte esecuzione di codice con diritti amministrativi (CVSS 7.2). La versione corretta più recente indicata è la 9.5.1; Kiteworks ha definito la 9.5.1 la versione attuale a fine settembre, nella quale sarebbero considerate tutte le vulnerabilità note. Chi aggiorna totemomail alla 9.5.1 raggiunge quindi il livello a cui fanno riferimento tutti gli advisory EPG del registro.

## Librerie aggiornate

Le librerie incluse di Apache Tomcat e Apache XML Graphics Batik sono state aggiornate. Le release note citano CVE-2026-50229, CVE-2026-53404, CVE-2026-53434, CVE-2026-55276, CVE-2026-55955, CVE-2026-55956 e CVE-2026-59083, senza attribuirle singolarmente alle due librerie. Inoltre, woodstox-core-asl è stato aggiornato a causa di una vulnerabilità; non è indicato alcun numero CVE.

## Miglioramento e correzioni di errori

- **Migrazione EPG:** durante la migrazione da un database Oracle, totemomail ora supporta Oracle Wallet. Se non è configurata una password del database, la migrazione si connette tramite il wallet.
- **RMS:** i messaggi potevano rimanere bloccati nell'elaborazione perché le azioni RMS attendevano senza limite l'autenticazione. Il tempo di attesa è ora limitato a un minuto. Ciò riguarda le installazioni con Microsoft Purview Information Protection (RMS).
- **TRE:** la funzione «Open Online» per le Registered Envelopes funziona di nuovo. Non funzionava dalla citata hardening della 9.3.1.
- **Rapporto sulle licenze:** per le licenze cloud, il rapporto conta di nuovo correttamente gli utenti interni con tipi di funzionalità predefiniti. L'articolo sul [limite di licenza e la pulizia tramite LDAP](/blog/totemomail-licensed-user-limit-ldap-cleanup) descrive l'impatto degli account orfani sul conteggio delle licenze.

## Installazione

La patch può essere installata su sistemi a partire da totemomail 7 b985 su Windows, Linux, Solaris e FreeBSD. Accedete alla console di amministrazione, caricate il file TPF fornito in `[Instanzname]` > «Update» e avviate l'aggiornamento. Con più istanze, l'operazione è necessaria singolarmente per ciascuna istanza. In seguito totemomail si riavvia autonomamente, se possibile; pianificate una finestra di manutenzione e create prima un backup o uno snapshot.

Finché l'aggiornamento non è installato, due misure riducono il rischio: rendere la console di amministrazione e l'Admin API raggiungibili solo dalla rete di gestione e limitare le connessioni in uscita del gateway alle destinazioni effettivamente necessarie. Nelle installazioni cloud, l'endpoint di metadati `169.254.169.254` rientra tra le destinazioni che il gateway non deve poter raggiungere. Entrambe le misure non sostituiscono la patch; solo l'aggiornamento risolve la vulnerabilità nel reset della password del Web Portal.

## Fonti

1.  totemomail Release Notes 9.5.1 (Build 101), totemo ag: guida di installazione e release note in PDF, disponibili per i clienti tramite il supporto del produttore; base di tutte le informazioni in questo articolo.

2.  [Kiteworks Security Advisories su GitHub](https://github.com/kiteworks/security-advisories/security/advisories): registro ufficiale con valori CVSS e versioni corrette per Email Protection Gateway, Core, MFT e Advanced Forms.

3.  [GHSA-rwpq-5xfv-54pv: Account takeover (CVE-2026-85066)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-rwpq-5xfv-54pv): advisory sull'errore di reset della password, CVSS 9.8, risolto in EPG 9.5.0.

4.  [GHSA-5xhq-9wq3-rvj6: Arbitrary code execution (CVE-2026-54154)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5xhq-9wq3-rvj6): vulnerabilità EPG più critica con CVSS 10.0, risolta in EPG 9.4.1.

5.  [P.K. Sharma: Kiteworks published 125 advisories](https://www.pk-sharma.com/briefing/kiteworks-125-advisories-which-is-the-shutdown-flaw): analisi della pubblicazione del 30 settembre 2026 con conteggio per prodotto e livello di gravità e le dichiarazioni di Kiteworks sulla 9.5.1.

6.  [Comprendere il routing della posta tra totemomail e Exchange Online](/blog/totemomail-m365): come totemomail elabora i messaggi in entrata; rilevante per stabilire quali e-mail possano attivare la vulnerabilità SSRF.

7.  [Limite di licenza Totemomail raggiunto: pulire gli utenti orfani tramite LDAP](/blog/totemomail-licensed-user-limit-ldap-cleanup): contesto sul conteggio delle licenze interessato dalla correzione del rapporto sulle licenze.
