---
title: "Midea V2, V3 e API cloud: cosa significa realmente per PortaSplit"
navTitle: "API cloud Midea V2"
description: "Il protocollo locale del dispositivo, gli endpoint privati dell'app e l'API ufficiale per partner usano nomi di versione simili. L'analisi delle fonti separa questi livelli e contestualizza l'avviso di dismissione."
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
translatedAt: 2026-10-31T03:32:45.152Z
translationReview: automatic
translationSourceHash: 551c9e10e7e9c6a84b6e439dd388d775dae5246f96cd6c71e46c6065e84c5fd7
image: ../images/midea-portasplit-home-assistant/portasplit-dashboard.png
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/midea-v2-v3-e-api-cloud-cosa-significano-davvero-per-portasplit
---

Nel contesto di Midea PortaSplit, «V2» indica diverse cose indipendenti tra loro. Esistono un protocollo locale V2 del dispositivo, numeri di versione negli endpoint privati dell'app e un'API V2 cloud-to-cloud ufficiale per i partner. Chi equipara questi livelli giunge inevitabilmente a conclusioni errate sul controllo locale.

Il progetto `Midea AC LAN` avverte nella sua [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) che le precedenti interfacce token verrebbero chiuse e sostituite da un'API V2 basata sul cloud. Un esame delle discussioni, del codice attuale e della documentazione ufficiale di Midea restituisce un quadro più differenziato:

> Esiste un'API V2 cloud-to-cloud ufficiale di Midea. Tuttavia, non è identica all'interfaccia token utilizzata da Home Assistant né al protocollo locale V2 o V3 del dispositivo. Non è documentata una dismissione ufficialmente annunciata del controllo locale di PortaSplit con una data concreta. Nel giugno 2026 è stato inoltre dimostrato che l'API token SmartHome, apparentemente dismessa, continuava a funzionare: la precedente richiesta della libreria della community era semplicemente incompleta.

Questa è la parte 3 della serie; [la parte 1](/blog/midea-portasplit-home-assistant) descrive la configurazione fino alla dashboard, [la parte 2](/blog/midea-portasplit-home-assistant-absichern) la protezione di token, key e rete domestica. Questo articolo è aggiornato al 25 ottobre 2026.

![Dashboard di Home Assistant della Midea PortaSplit in modalità raffreddamento: indicatori in alto, termostato a 22 °C, andamenti di temperatura ambiente, potenza assorbita, energia giornaliera, frequenza del compressore, funzionamento del compressore e velocità della ventola, seguiti da valori tecnici e stato.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## Perché la precedente classificazione deve essere corretta

In una versione precedente dell'[articolo su token e key](/blog/midea-portasplit-home-assistant-absichern) avevo riportato l'avvertimento del progetto `Midea AC LAN` sostanzialmente come una dismissione annunciata delle interfacce cloud. Ciò corrispondeva al testo della README del progetto, ma era formulato in modo troppo categorico come affermazione di fatto.

L'avvertimento continua a essere rilevante come indicazione di rischio. Tuttavia, non è una roadmap Midea pubblicata. Soprattutto, nel frattempo è disponibile nuovo materiale tecnico che mette in discussione una parte sostanziale dell'interpretazione precedente.

## Come funziona il controllo locale di PortaSplit

L'integrazione Home Assistant `Midea Smart AC` descrive esplicitamente la propria architettura come controllo locale. Con i dispositivi V3 più recenti, il cloud Midea viene utilizzato solo durante la configurazione per ottenere un token e una key specifici del dispositivo. In seguito, l'integrazione memorizza entrambi i valori localmente e non necessita di ulteriori connessioni cloud per il controllo effettivo. Il progetto lo documenta nella [«Note On Cloud Usage»](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

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

Per i dispositivi V3 configurati manualmente, `Midea Smart AC` richiede ID dispositivo, indirizzo IP, porta, token e key. La porta standard documentata è `6444/TCP`; token e key sono indicati rispettivamente come 128 e 64 caratteri esadecimali. Queste informazioni sono riportate nella [documentazione sulla configurazione manuale](https://github.com/mill1000/midea-ac-py#manual-configuration).

Nel tracker delle issue di `Midea AC LAN`, una PortaSplit è stata ad esempio rilevata come tipo di dispositivo `0xAC`, modello `00000Q1D` e versione del protocollo 3. Lo stesso utente è poi riuscito ad aggiungerla a Home Assistant tramite NetHome Plus. Il caso concreto è documentato in [Issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607).

La distinzione è decisiva:

- Il servizio cloud viene utilizzato per ottenere le credenziali di accesso locali.
- Il controllo successivo avviene direttamente nella LAN.
- Un malfunzionamento del servizio token impedisce quindi soprattutto nuove configurazioni.
- Non interrompe automaticamente una connessione locale già configurata.

Quest'ultimo punto corrisponde anche alla descrizione esplicita di [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## Da dove proviene l'avvertimento di dismissione

Il testo di avviso visibile oggi è stato inserito nella documentazione il 19 maggio 2025 con [Pull Request #578](https://github.com/wuwentao/midea_ac_lan/pull/578).

La motivazione, in sintesi, è la seguente:

- I token locali non avrebbero una data di scadenza.
- Diversi progetti Home Assistant utilizzerebbero crittografia dell'app riprodotta o estratta.
- Ne deriverebbe un problema di sicurezza.
- Midea chiuderebbe quindi gradualmente i precedenti servizi token.
- A lungo termine, il controllo locale V1 dovrebbe essere soppiantato da un'API V2 basata sul cloud.

Nel luglio 2025 la documentazione è stata nuovamente modificata tramite [Pull Request #639](https://github.com/wuwentao/midea_ac_lan/pull/639). Al posto del cloud SmartHome veniva ora indicato NetHome Plus come fonte token usata temporaneamente. L'avvertimento di dismissione vero e proprio è rimasto.

Tuttavia, la discussione sottostante è formulata con maggiore cautela rispetto alla README.

Nel [commento del maintainer di Midea AC LAN](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) si afferma sostanzialmente che NetHome Plus potrebbe essere solo una soluzione temporanea e che, secondo la sua interpretazione, Midea disporrebbe di un nuovo servizio V2 interamente basato sul cloud.

Il maintainer di `midea-msmart` ha risposto di aver anch'egli ipotizzato l'esistenza di una nuova API V2, ma di poterla esaminare solo in misura limitata poiché non dispone di propri dispositivi Midea. Ciò è riportato nel [commento di risposta diretto](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

La situazione delle fonti è quindi più chiara:

- L'avvertimento proviene da sviluppatori esperti della community.
- Si basa su cambiamenti osservati e sulla loro valutazione tecnica.
- Uno dei maintainer definisce esplicitamente la migrazione V2 come una propria interpretazione.
- L'altro parla di un'ipotesi.
- Né la pull request né la discussione rimandano a un annuncio ufficiale di Midea sulla dismissione o a una data.

Questo non rende l'avvertimento privo di valore. Ma lo rende un'analisi del rischio e non una roadmap del produttore confermata.

## La decisiva nuova scoperta del giugno 2026

Il 15 giugno 2026 è stata integrata nella libreria `midea-local` una correzione che modifica in modo sostanziale l'interpretazione precedente.

Il punto di partenza era l'errore:

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

Questo errore si era verificato durante la richiesta di token e key tramite il cloud SmartHome. L'accesso e l'elenco dei dispositivi continuavano a funzionare, ma la chiamata a `/v1/iot/secure/getToken` veniva rifiutata.

Inizialmente ciò sembrava indicare un'interfaccia dismessa o resa inutilizzabile. Un'analisi della richiesta dell'app ufficiale SmartHome ha però mostrato un'altra causa: oltre a `udpid`, l'app inviava anche il campo `applianceCodes`. La libreria della community non inviava questo campo.

La richiesta corretta ora contiene:

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

Lo sviluppatore ha testato la modifica con un account SmartHome reale e quattro climatizzatori V3 di tipo `0xAC`:

- Senza `applianceCodes`, il server rispondeva con l'errore 3004.
- Con `applianceCodes`, forniva token e key validi.
- I valori restituiti funzionavano successivamente per l'autenticazione locale V3.

L'analisi completa, i risultati dei test e il diff del codice sono documentati nella [`midea-local` Pull Request #470](https://github.com/midea-lan/midea-local/pull/470). Il commit immutabile corrispondente è [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

Anche nel codice sorgente attuale viene ancora utilizzato esattamente questo endpoint:

```text
/v1/iot/secure/getToken
```

Inoltre, ora viene inviato anche `applianceCodes`. Ciò è direttamente verificabile nel [`midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) attuale.

Dal 12 agosto 2026, `Midea AC LAN` utilizza la propria libreria `midea-lan` anziché `midea-local`; la versione attuale integra `midea-lan==2026.10.0` e continua a dichiararsi come integrazione `local_push`. Entrambe le informazioni sono riportate nel [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json) attuale. Anche `midea-lan` richiede token e key al cloud SmartHome tramite `/v1/iot/secure/getToken` con `applianceCodes`, come si può verificare nel [`midealan/cloud.py`](https://github.com/wuwentao/midea-lan/blob/main/midealan/cloud.py).

L'affermazione generalizzata secondo cui l'API token SmartHome sarebbe stata chiusa è quindi confutata almeno per gli account e i dispositivi testati nel giugno 2026. La formulazione corretta sarebbe:

> La precedente richiesta di token non funzionava più dopo una modifica del formato della richiesta previsto. Dopo l'adeguamento al formato utilizzato dall'app ufficiale, lo stesso endpoint V1 ha nuovamente fornito credenziali locali valide.

Differenze regionali, account diversi o tipi di dispositivi non supportati non sono quindi esclusi. Ma evidentemente non si trattava di una dismissione globale.

Con altri cloud Midea, la situazione è cambiata da allora. Il cloud Meiju cinese risponde a `/v1/iot/secure/getToken` dall'agosto 2026 con l'errore `40404`; token e key vengono ora forniti solo dall'endpoint dell'app `/v2/iot/secure/getToken`, che richiede inoltre un `homegroupId`. Le librerie sono state adeguate di conseguenza alla fine di agosto 2026, come documentato nella [`midea-local` Pull Request #717](https://github.com/midea-lan/midea-local/pull/717). Secondo la [README di `Midea AC LAN`](https://github.com/wuwentao/midea_ac_lan#1-important-notice), al momento non è più possibile recuperare token tramite account NetHome Plus; sono utilizzabili solo account personali Meiju e SmartHome ai quali il dispositivo è associato. Anche questo cambiamento riguarda un endpoint privato dell'app e non l'API V2 cloud-to-cloud ufficiale.

## Perché «V2» è così facile da fraintendere in questo caso

Nel contesto Midea vengono usate almeno tre designazioni di versione indipendenti tra loro.

| Termine | Significato |
| --- | --- |
| Protocollo locale V2/V3 | Generazione della comunicazione diretta tra integrazione e dispositivo |
| Endpoint dell'app V1/V2 | Numero di versione di un singolo endpoint HTTP nel backend delle app Midea |
| API V2 cloud-to-cloud | API ufficiale per partner e aziende terze autorizzate |

### V2 e V3 locali

Nel protocollo locale del dispositivo, V2 o V3 indica la generazione di comunicazione del dispositivo. I dispositivi V3 più recenti richiedono token e key per l'autenticazione locale. `Midea Smart AC` documenta questo requisito nella sua [guida alla configurazione](https://github.com/mill1000/midea-ac-py#manual-configuration).

Questa versione del protocollo non ha nulla a che vedere con l'API V2 cloud-to-cloud ufficiale.

### V1 e V2 negli URL dell'app

Anche all'interno della stessa app possono essere utilizzati contemporaneamente endpoint con numeri di versione diversi. Un `/v2/` nel percorso dell'URL non significa quindi che l'intera piattaforma sia stata convertita a una nuova architettura.

Il codice attuale di `midea-local` continua a utilizzare [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) per token e key. Altre funzioni possono comunque trovarsi in percorsi con versioni diverse.

### API V2 cloud-to-cloud ufficiale

Midea documenta effettivamente un'[API V2 cloud-to-cloud ufficiale](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Questa utilizza tra l'altro:

- OAuth 2.0
- `client_id` e `client_secret`
- access token e refresh token a breve durata
- firme HMAC-SHA256
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- richieste di stato e comandi di controllo basati sul cloud

Si tratta di un'interfaccia per partner controllata. Il necessario `client_secret` viene assegnato da Midea a un fornitore terzo. Un normale proprietario di PortaSplit non lo riceve semplicemente tramite il proprio account MSmartHome. I requisiti e le regole di firma sono descritti nella [documentazione V2 ufficiale](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Inoltre, questa API non è nata soltanto nel 2025. La documentazione contiene esempi di richieste con timestamp del 2018 e un commento Java del 18 aprile 2019. L'interfaccia V2 per partner esisteva quindi già molto prima dell'avvertimento in `Midea AC LAN`.

## Midea sostituisce effettivamente un'API V1, ma è un'altra

Midea gestisce anche una precedente interfaccia cloud-to-cloud ufficiale sotto `/v1/open/...`. La sua documentazione riporta esplicitamente l'avviso che non è più raccomandata, potrebbe essere dismessa in futuro e che dovrebbe essere utilizzata la nuova documentazione V2. Questo è indicato nella [documentazione Midea della vecchia API cloud-to-cloud](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

Questo avviso costituisce una reale migrazione ufficiale da V1 a V2. Tuttavia, riguarda gli endpoint per partner:

```text
/v1/open/...
           ↓
/v2/open/...
```

La richiesta token utilizzata dalle librerie Home Assistant, invece, è:

```text
/v1/iot/secure/getToken
```

E successivamente la connessione locale di PortaSplit non passa più attraverso un simile URL cloud, ma direttamente nella rete domestica.

Equiparare le tre interfacce solo in base al numero di versione «V1» non sarebbe quindi tecnicamente giustificato.

## Esiste già un'integrazione Home Assistant completamente basata sul cloud?

Con [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud) esiste ora un'integrazione della community che controlla i dispositivi Midea tramite il cloud anziché direttamente tramite LAN.

Tuttavia, nemmeno questo dimostra che l'API V2 ufficiale per partner abbia già sostituito il controllo locale. Il codice sorgente attuale di `Midea Auto Cloud` utilizza tra l'altro:

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

Questi endpoint sono visibili nel [`core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py) attuale.

L'integrazione riproduce quindi funzioni private dell'app o del cloud consumer. Non utilizza semplicemente l'interfaccia per partner documentata `/v2/open/...`.

Esiste quindi già un'alternativa basata sul cloud. Tuttavia, comporta anche le consuete dipendenze di un'integrazione cloud: accesso a Internet, account utente funzionante, server Midea disponibili ed endpoint privati ancora compatibili.

## Cosa significa concretamente per i proprietari di PortaSplit?

### Controllo locale già configurato

Per una PortaSplit già configurata, la situazione è relativamente poco critica. `Midea Smart AC` memorizza token e key localmente dopo la configurazione e, secondo la propria [documentazione cloud](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage), non richiede una connessione cloud per il controllo successivo.

Una dismissione del solo recupero dei token non interromperebbe quindi automaticamente la connessione locale esistente.

### Nuova configurazione o ripristino

Il rischio è maggiore in caso di:

- una nuova installazione di Home Assistant
- passaggio a un'altra integrazione
- backup perso o danneggiato
- sostituzione del modulo WLAN
- modifiche all'associazione del dispositivo
- un nuovo abbinamento, se le credenziali del dispositivo cambiano durante tale procedura

In questi casi, l'integrazione deve recuperare nuovamente token e key oppure l'utente deve indicarli manualmente. Il fatto che `Midea Smart AC` supporti una configurazione manuale è descritto nella relativa [documentazione di configurazione](https://github.com/mill1000/midea-ac-py#manual-configuration).

Non è documentato ufficialmente se un ripristino alle impostazioni di fabbrica o un nuovo abbinamento generino obbligatoriamente nuove credenziali per ogni PortaSplit e non dovrebbe quindi essere affermato in modo generalizzato.

### Una reale dismissione del controllo LAN

Affinché una PortaSplit già configurata non accetti più le credenziali memorizzate localmente, dovrebbe cambiare anche il comportamento del dispositivo o del modulo WLAN, ad esempio tramite un nuovo firmware o una procedura di autenticazione modificata.

La mera dismissione dell'endpoint cloud `/v1/iot/secure/getToken` non rimuove automaticamente le credenziali già presenti nel dispositivo e in Home Assistant. Ciò deriva dalla separazione tra recupero cloud una tantum e successivo controllo LAN documentata da [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Una simile futura modifica del dispositivo è tecnicamente possibile. Tuttavia, non ho trovato negli atti Midea pubblicamente accessibili un annuncio concreto o una data di dismissione specificamente per PortaSplit.

## Cosa continuerei a raccomandare

Nonostante le conclusioni ridimensionate, un backup rimane sensato.

Per i dispositivi V3, `Midea AC LAN` raccomanda esplicitamente di salvare al di fuori di HAOS la configurazione JSON generata. La raccomandazione attuale è riportata direttamente nella [README del progetto](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

Un backup è una protezione ragionevole contro modifiche al cloud, problemi di integrazione ed errori personali, ma non è un'indicazione che una dismissione sia imminente. La [parte 2](/blog/midea-portasplit-home-assistant-absichern#token-key-und-konfiguration-sichern) descrive come salvare token, key e configurazione.

## Valutazione basata sulle prove disponibili

L'avvertimento di `Midea AC LAN` dovrebbe essere preso sul serio, ma correttamente contestualizzato.

Documenta un plausibile rischio a lungo termine: Midea potrebbe considerare i token locali senza scadenza un problema di sicurezza, limitare ulteriormente l'ottenimento di tali token o legare maggiormente i dispositivi futuri al cloud.

Non è invece dimostrata una dismissione del controllo locale di PortaSplit ufficialmente annunciata e con una data stabilita.

L'attuale stato tecnico mostra persino il contrario di una dismissione già avvenuta: nel giugno 2026, l'endpoint token V1 ancora utilizzato forniva credenziali valide dopo che la richiesta era stata adattata al formato dell'app ufficiale SmartHome. La relativa correzione fa oggi parte della libreria utilizzata da `Midea AC LAN`.

Esiste anche l'API V2 cloud-to-cloud ufficiale di Midea. Tuttavia, è un'interfaccia per partner più vecchia e ad accesso limitato, non automaticamente il successore del protocollo locale di PortaSplit.

La conclusione sobria è quindi:

> Creare un backup, monitorare le integrazioni e tenere a mente le dipendenze dal cloud, ma non liquidare prematuramente il controllo locale di PortaSplit sulla base di un'ipotesi di dismissione non confermata.

## Cronologia delle modifiche

- **25 ottobre 2026:** Aggiunto: Midea AC LAN utilizza dall'agosto 2026 la libreria midea-lan (attualmente 2026.10.0), e il cloud Meiju fornisce token e key solo tramite l'endpoint /v2/iot/secure/getToken con homegroupId. Aggiornato lo stato dell'articolo al 25 ottobre 2026 e aggiunte fonti.

## Fonti

1.  [Midea AC LAN: README attuale e avviso di dismissione](https://github.com/wuwentao/midea_ac_lan#1-important-notice): testo dell'avvertimento, raccomandazione di backup e distinzione tra dispositivi V2 meno recenti e V3 più recenti.

2.  [Midea AC LAN PR #578 del 19 maggio 2025](https://github.com/wuwentao/midea_ac_lan/pull/578): introduzione dell'avvertimento sulla graduale dismissione dei servizi token e sulla presunta migrazione a un'API V2 basata sul cloud.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639): modifica della fonte token documentata a NetHome Plus.

4.  [midea-msmart Issue #201](https://github.com/mill1000/midea-msmart/issues/201): discussione sulla richiesta token SmartHome errata e sull'utilizzo temporaneo di NetHome Plus.

5.  [Commento del maintainer di Midea AC LAN sulla presunta migrazione V2](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457): contrassegna esplicitamente l'affermazione sul nuovo cloud V2 come propria interpretazione.

6.  [Risposta del maintainer di midea-msmart](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109): descrive l'esistenza di una nuova API V2 come un'ipotesi e fa riferimento alle limitate possibilità di reverse engineering.

7.  [midea-local PR #470 del 15 giugno 2026](https://github.com/midea-lan/midea-local/pull/470): analisi dell'errore 3004, acquisizione della richiesta dell'app ufficiale, aggiunta di `applianceCodes` e test riuscito con quattro climatizzatori V3.

8.  [Commit immutabile della correzione SmartHome-getToken](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5): diff esatto del codice della correzione integrata.

9.  [Codice cloud midea-local attuale](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py): endpoint ancora utilizzato `/v1/iot/secure/getToken` e campo di richiesta attuale `applianceCodes`.

10.  [Manifest attuale di Midea AC LAN](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json): versione utilizzata di `midea-lan` e classificazione come integrazione push locale.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py): documentazione del controllo locale, del recupero cloud una tantum per dispositivi V3 e della configurazione manuale con token e key.

12.  [Midea AC LAN Issue #607 su PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607): esempio concreto di PortaSplit con tipo di dispositivo `0xAC`, modello `00000Q1D`, versione del protocollo 3 e configurazione riuscita tramite NetHome Plus.

13.  [API V2 cloud-to-cloud ufficiale Midea](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html): OAuth2, Client-ID, Client-Secret, access e refresh token, procedura di firma ed endpoint `/v2/open/...`.

14.  [API V1 cloud-to-cloud ufficiale Midea](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html): avviso ufficiale che la vecchia interfaccia per partner `/v1/open/...` non è più raccomandata e potrebbe essere dismessa in futuro.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) e [codice cloud attuale](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py): integrazione della community per il controllo completo via cloud e gli endpoint privati V1 dell'app effettivamente utilizzati.

16.  [midea-local PR #717](https://github.com/midea-lan/midea-local/pull/717): passaggio del cloud Meiju da `/v1/iot/secure/getToken` (errore 40404) a `/v2/iot/secure/getToken` con `homegroupId`.

17.  [Codice cloud midea-lan attuale](https://github.com/wuwentao/midea-lan/blob/main/midealan/cloud.py): libreria utilizzata da Midea AC LAN dall'agosto 2026, richiesta token SmartHome tramite `/v1/iot/secure/getToken` con `applianceCodes`.
