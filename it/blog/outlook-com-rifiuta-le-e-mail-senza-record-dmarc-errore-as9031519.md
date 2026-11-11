---
title: "Outlook.com rifiuta le e-mail senza record DMARC: errore AS9031519"
navTitle: "Outlook.com: AS9031519"
description: "Dal 5 novembre 2026, Outlook.com, Hotmail e Live rifiutano i messaggi provenienti da domini senza record DMARC con 550 5.7.1 e l'aggiunta AS9031519, anche in caso di bassi volumi di invio. Cosa documenta Microsoft, cosa hanno misurato gli operatori, come verificare il proprio dominio e cosa mostra l'indice di sicurezza e-mail dei comuni svizzeri."
date: "2026-11-10"
kategorie: "SMTP e flusso e-mail"
timeToRead: "8 min di lettura"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
hauptthema: "smtp-mailflow"
produkte:
  - "exchange-online"
protokolle:
  - "mail-auth"
  - "dns"
  - "smtp"
  - "troubleshooting"
related:
  - dns-records-e-mail-stolpersteine
  - microsoft-365-compauth-reason-codes
slug: "outlook-com-rifiuta-le-e-mail-senza-record-dmarc-errore-as9031519"
translationId: "article-84656f0bf3da2006"
translationOf: outlook-com-dmarc-ablehnung-as9031519
url: https://rafaelpfister.ch/it/blog/outlook-com-rifiuta-le-e-mail-senza-record-dmarc-errore-as9031519
translationSourceHash: afec1593a28f2ec6ab14b2af5b064bae9b03ef23d0d091f8b659713eff887222
translationModel: gpt-5.6-terra
translatedAt: 2026-11-11T03:20:18.730Z
translationReview: automatic
---

# Outlook.com rifiuta le e-mail senza record DMARC: errore AS9031519

Dal 5 novembre 2026, i messaggi destinati alle caselle di posta di Outlook.com, Hotmail, Live e MSN vengono restituiti con un nuovo errore se il dominio del mittente non ha pubblicato un record DMARC:

```text
550 5.7.1 Message rejected: Message lacks SPF/DKIM/DMARC Auth Records
(AS9031519)
```

Dati aggiornati al 10 novembre 2026.

## Cosa documenta Microsoft

Le linee guida per i postmaster di Outlook.com continuano a contenere solo la regola del 5 maggio 2025: i domini che inviano più di 5000 messaggi al giorno a account Outlook.com devono superare SPF e DKIM e pubblicare un record DMARC con almeno `p=none`, allineato tramite SPF o DKIM. I messaggi che non soddisfano questi requisiti finiscono dapprima nella cartella Posta indesiderata e possono successivamente essere rifiutati con il seguente errore:

```text
550; 5.7.515 Access denied, sending domain [SendingDomain] does not meet
the required authentication level.
```

Per tutti gli altri mittenti, la stessa pagina formula soltanto una raccomandazione. L'aggiunta `AS9031519` e il testo dell'errore `Message lacks SPF/DKIM/DMARC Auth Records` non compaiono né nelle linee guida per i postmaster né nell'articolo di supporto relativo a 5.7.515.

Microsoft si è espresso sulla situazione attuale soltanto nei forum Microsoft Q&A. Un moderatore contrassegnato come «Microsoft External Staff» ha scritto il 9 novembre che è in corso un problema nel servizio Exchange relativo alla consegna a Outlook.com, Hotmail.com e Live.com. È classificato come degrado del servizio e Microsoft ne sta indagando la causa. Il contributo non indica né un numero di incidente, né una causa, né una data per la risoluzione. Non sono disponibili un post sul blog, una voce nelle linee guida per i postmaster o un avviso pubblico sullo stato del servizio.

## Cosa hanno misurato gli operatori

I dati più attendibili provengono da provider di posta che hanno analizzato i propri log e li hanno pubblicati sulla mailing list mailop e nei propri blog (MailRoute, Mythic Beasts, MXroute, Fastmail, one.com, EasyDMARC). Le osservazioni coincidono:

- **Causa scatenante:** il dominio del mittente non ha un record DMARC. SPF e DKIM erano validi e allineati nei casi documentati. Sebbene il testo dell'errore menzioni tutte e tre le procedure, il rifiuto avviene per l'assenza del record DMARC.
- **Volume:** sono interessati anche domini con pochi messaggi alla settimana, ben al di sotto della soglia di 5000 al giorno.
- **Destinatari:** solo le caselle di posta consumer (outlook.com, hotmail.com, live.com, msn.com e varianti nazionali come hotmail.de o outlook.be). Secondo le segnalazioni disponibili, i tenant Exchange Online come destinatari non rifiutano con questo errore.
- **Irregolare:** viene rifiutata solo una parte dei messaggi. Un nuovo invio dello stesso messaggio può andare a buon fine.
- **Risoluzione:** nei casi documentati è stato sufficiente un record con `v=DMARC1; p=none`. Dopo la pubblicazione, Outlook.com ha ripreso ad accettare i messaggi in un periodo compreso tra pochi minuti e circa un'ora e mezza.

Due analisi delimitano ulteriormente la causa scatenante: presso one.com venivano rifiutati solo i domini la cui interrogazione di `_dmarc.<domain>` terminava con `NXDOMAIN`. I domini per i quali il nome esisteva ma non era presente alcun record TXT (`NOERROR` senza risposta) non erano interessati. MXroute ha rilevato in un log 367 domini mittenti rifiutati, 286 dei quali con `NXDOMAIN` su `_dmarc`. Questa delimitazione si basa sui dati di singoli operatori e non è stata chiarita in modo definitivo.

Dal pomeriggio del 9 novembre, alcuni interessati segnalano che i rifiuti sono cessati. Microsoft non lo ha confermato.

## Caso particolare: Microsoft 365 come mittente

I tenant in Microsoft 365 sono colpiti in modo superiore alla media come mittenti. Microsoft 365 non crea alcun record DMARC per un dominio proprio e, finché DKIM non è attivato per il dominio, Exchange Online firma con il dominio `onmicrosoft.com` del tenant. Il rapporto di mancata consegna racchiude inoltre il rifiuto in un secondo codice di stato che fa riferimento a SPF:

```text
550 5.7.23 The message was rejected because of Sender Policy Framework
violation -> 550 5.7.1 Message rejected: Message lacks SPF/DKIM/DMARC
Auth Records (AS9031519)
```

In questi casi, il record SPF è di norma corretto. Fa fede l'errore interno dopo la freccia.

## Verificare il proprio dominio

Viene interrogato il record TXT sotto `_dmarc.<domain>`:

```bash
dig +short TXT _dmarc.example.com
```

```powershell
Resolve-DnsName -Type TXT -Name _dmarc.example.com
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `+short` | `dig` restituisce solo il contenuto del record |
| `TXT` o `-Type TXT` | tipo di record dell'interrogazione |
| `_dmarc.example.com` o `-Name` | nome interrogato: l'etichetta `_dmarc` prima del dominio del mittente |

</details>

Un output vuoto con `dig +short` o l'errore `DNS name does not exist` con `Resolve-DnsName` significa che non esiste alcun record DMARC. Per i sottodomini vale quanto segue: se invia `news.example.com` e non dispone di un proprio record, secondo RFC 7489 conta il record del dominio organizzativo `example.com`.

Il [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) in questa pagina verifica DMARC insieme a SPF, DKIM e all'allineamento in un'unica operazione.

## Misura immediata e il percorso successivo

Se manca il record, questa voce ripristina la consegna a Outlook.com:

```text
_dmarc.example.com.  3600  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com"
```

`p=none` non cambia nulla nel trattamento delle vostre e-mail presso altri destinatari e può quindi essere pubblicato senza preparativi. Il tag `rua` non è necessario per l'accettazione da parte di Outlook.com. Senza `rua`, tuttavia, non riceverete rapporti aggregati e senza questi rapporti manca la base per ogni passaggio successivo. L'indirizzo deve raggiungere una casella di posta o un servizio di analisi che elabori i rapporti XML.

`p=none` non protegge il dominio da abusi. Il record soddisfa soltanto la condizione di accettazione. Per una protezione efficace occorrono questi passaggi:

1. Censire tutti i sistemi che inviano con il dominio (server di posta, servizio newsletter, ERP, negozio online, scanner).
2. Mantenere SPF completo e valido per questi sistemi, con al massimo dieci lookup DNS.
3. Attivare DKIM con il proprio dominio, in Microsoft 365 per ogni dominio nel portale Defender.
4. Analizzare i rapporti aggregati per alcune settimane e correggere le fonti non allineate.
5. Elevare la policy a `p=quarantine` e successivamente a `p=reject`.

La [guida per amministratori DNS](https://rafaelpfister.ch/blog/dns-records-e-mail-stolpersteine) descrive gli errori tipici nella pubblicazione di questi record (record SPF duplicati, chiavi DKIM troppo lunghe, nome della zona aggiunto automaticamente).

## L'impatto in Svizzera: 152 comuni in cinque giorni

L'[indice di sicurezza e-mail Svizzera](https://rafaelpfister.ch/open-data/mail-security-index) verifica ogni notte i record DNS di 3364 organizzazioni svizzere, tra cui 2078 comuni. Il [registro delle modifiche](https://rafaelpfister.ch/open-data/mail-security-index/aenderungen) mostra dal 6 novembre un'ondata di record DMARC appena pubblicati. La data è in ogni caso quella del confronto notturno.

| Confronto | Organizzazioni con nuovo record DMARC | di cui comuni |
|---|---|---|
| Dal 19 ottobre al 5 novembre | al massimo 5 al giorno | al massimo 4 al giorno |
| 6 novembre | 4 | 4 |
| 7 novembre | 22 | 21 |
| 8 novembre | 1 | 1 |
| 9 novembre | 2 | 2 |
| 10 novembre | 140 | 124 |

In cinque giorni, 169 organizzazioni hanno pubblicato per la prima volta un record DMARC, di cui 152 comuni. Il numero di organizzazioni senza DMARC nell'indice è sceso da 817 a 650. Il momento coincide con i rifiuti presso Outlook.com. Non ho chiesto ai comuni stessi le ragioni.

I record hanno le seguenti caratteristiche:

- **Policy:** 139 dei 152 comuni hanno impostato `p=none`, cinque `p=quarantine`, tre `p=reject`. Per cinque, il record è costituito soltanto da `v=DMARC1` senza tag `p=` ed è quindi non valido.
- **Rapporti:** 114 record contengono esclusivamente `v=DMARC1; p=none`. Complessivamente, 124 dei 152 comuni non hanno indicato un indirizzo `rua` e non ricevono quindi rapporti.
- **Modifiche centralizzate:** le modifiche si verificano in blocchi. Il 10 novembre si sono aggiunti 32 comuni ginevrini, i cui server di posta operano tutti sotto `acg-geneve.ch`, oltre a 26 comuni zurighesi e 19 friburghesi. Il 7 novembre erano otto comuni nidvaldesi e sette ticinesi. Il raggruppamento per server di posta e cantone suggerisce che i fornitori di servizi informatici abbiano impostato i record per tutti i comuni assistiti in un'unica operazione.

Per la consegna a Outlook.com questi record sono sufficienti. Non proteggono ancora i domini comunali da mittenti falsificati: senza `rua` mancano i dati per elevare in seguito la policy. 537 dei 2078 comuni nell'indice non hanno ancora alcun record DMARC.

## Valutazione

Non è chiaro se Microsoft ritirerà il rifiuto o lo confermerà come regola. Per la pianificazione, ciò ha poca importanza: Google e Yahoo richiedono DMARC dai mittenti di massa dal febbraio 2024, Microsoft dal maggio 2025, e le linee guida per i postmaster di Outlook.com lo raccomandano a tutti i mittenti. Negli ultimi giorni, un dominio senza record DMARC non era più consegnabile in modo affidabile presso uno dei maggiori provider di posta. Il record con `p=none` e `rua` si pubblica in pochi minuti e non ha effetti collaterali sulla consegna.

## Fonti

1.  [Outlook.com Postmaster: Policies, Practices, and Guidelines](https://substrate.office.com/ip-domain-management-snds/postmaster/Policies): Requisiti ufficiali per i mittenti; indica la soglia di 5000 messaggi al giorno, l'errore 5.7.515 e la raccomandazione a tutti i mittenti.

2.  [Microsoft Support: Fix NDR error 550 5.7.515 in Outlook.com](https://support.microsoft.com/en-us/outlook/fix-ndr-error-550-5-7-515-in-outlook-com): Articolo di supporto sull'errore documentato per mittenti con volumi elevati.

3.  [Microsoft Q&A: SPF Validation Error](https://learn.microsoft.com/en-us/answers/questions/6063210/spf-validation-error): Discussione nel forum con la risposta del moderatore del 9 novembre 2026 sul problema in corso; contributo nel forum, non un avviso ufficiale sullo stato del servizio.

4.  [MailRoute: Outlook.com Is Quietly Rejecting Email From Domains Without DMARC](https://mailroute.net/blog/outlook-rejecting-email-without-dmarc): Analisi dei log in uscita di un provider di posta a partire dal 5 novembre 2026.

5.  [Mythic Beasts: Microsoft are rejecting your mail](https://www.mythic-beasts.com/blog/2026/11/09/microsoft-are-rejecting-your-mail/): Rapporto di un hosting provider con oltre 500 rifiuti il primo giorno e risoluzione tramite `p=none`.

6.  [mailop: Microsoft requiring DMARC now or just us?](https://www.mail-archive.com/mailop@mailop.org/msg27569.html): Discussione della mailing list degli operatori con le misurazioni di MXroute, Fastmail e one.com, inclusa la delimitazione a `NXDOMAIN`.

7.  [EasyDMARC: Microsoft Rejecting Emails Without DMARC](https://easydmarc.com/blog/microsoft-rejecting-emails-without-dmarc/): Test riprodotto con SPF e DKIM validi e allineati, senza record DMARC.

8.  [heise online: Outlook.com: Die Mail ist echt und sie kommt trotzdem nicht an](https://www.heise.de/news/Outlook-com-Die-Mail-ist-echt-und-sie-kommt-trotzdem-nicht-an-11517776.html): Riassunto in lingua tedesca delle segnalazioni del 9 novembre 2026.

9.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489): Specifica, tra l'altro, relativa alla determinazione del record tramite il dominio organizzativo.

10.  [Mail-Sicherheitsindex Schweiz: Änderungsprotokoll](https://rafaelpfister.ch/open-data/mail-security-index/aenderungen): Modifiche quotidiane ai record DNS di 3364 organizzazioni svizzere; base delle cifre relative ai comuni.
