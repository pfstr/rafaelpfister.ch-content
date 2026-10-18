---
title: "Crittografia e-mail: metodi, chiavi e modelli operativi"
blatt: "verschluesselung"
description: "Crittografia e-mail per amministratori di messaggistica: SMTP-TLS, MTA-STS, DANE e REQUIRETLS, S/MIME e OpenPGP, modelli client, gateway e portale, ciclo di vita di chiavi e certificati, metadati, monitoraggio, ripristino e diagnostica."
fakten:
  - label: Domanda fondamentale
    wert: Quali dati sono protetti tra quali confini di fiducia?
    href: https://www.nist.gov/publications/trustworthy-email-0
  - label: Trasporto
    wert: SMTP STARTTLS protegge esattamente un hop
    href: https://datatracker.ietf.org/doc/html/rfc3207
  - label: Policy di trasporto
    wert: DANE · MTA-STS · REQUIRETLS
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-177r1.pdf
  - label: S/MIME
    wert: MIME + CMS + X.509/PKIX
    href: https://datatracker.ietf.org/doc/html/rfc8551
  - label: OpenPGP
    wert: Formato dei pacchetti, chiavi, firme e crittografia
    href: https://datatracker.ietf.org/doc/html/rfc9580
  - label: Firma
    wert: Autenticità e integrità; nessuna riservatezza
    href: https://datatracker.ietf.org/doc/html/rfc8551
  - label: Modello client
    wert: chiavi private all'endpoint; il server di posta memorizza testo cifrato
    href: https://www.nccoe.nist.gov/publication/1800-6/VolC/
  - label: Modello gateway
    wert: testo in chiaro nella zona controllata e nel gateway
    href: https://www.seppmail.com/products/
  - label: Modello portale
    wert: notifica via e-mail; contenuto in archivio web con identità propria
    href: https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm
  - label: Gestione delle chiavi
    wert: Inventario · protezione · distribuzione · rotazione · revoca · ripristino
    href: https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final
  - label: Metadati visibili
    wert: L'envelope SMTP e gli header esterni restano visibili a seconda del modello
    href: https://datatracker.ietf.org/doc/html/rfc8551
  - label: Regola di ripristino
    wert: Se necessario, proteggere le chiavi di crittografia; non replicare le chiavi di firma
    href: https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf
werbung:
  - tools
  - newsletter
ctaThemen:
  - e-mail-verschluesselung
  - seppmail
translationSourceHash: 915d86b5c19b97fc5564faf33053144b692f76fd3d6c732fd955f466d25c4983
translationModel: gpt-5.6-terra
translatedAt: 2026-10-16T09:06:39.593Z
translationReview: required
---

# Crittografia e-mail: metodi, chiavi e modelli operativi

La domanda «Questa e-mail è crittografata?» è tecnicamente incompleta. Un messaggio può essere leggibile sul client, memorizzato come testo cifrato sul server di posta, decrittografato su un gateway, protetto tra due MTA tramite [TLS](/kb/tls) e nuovamente archiviato non crittografato presso il destinatario. Una risposta attendibile indica quindi **oggetto protetto, endpoint, titolari delle chiavi e punti in chiaro**. NIST considera i meccanismi di trasporto, di dominio e di contenuto proprio per questo motivo come controlli cooperanti ma distinti ([NIST SP 800-177 Rev. 1](https://www.nist.gov/publications/trustworthy-email-0)).

La **crittografia del trasporto** protegge una connessione. SMTP STARTTLS viene negoziato nuovamente per ogni hop e termina al rispettivo MTA. La **crittografia del contenuto** protegge un'entità MIME con S/MIME o OpenPGP e può sopravvivere a più hop di trasporto e archiviazione. Il fatto che funzioni davvero end-to-end non dipende solo dal formato: se un gateway aziendale decrittografa per conto degli utenti, gli endpoint crittografici si trovano nei gateway e non nei client utente ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

Una **firma digitale** è un'altra prestazione di sicurezza. Collega i dati a una chiave di firma privata e consente di verificare integrità e identità; non rende segreto il contenuto. Viceversa, un messaggio decrittografato correttamente non dice ancora chi lo abbia creato. In S/MIME vengono distinti CMS `SignedData` e `EnvelopedData` oppure `AuthEnvelopedData`. OpenPGP separa anch'esso i pacchetti di firma, chiave di sessione e dati crittografati ([RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 8551, S/MIME Content Types](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 9580, Packet Syntax](https://datatracker.ietf.org/doc/html/rfc9580)).

La spiegazione segue un messaggio e chiede a ogni hop dove sia presente il testo in chiaro e chi controlli le chiavi. Crittografia del trasporto, procedure end-to-end, gateway, portali, metadati, diagnostica e ripristino vengono confrontati su questa base.

La crittografia e-mail può proteggere l'hop di trasporto, il contenuto del messaggio o l'accesso a un portale. Questi modelli hanno titolari delle chiavi diversi e punti diversi in cui il testo è disponibile in chiaro. L'articolo segue quindi prima il percorso di protezione e solo successivamente i singoli prodotti e metodi.

## Approccio architetturale: modelli di protezione e confini del testo in chiaro

La crittografia e-mail si differenzia soprattutto per i due punti che cifrano e decifrano. La tabella parte quindi dal normale percorso del messaggio e mostra poi dove il testo in chiaro rimane tecnicamente visibile.

| Modello | Endpoint crittografici | Il testo in chiaro è presente almeno | Stato operativo primario |
|---|---|---|---|
| SMTP-TLS opportunistico | due MTA adiacenti | su ogni MTA coinvolto | certificati, capacità TLS, coda e retry |
| TLS di trasporto obbligatorio | due MTA più policy pubblicata o locale | su ogni MTA coinvolto | MTA-STS/DANE/REQUIRETLS, DNS/HTTPS e report |
| S/MIME/OpenPGP da client a client | MUA o modulo crittografico dell'utente | prima della cifratura e dopo la decifratura all'endpoint | chiavi private, certificati/chiavi, compatibilità del client |
| da gateway a gateway | gateway di posta delle organizzazioni | sistema di posta interno, elaborazione del gateway, zona di destinazione | policy, directory, archivio chiavi del gateway, coda e mapping |
| portale web | mittente/gateway e applicazione web del portale | gateway/portale, browser dopo l'accesso | oggetto messaggio, account destinatario, OTP, link, conservazione |

Nessuno di questi modelli è genericamente «più forte». Il modello da client a client riduce i punti in chiaro nell'infrastruttura, ma rende più difficili il controllo antimalware, il journaling, l'eDiscovery e il ripristino. L'esempio pratico NCCoE esegue quindi la verifica S/MIME sul client utente con i relativi trust anchor, dopo che i controlli di trasporto e basati sul dominio sono terminati sull'MTA ([NCCoE: DNS-Based Email Security](https://www.nccoe.nist.gov/publication/1800-6/VolC/)). I gateway centralizzano policy e chiavi, ma diventano essi stessi punti altamente privilegiati per il testo in chiaro e le chiavi. I portali raggiungono destinatari senza crittografia della posta, ma creano un'ulteriore procedura web, di identità e di archiviazione. La scelta dipende dalla classificazione dei dati, dal modello di minaccia, dalla conservazione, dai requisiti legali e dalle capacità di entrambe le parti comunicanti, non dal nome del prodotto.

<iframe class="kb-infographic kb-infographic--compact" style="aspect-ratio: 1400 / 1076; max-width: 720px; margin-inline: auto" src="/images/kb-interaktiv-verschluesselung.svg?v=20260813" title="Interaktive Infografik: E-Mail-Verschlüsselung mit Transport-, Client-, Gateway- und Portalmodell, Klartextgrenzen, Schlüsselzustand, Metadaten und Admin-Kontrollpunkten" loading="lazy">
  <a href="/images/kb-interaktiv-verschluesselung.svg?v=20260813">Aprire direttamente il grafico interattivo</a>.
</iframe>

## SMTP-TLS: protezione per hop di trasporto

[SMTP](/kb/smtp) è stato originariamente progettato senza riservatezza obbligatoria. STARTTLS estende una sessione SMTP esistente: dopo il completamento dell'handshake TLS, le capacità vengono nuovamente negoziate tramite EHLO. In modalità opportunistica, l'MTA mittente può ripiegare sul testo in chiaro in assenza di STARTTLS o in caso di errore del certificato. Ciò migliora la protezione contro l'intercettazione passiva, ma senza una policy aggiuntiva non fornisce una garanzia affidabile contro downgrade o un attaccante attivo ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)).

**MTA-STS** pubblica dapprima un'indicazione DNS TXT e recupera la policy effettiva tramite HTTPS. In modalità `enforce` i nomi MX indicati devono essere convalidati tramite PKIX; gli errori causano consegna ritardata anziché testo in chiaro. **DANE per SMTP** associa informazioni sui certificati o sulle chiavi a record TLSA convalidati tramite DNSSEC. **REQUIRETLS** è un contrassegno SMTP e di messaggio con cui un mittente richiede TLS per l'ulteriore trasmissione. Questi metodi hanno dipendenze e ambiti diversi e non devono essere riuniti in un'unica casella «TLS richiesto» ([RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8689](https://datatracker.ietf.org/doc/html/rfc8689)).

SMTP TLS Reporting, **TLS-RPT**, fornisce report aggregati sulle verifiche riuscite e fallite delle policy TLS. Non dimostra una singola consegna e non contiene alcun messaggio; aiuta a riconoscere errori di certificato, DNS e policy prima che le code crescano a lungo. NIST raccomanda ai domini riceventi di segnalare DANE, MTA-STS o entrambi e descrive le diverse dipendenze da DNSSEC, HTTPS e PKIX ([RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460), [NIST SP 800-177 Rev. 1, sezione 5.2](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-177r1.pdf)).

SMTP-TLS termina al server di posta successivo. Se il contenuto del messaggio deve rimanere protetto attraverso più hop, si utilizzano metodi basati sui messaggi quali S/MIME o OpenPGP.

## S/MIME: CMS, X.509 e MIME

S/MIME non è una singola libreria di crittografia, ma una combinazione di più standard. MIME struttura le parti del messaggio; CMS definisce contenitori firmati e cifrati; X.509/PKIX associa le chiavi pubbliche alle identità e agli anchor di fiducia tramite certificati. Per ciascun destinatario, il mittente cifra la chiave simmetrica del contenuto con la relativa chiave pubblica o con un metodo concordato di gestione delle chiavi. Per la lettura è necessaria la chiave privata corrispondente ([RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280)).

L'amministratore deve separare due ruoli di chiave. Una **chiave di firma** deve essere disponibile solo al firmatario; una copia centrale consentirebbe tecnicamente a terzi di creare nuove firme con la sua identità. Una **chiave di decrittografia** può essere protetta o sottoposta a escrow secondo la policy dell'organizzazione, affinché i vecchi messaggi aziendali restino leggibili dopo perdita del dispositivo, cambio di ruolo o rinnovo del certificato. RFC 8551 sottolinea esplicitamente che una persona può avere più coppie di chiavi attive per scopi diversi; NIST distingue la protezione e il ripristino per tipo di chiave ([RFC 8551, Multiple Key Pairs](https://datatracker.ietf.org/doc/html/rfc8551), [NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).

Prima dell'invio, il mittente necessita di un certificato di crittografia idoneo e valido per **ogni** destinatario. La distribuzione può avvenire tramite una directory, messaggi firmati, una piattaforma di certificati o manualmente. Le liste di distribuzione sono particolarmente impegnative, poiché espansione, visibilità e disponibilità dei certificati di tutti i membri devono combaciare. Microsoft documenta per Exchange gli attributi `userSMIMECertificate` e `userCertificate` nonché la raccolta di certificati per la verifica della fiducia ([Microsoft: S/MIME in Exchange Server](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/smime/smime), [Microsoft: S/MIME in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo)).

## OpenPGP: il formato non è il modello di fiducia

RFC 9580 definisce pacchetti OpenPGP, chiavi, fingerprint, firme, chiavi di sessione e dati cifrati. Secondo lo standard, un portachiavi è soltanto una raccolta di chiavi; il modo in cui un'organizzazione trova le chiavi, verifica i proprietari e decide la fiducia è esterno a questo formato dati. Il diffuso storico **Web of Trust** è quindi una possibile modalità operativa, ma non una definizione sinonimica di OpenPGP. Altri ambienti utilizzano fingerprint verificati manualmente, chiavi firmate dall'organizzazione, WKD, servizi di directory o gateway gestiti centralmente ([RFC 9580, keyring e fingerprint](https://datatracker.ietf.org/doc/html/rfc9580)).

Per le e-mail, PGP/MIME incapsula firme e testo cifrato in strutture MIME. Inline-PGP protegge invece solo parti di testo selezionate e porta facilmente a problemi di interoperabilità con HTML, allegati, set di caratteri e canonicalizzazione delle righe. RFC 3156 standardizza `multipart/encrypted` e `multipart/signed` per OpenPGP. Occorre testare con le controparti reali se un client supporta pienamente le chiavi moderne RFC 9580, AEAD, le curve utilizzate e PGP/MIME ([RFC 3156](https://datatracker.ietf.org/doc/html/rfc3156), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

Un Key ID breve non costituisce una prova d'identità sufficiente. RFC 9580 avverte che i Key ID non devono essere considerati univoci; per le verifiche si utilizza il fingerprint completo. Certificati di revoca, scadenza, subkey, nuovi fingerprint e la distribuzione sicura di una rotazione fanno quindi parte dello stesso ciclo di vita della prima emissione della chiave.

S/MIME e OpenPGP possono essere applicati sul dispositivo terminale o centralmente sul gateway. Il gateway semplifica policy e gestione delle chiavi, ma ottiene in cambio accesso al testo in chiaro.

## Modello gateway: policy centrale e potere centrale sul testo in chiaro

Un gateway di posta accetta normali messaggi interni, classifica mittente, destinatario e dati, cerca chiavi S/MIME o OpenPGP appropriate e seleziona crittografia, firma, TLS o consegna tramite portale. In ingresso, se necessario, decrittografa prima del trasferimento al sistema di posta interno. Prodotti come [SEPPmail](/kb/seppmail), [Totemomail oppure Kiteworks EPG](/kb/totemomail) e [HIN](/kb/hin) utilizzano piattaforme diverse, ma la stessa fondamentale delega di fiducia: l'organizzazione delega la crittografia dall'endpoint dell'utente a un componente dell'infrastruttura ([SEPPmail: prodotti](https://www.seppmail.com/products/), [HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)).

Ciò non risolve da sé la distribuzione delle chiavi. Il gateway necessita comunque di un'associazione comprovata tra l'identità del destinatario e la chiave pubblica, di una priorità tra più metodi, di regole per chiavi scadute o revocate e di un fallback definito. Lo scambio automatico di chiavi tra gateway può semplificare questi passaggi, ma è esso stesso un protocollo di fiducia con identità delle controparti, cache, scadenza e stati di errore.

L'ordine della policy decide quali controlli vedono il testo in chiaro. Se un gateway in ingresso decrittografa, antimalware, DLP e archiviazione possono operare successivamente. Se un gateway in uscita cifra troppo presto, gli scanner a valle sono ciechi. Se un gateway modifica un'entità MIME già firmata, la firma del contenuto o [DKIM](/kb/mail-auth) può diventare non valida. RFC 6376 rende gli header canonizzati e il body l'oggetto della firma; firma, crittografia, disclaimer, riscrittura e DKIM devono quindi essere ordinati in una pipeline documentata ([RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376)).

Se presso il destinatario esterno non è disponibile una chiave adatta, un portale può memorizzare il messaggio e controllare l'accesso tramite autenticazione web. Il problema passa così dalla crittografia della posta all'identità web e all'esercizio della piattaforma.

## Consegna tramite portale: applicazione web invece della chiave del destinatario

Con la consegna tramite portale, il destinatario riceve una normale e-mail di notifica e richiama il contenuto protetto tramite un'applicazione web. L'endpoint crittografico è il portale, non il client di posta del destinatario. Il compito tecnico si sposta quindi dalla distribuzione di chiavi pubbliche all'identità web, alla gestione delle sessioni, all'archivio di messaggi e allegati, al link di notifica, alla conservazione e al canale di risposta sicuro.

Come esempio concreto, HIN documenta la registrazione, il numero di cellulare e il codice SMS, nonché l'opzione di ricordare un dispositivo per accessi successivi. Questo esempio dimostra il modello operativo, non un'implementazione universale del portale. Altri prodotti utilizzano password, codice monouso tramite canale separato, identità federata o un accesso cliente già esistente ([HIN: posta a non membri](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)).

Per gli amministratori, un portale richiede almeno le seguenti evidenze:

- Quali dati sono contenuti nella notifica non protetta e nel token URL?
- Come vengono protetti registrazione iniziale, associazione dell'identità, tentativi falliti, memorizzazione del dispositivo e recupero dell'account?
- Dove risiedono messaggi, allegati, chiavi e dati di audit, e quando vengono eliminati?
- Un messaggio rimane recuperabile dopo scadenza, eliminazione dell'utente o cambio di tenant?
- Come vengono registrati risposta sicura, inoltro, download e conferma di lettura?
- Cosa accade se il portale non è raggiungibile, il servizio SMS è guasto o l'account del destinatario è compromesso?

Uno stato HTTP `200` dimostra solo che un'interfaccia web risponde. Un test sintetico del portale deve coprire notifica, registrazione o login, secondo fattore, apertura, allegato, risposta e il record di audit previsto.

## Metadati, header e archiviazione

La crittografia del contenuto non protegge automaticamente l'envelope SMTP. Gli MTA che consegnano necessitano almeno degli indirizzi dei destinatari; i sistemi generano righe `Received` ed elaborano informazioni su dimensione e routing. Anche con S/MIME gli header esterni rimangono normalmente visibili. RFC 8551 consente di incapsulare un messaggio completo come `message/rfc822` all'interno dell'entità MIME protetta, ma richiede una rappresentazione distinguibile degli header interni ed esterni. Il fatto che oggetto, visualizzazione del mittente ed elenco destinatari siano realmente protetti è una funzione del client e del gateway, non una conseguenza garantita dell'icona del lucchetto ([RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322), [RFC 8551, Header Protection](https://datatracker.ietf.org/doc/html/rfc8551)).

L'archiviazione deve stabilire **quale rappresentazione** viene conservata: testo in chiaro originale, entità originale firmata, originale cifrato, copia di lavoro decrittografata o oggetto del portale. Il solo testo in chiaro semplifica l'eDiscovery, ma perde la prova crittografica originale. Il solo testo cifrato può diventare illeggibile dopo la perdita della chiave. La convalida della firma in un momento successivo richiede byte originali, catena di certificati, informazioni di fiducia e revoca dell'epoca e una base temporale; un certificato oggi scaduto non rende automaticamente priva di valore una firma allora valida.

## Ciclo di vita di chiavi e certificati

Un servizio crittografico produttivo necessita di un inventario che vada oltre le date di scadenza. NIST SP 800-57 tratta generazione, attivazione, protezione, inventario, distribuzione, utilizzo, rotazione, revoca, archivio, ripristino e distruzione come attività di gestione interconnesse ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).

| Fase | Domanda dell'amministratore | Evidenza |
|---|---|---|
| Identità | Chi o quale ruolo viene associato alla chiave? | processo di enrollment verificato, Subject/SAN/User ID, approvazione |
| Generazione | Dove viene creata la chiave privata? | client, HSM, gateway o CA; algoritmo ed esportabilità |
| Distribuzione | Come ottiene il mittente la corretta chiave pubblica? | directory, mail firmata, WKD, servizio di certificati, verifica del fingerprint |
| Utilizzo | La chiave può firmare, decrittografare o entrambe le cose? | Key Usage/EKU, flag OpenPGP, policy e client effettivo |
| Rotazione | Come si sovrappongono chiave vecchia e nuova? | pubblicazione prima del passaggio, test su tutte le piattaforme di destinazione |
| Revoca | Come apprendono verificatori e mittenti di una compromissione? | CRL/OCSP o revoca OpenPGP, policy di cache e di errore |
| Ripristino | Chi può decrittografare i contenuti storici? | procedura di escrow, approvazione a quattro occhi, audit, test di ripristino |
| Distruzione | Quando termina intenzionalmente la decrittografabilità? | prova di conservazione e cancellazione per chiavi e backup |

La verifica del percorso S/MIME comprende anchor di fiducia, certificati intermedi, periodo di validità, nome o identità e-mail, Key Usage/Extended Key Usage, algoritmo e stato di revoca. OCSP e CRL sono a loro volta dipendenze di rete e cache. Con un soft-fail, una verifica rimane possibile nonostante informazioni di revoca non raggiungibili; con un hard-fail, un problema infrastrutturale può bloccare tutti i messaggi firmati o cifrati. Questa decisione deve far parte della policy e del monitoraggio ([RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 6960](https://datatracker.ietf.org/doc/html/rfc6960)).

La diagnostica inizia dal modello di protezione scelto: hop TLS, firma e crittografia del contenuto oppure accesso al portale. Solo successivamente si verificano certificato, chiave, policy e percorso di consegna.

## Monitoraggio e analisi dei guasti

Il monitoraggio deve essere separato per modello di protezione. Per TLS di trasporto contano protocolli offerti e negoziati, errori di policy, validazione del certificato, TLS-RPT e anzianità della coda. Per S/MIME/OpenPGP contano chiavi del destinatario mancanti, percorsi o firme non validi, fallback, rotazione ed errori di decrittografia. Per i gateway si aggiungono decisione di policy, Message-ID, fingerprint della chiave, elaborazione del testo in chiaro e hop successivo. Per i portali, registrazione, MFA/OTP, archiviazione, scadenza dei link, login, apertura del messaggio e percorso di risposta sono segnali distinti.

Un protocollo d'incidente sensato risponde a:

1. Quale messaggio e quale percorso del destinatario sono interessati?
2. Quale modello di protezione e quale policy sono stati scelti?
3. Quale chiave pubblica o certificato è stato utilizzato, con quale fingerprint e stato?
4. Dove il messaggio è stato l'ultima volta leggibile, cifrato, memorizzato o in coda?
5. Il risultato è stato consegna, deposito sul portale, quarantena, fallback, retry o errore permanente?
6. Il risultato può essere riprodotto con un'identità di test controllata senza esporre dati di produzione?

## Backup e ripristino

Dati cifrati e chiavi devono essere pianificati insieme, ma con misure di protezione separate. Un backup di messaggi cifrati senza le chiavi private di decrittografia necessarie a lungo termine può essere correttamente protetto ma comunque non recuperabile dal punto di vista operativo. Viceversa, un backup di key escrow insufficientemente protetto trasforma molti segreti individuali in un punto di attacco centrale. NIST tratta l'uso consentito di chiavi private recuperate in funzione del tipo di chiave e del periodo crittografico ([NIST SP 800-57 Part 1 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf)).

L'ambito minimo di ripristino comprende:

- messaggi originali cifrati ed eventualmente entità MIME firmate con valore probatorio;
- chiavi private di **decrittografia** con metadati, passphrase, ruoli e procedure di approvazione;
- chiavi pubbliche, catene di certificati, trust anchor, dati di revoca e convalida storica;
- policy di gateway e portale, routing, associazione delle chiavi, dati utente e oggetti messaggio;
- fonti di audit e temporali affinché le decisioni precedenti restino tracciabili;
- un test di ripristino con chiavi vecchie e nuove, più destinatari, allegati e verifica della firma.

Le chiavi private di firma non dovrebbero essere copiate come meccanismo generale di ripristino. Se un'organizzazione deve firmare automaticamente o per delega, un'identità organizzativa esplicita con una propria policy è preferibile alla copia inosservata di una chiave personale. Altrimenti, capacità di ripristino e non ripudio entrano in conflitto.

## Strumenti diagnostici

La ricerca degli errori segue il modello di protezione scelto. Per la crittografia del trasporto vengono verificati DNS e TLS; per le procedure end-to-end seguono certificati, chiavi ed elaborazione client; per i gateway si aggiungono il relativo stato di policy e coda.

### MTA-STS, TLS-RPT e DANE nel DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für E-Mail-Transportpolicy-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type TXT _mta-sts.example.ch
Resolve-DnsName -Type TXT _smtp._tls.example.ch
Resolve-DnsName -Type TLSA _25._tcp.mx.example.ch
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short TXT _mta-sts.example.ch
dig +short TXT _smtp._tls.example.ch
dig +dnssec TLSA _25._tcp.mx.example.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) mostrano le policy di trasporto pubblicate. Per DANE deve inoltre essere dimostrato lo stato di validazione DNSSEC; un testo TLSA semplicemente visibile non è ancora un record autenticato.

### STARTTLS e catena di certificati

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection mx.example.ch -Port 25 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://mx.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz mx.example.ch 25
openssl s_client -starttls smtp -connect mx.example.ch:25 \
  -servername mx.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) verificano il percorso TCP. [`curl`](https://curl.se/docs/manpage.html) e [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) mostrano STARTTLS, handshake e catena. Se un MTA di produzione effettua fallback o accoda in caso di errori risulta solo dalla sua policy e dai log.

### Certificato S/MIME e finalità d'uso

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für S-MIME-Zertifikatsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
certutil.exe -dump .\recipient-smime.cer
certutil.exe -verify -urlfetch .\recipient-smime.cer
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
openssl x509 -in recipient-smime.pem -noout \
  -subject -issuer -serial -fingerprint -dates -purpose -ext keyUsage -ext extendedKeyUsage
openssl verify -show_chain -CAfile trust-chain.pem recipient-smime.pem
```

  </div>
</div>

[`certutil`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/certutil) e [`openssl x509`](https://docs.openssl.org/master/man1/openssl-x509/) mostrano identità, fingerprint, validità e finalità d'uso; [`openssl verify`](https://docs.openssl.org/master/man1/openssl-verify/) verifica un truststore indicato. Il test deve essere ripetuto con gli stessi trust anchor e le stesse regole di revoca del client o gateway di produzione.

### Fingerprint OpenPGP e struttura della chiave

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für OpenPGP-Schlüsselprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
gpg.exe --show-keys --with-fingerprint --with-subkey-fingerprint .\recipient.asc
gpg.exe --list-packets .\message.pgp
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
gpg --show-keys --with-fingerprint --with-subkey-fingerprint recipient.asc
gpg --list-packets message.pgp
```

  </div>
</div>

[`gpg`](https://www.gnupg.org/documentation/manuals/gnupg24/gpg.1.html) mostra fingerprint completi, subkey, capacità e struttura dei pacchetti. `--list-packets` analizza solo il formato; non conferma né proprietario né fiducia. I fingerprint devono essere verificati tramite un secondo canale autenticato o una procedura organizzativa definita.

### Percorso del portale e cattura dei pacchetti

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Portal- und Paketpfaddiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://securemail.example.ch/
pktmon filter remove
pktmon filter add SecureMail -p 25
pktmon start --capture --pkt-size 0 --file-name securemail.etl
pktmon stop
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://securemail.example.ch/
tcpdump -ni any -s 0 -w securemail.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest) e [`curl`](https://curl.se/docs/manpage.html) verificano solo la raggiungibilità HTTP/TLS. [`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) e [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) acquisiscono solo nel punto di misurazione scelto. I dati dei pacchetti possono contenere testo SMTP in chiaro, indirizzi e metadati del portale e devono essere protetti di conseguenza.

## Storia tecnica

PEM, Privacy Enhanced Mail, standardizzò all'inizio degli anni 1990 la posta Internet protetta crittograficamente, ma non raggiunse un'ampia interoperabilità. S/MIME combinò successivamente la struttura CMS derivata da PKCS con MIME e X.509. Gli standard S/MIME si sono evoluti attraverso più generazioni fino alla versione 4.0 pubblicata nel 2019; CMS stesso è standardizzato in RFC 5652 ([RFC 1421](https://datatracker.ietf.org/doc/html/rfc1421), [RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652)).

OpenPGP nacque dal formato di pacchetti PGP e fu inizialmente standardizzato in RFC 2440, poi in RFC 4880. RFC 9580 ha sostituito RFC 4880 nel 2024 e ha modernizzato, tra l'altro, versioni delle chiavi, fingerprint e procedure per la crittografia autenticata. PGP/MIME secondo RFC 3156 rimane l'incapsulamento e-mail; l'evoluzione del formato non elimina tuttavia il compito organizzativo di regolare identità delle chiavi e ripristino ([RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580), [RFC 3156](https://datatracker.ietf.org/doc/html/rfc3156)).

SMTP STARTTLS è stato standardizzato nel 1999 in RFC 2487 e aggiornato nel 2002 da RFC 3207. DANE per SMTP, MTA-STS, TLS-RPT e REQUIRETLS hanno successivamente aggiunto livelli di policy e reporting contro downgrade e configurazioni errate. Parallelamente, le aziende hanno introdotto gateway e portali web perché i destinatari esterni non dispongono in modo affidabile di chiavi S/MIME o OpenPGP compatibili. La crittografia e-mail è così diventata meno una singola funzionalità client e più un sistema distribuito composto da SMTP, DNS, HTTPS, PKI, directory, policy, identità web e ripristino ([RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461), [RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460), [RFC 8689](https://datatracker.ietf.org/doc/html/rfc8689)).

## Fonti

- [NIST – SP 800-177 Rev. 1, Trustworthy Email](https://www.nist.gov/publications/trustworthy-email-0)
- [IETF RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [IETF RFC 8551 – S/MIME 4.0 Message Specification](https://datatracker.ietf.org/doc/html/rfc8551)
- [IETF RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [IETF RFC 5652 – Cryptographic Message Syntax](https://datatracker.ietf.org/doc/html/rfc5652)
- [NCCoE – DNS-Based Email Security Practice Guide](https://www.nccoe.nist.gov/publication/1800-6/VolC/)
- [IETF RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [IETF RFC 8689 – SMTP REQUIRETLS](https://datatracker.ietf.org/doc/html/rfc8689)
- [IETF RFC 8460 – SMTP TLS Reporting](https://datatracker.ietf.org/doc/html/rfc8460)
- [NIST – SP 800-177 Rev. 1, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-177r1.pdf)
- [IETF RFC 5280 – Internet X.509 PKI Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [NIST – SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [Microsoft Learn – S/MIME in Exchange Server](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/smime/smime)
- [Microsoft Learn – S/MIME in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo)
- [IETF RFC 3156 – MIME Security with OpenPGP](https://datatracker.ietf.org/doc/html/rfc3156)
- [SEPPmail – prodotti](https://www.seppmail.com/products/)
- [HIN – HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)
- [IETF RFC 6376 – DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [HIN Support – posta a non membri](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)
- [IETF RFC 5322 – Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322)
- [IETF RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960)
- [NIST – SP 800-57 Part 1 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – pagina man di dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – pagina man di nc](https://man.openbsd.org/nc)
- [curl – pagina man della riga di comando](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Microsoft Learn – certutil](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/certutil)
- [OpenSSL – x509](https://docs.openssl.org/master/man1/openssl-x509/)
- [OpenSSL – verify](https://docs.openssl.org/master/man1/openssl-verify/)
- [GnuPG – manuale gpg](https://www.gnupg.org/documentation/manuals/gnupg24/gpg.1.html)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – pktmon](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – pagina del manuale](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [IETF RFC 1421 – Privacy Enhancement for Internet Electronic Mail](https://datatracker.ietf.org/doc/html/rfc1421)
