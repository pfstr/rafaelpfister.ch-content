---
title: "compauth in Microsoft 365: Composite Authentication e tutti i Reason Code"
navTitle: "Codici compauth"
description: "Microsoft 365 integra SPF, DKIM e DMARC con una propria valutazione: compauth. Cosa verifica Composite Authentication, cosa significano pass, softpass, fail e none e quale causa si cela dietro ogni Reason Code, da 000 a 905."
date: "2026-08-26"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min di lettura"
themen:
  - microsoft-365-exchange
hauptthema: "microsoft-365-exchange"
produkte:
  - "exchange-online"
protokolle:
  - "mail-auth"
  - "troubleshooting"
related:
  - exchange-authmechanism-10-authas-internal
  - exchange-hybrid-header-intern-extern
  - dns-records-e-mail-stolpersteine
slug: "compauth-in-microsoft-365-composite-authentication-e-tutti-i-reason-code"
translationId: "article-a9dceac9ee095bbd"
translationOf: microsoft-365-compauth-reason-codes
url: https://rafaelpfister.ch/it/blog/compauth-in-microsoft-365-composite-authentication-e-tutti-i-reason-code
translationSourceHash: a37557eaef3ea6605e72281d81c56154d6062ae726ef646baa906c2d7d9927a4
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:21:07.131Z
translationReview: automatic
---

# compauth in Microsoft 365: Composite Authentication e tutti i Reason Code

Nell'intestazione `Authentication-Results` di un messaggio ricevuto in Microsoft 365, accanto ai risultati standard di SPF, DKIM e DMARC, è presente un campo proprietario di Microsoft:

```text
Authentication-Results: spf=pass (sender IP is 192.0.2.10)
  smtp.mailfrom=example.com; dkim=pass (signature was verified)
  header.d=example.com; dmarc=pass action=none header.from=example.com;
  compauth=pass reason=100
```

`compauth` sta per Composite Authentication: Microsoft 365 combina i risultati di SPF, DKIM e DMARC con altri segnali del messaggio in una valutazione complessiva dell'affidabilità dell'indirizzo From visibile. La base della valutazione è il dominio From, ossia l'indirizzo che i destinatari vedono nel client di posta. In questo modo Microsoft colma la lacuna che si crea quando un dominio mittente non ha pubblicato record di autenticazione o li ha pubblicati incompleti: anche senza una policy DMARC, viene verificato implicitamente se l'email corrisponde al dominio dichiarato.

## I quattro risultati

- `compauth=pass`: Il messaggio ha superato l'autenticazione esplicita (DMARC) o implicita.
- `compauth=softpass`: Il controllo implicito è stato superato con un livello di sicurezza inferiore.
- `compauth=fail`: Il messaggio non ha superato il controllo esplicito o implicito.
- `compauth=none`: Non è stato eseguito alcun controllo Composite oppure è stato saltato.

Un `compauth=fail` non comporta automaticamente la quarantena o la cartella Posta indesiderata. È un segnale di input per la decisione del filtro; per il trattamento effettivo sono determinanti `CAT` e altri campi nell'intestazione `X-Forefront-Antispam-Report`. Viceversa: chi vuole sapere perché compauth ha preso questa decisione deve consultare il codice `reason` direttamente dopo il risultato.

## Panoramica dei Reason Code

Il codice a tre cifre indica la regola che ha portato al risultato. La prima cifra raggruppa i codici: 0xx e 6xx sono insuccessi, 1xx e 7xx sono controlli superati, 2xx è softpass, 3xx, 4xx e 9xx indicano che non è stato eseguito alcun controllo oppure che è stato saltato.

| Codice | Significato |
|---|---|
| `000` | Fallimento esplicito: DMARC fail con una policy `p=quarantine` o `p=reject`. |
| `001` | Fallimento implicito: il dominio non pubblica record di autenticazione oppure solo record deboli (SPF `~all`/`?all`, DMARC `p=none`). |
| `002` | L'organizzazione ha vietato esplicitamente, per questa coppia mittente/dominio, l'invio di email spoofed (voce gestita manualmente). |
| `010` | DMARC fail con `p=reject`/`p=quarantine`, e il dominio mittente è un Accepted Domain interno (spoofing della propria organizzazione). |
| `100` | SPF o DKIM superati, i domini MAIL FROM e From sono allineati. |
| `101` | Il messaggio è firmato DKIM dal dominio From. |
| `102` | I domini MAIL FROM e From sono allineati, SPF superato. |
| `103` / `104` | Il dominio From corrisponde al record PTR (reverse lookup) dell'indirizzo IP di consegna. |
| `108` | DKIM fail dovuto a una modifica del corpo del messaggio in precedenti passaggi legittimi, ad esempio nell'ambiente OnPrem interno. |
| `109` | Il dominio non ha un record DMARC, ma il controllo sarebbe stato superato. |
| `111` | Nonostante un errore temporaneo o permanente DMARC, il dominio SPF o DKIM è allineato con il dominio From. |
| `112` | Un timeout DNS ha impedito il recupero del record DMARC. |
| `115` | L'email proviene da un'organizzazione Microsoft 365 nella quale il dominio From è configurato come Accepted Domain. |
| `116` | Il record MX del dominio From corrisponde al record PTR dell'IP di consegna. |
| `130` | Un ARC sealer configurato come attendibile ha sovrascritto il DMARC fail. |
| `201` / `202` | Softpass: il dominio From corrisponde al record PTR oppure alla relativa subnet. |
| `3xx` / `4xx` / `9xx` | Nessun controllo Composite eseguito oppure controllo saltato. |
| `501` / `502` | DMARC non applicato perché si tratta di un NDR valido. |
| `601` | Fallimento implicito: il dominio mittente è un Accepted Domain interno (self-spoofing, frequente con Direct Send). |
| `701`–`704` | DMARC non applicato perché l'organizzazione riceve da questa infrastruttura email dimostrabilmente legittime. |
| `905` | DMARC non applicato a causa di routing complesso, ad esempio email Internet via Exchange OnPrem o un servizio di terze parti prima di Microsoft 365. |

## I casi più frequenti nella pratica

**`compauth=fail reason=001`** è il caso standard per i domini senza autenticazione o con autenticazione debole. La correzione spetta al mittente: pubblicare SPF con `-all`, firma DKIM e una policy DMARC. Finché i record mancano, la recapitalità dipende dai segnali di reputazione.

**`compauth=fail reason=601`** compare quando arrivano dall'esterno email con il proprio dominio come mittente, tipicamente con Direct Send: dispositivi multifunzione, applicazioni o fornitori consegnano direttamente all'MX senza un connector autenticato. La correzione consiste in un Inbound Connector configurato correttamente oppure nell'inclusione della fonte nel proprio SPF.

**`compauth=fail reason=000` o `010`** significa che DMARC è stato applicato regolarmente. Se accanto compare `action=oreject`, Microsoft 365 ha tradotto la policy reject del mittente in una consegna in quarantena. Non c'è nulla da correggere, salvo che il mittente sia legittimo e la sua autenticazione sia difettosa.

**`reason=108`** e **`reason=130`** riguardano scenari di inoltro e gateway: una stazione intermedia ha modificato l'email oppure un ARC sealer attendibile ha conservato i risultati di verifica originali. Chi gestisce un gateway davanti a Microsoft 365 dovrebbe configurare il relativo ARC sealing come attendibile nella configurazione antispam; altrimenti le email legittime continueranno a fallire DMARC.

## Leggere compauth nell'header

Nella pratica `compauth` è raramente isolato: solo l'interazione con i singoli risultati SPF, DKIM e DMARC, l'allineamento dei domini coinvolti e la catena `Received` forniscono il quadro completo. L'[analizzatore di header email](/tools/header-analyzer) su questo sito decodifica `compauth` con il relativo Reason Code direttamente nel browser e mostra affiancati i domini corrispondenti (From, Envelope-From, `d=`) per la valutazione dell'allineamento; l'header incollato non lascia il browser.

## Fonti

1.  [Microsoft Learn: Anti-spam message headers](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Riferimento ufficiale dei campi Authentication-Results e della tabella completa dei compauth Reason Code.

2.  [Microsoft Learn: Security Operations guide for email authentication](https://learn.microsoft.com/en-us/defender-office-365/email-auth-sec-ops-guide): Procedura per gli errori di autenticazione dal punto di vista SecOps.

3.  [Microsoft Learn: Configure trusted ARC sealers](https://learn.microsoft.com/en-us/defender-office-365/email-authentication-arc-configure): Configurazione di ARC sealer attendibili per scenari di gateway e inoltro (Reason Code 130).

4.  [Microsoft Learn: Spam confidence level (SCL)](https://learn.microsoft.com/en-us/defender-office-365/anti-spam-spam-confidence-level-scl-about): Distinzione tra il segnale compauth e la decisione effettiva del filtro.
