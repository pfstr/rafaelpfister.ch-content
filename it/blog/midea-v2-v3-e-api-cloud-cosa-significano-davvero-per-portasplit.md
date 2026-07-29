---
title: "Midea V2, V3 e API cloud: cosa significano davvero per PortaSplit"
navTitle: "API cloud Midea V2"
description: "Il protocollo locale del dispositivo, gli endpoint privati dell'app e l'API ufficiale per partner utilizzano nomi di versione simili. L'analisi delle fonti distingue questi livelli e contestualizza l'avviso di dismissione."
date: "2026-07-25"
kategorie: "Home Assistant e IoT"
timeToRead: "11 min di lettura"
themen:
  - "smart-home-iot"
related:
  - "midea-portasplit-home-assistant"
  - "midea-portasplit-home-assistant-einrichten"
draft: false
slug: "midea-v2-v3-e-api-cloud-cosa-significano-davvero-per-portasplit"
translationOf: "midea-v2-cloud-api-portasplit-home-assistant"
url: "https://rafaelpfister.ch/it/blog/midea-v2-v3-e-api-cloud-cosa-significano-davvero-per-portasplit"
---

Nel contesto di Midea PortaSplit, «V2» indica diverse cose indipendenti tra loro. Esistono un protocollo locale del dispositivo V2, numeri di versione negli endpoint privati dell'app e un'API ufficiale cloud-to-cloud V2 per i partner. Chi equipara questi livelli giunge inevitabilmente a conclusioni errate sul controllo locale.

Il progetto `Midea AC LAN` avverte nella sua [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) che le precedenti interfacce token sarebbero state chiuse e sostituite da un'API V2 basata sul cloud. Un esame delle discussioni, del codice attuale e della documentazione ufficiale Midea restituisce un quadro più differenziato:

> Esiste un'API ufficiale Midea cloud-to-cloud V2. Tuttavia, non è identica all'interfaccia token utilizzata da Home Assistant, né al protocollo locale del dispositivo V2 o V3. Non è documentata una dismissione annunciata ufficialmente del controllo locale di PortaSplit con una data concreta. Nel giugno 2026 è stato inoltre dimostrato che la presunta API token SmartHome dismessa continuava a funzionare: la precedente richiesta della libreria della community era semplicemente incompleta.

Questo articolo è aggiornato al 25 luglio 2026.

## Perché la classificazione precedente deve essere corretta

Nel [primo articolo sulla questione dei token cloud](/blog/midea-portasplit-home-assistant) avevo riportato, in sostanza, l'avvertimento del progetto `Midea AC LAN` come una dismissione annunciata delle interfacce cloud. Ciò corrispondeva al testo della README del progetto, ma era formulato in modo troppo categorico come affermazione di fatto.

L'avvertimento resta rilevante come indicazione di rischio. Non è però una roadmap Midea pubblicata. Soprattutto, nel frattempo è disponibile nuovo materiale tecnico che mette in discussione una parte sostanziale dell'interpretazione precedente.

## Come funziona il controllo locale di PortaSplit

L'integrazione Home Assistant `Midea Smart AC` descrive esplicitamente la propria architettura come controllo locale. Sui dispositivi V3 più recenti, il cloud Midea viene utilizzato solo durante la configurazione per ottenere un token e una chiave specifici del dispositivo. In seguito, l'integrazione memorizza entrambi i valori localmente e non necessita di ulteriori connessioni cloud per il controllo effettivo. Il progetto lo documenta in [«Note On Cloud Usage"](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

In forma semplificata, il flusso è il seguente:

```text
Einrichtung:

Home Assistant
    │
    ├── Anmeldung an einer Midea-Cloud
    ├── Abruf von Geräte-ID, Token und Key
    └── lokale Speicherung der Zugangsdaten

Normalbetrieb:

Home Assistant
    │
    └── lokale TCP-Verbindung zur PortaSplit
```

Per i dispositivi V3 configurati manualmente, `Midea Smart AC` richiede ID del dispositivo, indirizzo IP, porta, token e chiave. La porta standard documentata è `6444/TCP`; token e chiave sono indicati rispettivamente come 128 e 64 caratteri esadecimali. Queste informazioni sono disponibili nella [documentazione sulla configurazione manuale](https://github.com/mill1000/midea-ac-py#manual-configuration).

Nel tracker delle issue di `Midea AC LAN`, una PortaSplit è stata ad esempio riconosciuta come dispositivo di tipo `0xAC`, modello `00000Q1D` e versione del protocollo 3. Lo stesso utente ha poi potuto aggiungerla a Home Assistant tramite NetHome Plus. Il caso concreto è documentato in [Issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607).

La separazione decisiva è questa:

- Il servizio cloud viene utilizzato per ottenere le credenziali locali.
- Il controllo successivo avviene direttamente nella LAN.
- Un guasto del servizio token impedisce quindi soprattutto nuove configurazioni.
- Non interrompe automaticamente una connessione locale già configurata.

Quest'ultimo punto corrisponde anche alla descrizione esplicita di [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## Da dove proviene l'avviso di dismissione

Il testo di avviso attualmente visibile è stato aggiunto alla documentazione il 19 maggio 2025 con [Pull Request #578](https://github.com/wuwentao/midea_ac_lan/pull/578).

In sintesi, la motivazione è la seguente:

- I token locali non avrebbero una data di scadenza.
- Vari progetti Home Assistant utilizzerebbero una crittografia dell'app riprodotta o estratta.
- Da ciò deriverebbe un problema di sicurezza.
- Midea chiuderebbe quindi gradualmente i precedenti servizi token.
- A lungo termine, il controllo locale V1 sarebbe destinato a essere sostituito da un'API V2 basata sul cloud.

Nel luglio 2025, la documentazione è stata nuovamente modificata tramite [Pull Request #639](https://github.com/wuwentao/midea_ac_lan/pull/639). Al posto del cloud SmartHome, veniva ora indicato NetHome Plus come fonte token temporaneamente utilizzata. L'avviso di dismissione vero e proprio è rimasto.

La discussione sottostante è tuttavia formulata con maggiore cautela rispetto alla README.

Nel [commento del maintainer di Midea-AC-LAN](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) si afferma in sostanza che NetHome Plus potrebbe essere solo una soluzione temporanea e che, secondo la sua interpretazione, Midea dispone di un nuovo servizio V2 interamente basato sul cloud.

Il maintainer di `midea-msmart` ha risposto di aver anch'egli ipotizzato l'esistenza di una nuova API V2, ma di poterla analizzare solo limitatamente per mancanza di dispositivi Midea propri. Ciò è riportato nel [commento di risposta diretto](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

Il quadro delle fonti è quindi più chiaro:

- L'avvertimento proviene da sviluppatori esperti della community.
- Si basa su modifiche osservate e sulla loro valutazione tecnica.
- Uno dei maintainer definisce esplicitamente la migrazione V2 come la propria interpretazione.
- L'altro parla di un'ipotesi.
- Né la pull request né la discussione rimandano a un annuncio ufficiale Midea di dismissione o a una data.

Ciò non rende l'avvertimento privo di valore. Lo rende però un'analisi del rischio, non una roadmap del produttore confermata.

## Il nuovo riscontro decisivo del giugno 2026

Il 15 giugno 2026 è stata integrata nella libreria `midea-local` una correzione che modifica sostanzialmente l'interpretazione precedente.

Il punto di partenza era l'errore:

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

Questo errore si era verificato durante la richiesta di token e chiave tramite il cloud SmartHome. Il login e l'elenco dei dispositivi continuavano a funzionare, ma la chiamata di `/v1/iot/secure/getToken` veniva respinta.

Inizialmente ciò sembrava indicare un'interfaccia dismessa o resa inutilizzabile. L'analisi della richiesta dell'app ufficiale SmartHome ha però mostrato una causa diversa: oltre a `udpid`, l'app inviava anche il campo `applianceCodes`. La libreria della community non inviava questo campo.

La richiesta corretta contiene ora:

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

Lo sviluppatore ha testato la modifica con un account SmartHome reale e quattro climatizzatori V3 di tipo `0xAC`:

- Senza `applianceCodes`, il server rispondeva con l'errore 3004.
- Con `applianceCodes`, forniva token e chiavi validi.
- I valori restituiti funzionavano poi per l'autenticazione locale V3.

L'indagine completa, i risultati dei test e il diff del codice sono documentati nella [`midea-local` Pull Request #470](https://github.com/midea-lan/midea-local/pull/470). Il commit immutabile associato è [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

Anche nel codice sorgente attuale continua a essere utilizzato esattamente questo endpoint:

```text
/v1/iot/secure/getToken
```

Inoltre, ora viene inviato anche `applianceCodes`. Questo è verificabile direttamente nel [`midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) attuale.

La versione attuale di `Midea AC LAN` integra `midea-local==6.11.0` e continua a dichiararsi un'integrazione `local_push`. Entrambe le informazioni sono presenti nel [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json) attuale.

L'affermazione generica secondo cui l'API token SmartHome sarebbe stata chiusa è quindi confutata, almeno per gli account e i dispositivi testati nel giugno 2026. La formulazione corretta sarebbe:

> La precedente richiesta token non funzionava più dopo una modifica del formato della richiesta previsto. Dopo l'adeguamento al formato utilizzato dall'app ufficiale, lo stesso endpoint V1 ha nuovamente fornito credenziali locali valide.

Non sono per questo esclusi differenze regionali, account diversi o tipi di dispositivo non supportati. Ma non si trattava evidentemente di una dismissione globale.

## Perché «V2» viene così facilmente frainteso in questo contesto

Nell'ecosistema Midea vengono utilizzate almeno tre denominazioni di versione indipendenti tra loro.

| Termine | Significato |
| --- | --- |
| Protocollo locale V2/V3 | Generazione della comunicazione diretta tra integrazione e dispositivo |
| Endpoint dell'app V1/V2 | Numero di versione di un singolo endpoint HTTP nel backend delle app Midea |
| API cloud-to-cloud V2 | API ufficiale per partner destinata a terze parti autorizzate |

### V2 e V3 locali

Nel protocollo locale del dispositivo, V2 e V3 indicano la generazione di comunicazione del dispositivo. I dispositivi V3 più recenti richiedono token e chiave per l'autenticazione locale. `Midea Smart AC` documenta questo requisito nella sua [guida alla configurazione](https://github.com/mill1000/midea-ac-py#manual-configuration).

Questa versione del protocollo non ha nulla a che fare con l'API ufficiale cloud-to-cloud V2.

### V1 e V2 negli URL dell'app

Anche all'interno della stessa app possono essere usati contemporaneamente endpoint con numeri di versione diversi. Un `/v2/` nel percorso URL non significa quindi che l'intera piattaforma sia stata convertita a una nuova architettura.

Il codice attuale di `midea-local` continua a utilizzare [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) per token e chiave. Altre funzioni possono comunque trovarsi in percorsi con versioni diverse.

### API ufficiale cloud-to-cloud V2

Midea documenta effettivamente una [API ufficiale cloud-to-cloud V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Questa utilizza, tra l'altro:

- OAuth 2.0
- `client_id` e `client_secret`
- access token e refresh token a breve durata
- firme HMAC-SHA256
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- interrogazioni di stato e comandi di controllo basati sul cloud

Si tratta di un'interfaccia per partner controllata. Il necessario `client_secret` viene assegnato da Midea a un fornitore terzo. Un normale proprietario di PortaSplit non lo ottiene semplicemente tramite il proprio account MSmartHome. I requisiti e le regole di firma sono descritti nella [documentazione ufficiale V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Inoltre, questa API non è nata soltanto nel 2025. La documentazione contiene esempi di richieste con timestamp del 2018 e un commento Java del 18 aprile 2019. L'interfaccia V2 per partner esisteva dunque già molto prima dell'avvertimento in `Midea AC LAN`.

## Midea sostituisce effettivamente un'API V1, ma un'altra

Midea gestisce anche una vecchia interfaccia ufficiale cloud-to-cloud sotto `/v1/open/...`. La relativa documentazione riporta esplicitamente che non è più raccomandata, che potrebbe essere dismessa in futuro e che occorre utilizzare la nuova documentazione V2. Questo è indicato nella [documentazione Midea della vecchia API cloud-to-cloud](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

Questo avviso rappresenta una reale migrazione ufficiale da V1 a V2. Riguarda però gli endpoint per partner:

```text
/v1/open/...
           ↓
/v2/open/...
```

La richiesta token utilizzata dalle librerie Home Assistant è invece:

```text
/v1/iot/secure/getToken
```

E la connessione locale di PortaSplit non passa poi più attraverso un simile URL cloud, ma direttamente nella rete domestica.

Equiparare le tre interfacce solo in base al numero di versione «V1» non sarebbe quindi tecnicamente giustificato.

## Esiste già un'integrazione Home Assistant completamente basata sul cloud?

Con [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud) esiste ormai un'integrazione della community che controlla i dispositivi Midea tramite il cloud invece che direttamente tramite la LAN.

Anche questo, tuttavia, non dimostra che l'API V2 ufficiale per partner abbia già sostituito il controllo locale. Il codice sorgente attuale di `Midea Auto Cloud` utilizza tra l'altro:

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

Questi endpoint sono visibili nel [`core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py) attuale.

L'integrazione replica dunque funzioni private dell'app o del cloud consumer. Non utilizza semplicemente l'interfaccia per partner documentata `/v2/open/...`.

Esiste quindi già un'alternativa basata sul cloud. Ma comporta anche le normali dipendenze di un'integrazione cloud: accesso a Internet, account utente funzionante, server Midea disponibili ed endpoint privati ancora compatibili.

## Cosa significa concretamente per i proprietari di PortaSplit?

### Controllo locale già configurato

Per una PortaSplit già configurata, la situazione è relativamente tranquilla. `Midea Smart AC` memorizza token e chiave localmente dopo la configurazione e, secondo la propria [documentazione cloud](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage), non richiede una connessione cloud per il controllo successivo.

La dismissione della sola richiesta token non interromperebbe quindi automaticamente la connessione locale esistente.

### Nuova configurazione o ripristino

Il rischio è maggiore in caso di:

- una nuova installazione Home Assistant
- passaggio a un'altra integrazione
- backup perso o danneggiato
- sostituzione del modulo Wi-Fi
- modifiche all'associazione del dispositivo
- un nuovo abbinamento, se comporta una modifica delle credenziali del dispositivo

In questi casi, l'integrazione deve ottenere nuovamente token e chiave, oppure l'utente deve inserirli manualmente. Il fatto che `Midea Smart AC` supporti una configurazione manuale è descritto nella sua [documentazione di configurazione](https://github.com/mill1000/midea-ac-py#manual-configuration).

Non è documentato ufficialmente se un ripristino di fabbrica o un nuovo abbinamento generi necessariamente nuove credenziali per ogni PortaSplit; non dovrebbe pertanto essere affermato in modo generalizzato.

### Una vera dismissione del controllo LAN

Affinché una PortaSplit già configurata non accetti più le credenziali memorizzate localmente, dovrebbe cambiare anche il comportamento del dispositivo o del modulo Wi-Fi, ad esempio tramite un nuovo firmware o una procedura di autenticazione modificata.

La semplice dismissione dell'endpoint cloud `/v1/iot/secure/getToken` non rimuove automaticamente le credenziali già presenti nel dispositivo e in Home Assistant. Ciò deriva dalla separazione tra recupero cloud una tantum e successivo controllo LAN documentata da [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Una futura modifica del dispositivo di questo tipo è tecnicamente possibile. Tuttavia, non ho trovato nella documentazione Midea pubblicamente accessibile un annuncio concreto o una data di dismissione specifica per PortaSplit.

## Cosa continuerei a raccomandare

Nonostante le conclusioni ridimensionate, un backup resta sensato.

Per i dispositivi V3, `Midea AC LAN` raccomanda esplicitamente di salvare la configurazione JSON generata al di fuori di HAOS. La raccomandazione attuale è riportata direttamente nella [README del progetto](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

Si applicano le seguenti regole:

- Trattare token e chiave come password.
- Non caricare il file JSON in un repository Git pubblico.
- Non pubblicare log di debug non oscurati.
- Crittografare il backup.
- Creare inoltre un backup completo di Home Assistant.
- Verificare il funzionamento attuale prima di aggiornamenti firmware e integrazioni.
- Testare nuovamente il controllo locale dopo gli aggiornamenti.

Un backup è una protezione ragionevole contro modifiche cloud, problemi dell'integrazione ed errori propri. Non è però un'indicazione che una dismissione sia imminente. Come configurare correttamente una PortaSplit e proteggerla nella rete domestica è illustrato nella [parte pratica sulla configurazione](/blog/midea-portasplit-home-assistant-einrichten).

## Valutazione sulla base delle prove disponibili

L'avvertimento di `Midea AC LAN` va preso sul serio, ma inquadrato correttamente.

Documenta un rischio plausibile a lungo termine: Midea potrebbe considerare un problema di sicurezza i token locali senza scadenza, limitare ulteriormente l'ottenimento di tali token o vincolare maggiormente al cloud i dispositivi futuri.

Non è invece comprovata una dismissione ufficialmente annunciata e programmata del controllo locale di PortaSplit.

Lo stato tecnico attuale mostra addirittura il contrario di una dismissione già avvenuta: nel giugno 2026, l'endpoint token V1 ancora utilizzato ha fornito credenziali valide dopo che la richiesta era stata adeguata al formato dell'app ufficiale SmartHome. La correzione corrispondente fa oggi parte della libreria utilizzata da `Midea AC LAN`.

Esiste anche l'API ufficiale Midea cloud-to-cloud V2. Ma è un'interfaccia per partner più datata, ad accesso limitato, e non è automaticamente il successore del protocollo locale PortaSplit.

La conclusione sobria è quindi:

> Creare un backup, monitorare le integrazioni e tenere a mente le dipendenze dal cloud, ma non abbandonare prematuramente il controllo locale di PortaSplit sulla base di un'ipotesi di dismissione non confermata.

## Fonti

1.  [Midea AC LAN: README attuale e avviso di dismissione](https://github.com/wuwentao/midea_ac_lan#1-important-notice): testo dell'avviso, raccomandazione di backup e distinzione tra dispositivi V2 più vecchi e V3 più recenti.

2.  [Midea AC LAN PR #578 del 19 maggio 2025](https://github.com/wuwentao/midea_ac_lan/pull/578): introduzione dell'avvertimento sulla graduale dismissione dei servizi token e sulla presunta migrazione verso un'API V2 basata sul cloud.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639): passaggio della fonte token documentata a NetHome Plus.

4.  [midea-msmart Issue #201](https://github.com/mill1000/midea-msmart/issues/201): discussione sulla richiesta token SmartHome difettosa e sull'uso temporaneo di NetHome Plus.

5.  [Commento del maintainer di Midea-AC-LAN sulla presunta migrazione V2](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457): indica esplicitamente l'affermazione sul nuovo cloud V2 come propria interpretazione.

6.  [Risposta del maintainer di midea-msmart](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109): descrive l'esistenza di una nuova API V2 come un'ipotesi e segnala le limitate possibilità di reverse engineering.

7.  [midea-local PR #470 del 15 giugno 2026](https://github.com/midea-lan/midea-local/pull/470): analisi dell'errore 3004, acquisizione della richiesta dell'app ufficiale, aggiunta di `applianceCodes` e test riuscito con quattro climatizzatori V3.

8.  [Commit immutabile della correzione SmartHome-getToken](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5): diff esatto del codice della correzione integrata.

9.  [Codice cloud midea-local attuale](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py): endpoint ancora utilizzato `/v1/iot/secure/getToken` e campo di richiesta attuale `applianceCodes`.

10.  [Manifest attuale di Midea AC LAN](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json): versione utilizzata di `midea-local` e classificazione come integrazione push locale.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py): documentazione del controllo locale, del recupero cloud una tantum per dispositivi V3 e della configurazione manuale con token e chiave.

12.  [Midea AC LAN Issue #607 su PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607): esempio concreto di PortaSplit con dispositivo di tipo `0xAC`, modello `00000Q1D`, versione del protocollo 3 e configurazione riuscita tramite NetHome Plus.

13.  [API ufficiale Midea cloud-to-cloud V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html): OAuth2, ID client, segreto client, access e refresh token, procedura di firma ed endpoint `/v2/open/...`.

14.  [API ufficiale Midea cloud-to-cloud V1](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html): avviso ufficiale che la vecchia interfaccia per partner `/v1/open/...` non è più raccomandata e potrebbe essere dismessa in futuro.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) e [codice cloud attuale](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py): integrazione della community per il controllo cloud completo e gli endpoint privati dell'app V1 effettivamente utilizzati.
