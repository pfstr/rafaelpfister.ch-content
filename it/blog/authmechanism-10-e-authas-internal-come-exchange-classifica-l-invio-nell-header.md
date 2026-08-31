---
title: "AuthMechanism 10 e AuthAs Internal: come Exchange classifica l'invio nell'header"
navTitle: "AuthMechanism 10"
description: "L'header X-MS-Exchange-Organization-AuthMechanism documenta come un server mittente si è autenticato. Il valore 10 indica un Receive Connector con Externally Secured e classifica le e-mail esterne come interne, con conseguenze per filtri antispam, regole di flusso della posta e protezione dallo spoofing."
date: "2026-08-26"
featured: "2026-08-27"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "8 min di lettura"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-hybrid"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-hybrid-header-intern-extern
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
slug: "authmechanism-10-e-authas-internal-come-exchange-classifica-l-invio-nell-header"
translationId: "article-0df383d5c49016da"
translationOf: exchange-authmechanism-10-authas-internal
url: https://rafaelpfister.ch/it/blog/authmechanism-10-e-authas-internal-come-exchange-classifica-l-invio-nell-header
translationSourceHash: 5a9335a90afc9bf7df78b908f71b679f64c29f3b9e96bd7f25bcc916123b82df
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:17:53.089Z
translationReview: automatic
---

# AuthMechanism 10 e AuthAs Internal: come Exchange classifica l'invio nell'header

Nell'analisi di casi di spam, spoofing e flusso della posta in ambienti Exchange, sono determinanti tre intestazioni che Exchange aggiunge alla ricezione:

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthSource: EX01.example.com
X-MS-Exchange-Organization-AuthMechanism: 10
```

`AuthAs` registra in quale veste il mittente si è presentato al trasporto. `AuthSource` indica il server che ha effettuato la valutazione. `AuthMechanism` documenta il meccanismo con cui è avvenuta l'autenticazione. Insieme determinano se Exchange tratta un messaggio come interno o esterno, e questa classificazione ha conseguenze significative.

## Perché la classificazione è importante

`AuthAs` presenta in pratica due valori: `Internal` e `Anonymous`. Un messaggio classificato come `Internal` viene trattato diversamente dalla posta esterna:

- Le regole di flusso della posta con la condizione «mittente esterno all'organizzazione» non vengono applicate.
- Il messaggio può essere recapitato a gruppi di distribuzione e cassette postali che richiedono mittenti autenticati (`RequireSenderAuthenticationEnabled`).
- I controlli antispam e antispoofing sono meno severi o vengono omessi; negli ambienti ibridi non viene aggiunto il disclaimer esterno e Outlook non mostra l'avviso «Esterno».
- Il nome visualizzato viene risolto dalla rubrica e l'e-mail appare ai destinatari come posta interna.

Proprio per questo la domanda «AuthAs Internal o Anonymous?» deve essere all'inizio di ogni analisi dell'header: consente di chiarire perché un'e-mail di spoofing evidente abbia superato il filtro antispam o perché una regola di flusso della posta non sia mai stata attivata.

## I valori di AuthMechanism

Microsoft non documenta completamente in pubblico la codifica di `AuthMechanism`. Due valori sono rilevanti e ben documentati per la risoluzione dei problemi:

| Valore | Significato |
|---|---|
| `04` | Traffico Exchange autenticato: da cassetta postale a cassetta postale all'interno dell'organizzazione e traffico ibrido tramite i connettori configurati da Hybrid Configuration Wizard. |
| `10` | Receive Connector con l'opzione di autenticazione `ExternalAuthoritative` («Protetto esternamente» / «Externally secured»): la connessione è considerata protetta al di fuori di Exchange e tutto ciò che viene inviato tramite essa viene trattato come interno. |

Altri valori compaiono negli header, ma non dispongono di riferimenti ufficiali. Nella pratica basta la distinzione: `04` indica un'autenticazione Exchange effettiva, `10` indica attendibilità basata sulla configurazione del connettore.

## Cosa significa davvero Externally Secured

L'opzione `ExternalAuthoritative` su un Receive Connector comunica a Exchange che la protezione di questa connessione è gestita da qualcun altro, ad esempio un firewall, un segmento di rete dedicato o IPsec. Exchange non effettua quindi ulteriori controlli, ma tratta ogni invio tramite questo connettore come autenticato e interno, incluso il diritto di utilizzare qualsiasi indirizzo mittente interno.

È pensata per pochi scenari, ad esempio un server applicativo completamente attendibile nel proprio data center. Diventa problematica quando il connettore punta a un gateway di posta a monte o a un filtro antispam nella DMZ, attraverso il quale arriva anche posta da Internet. Dopo l'invio, ogni e-mail esterna presenta quindi:

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthMechanism: 10
```

Le conseguenze: le e-mail esterne sono considerate interne, le regole di flusso della posta per mittenti esterni non vengono applicate, la protezione dallo spoofing per il proprio dominio è inefficace e chiunque raggiunga il gateway può recapitare ai destinatari, usando indirizzi mittente interni, messaggi che in realtà richiedono mittenti autenticati.

## Individuare i connettori interessati

La Exchange Management Shell mostra quali Receive Connector sono configurati con `ExternalAuthoritative`:

```powershell
Get-ReceiveConnector | Where-Object {
  $_.AuthMechanism -match "ExternalAuthoritative"
} | Format-Table Identity, RemoteIPRanges, AuthMechanism, PermissionGroups
```

Per ogni risultato, verificare quali `RemoteIPRanges` sono configurati e se i sistemi a valle necessitano effettivamente di questa attendibilità. Un gateway che deve soltanto inoltrare e-mail non ne ha bisogno.

## L'alternativa per gli scenari di relay

Se un sistema deve soltanto effettuare relay anonimo tramite Exchange (stampanti, applicazioni, monitoraggio), un connettore relay anonimo è la soluzione più pulita: invio anonimo più il diritto di recapitare a destinatari arbitrari, ma senza la classificazione Internal.

```powershell
New-ReceiveConnector -Name "Anonymous Relay" -TransportRole FrontendTransport `
  -RemoteIPRanges 192.0.2.10 -Bindings 0.0.0.0:25 -Usage Custom -PermissionGroups AnonymousUsers

Get-ReceiveConnector "EX01\Anonymous Relay" | Add-ADPermission `
  -User "NT AUTHORITY\ANONYMOUS LOGON" -ExtendedRights "ms-Exch-SMTP-Accept-Any-Recipient"
```

Le e-mail inviate tramite questo connettore rimangono `AuthAs: Anonymous`, passano attraverso i controlli normali e non possono simulare mittenti interni. `ExternalAuthoritative` resta riservato ai sistemi ai quali si desidera concedere consapevolmente il diritto di usare indirizzi mittente interni.

## Leggere gli header nel contesto

Per capire più rapidamente se un messaggio concreto è stato classificato come interno o esterno e da quale percorso è arrivato, è necessario leggere l'header completo: `AuthAs`, `AuthMechanism` e `AuthSource` insieme alla catena `Received`. L'[analizzatore di header delle e-mail](/tools/header-analyzer) su questo sito valuta direttamente nel browser questi campi e evidenzia la classificazione ibrida nel percorso di recapito; l'header non lascia il browser.

L'articolo [Interna o esterna? Classificare le e-mail Exchange ibride nell'header](/blog/exchange-hybrid-header-intern-extern) tratta come la classificazione venga mantenuta tra Exchange Online e OnPrem negli ambienti ibridi e come riconoscere un'assegnazione errata.

## Fonti

1.  [Microsoft Q&A: Exchange 2016 mail flow rule, which header is checked for "outside the organization"?](https://learn.microsoft.com/en-us/answers/questions/54418/exchange-2016-mail-flow-rule-which-header-is-check): associazione di AuthAs e AuthMechanism 10 alla configurazione Externally Secured e relativo effetto sulle regole di flusso della posta.

2.  [Microsoft Tech Community: Demystifying and troubleshooting hybrid mail flow: when is a message internal?](https://techcommunity.microsoft.com/blog/exchange/demystifying-and-troubleshooting-hybrid-mail-flow-when-is-a-message-internal/1420838): descrizione ufficiale della classificazione Internal e delle sue conseguenze nel flusso della posta ibrido.

3.  [msxfaq: X-MS-Exchange-Organization-AuthAs](https://www.msxfaq.de/cloud/exchangeonline/transport/x-ms-exchange-organization-authas.htm): valori AuthAs, AuthSource e AuthMechanism osservati in diversi scenari di invio.

4.  [Microsoft Learn: Allow anonymous relay on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/allow-anonymous-relay): configurazione del connettore relay anonimo come alternativa a Externally Secured.
