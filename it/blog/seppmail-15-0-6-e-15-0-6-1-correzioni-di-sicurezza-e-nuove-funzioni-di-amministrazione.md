---
title: "SEPPmail 15.0.6 e 15.0.6.1: correzioni di sicurezza e nuove funzioni di amministrazione"
navTitle: "SEPPmail 15.0.6"
description: "Nel luglio 2026, SEPPmail ha rilasciato la patch release 15.0.6 e l'hotfix 15.0.6.1. Oltre a vulnerabilità corrette nella generazione di PDF e nell'elaborazione PGP, le release introducono un campo MFA separato, l'autenticazione LDAP per la GUI di amministrazione e correzioni a RuleEngine, Webmail e API REST."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "5 min di lettura"
themen:
  - seppmail
slug: "seppmail-15-0-6-e-15-0-6-1-correzioni-di-sicurezza-e-nuove-funzioni-di-amministrazione"
translationId: "article-3046fc35b259929b"
draft: false
translationOf: seppmail-releases-15-0-6-und-15-0-6-1
translationSourceHash: 636a7246234584a2b5797f53239fe65129de0f4463b8f773d0a7d9ed06d61f91
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:15:13.876Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/seppmail-15-0-6-e-15-0-6-1-correzioni-di-sicurezza-e-nuove-funzioni-di-amministrazione
---

# SEPPmail 15.0.6 e 15.0.6.1: correzioni di sicurezza e nuove funzioni di amministrazione

Il 21 luglio 2026, SEPPmail ha rilasciato la patch release 15.0.6 e, un giorno dopo, l'hotfix 15.0.6.1. La patch release chiude diverse vulnerabilità, aggiorna OpenSSH e OpenSSL e apporta miglioramenti tangibili all'amministrazione. L'hotfix corregge due errori nella RuleEngine introdotti o resi visibili con la versione 15.0.6. Le modifiche riguardano anche le appliance utilizzate come HIN Mailgateway, poiché si basano sullo stesso firmware SEPPmail.

## Hotfix 15.0.6.1 del 22 luglio 2026

L'hotfix risolve due aspetti nella RuleEngine. In primo luogo, un valore non definito nell'oggetto Message impediva la scrittura delle voci di log nel Mail Log. I messaggi interessati attraversavano quindi il sistema senza essere registrati. In secondo luogo, la RuleEngine ora riconosce la direzione delle e-mail archiviate, affinché la relativa consegna venga gestita correttamente.

Chi ha già installato la versione 15.0.6 o sta pianificando l'aggiornamento dovrebbe passare direttamente alla 15.0.6.1.

Anche le appliance HIN sembrano aver ricevuto l'hotfix: un HIN Mailgateway con la versione installata 15.0.6-RC-42-g278c81f84 segnala ora 15.0.6-RC-88-g916e513cc come prossima versione nel ramo 15.0. Le denominazioni RC del firmware HIN non possono essere associate direttamente a una release SEPPmail, ma il momento in cui l'aggiornamento viene proposto fa pensare all'hotfix.

## Correzioni di sicurezza nella 15.0.6

La parte più importante della patch release consiste in tre correzioni all'architettura di sicurezza:

- È stata chiusa una possibile vulnerabilità di path traversal nella generazione di PDF. È stata individuata da InfoGuard.
- Tutto il contenuto decrittografato tramite PGP viene ora codificato in Base64 per impedire la MIME-Structure-Injection.
- La funzione hashencrypt è stata convertita ad AES-256-CBC con PBKDF2.

A queste si aggiungono librerie aggiornate: OpenSSH 10.4 e OpenSSL 3.0.21 correggono complessivamente oltre venti CVE. Già solo per questi aspetti, l'aggiornamento è consigliabile per i sistemi produttivi.

## Nuove funzioni per l'amministrazione

Tre modifiche nella GUI di amministrazione risultano evidenti nell'uso quotidiano:

- **Campo di inserimento MFA separato:** Il secondo fattore non deve più essere aggiunto alla password, ma dispone di un proprio campo. Questo elimina una fonte di errori di lunga data durante il login.
- **Autenticazione LDAP per la GUI di amministrazione:** Gli amministratori possono ora autenticarsi tramite un server LDAP esterno, invece di gestire account locali sull'appliance. La configurazione è descritta nell'articolo sul [collegamento della GUI di amministrazione ad Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung). Verifico ancora se anche HIN Mailgateway ha ricevuto questa funzione e aggiungerò successivamente l'informazione all'articolo; poiché HIN utilizza la stessa base firmware, lo presumo.
- **Pulsante AutoRenew per MPKI:** Nelle impostazioni del connettore MPKI, il rinnovo automatico del certificato può essere avviato manualmente tramite «Trigger AutoRenew...». 

Inoltre, l'appliance utilizza ora in modo coerente fusi orari validi (predefinito: Europe/Zurich), e il System Object ID in System >> Advanced View viene validato come OID valido.

## Elaborazione della posta e Webmail

Nella RuleEngine sono stati corretti quattro aspetti. La gestione dell'oggetto funziona ora anche con una codifica sconosciuta. I messaggi vengono respinti quando una firma è richiesta esplicitamente ma non può essere creata; in precedenza tali messaggi potevano proseguire senza firma. Le copie archiviate passano ora attraverso la funzione di consegna e ricevono quindi intestazioni ARC. Inoltre, per i messaggi PGP privi di dati MDC, gli errori MDC vengono ignorati anziché interferire con l'elaborazione.

Nel Webmail (GINA) sono stati corretti quattro errori: l'eliminazione automatica degli account non registrati dopo la scadenza del periodo di tolleranza funziona nuovamente, la funzione hashdecrypt restituiva in determinati casi un risultato di decrittografia falsamente positivo, l'aggiunta di un allegato svuotava i campi A e CC e la visualizzazione dell'ora nei log SMS era errata.

## API REST, cluster e backup

L'API REST riceve correzioni a diversi endpoint: /system/ifaliasconfig (gestione dei valori null), /system/applySysconfig (configurazione degli accessi), /crypto/domain/{domainName} (caricamento di certificati di dominio) nonché GET e POST /ssl/csr. Il timeout per le chiamate REST è stato aumentato da 300 a 900 secondi, rendendo più affidabili le richieste di lunga durata, come modifiche di configurazione più ampie.

Nel funzionamento in cluster, un IP CARP esistente bloccava finora le impostazioni IP di un membro appena aggiunto; il problema è stato risolto. Prima della creazione quotidiana dello snapshot, il backup verifica ora anche la presenza di un database corrotto prima di scrivere lo snapshot.

## Collegamento con il problema di login nella 15.0.5

Durante l'aggiornamento di un cluster alla 15.0.5, il login poteva non funzionare su entrambi i nodi. Il comportamento dell'errore e il ripristino sono descritti nell'articolo sul [problema di login dopo l'aggiornamento alla 15.0.5](/blog/hin-update-issue-version-15.0.5). All'epoca il produttore conosceva già il problema e aveva annunciato una correzione per una versione successiva.

Nelle release notes della 15.0.6 compare ora esattamente una voce compatibile con questo comportamento: «prevent password rehashing when cluster members use different firmware versions». Durante un aggiornamento del cluster, i nodi funzionano inevitabilmente temporaneamente con versioni firmware differenti. Se un nodo ricalcola gli hash delle password in questa fase e li replica nel cluster, gli hash non sono più compatibili con l'altra versione e il login fallisce su entrambi i nodi, proprio come nel guasto osservato allora. Le release notes non menzionano esplicitamente il problema di login, ma la voce copre esattamente la configurazione che lo aveva causato. La causa è quindi affrontata nella 15.0.6; la procedura di emergenza necessaria nella 15.0.5, con scioglimento del cluster, dovrebbe risultare superflua negli aggiornamenti futuri.

## Correzioni minori

Nel Mail Log è stato corretto l'ordinamento per data, che in precedenza ordinava alfabeticamente anziché cronologicamente, e la dimensione visualizzata dei messaggi LFT è nuovamente corretta. Gli accessi a intestazioni X inesistenti non vengono più registrati. Il connettore CertCentral di MPKI gestisce in modo più robusto gli errori di input e REST.

## Valutazione

I due errori della RuleEngine corretti dall'hotfix suggeriscono di saltare la 15.0.6 e utilizzare direttamente la 15.0.6.1. Nei cluster, creare snapshot di entrambi i nodi prima dell'aggiornamento e rispettare l'ordine di aggiornamento indicato nella documentazione del produttore. Il problema di login nella 15.0.5 ha mostrato perché questa preparazione non è una mera formalità.

## Fonti

1.  [Documentazione SEPPmail – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Release notes ufficiali per 15.0.6 e 15.0.6.1 con tutti i singoli punti.

2.  [HIN Mailgateway 15.0.5: risolvere il problema di login dopo l'aggiornamento del cluster](/blog/hin-update-issue-version-15.0.5): Perché gli snapshot e il corretto ordine di aggiornamento nel cluster sono decisivi.
