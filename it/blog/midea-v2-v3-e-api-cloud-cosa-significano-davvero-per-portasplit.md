---
title: "Midea V2, V3 e API cloud: cosa significa davvero per PortaSplit"
navTitle: "API cloud Midea V2"
description: "Il protocollo locale del dispositivo, gli endpoint privati dell'app e l'API ufficiale per partner utilizzano nomi di versione simili. L'analisi delle fonti distingue questi livelli e contestualizza l'avviso di dismissione."
date: "2026-07-25"
kategorie: "Home Assistant e IoT"
timeToRead: "11 min di lettura"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-absichern
  - midea-portasplit-home-assistant
draft: false
slug: "midea-v2-v3-e-api-cloud-cosa-significano-davvero-per-portasplit"
translationOf: "midea-v2-cloud-api-portasplit-home-assistant"
translationId: article-f504b2af00493864
translationModel: gpt-5.6-terra
translatedAt: 2026-10-09T10:44:51.022Z
translationReview: automatic
translationSourceHash: fb685f86fac63fa4efa6770539bc7b23d1376fc51c6cca5993d68a517872975f
image: ../images/midea-portasplit-home-assistant/portasplit-dashboard.png
url: https://rafaelpfister.ch/it/blog/midea-v2-v3-e-api-cloud-cosa-significano-davvero-per-portasplit
---

Nel contesto di Midea PortaSplit, «V2» indica diverse cose tra loro indipendenti. Esistono un protocollo locale V2 del dispositivo, numeri di versione negli endpoint privati dell'app e un'API V2 cloud-to-cloud ufficiale per partner. Chi equipara questi livelli giunge inevitabilmente a conclusioni errate sul controllo locale.

Il progetto `Midea AC LAN` avverte nella sua [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) che le precedenti interfacce basate su token sarebbero state chiuse e sostituite da un'API V2 basata sul cloud. Un esame delle discussioni, del codice attuale e della documentazione ufficiale Midea restituisce un quadro più articolato:

> Esiste un'API V2 cloud-to-cloud ufficiale di Midea. Tuttavia, non è identica all'interfaccia basata su token utilizzata da Home Assistant, né al protocollo locale V2 o V3 del dispositivo. Non è documentata una dismissione ufficialmente annunciata del controllo locale della PortaSplit con una data concreta. Nel giugno 2026 è stato inoltre dimostrato che la presunta API SmartHome basata su token dismessa continuava a funzionare: la precedente richiesta della libreria della community era semplicemente incompleta.

Questa è la parte 3 della serie; [parte 1](/blog/midea-portasplit-home-assistant) descrive la configurazione fino alla dashboard, [parte 2](/blog/midea-portasplit-home-assistant-absichern) la protezione di token, key e rete domestica. Questo articolo è aggiornato al 25 luglio 2026.

![Dashboard di Home Assistant della Midea PortaSplit in modalità raffreddamento: indicatori in alto, termostato a 22 °C, andamenti di temperatura ambiente, assorbimento di potenza, energia giornaliera, frequenza del compressore, funzionamento del compressore e velocità della ventola, seguiti da valori tecnici e stato.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## Perché la precedente classificazione deve essere corretta

In una versione precedente dell'[articolo su token e key](/blog/midea-portasplit-home-assistant-absichern) avevo riportato l'avviso del progetto `Midea AC LAN` come l'annunciata dismissione delle interfacce cloud. Ciò corrispondeva al testo della README del progetto, ma era formulato in modo troppo categorico come affermazione di fatto.

L'avviso rimane rilevante come segnalazione di rischio. Tuttavia, non è una roadmap Midea pubblicata. Soprattutto, nel frattempo è disponibile nuovo materiale tecnico che mette in discussione una parte sostanziale dell'interpretazione precedente.

## Come funziona il controllo locale di PortaSplit

L'integrazione Home Assistant `Midea Smart AC` descrive esplicitamente la propria architettura come controllo locale. Nei dispositivi V3 più recenti, il cloud Midea viene utilizzato solo durante la configurazione per ottenere un token e una key specifici del dispositivo. In seguito, l'integrazione salva entrambi i valori localmente e non necessita di ulteriori connessioni cloud per il controllo effettivo. Il progetto lo documenta in ["Note On Cloud Usage"](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

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

Per i dispositivi V3 configurati manualmente, `Midea Smart AC` richiede ID dispositivo, indirizzo IP, porta, token e key. La porta standard documentata è `6444/TCP`; token e key sono indicati rispettivamente come 128 e 64 caratteri esadecimali. Queste informazioni sono riportate nella [documentazione della configurazione manuale](https://github.com/mill1000/midea-ac-py#manual-configuration).

Nel tracker delle issue di `Midea AC LAN`, una PortaSplit è stata ad esempio rilevata come tipo di dispositivo `0xAC`, modello `00000Q1D` e versione del protocollo 3. Lo stesso utente ha poi potuto aggiungerla a Home Assistant tramite NetHome Plus. Il caso concreto è documentato in [Issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607).

La distinzione decisiva è:

- Il servizio cloud viene utilizzato per ottenere le credenziali locali.
- Il controllo successivo avviene direttamente nella LAN.
- Un guasto del servizio token impedisce quindi soprattutto nuove configurazioni.
- Non interrompe automaticamente una connessione locale già configurata.

Quest'ultimo punto corrisponde anche alla descrizione esplicita di [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## Da dove proviene l'avviso di dismissione

Il testo dell'avviso oggi visibile è stato aggiunto alla documentazione il 19 maggio 2025 con [Pull Request #578](https://github.com/wuwentao/midea_ac_lan/pull/578).

In sintesi, la motivazione è la seguente:

- I token locali non avrebbero una data di scadenza.
- Diversi progetti Home Assistant userebbero crittografia dell'app emulata o estratta.
- Ne deriverebbe un problema di sicurezza.
- Midea chiuderebbe quindi gradualmente i precedenti servizi token.
- A lungo termine, il controllo locale V1 dovrebbe essere soppiantato da un'API V2 basata sul cloud.

Nel luglio 2025 la documentazione è stata nuovamente modificata tramite [Pull Request #639](https://github.com/wuwentao/midea_ac_lan/pull/639). Al posto del cloud SmartHome veniva ora indicato NetHome Plus come fonte temporanea dei token. L'avviso di dismissione vero e proprio rimaneva.

La discussione sottostante è tuttavia formulata con maggiore cautela rispetto alla README.

Nel [commento del maintainer di Midea AC LAN](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) si afferma sostanzialmente che NetHome Plus potrebbe essere solo una soluzione temporanea e che, secondo la sua comprensione, Midea dispone di un nuovo servizio V2 completamente basato sul cloud.

Il maintainer di `midea-msmart` ha risposto di aver anch'egli supposto l'esistenza di una nuova API V2, ma di poterla analizzare solo in misura limitata per mancanza di dispositivi Midea propri. Questo è riportato nel [commento di risposta diretto](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

La situazione delle fonti è quindi più chiara:

- L'avviso proviene da esperti sviluppatori della community.
- Si basa su cambiamenti osservati e sulla loro valutazione tecnica.
- Uno dei maintainer definisce esplicitamente la migrazione a V2 come la propria interpretazione.
- L'altro parla di un'ipotesi.
- Né la pull request né la discussione collegano un annuncio ufficiale Midea di dismissione o una data.

Questo non rende l'avviso privo di valore. Lo rende però un'analisi del rischio, non una roadmap del produttore confermata.

## La decisiva nuova scoperta del giugno 2026

Il 15 giugno 2026 è stata integrata nella libreria `midea-local` una correzione che modifica sostanzialmente l'interpretazione precedente.

Il punto di partenza era l'errore:

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

Questo errore si era verificato durante la richiesta di token e key tramite il cloud SmartHome. Il login e l'elenco dei dispositivi continuavano a funzionare, ma la chiamata a `/v1/iot/secure/getToken` veniva respinta.

Inizialmente ciò sembrava indicare un'interfaccia dismessa o resa inutilizzabile. Un'analisi della richiesta dell'app ufficiale SmartHome ha però mostrato un'altra causa: oltre a `udpid`, l'app inviava il campo `applianceCodes`. La libreria della community non inviava questo campo.

La richiesta corretta ora contiene:

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

Lo sviluppatore ha testato la modifica con un vero account SmartHome e quattro condizionatori V3 di tipo `0xAC`:

- Senza `applianceCodes`, il server rispondeva con l'errore 3004.
- Con `applianceCodes`, forniva token e key validi.
- I valori restituiti funzionavano poi per l'autenticazione locale V3.

L'indagine completa, i risultati dei test e il diff del codice sono documentati in [`midea-local` Pull Request #470](https://github.com/midea-lan/midea-local/pull/470). Il commit immutabile associato è [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

Anche il codice sorgente attuale continua a utilizzare esattamente questo endpoint:

```text
/v1/iot/secure/getToken
```

Inoltre, ora viene inviato anche `applianceCodes`. Ciò è direttamente verificabile nell'attuale [`midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py).

La versione attuale di `Midea AC LAN` integra `midea-local==6.11.0` e continua a definirsi un'integrazione `local_push`. Entrambi gli aspetti sono indicati nell'attuale [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json).

L'affermazione generalizzata secondo cui l'API SmartHome basata su token sarebbe stata chiusa è quindi confutata, almeno per gli account e i dispositivi testati nel giugno 2026. Più correttamente:

> La precedente richiesta di token non ha più funzionato dopo una modifica del formato di richiesta previsto. Dopo l'adeguamento al formato utilizzato dall'app ufficiale, lo stesso endpoint V1 ha nuovamente fornito credenziali locali valide.

Non sono esclusi differenze regionali, account diversi o tipi di dispositivo non supportati. Ma evidentemente non si è trattato di una dismissione globale.

## Perché qui «V2» viene così facilmente frainteso

Nell'ambiente Midea vengono utilizzate almeno tre denominazioni di versione tra loro indipendenti.

| Termine | Significato |
| --- | --- |
| Protocollo locale V2/V3 | Generazione della comunicazione diretta tra integrazione e dispositivo |
| Endpoint dell'app V1/V2 | Numero di versione di un singolo endpoint HTTP nel backend delle app Midea |
| API V2 cloud-to-cloud | API ufficiale per partner per aziende terze autorizzate |

### V2 e V3 locali

Nel protocollo locale del dispositivo, V2 o V3 identifica la generazione di comunicazione del dispositivo. I dispositivi V3 più recenti richiedono token e key per l'autenticazione locale. `Midea Smart AC` documenta questo requisito nella sua [guida alla configurazione](https://github.com/mill1000/midea-ac-py#manual-configuration).

Questa versione del protocollo non ha nulla a che fare con l'API V2 cloud-to-cloud ufficiale.

### V1 e V2 negli URL delle app

Anche nella stessa app possono essere utilizzati contemporaneamente endpoint con numeri di versione diversi. Un `/v2/` nel percorso URL non significa quindi che l'intera piattaforma sia stata migrata a una nuova architettura.

L'attuale codice di `midea-local` continua a utilizzare per token e key [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py). Altre funzioni possono comunque trovarsi in percorsi con versioni diverse.

### API V2 cloud-to-cloud ufficiale

Midea documenta effettivamente un'[API V2 cloud-to-cloud ufficiale](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Questa utilizza tra l'altro:

- OAuth 2.0
- `client_id` e `client_secret`
- Access token e refresh token a breve durata
- firme HMAC-SHA256
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- interrogazioni dello stato e comandi di controllo basati sul cloud

Si tratta di un'interfaccia per partner controllata. Il necessario `client_secret` viene assegnato da Midea a un fornitore terzo. Un normale proprietario di una PortaSplit non lo riceve semplicemente tramite il proprio account MSmartHome. I requisiti e le regole di firma sono descritti nella [documentazione ufficiale V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Inoltre, questa API non è nata solo nel 2025. La documentazione contiene esempi di richieste con timestamp del 2018 e un commento Java del 18 aprile 2019. L'interfaccia partner V2 esisteva dunque già molto prima dell'avviso in `Midea AC LAN`.

## Midea sostituisce effettivamente un'API V1, ma un'altra

Midea gestisce anche una precedente interfaccia cloud-to-cloud ufficiale sotto `/v1/open/...`. La relativa documentazione reca esplicitamente l'avviso che non è più consigliata, potrebbe essere dismessa in futuro e che dovrebbe essere utilizzata la nuova documentazione V2. Ciò è indicato nella [documentazione Midea della vecchia API cloud-to-cloud](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

Questo avviso è una vera migrazione ufficiale da V1 a V2. Riguarda tuttavia gli endpoint dei partner:

```text
/v1/open/...
           ↓
/v2/open/...
```

La richiesta di token utilizzata dalle librerie Home Assistant è invece:

```text
/v1/iot/secure/getToken
```

E la connessione locale della PortaSplit non passa poi più attraverso un simile URL cloud, bensì direttamente nella rete domestica.

Equiparare le tre interfacce solo in base al numero di versione «V1» non sarebbe quindi tecnicamente giustificato.

## Esiste già un'integrazione Home Assistant completamente basata sul cloud?

Con [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud) esiste ormai un'integrazione della community che controlla i dispositivi Midea tramite il cloud invece che direttamente tramite la LAN.

Anche questo, però, non dimostra che l'API V2 ufficiale per partner abbia già sostituito il controllo locale. Il codice sorgente attuale di `Midea Auto Cloud` utilizza tra l'altro:

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

Questi endpoint sono consultabili nell'attuale [`core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py).

L'integrazione emula quindi funzioni private dell'app o del cloud consumer. Non utilizza semplicemente l'interfaccia partner documentata `/v2/open/...`.

Esiste quindi già un'alternativa basata sul cloud. Ma comporta anche le normali dipendenze di un'integrazione cloud: accesso a Internet, account utente funzionante, server Midea disponibili ed endpoint privati ancora compatibili.

## Cosa significa concretamente per i proprietari di PortaSplit?

### Controllo locale già configurato

Per una PortaSplit già configurata, la situazione è relativamente poco critica. Dopo la configurazione, `Midea Smart AC` salva token e key localmente e, secondo la propria [documentazione cloud](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage), non necessita di una connessione cloud per il controllo successivo.

La dismissione del solo recupero dei token non interromperebbe quindi automaticamente la connessione locale esistente.

### Nuova configurazione o ripristino

Il rischio è maggiore in caso di:

- una nuova installazione di Home Assistant
- passaggio a un'altra integrazione
- backup perso o danneggiato
- sostituzione del modulo WLAN
- modifiche all'associazione del dispositivo
- un nuovo abbinamento, qualora comporti la modifica delle credenziali del dispositivo

In questi casi, l'integrazione deve recuperare nuovamente token e key oppure l'utente deve indicarli manualmente. Il fatto che `Midea Smart AC` supporti una configurazione manuale è descritto nella relativa [documentazione di configurazione](https://github.com/mill1000/midea-ac-py#manual-configuration).

Non è ufficialmente documentato se un ripristino di fabbrica o un nuovo abbinamento generino necessariamente nuove credenziali per ogni PortaSplit; non dovrebbe quindi essere affermato in modo generalizzato.

### Una vera dismissione del controllo LAN

Affinché una PortaSplit già configurata non accetti più le credenziali salvate localmente, dovrebbe inoltre cambiare il comportamento del dispositivo o del modulo WLAN, ad esempio tramite un nuovo firmware o una procedura di autenticazione modificata.

La semplice dismissione dell'endpoint cloud `/v1/iot/secure/getToken` non rimuove automaticamente le credenziali già presenti nel dispositivo e in Home Assistant. Ciò deriva dalla separazione tra recupero cloud una tantum e successivo controllo LAN, documentata da [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Un simile futuro cambiamento del dispositivo è tecnicamente possibile. Tuttavia, non ho trovato negli atti Midea pubblicamente accessibili un annuncio concreto o una data di dismissione specifica per PortaSplit.

## Cosa continuerei a raccomandare

Nonostante le conclusioni relativizzanti, un backup rimane utile.

Per i dispositivi V3, `Midea AC LAN` raccomanda espressamente di salvare la configurazione JSON generata al di fuori di HAOS. La raccomandazione attuale è riportata direttamente nella [README del progetto](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

Un backup è una protezione ragionevole contro modifiche del cloud, problemi di integrazione e propri errori, ma non indica che una dismissione sia imminente. La [parte 2](/blog/midea-portasplit-home-assistant-absichern#token-key-und-konfiguration-sichern) descrive come salvare token, key e configurazione.

## Valutazione sulla base delle prove disponibili

L'avviso di `Midea AC LAN` va preso sul serio, ma inquadrato correttamente.

Documenta un rischio plausibile a lungo termine: Midea potrebbe considerare i token locali senza scadenza un problema di sicurezza, limitare ulteriormente il recupero di tali token o vincolare più strettamente al cloud i dispositivi futuri.

Non è invece dimostrata una dismissione ufficialmente annunciata e datata del controllo locale della PortaSplit.

L'attuale situazione tecnica mostra persino il contrario di una dismissione già avvenuta: nel giugno 2026, l'endpoint V1 per token ancora utilizzato ha fornito credenziali valide dopo che la richiesta era stata adeguata al formato dell'app ufficiale SmartHome. La relativa correzione fa oggi parte della libreria utilizzata da `Midea AC LAN`.

Esiste anche l'API V2 cloud-to-cloud ufficiale di Midea. Tuttavia, si tratta di una più vecchia interfaccia per partner con accesso limitato e non automaticamente del successore del protocollo locale PortaSplit.

La conclusione sobria è quindi:

> Creare un backup, monitorare le integrazioni e tenere presenti le dipendenze dal cloud, ma non dare per spacciato prematuramente il controllo locale della PortaSplit sulla base di un'ipotesi di dismissione non confermata.

## Fonti

1.  [Midea AC LAN: README attuale e avviso di dismissione](https://github.com/wuwentao/midea_ac_lan#1-important-notice): testo dell'avviso, raccomandazione per il backup e distinzione tra dispositivi V2 più vecchi e V3 più recenti.

2.  [Midea AC LAN PR #578 del 19 maggio 2025](https://github.com/wuwentao/midea_ac_lan/pull/578): introduzione dell'avviso sulla graduale dismissione dei servizi token e sulla presunta migrazione a un'API V2 basata sul cloud.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639): passaggio della fonte documentata dei token a NetHome Plus.

4.  [midea-msmart Issue #201](https://github.com/mill1000/midea-msmart/issues/201): discussione sulla richiesta difettosa di token SmartHome e sull'uso temporaneo di NetHome Plus.

5.  [Commento del maintainer di Midea AC LAN sulla presunta migrazione V2](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457): qualifica esplicitamente l'affermazione sul nuovo cloud V2 come sua interpretazione personale.

6.  [Risposta del maintainer di midea-msmart](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109): descrive l'esistenza di una nuova API V2 come un'ipotesi e richiama le limitate possibilità di reverse engineering.

7.  [midea-local PR #470 del 15 giugno 2026](https://github.com/midea-lan/midea-local/pull/470): analisi dell'errore 3004, cattura della richiesta dell'app ufficiale, aggiunta di `applianceCodes` e test riuscito con quattro condizionatori V3.

8.  [Commit immutabile della correzione SmartHome-getToken](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5): diff esatto del codice della correzione integrata.

9.  [Codice cloud midea-local attuale](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py): endpoint `/v1/iot/secure/getToken` ancora utilizzato e campo di richiesta attuale `applianceCodes`.

10.  [Manifest attuale di Midea AC LAN](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json): versione utilizzata di `midea-local` e classificazione come integrazione push locale.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py): documentazione del controllo locale, del recupero cloud una tantum per i dispositivi V3 e della configurazione manuale con token e key.

12.  [Midea AC LAN Issue #607 sulla PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607): esempio concreto di PortaSplit con tipo di dispositivo `0xAC`, modello `00000Q1D`, versione del protocollo 3 e configurazione riuscita tramite NetHome Plus.

13.  [API V2 cloud-to-cloud ufficiale Midea](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html): OAuth2, ID client, client secret, access e refresh token, procedura di firma ed endpoint `/v2/open/...`.

14.  [API V1 cloud-to-cloud ufficiale Midea](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html): avviso ufficiale che la vecchia interfaccia partner `/v1/open/...` non è più raccomandata e potrebbe essere dismessa in futuro.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) e [codice cloud attuale](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py): integrazione della community per il controllo completo via cloud e gli endpoint privati V1 dell'app effettivamente utilizzati.
