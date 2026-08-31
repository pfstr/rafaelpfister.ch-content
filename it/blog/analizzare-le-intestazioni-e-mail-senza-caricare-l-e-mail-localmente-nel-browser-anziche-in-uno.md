---
title: "Analizzare le intestazioni e-mail senza caricare l’e-mail: localmente nel browser anziché in uno strumento web"
navTitle: "Analizzare le intestazioni localmente"
description: "Le intestazioni e-mail contengono nomi host interni, indirizzi IP e dati personali. Chi le incolla in uno strumento online trasmette queste informazioni a un server di terzi. Perché l’analisi non richiede un server e cosa può fare uno strumento eseguito localmente nel browser."
date: "2026-08-26"
kategorie: "SMTP e flusso di posta"
timeToRead: "7 min di lettura"
themen:
  - smtp-mailflow
hauptthema: "smtp-mailflow"
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "mail-auth"
  - "troubleshooting"
related:
  - microsoft-365-compauth-reason-codes
  - exchange-hybrid-header-intern-extern
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
slug: "analizzare-le-intestazioni-e-mail-senza-caricare-l-e-mail-localmente-nel-browser-anziche-in-uno"
translationId: "article-cad792e705cee24e"
translationOf: e-mail-header-analysieren-ohne-upload
url: https://rafaelpfister.ch/it/blog/analizzare-le-intestazioni-e-mail-senza-caricare-l-e-mail-localmente-nel-browser-anziche-in-uno
translationSourceHash: 11c4e7d120ea34ca557f0136b93120e5e8e9d72dc7350fd2df7880b23ff46649
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:16:33.305Z
translationReview: automatic
---

# Analizzare le intestazioni e-mail senza caricare l’e-mail: localmente nel browser anziché in uno strumento web

Il modo abituale di analizzare un’intestazione e-mail è questo: copiare l’intestazione dal client di posta, incollarla in uno strumento online e farla analizzare. È pratico, ma l’intera intestazione viene così inviata al server del gestore dello strumento. Pochi sono consapevoli di ciò che viene effettivamente trasmesso.

## Cosa contiene davvero un’intestazione

Un’intestazione completa di un’e-mail proveniente da un ambiente aziendale contiene tipicamente:

- **Nomi host interni e indirizzi IP:** Ogni riga `Received` documenta un server nel percorso di consegna, inclusi i server Exchange interni, i gateway e i load balancer con FQDN e spesso indirizzo IP privato. Nel complesso ne risulta uno schema dell’infrastruttura di posta.
- **Dati personali:** Indirizzi del mittente e del destinatario, nomi visualizzati, oggetto, Message-ID e, a seconda del client, l’indirizzo IP del mittente originario.
- **Software e versioni:** Le righe Received e le intestazioni specifiche del prodotto indicano i prodotti utilizzati, in parte con le relative versioni.
- **Valutazioni interne all’organizzazione:** In Microsoft 365, ad esempio, la valutazione completa di spam e autenticazione, gli identificativi del tenant e la classificazione interna del messaggio.

Per un aggressore si tratta di materiale utile alla preparazione, mentre per la protezione dei dati sono dati personali: mittente, destinatario e oggetto di un messaggio concreto. Ai sensi della legge sulla protezione dei dati riveduta, il trattamento da parte di uno strumento online estero resta una comunicazione a terzi, in caso di dubbio all’estero. Nel caso di un’intestazione tratta da una richiesta di supporto di un cliente, la questione si fa ancora più delicata: inserire i suoi dati in uno strumento web di terzi è difficilmente giustificabile senza una base giuridica o il consenso.

## L’analisi non richiede un server

Il punto decisivo è questo: un’intestazione è puro testo e la sua valutazione è puro parsing. Ordinare cronologicamente la catena Received, calcolare le differenze tra timestamp, decodificare `Authentication-Results`, confrontare domini: nulla di tutto questo richiede una componente server. Tutto viene eseguito in JavaScript nel browser, senza che l’intestazione lasci il dispositivo.

Uno strumento costruito in questo modo si distingue sostanzialmente, sotto il profilo della sicurezza, da un classico analizzatore online: non vi sono trasmissione, memorizzazione presso il gestore né file di log con intestazioni di terzi. L’analisi dell’intestazione di un cliente resta quindi allo stesso livello dell’apertura del file in un editor locale, ma più leggibile.

## Cosa può fare uno strumento locale

Il [Mail-Header-Analyzer](/tools/header-analyzer) su questo sito è costruito secondo questo principio. L’intestazione incollata viene analizzata esclusivamente localmente nel browser. Le funzionalità dimostrano che non si perde nulla:

- **Percorso di consegna con tempi di transito:** La catena `Received` viene ordinata cronologicamente, viene calcolato il tempo di permanenza per ogni stazione e viene evidenziato il tratto più lungo. In questo modo è visibile dove una consegna lenta si è effettivamente bloccata. Vengono rilevati e indicati gli sfasamenti dell’orologio tra server.
- **Crittografia del trasporto per hop:** La versione TLS e il cipher vengono letti dalle righe Received, dove il server ricevente li registra; Microsoft, Postfix ed Exim utilizzano formati diversi.
- **Autenticazione:** Risultati SPF, DKIM e DMARC da `Authentication-Results` (RFC 8601), inclusi dettagli quali `header.d`, `smtp.mailfrom` e `compauth` di Microsoft con codice motivo.
- **Allineamento DMARC:** Dominio From, Envelope-From e dominio DKIM sono affiancati e valutati secondo l’allineamento strict e relaxed.
- **Integrità ARC e DKIM:** Tracce dedicate nel grafico del flusso mostrano da dove a dove l’hash DKIM era integro e da quale stazione la catena ARC conserva i risultati della verifica.
- **Ambienti Microsoft:** I campi del filtro antispam (`X-Forefront-Antispam-Report`, SCL, CAT) vengono decodificati; le transizioni tra tenant e la classificazione ibrida vengono evidenziate nel percorso di consegna.

Una limitazione vale per ogni strumento di analisi delle intestazioni, locale o meno: mostra la valutazione documentata del server ricevente, non una propria verifica. L’intestazione non può dire se un record SPF abbia ancora oggi lo stesso aspetto che aveva al momento della ricezione.

## Inquadramento degli altri strumenti

Anche alcuni altri fornitori effettuano ormai l’analisi lato client; uno sguardo all’informativa sulla privacy e alla console di rete del browser chiarisce se, durante l’incollamento, non venga effettivamente inviata alcuna richiesta con il contenuto dell’intestazione. Per i classici analizzatori lato server vale una semplice regola: non inserire intestazioni provenienti da ambienti di produzione o da terzi, ma al massimo esempi anonimizzati.

Per analisi regolari di intestazioni relative a incidenti o richieste di supporto, uno strumento eseguito localmente è quindi la scelta naturale: non ci si deve porre la domanda su dove siano finiti i dati.

## Fonti

1.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://datatracker.ietf.org/doc/html/rfc8601): Standard per l’intestazione Authentication-Results, alla base della valutazione dell’autenticazione.

2.  [RFC 5321: Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321): Definizione delle righe Received (Trace Information), dalle quali è possibile ricostruire il percorso di consegna e i tempi di transito.

3.  [Microsoft Learn: Anti-spam message headers](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Riferimento ai campi delle intestazioni specifici di Microsoft 365 che un analizzatore decodifica.
