---
title: "SEPPmail 15.0.6 e 15.0.6.1: correzioni di sicurezza e nuove funzioni di amministrazione"
navTitle: "SEPPmail 15.0.6"
description: "SEPPmail ha rilasciato a luglio 2026 la patch release 15.0.6 e l'hotfix 15.0.6.1. Oltre alle vulnerabilità corrette nella generazione di PDF e nell'elaborazione PGP, le release introducono un campo MFA separato, l'autenticazione LDAP per la GUI di amministrazione e correzioni a RuleEngine, Webmail e REST API."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "5 min di lettura"
themen:
  - seppmail
slug: "seppmail-15-0-6-e-15-0-6-1-correzioni-di-sicurezza-e-nuove-funzioni-di-amministrazione"
translationId: "article-3046fc35b259929b"
draft: false
translationOf: seppmail-releases-15-0-6-und-15-0-6-1
url: https://rafaelpfister.ch/it/blog/seppmail-15-0-6-e-15-0-6-1-correzioni-di-sicurezza-e-nuove-funzioni-di-amministrazione
translationSourceHash: 5cf19b84bb90403b0a7e2795222b8f853c29c3fe562429df8538e703e565217a
translationModel: gpt-5.6-terra
translatedAt: 2026-07-30T12:49:46.205Z
translationReview: automatic
---

# SEPPmail 15.0.6 e 15.0.6.1: correzioni di sicurezza e nuove funzioni di amministrazione

SEPPmail ha rilasciato il 21 luglio 2026 la patch release 15.0.6 e, un giorno dopo, l'hotfix 15.0.6.1. La patch release chiude diverse vulnerabilità, aggiorna OpenSSH e OpenSSL e introduce notevoli miglioramenti per l'amministrazione. L'hotfix corregge due errori nella RuleEngine introdotti o resi visibili con la versione 15.0.6. Le modifiche riguardano anche le appliance utilizzate come HIN Mailgateway, poiché si basano sullo stesso firmware SEPPmail.

## Hotfix 15.0.6.1 del 22 luglio 2026

L'hotfix risolve due aspetti nella RuleEngine. In primo luogo, un valore non definito nell'oggetto Message impediva la scrittura delle voci di log nel Mail-Log. I messaggi interessati attraversavano quindi il sistema senza essere registrati. In secondo luogo, la RuleEngine ora riconosce la direzione delle e-mail archiviate, affinché la loro consegna venga gestita correttamente.

Chi ha già installato la versione 15.0.6 o prevede l'aggiornamento dovrebbe passare direttamente alla 15.0.6.1.

A quanto pare, anche le appliance HIN hanno ricevuto l'hotfix: un HIN Mailgateway con la versione installata 15.0.6-RC-42-g278c81f84 segnala ora 15.0.6-RC-88-g916e513cc come prossima versione nel ramo 15.0. Le denominazioni RC del firmware HIN non possono essere associate direttamente a una release SEPPmail, ma il momento dell'offerta fa pensare all'hotfix.

## Correzioni di sicurezza nella 15.0.6

La parte più importante della patch release consiste in tre correzioni all'architettura di sicurezza:

- È stata chiusa una possibile vulnerabilità di path traversal nella generazione di PDF. È stata individuata da InfoGuard.
- Tutti i contenuti decifrati tramite PGP vengono ora codificati in Base64 per impedire la MIME-Structure-Injection.
- La funzione hashencrypt è stata convertita ad AES-256-CBC con PBKDF2.

Si aggiungono librerie aggiornate: OpenSSH 10.4 e OpenSSL 3.0.21 correggono insieme oltre venti CVE. Già solo per questi punti, l'aggiornamento è consigliabile per i sistemi in produzione.

## Nuove funzioni per l'amministrazione

Tre modifiche nella GUI di amministrazione si notano nell'uso quotidiano:

- **Campo di inserimento MFA separato:** il secondo fattore non deve più essere aggiunto alla password, ma dispone di un proprio campo. Questo elimina un ostacolo di lunga data durante il login.
- **Autenticazione LDAP per la GUI di amministrazione:** gli amministratori possono ora autenticarsi tramite un server LDAP esterno, anziché gestire account locali sull'appliance. La configurazione è descritta nell'articolo sul [collegamento della GUI di amministrazione ad Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung). Sto ancora verificando se anche HIN Mailgateway abbia ricevuto questa funzione e completerò l'articolo in seguito; dato che HIN utilizza la stessa base firmware, lo presumo.
- **Pulsante AutoRenew per MPKI:** nelle impostazioni del connettore MPKI, il rinnovo automatico del certificato può essere avviato manualmente tramite «Trigger AutoRenew...».

Inoltre, l'appliance ora utilizza sistematicamente fusi orari validi (predefinito: Europe/Zurich) e il System Object ID in System >> Advanced View viene validato come OID valido.

## Elaborazione della posta e Webmail

Nella RuleEngine sono stati corretti quattro aspetti. La gestione dell'oggetto funziona ora anche con encoding sconosciuto. I messaggi vengono respinti se una firma è richiesta esplicitamente ma non può essere creata; finora tali messaggi potevano proseguire senza firma. Le copie archiviate passano ora attraverso la funzione di consegna e ricevono quindi header ARC. E per i messaggi PGP senza dati MDC, gli errori MDC vengono ignorati anziché interferire con l'elaborazione.

Nel Webmail (GINA) sono stati corretti quattro errori: l'eliminazione automatica degli account non registrati dopo la scadenza del periodo di tolleranza funziona nuovamente, la funzione hashdecrypt in determinati casi restituiva un risultato di decrittazione falsamente positivo, l'aggiunta di un allegato svuotava i campi A e CC, e la visualizzazione dell'ora nei log SMS era errata.

## REST API, cluster e backup

La REST API riceve correzioni a diversi endpoint: /system/ifaliasconfig (gestione dei valori null), /system/applySysconfig (configurazione degli accessi), /crypto/domain/{domainName} (upload dei certificati di dominio) nonché GET e POST /ssl/csr. Il timeout per le chiamate REST è stato aumentato da 300 a 900 secondi, rendendo più affidabili le richieste di lunga durata, come le modifiche di configurazione più ampie.

Nel funzionamento in cluster, un IP CARP esistente bloccava finora le impostazioni IP di un nuovo membro aggiunto; il problema è stato risolto. Prima della creazione giornaliera dello snapshot, il backup verifica ora inoltre la presenza di un database corrotto prima di scrivere lo snapshot.

## Relazione con il malfunzionamento del login nella 15.0.5

Durante l'aggiornamento di un cluster alla 15.0.5, il login poteva smettere di funzionare su entrambi i nodi. Il problema e il ripristino sono descritti nell'articolo sul [malfunzionamento del login dopo l'aggiornamento alla 15.0.5](/blog/hin-update-issue-version-15.0.5). Il produttore conosceva già il problema e all'epoca aveva annunciato una correzione per una versione successiva.

Nelle release notes della 15.0.6 è ora presente esattamente una voce compatibile con questo problema: «prevent password rehashing when cluster members use different firmware versions». Durante un aggiornamento del cluster, i nodi operano inevitabilmente e temporaneamente con versioni firmware diverse. Se in questa fase un nodo ricalcola gli hash delle password e li replica nel cluster, gli hash non corrispondono più sull'altra versione e il login fallisce su entrambi i nodi, esattamente come nel malfunzionamento osservato allora. Le release notes non menzionano esplicitamente il malfunzionamento del login, ma la voce copre esattamente la configurazione che lo aveva causato. La causa è quindi affrontata nella 15.0.6; la procedura di emergenza necessaria nella 15.0.5, con scioglimento del cluster, dovrebbe diventare superflua nei futuri aggiornamenti.

## Correzioni minori

Nel Mail-Log è stato corretto l'ordinamento per data, che finora era alfabetico anziché cronologico, e la dimensione visualizzata dei messaggi LFT è nuovamente corretta. Gli accessi a X-Header inesistenti non vengono più registrati. Il connettore CertCentral dell'MPKI gestisce in modo più robusto gli errori di input e REST.

## Valutazione

I due errori RuleEngine corretti dall'hotfix suggeriscono di saltare la 15.0.6 e utilizzare direttamente la 15.0.6.1. Per i cluster, create snapshot di entrambi i nodi prima dell'aggiornamento e rispettate l'ordine di aggiornamento indicato nella documentazione del produttore. Il malfunzionamento del login nella 15.0.5 ha dimostrato perché questa preparazione non è una mera formalità.

## Fonti

1.  [Documentazione SEPPmail – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): release notes ufficiali della 15.0.6 e della 15.0.6.1 con tutti i singoli punti.

2.  [HIN Mailgateway 15.0.5: risolvere il malfunzionamento del login dopo l'aggiornamento del cluster](/blog/hin-update-issue-version-15.0.5): perché gli snapshot e il corretto ordine di aggiornamento nel cluster sono determinanti.
