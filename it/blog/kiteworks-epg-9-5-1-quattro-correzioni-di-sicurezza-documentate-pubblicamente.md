---
title: "Kiteworks EPG 9.5.1: quattro correzioni di sicurezza documentate pubblicamente"
navTitle: "Kiteworks EPG 9.5.1"
description: "Kiteworks Email Protection Gateway 9.5.1 risolve quattro vulnerabilità documentate pubblicamente, tra cui una compromissione dell’account con CVSS 9.4. Le installazioni precedenti alla 9.5.1 dovrebbero essere aggiornate a questa o a una versione successiva."
date: "2026-11-06"
kategorie: "Totemomail"
timeToRead: "3 min di lettura"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "releases"
slug: "kiteworks-epg-9-5-1-quattro-correzioni-di-sicurezza-documentate-pubblicamente"
translationId: article-d4b9d984c72a54ea
translationOf: kiteworks-epg-releases-9-5-1
url: https://rafaelpfister.ch/it/blog/kiteworks-epg-9-5-1-quattro-correzioni-di-sicurezza-documentate-pubblicamente
translationSourceHash: 7d961f479006e9eb0da1a74925d77e25e7205a64a1ed783dccb55b6eed8c2c5c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-07T03:19:36.109Z
translationReview: automatic
---

# Kiteworks EPG 9.5.1: quattro correzioni di sicurezza documentate pubblicamente

Kiteworks prosegue la precedente tecnologia totemo come Email Protection Gateway (EPG). La versione 9.5.1 era l’attuale versione Kiteworks al più tardi il 25 settembre 2026; il produttore non indica una data di rilascio pubblica. Gli advisory del produttore pubblicati il 30 settembre documentano quattro correzioni per EPG 9.5.1: una compromissione dell’account, due vulnerabilità non autenticate per la scrittura di file o l’attivazione di un Denial of Service, nonché una vulnerabilità che consentiva agli amministratori di scrivere file arbitrari. La classificazione per l’omonima versione totemomail è disponibile nell’articolo su [totemomail 9.5.1](/blog/totemomail-9-5-1-sicherheitsupdate).

## Panoramica delle versioni

| Versione | Data | Tipo | Punti principali |
|---|---|---|---|
| 9.5.1 | confermata pubblicamente al più tardi il 25.09.2026 | aggiornamento di sicurezza | Quattro correzioni EPG documentate pubblicamente, tra cui CVE-2026-102149 con CVSS 9.4 |

## Correzioni di sicurezza nella 9.5.1

Secondo Kiteworks, tutti e quattro gli advisory riguardano tutte le versioni EPG precedenti alla 9.5.1 e indicano la 9.5.1 come versione corretta. Gli advisory sono stati pubblicati il 30 settembre 2026. Descrivono retrospettivamente lo stato di sicurezza del rilascio, non la sua data di pubblicazione esatta.

- **CVE-2026-102149, CVSS 9.4 (critico):** Una debolezza nel controllo degli accessi poteva consentire a un aggressore di accedere agli account utente tramite la rete. La valutazione CVSS non richiede né privilegi né interazione dell’utente. Kiteworks attribuisce la causa a CWE-306 (autenticazione mancante per funzioni critiche) e CWE-639 (aggiramento dell’autorizzazione tramite una chiave controllata dall’utente).
- **CVE-2026-102143, CVSS 7.5 (alto):** Un aggressore non autenticato poteva scrivere file sull’appliance. Kiteworks cita un problema di autenticazione nonché CWE-306 e CWE-434 (caricamento illimitato di tipi di file pericolosi).
- **CVE-2026-102146, CVSS 6.5 (medio):** Un amministratore poteva scrivere file arbitrari. Kiteworks descrive un errore di autorizzazione; la metrica CVSS richiede privilegi elevati. Il produttore indica come debolezze CWE-73 (controllo esterno del nome file o del percorso) e CWE-1336 (neutralizzazione insufficiente in un template engine).
- **CVE-2026-102144, CVSS 5.3 (medio):** Un aggressore nella rete poteva attivare un Denial of Service. L’advisory cita l’autenticazione mancante per funzioni critiche e il consumo incontrollato di risorse (CWE-306, CWE-400). Anche questa valutazione non richiede privilegi né interazione dell’utente.

Per gli operatori, la compromissione dell’account è la correzione prioritaria, poiché secondo l’advisory è possibile senza un account esistente. Anche le altre due vulnerabilità senza autenticazione riguardano tutte le versioni precedenti alla 9.5.1. Gli advisory del produttore non pubblicano dettagli tecnici su endpoint, configurazioni o soluzioni alternative.

## Procedura consigliata

Aggiornate EPG almeno alla versione 9.5.1 se utilizzate una versione precedente. Kiteworks indica esplicitamente per tutti e quattro gli advisory l’aggiornamento alla 9.5.1 o successiva come rimedio. Se è già stata approvata una versione successiva del produttore, anch’essa soddisfa questa versione minima. Kiteworks non ha pubblicato nelle fonti consultate un percorso di aggiornamento pubblico, problemi noti o termini di supporto per EPG 9.5.1.

## Fonti

1.  [Kiteworks: Precautionary Shutdown Advisory for Customers](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): Comunicazione del produttore del 25 settembre 2026 che indica la 9.5.1 come versione attuale.

2.  [Kiteworks Security Advisories](https://github.com/kiteworks/security-advisories/security/advisories): Panoramica ufficiale degli advisory pubblicati il 30 settembre 2026, compresi gli elementi EPG per la 9.5.1.

3.  [GHSA-c9w5-4frw-7wqq: Account takeover (CVE-2026-102149)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-c9w5-4frw-7wqq): Debolezza critica nel controllo degli accessi, CVSS 9.4, corretta in EPG 9.5.1.

4.  [GHSA-3p9g-jh62-8f89: Unauthorized file modification (CVE-2026-102143)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-3p9g-jh62-8f89): Scrittura di file senza autenticazione, CVSS 7.5, corretta in EPG 9.5.1.

5.  [GHSA-5pgq-v8g2-rg2f: Unauthorized file modification (CVE-2026-102146)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5pgq-v8g2-rg2f): Scrittura di file arbitrari con privilegi di amministratore, CVSS 6.5, corretta in EPG 9.5.1.

6.  [GHSA-wwhf-5862-rjxq: Denial of service (CVE-2026-102144)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-wwhf-5862-rjxq): Denial of Service senza autenticazione, CVSS 5.3, corretto in EPG 9.5.1.

7.  [Kiteworks: EPG with totemo Integration](https://www.kiteworks.com/video-why-kiteworks-and-totemo-are-better-together/): Indicazione del produttore sulla prosecuzione e ridenominazione della tecnologia totemo come Email Protection Gateway.
