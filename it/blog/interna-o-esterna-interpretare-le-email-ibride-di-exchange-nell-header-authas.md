---
title: "Interna o esterna? Interpretare le email ibride di Exchange nell'header: AuthAs, MessageDirectionality e X-originatorOrg"
navTitle: "Leggere gli header ibridi"
description: "Negli ambienti ibridi di Exchange, la classificazione degli header determina se un'email viene trattata come interna. Quali intestazioni determinano la classificazione, come funziona l'attribuzione del tenant tramite certificato e connettore e come riconoscere un messaggio instradato in modo errato."
date: "2026-08-26"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "10 min di lettura"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange-hybrid"
  - "hybrid-mailfluss"
  - "exchange-online"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-authmechanism-10-authas-internal
  - microsoft-365-compauth-reason-codes
  - einliefernde-ip-adressen-aggregieren
slug: "interna-o-esterna-interpretare-le-email-ibride-di-exchange-nell-header-authas"
translationId: "article-c8d7859be8dbfe63"
translationOf: exchange-hybrid-header-intern-extern
url: https://rafaelpfister.ch/it/blog/interna-o-esterna-interpretare-le-email-ibride-di-exchange-nell-header-authas
translationSourceHash: 5a0eccedd4b1a194461602319f5f1a8f59de204c1710e261c2358591bb720dfb
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:19:17.611Z
translationReview: automatic
---

# Interna o esterna? Interpretare le email ibride di Exchange nell'header: AuthAs, MessageDirectionality e X-originatorOrg

In un ambiente ibrido, le email tra Exchange OnPrem e Exchange Online devono essere trattate come posta interna: nessun filtro antispam intermedio, nessuna indicazione «Esterna», recapito a gruppi di distribuzione protetti, nomi visualizzati risolti. A determinare se ciò funziona non è il dominio del mittente, bensì una manciata di intestazioni che devono essere preservate nel percorso tra i due mondi. Chi sa leggerle può rispondere alle più comuni domande sugli ambienti ibridi direttamente dall'header: l'email è arrivata tramite il connettore ibrido? Perché è stata classificata come esterna? E a quale tenant è stata attribuita?

## Le intestazioni coinvolte

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthSource: EX01.example.com
X-MS-Exchange-Organization-MessageDirectionality: Originating
X-OriginatorOrg: example.com
```

**`AuthAs`** indica la classificazione: `Internal` oppure `Anonymous`. È il risultato degli altri segnali e l'indicatore più diretto di come Exchange ha trattato il messaggio.

**`AuthSource`** indica il FQDN del server che ha effettuato la classificazione: un server OnPrem proprio, un server mailbox in Exchange Online oppure un frontend EOP. Da questo si può capire da quale lato è avvenuta la valutazione.

**`MessageDirectionality`** distingue `Originating` (il messaggio è stato creato all'interno dell'organizzazione, in Exchange Online o tramite un Inbound Connector autenticato) da `Incoming` (il messaggio è arrivato dall'esterno).

**`X-OriginatorOrg`** identifica l'organizzazione mittente dal punto di vista di Exchange Online: il dominio Accepted predefinito o corrispondente del tenant mittente. L'header viene impostato durante l'invio da Exchange Online tramite l'estensione SMTP XOORG ed è vincolato alla combinazione di certificato TLS EOP, configurazione del connettore e dominio Accepted. Non può quindi essere falsificato semplicemente inviandolo: un `X-OriginatorOrg` consegnato dall'esterno senza la corrispondente relazione di attendibilità non viene riconosciuto come tale.

A questi si aggiungono gli header `X-MS-Exchange-CrossTenant-*`, che Exchange Online appone nel passaggio tra tenant, tra cui `X-MS-Exchange-CrossTenant-AuthAs`. Riflettono la classificazione dal punto di vista del tenant destinatario.

## Come funziona tecnicamente la relazione di attendibilità

La classificazione Internal oltre il confine dell'organizzazione si basa su due componenti configurati da Hybrid Configuration Wizard:

Primo, l'**Inbound Connector** in Exchange Online di tipo OnPremises, che identifica la fonte di invio tramite il certificato TLS (`TlsSenderCertificateName`) oppure l'indirizzo IP. In base a questa associazione, Exchange Online decide anche a quale tenant attribuire una consegna (attribution).

Secondo, il flag **`CloudServicesMailEnabled`** sui connettori di entrambi i lati. Fa sì che gli header `X-MS-Exchange-Organization-*` (header cross-premises) vengano preservati nel passaggio, anziché rimossi come avviene per la posta esterna. Se il flag manca oppure l'email segue un percorso senza questa configurazione, gli header vanno persi e l'email arriva come `Anonymous`.

Ne deriva la regola diagnostica più importante: un'email ibrida è interna solo se ha effettivamente seguito il percorso configurato dall'HCW.

## Caso 1: l'email arriva come Anonymous, benché dovrebbe essere interna

È il problema più frequente: le email dalle cassette postali OnPrem appaiono in Exchange Online come esterne, con controllo antispam, contrassegno «Esterna» o rifiuto da parte di gruppi di distribuzione protetti. Le cause, in ordine di frequenza decrescente:

- **Percorso errato:** l'email non è passata attraverso il connettore ibrido, bensì tramite l'MX, quindi attraverso EOP come posta Internet, oppure tramite un gateway a monte che rimuove gli header cross-premises o termina la connessione TLS. Nell'header ciò è visibile nella catena `Received`: invece del passaggio diretto da OnPrem a `*.mail.protection.outlook.com` tramite il connettore, compaiono stazioni intermedie.
- **Sostituzione del certificato:** il certificato OnPrem è stato rinnovato, ma `TlsSenderCertificateName` nell'Inbound Connector non è stato aggiornato. L'identificazione tramite certificato non funziona più.
- **Configurazione del connettore modificata:** `CloudServicesMailEnabled` è stato disattivato durante il troubleshooting oppure un connettore creato manualmente sostituisce il connettore HCW senza le impostazioni necessarie.

La verifica sul lato Exchange Online:

```powershell
Get-InboundConnector | Format-List Name, ConnectorType,
  TlsSenderCertificateName, SenderIPAddresses, CloudServicesMailEnabled
```

Nel Message Trace, il campo `ConnectorName` indica se il messaggio è stato effettivamente consegnato tramite il connettore previsto.

## Caso 2: attribuzione al tenant errato

Exchange Online attribuisce ogni messaggio in ingresso a un tenant; l'header `X-EOPTenantAttributedMessage` contiene il GUID del tenant attribuito. Se due tenant utilizzano lo stesso `TlsSenderCertificateName` o gli stessi `SenderIPAddresses` nei rispettivi Inbound Connector, ad esempio presso un fornitore di servizi gateway condiviso o dopo una migrazione, un messaggio può essere attribuito al tenant sbagliato. In tal caso non compare nel Message Trace del proprio tenant ed è soggetto a regole di trasporto altrui.

Il GUID del proprio tenant viene fornito da `Get-OrganizationConfig | Select-Object GUID`; se non corrisponde a quello nell'header, gli identificatori dei connettori devono essere separati: un certificato dedicato o intervalli IP dedicati per ogni tenant.

## Caso 3: un'email classificata come esterna viene comunque trattata come interna

Il caso opposto si verifica OnPrem: un Receive Connector con l'opzione `ExternalAuthoritative` («Externally secured») classifica come interno tutto ciò che viene consegnato tramite esso, riconoscibile da `AuthAs: Internal` in combinazione con `AuthMechanism: 10`. Se un tale connettore punta a un gateway attraverso il quale passa anche posta Internet, la posta esterna viene considerata interna, con tutte le conseguenze per i filtri antispam e la protezione dallo spoofing. I dettagli e le contromisure sono descritti nell'articolo [AuthMechanism 10 e AuthAs Internal](/blog/exchange-authmechanism-10-authas-internal).

## Leggere l'header nel suo insieme

Per classificare un messaggio concreto occorre considerare insieme tutti i segnali: la catena `Received` per il percorso effettivo, `AuthAs`/`AuthSource`/`MessageDirectionality` per la classificazione, `X-OriginatorOrg` e gli header CrossTenant per l'organizzazione di origine. L'[analizzatore di header email](/tools/header-analyzer) su questo sito analizza questi campi direttamente nel browser e contrassegna il passaggio tra tenant e la classificazione ibrida nel percorso di consegna; l'header non lascia il browser.

## Fonti

1.  [Microsoft Tech Community: Demystifying and troubleshooting hybrid mail flow: when is a message internal?](https://techcommunity.microsoft.com/blog/exchange/demystifying-and-troubleshooting-hybrid-mail-flow-when-is-a-message-internal/1420838): Descrizione ufficiale della classificazione Internal, degli header coinvolti e dei requisiti dei connettori.

2.  [Microsoft Tech Community: Advanced Office 365 Routing: Locking Down Exchange On-Premises when MX points to Office 365](https://techcommunity.microsoft.com/blog/exchange/advanced-office-365-routing-locking-down-exchange-on-premises-when-mx-points-to-/609238): Funzionamento di XOORG e X-OriginatorOrg nel routing tra Exchange Online e OnPrem.

3.  [Microsoft Learn (archivio): Use headers to determine which Exchange Online tenant a message was attributed to](https://learn.microsoft.com/en-us/archive/blogs/eopfieldnotes/use-headers-to-determine-which-exchange-online-tenant-a-message-was-attributed-to): X-EOPTenantAttributedMessage e la procedura in caso di attribuzione al tenant errato.

4.  [Microsoft Learn: Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): Riferimento ai tipi di Inbound Connector, TlsSenderCertificateName e attribution.
