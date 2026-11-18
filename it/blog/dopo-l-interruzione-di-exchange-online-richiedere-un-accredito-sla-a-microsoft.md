---
title: "Dopo l'interruzione di Exchange Online: richiedere un accredito SLA a Microsoft"
navTitle: "Accredito SLA"
description: "L'interruzione di Exchange Online del 12 e 13 novembre 2026 è durata circa 22 ore e per molti tenant ha quindi portato la disponibilità al di sotto del 99,9 per cento garantito. Microsoft non accredita automaticamente il credito: va richiesto entro il 31 dicembre 2026. Come calcolare la disponibilità, quali prove richiede Microsoft e perché le piccole aziende devono passare dal partner mentre le grandi aziende si rivolgono direttamente a Microsoft."
date: "2026-11-18"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min di lettura"
themen:
  - microsoft-365-exchange
produkte:
  - "exchange-online"
protokolle:
  - "lizenzierung"
  - "powershell"
slug: "dopo-l-interruzione-di-exchange-online-richiedere-un-accredito-sla-a-microsoft"
translationId: "article-0e96d35615b0d8d9"
translationOf: microsoft-sla-gutschrift-einfordern
url: https://rafaelpfister.ch/it/blog/dopo-l-interruzione-di-exchange-online-richiedere-un-accredito-sla-a-microsoft
translationSourceHash: 537caf78f05fd364041d0a3155f020be11b80f4c936d732e61f0a01f443853c1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T09:21:51.756Z
translationReview: automatic
---

# Dopo l'interruzione di Exchange Online: richiedere un accredito SLA a Microsoft

Dal 12 novembre 2026 alle 18:54 al 13 novembre 2026 alle 17:10 (ora svizzera), molte organizzazioni non hanno avuto accesso, o lo hanno avuto solo in modo intermittente, alle proprie caselle di posta di Exchange Online. Microsoft ha registrato l'interruzione nell'Admin Center con i codici EX1550543 (Exchange Online) e MO1550821 (altri servizi Microsoft 365). Ho documentato l'andamento nel [live ticker sull'interruzione](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11). Per Exchange Online, Microsoft garantisce una disponibilità mensile del 99,9 per cento. Un'interruzione di oltre 22 ore è nettamente al di sotto di tale soglia. Tuttavia, l'accredito viene riconosciuto solo se lo si richiede, e la procedura dipende da come la vostra organizzazione ha acquistato Microsoft 365.

## Cosa garantisce lo SLA

Fa fede il «Service Level Agreement for Microsoft Online Services», pubblicato nuovamente da Microsoft ogni mese. Questo articolo si basa sulla versione del 17 novembre 2026. Per Exchange Online, lo SLA definisce come tempo di inattività qualsiasi periodo in cui gli utenti non possono inviare o ricevere e-mail con Outlook sul Web. Per Exchange Online non sono previste finestre di manutenzione pianificate.

La disponibilità viene calcolata in minuti utente:

```text
Verfügbarkeit (%) = (Benutzerminuten - Ausfallminuten) / Benutzerminuten x 100
```

I minuti utente corrispondono ai minuti del mese moltiplicati per il numero di utenti. I minuti di inattività corrispondono alla durata di ciascun incidente moltiplicata per il numero di utenti interessati. Se la disponibilità scende sotto una soglia, viene riconosciuto un accredito sulla quota mensile del servizio interessato:

| Disponibilità nel mese | Accredito |
|---|---|
| inferiore al 99,9 % | 25 % |
| inferiore al 99 % | 50 % |
| inferiore al 95 % | 100 % |

Per Microsoft Teams, SharePoint Online e OneDrive valgono le stesse soglie, con definizioni proprie del tempo di inattività. Per Teams conta solo quando gli utenti non possono chattare o avviare riunioni. Durante EX1550543, secondo Microsoft chat e riunioni hanno funzionato per lo più; a essere compromessi erano calendari e inviti. Una richiesta per Teams avrebbe quindi poche probabilità di successo.

Oltre alla disponibilità della casella di posta, Exchange Online dispone di altri livelli di servizio, quali il tempo di recapito all'interno di Microsoft 365 e l'accettazione delle e-mail in entrata. Se un incidente ne coinvolge più di uno, secondo lo SLA dovete sceglierne uno. Per l'interruzione di novembre, la disponibilità della casella tramite Outlook sul Web è il parametro più ovvio.

## Cosa significa l'interruzione in percentuale

Novembre 2026 conta 30 giorni, cioè 43'200 minuti per utente. Ne risultano le seguenti soglie per utente:

| Soglia | Tempo di inattività consentito a novembre |
|---|---|
| 99,9 % | 43 minuti |
| 99 % | 7 ore e 12 minuti |
| 95 % | 36 ore |

L'interruzione è durata 1'336 minuti dalla prima segnalazione alla risoluzione. Due esempi di calcolo per un'organizzazione con 200 caselle di posta:

| Caso | Minuti di inattività | Disponibilità | Accredito |
|---|---|---|---|
| Tutti i 200 utenti senza accesso per l'intera durata dell'interruzione | 200 x 1'336 = 267'200 | 96,91 % | 50 % |
| 120 utenti senza accesso per 10 ore | 120 x 600 = 72'000 | 99,17 % | 25 % |

Il numero totale di minuti utente è in entrambi i casi 200 x 43'200 = 8'640'000. È determinante quanti utenti siano stati effettivamente interessati e per quanto tempo. Microsoft lo verifica con la propria telemetria, ma nella richiesta esige i vostri dati al riguardo. Molti tenant in Europa sono stati interessati solo dalla tarda serata del 12 novembre o solo in modo intermittente. Calcolate pertanto usando i periodi che potete documentare, non l'intera durata indicata nella comunicazione Microsoft.

## Cos'è l'accredito e cosa non è

Lo SLA disciplina alcuni aspetti che dovrebbero essere chiari prima della richiesta:

1.  **Solo su richiesta.** Microsoft non verifica né concede accrediti di propria iniziativa. Secondo lo SLA, una comunicazione di stato di Microsoft non costituisce esplicitamente conferma che un livello di servizio non sia stato rispettato.

2.  **Accredito, non rimborso.** L'accredito viene compensato con una fattura futura. È limitato alla quota mensile del servizio interessato.

3.  **Unico rimedio.** Secondo lo SLA, gli accrediti sono l'unico rimedio per i problemi di disponibilità. Microsoft non rimborsa perdite di fatturato, ore supplementari o altri costi conseguenti. Non potete ridurre autonomamente le fatture.

4.  **Solo il servizio interessato.** Se avete una suite come Microsoft 365 E3 o Business Premium, l'accredito riguarda la quota di Exchange Online, non l'intero prezzo della licenza. Microsoft determina tale quota durante la verifica; lo SLA non fornisce alcuna formula.

5.  **Nessun accredito per offerte gratuite e di prova.** Sono esclusi versioni di prova, preview e servizi gratuiti, nonché licenze donate a organizzazioni non profit. I clienti privati con Microsoft 365 Personal o Family non rientrano in questo SLA.

6.  **Contratti Open in giorni.** Chi ha ottenuto licenze tramite Open, Open Value o Open Value Subscription riceve un eventuale accredito sotto forma di giorni d'uso aggiuntivi anziché come importo monetario.

7.  **Termine.** Per i servizi Microsoft 365, la richiesta completa di tutte le informazioni deve pervenire entro la fine del mese successivo a quello dell'incidente. Per l'interruzione di novembre, il termine è il **31 dicembre 2026**. Per Azure si applica un termine separato di 60 giorni.

## Quali informazioni richiede Microsoft

Lo SLA indica cinque informazioni obbligatorie. Se mancano, Microsoft respinge la richiesta:

1.  una descrizione precisa dell'incidente,
2.  data, ora e durata del tempo di inattività,
3.  le risorse interessate (per Exchange Online: tenant e servizio),
4.  numero e località degli utenti interessati,
5.  una descrizione degli errori verificatisi.

L'identificativo dell'incidente (EX1550543) e l'ID tenant devono sempre essere inclusi. Entrambi possono essere interrogati con Microsoft Graph PowerShell:

```powershell
Connect-MgGraph -Scopes "ServiceHealth.Read.All", "Organization.Read.All"
(Get-MgOrganization).Id
Get-MgServiceAnnouncementIssue -ServiceHealthIssueId "EX1550543" |
    Select-Object Id, Title, StartDateTime, EndDateTime, Classification
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Scopes "ServiceHealth.Read.All"` | autorizzazione per leggere le comunicazioni sull'integrità del servizio |
| `-Scopes "Organization.Read.All"` | autorizzazione per leggere i dati dell'organizzazione, incluso l'ID tenant |
| `(Get-MgOrganization).Id` | restituisce l'ID tenant |
| `-ServiceHealthIssueId "EX1550543"` | recupera esattamente questo incidente |
| `Select-Object …` | mostra identificativo, titolo, inizio, fine e classificazione dell'incidente |

</details>

La classificazione (`Classification`) indica se Microsoft gestisce l'incidente come `incident` o come `advisory`. Secondo la documentazione per partner di Microsoft, gli avvisi di livello «Advisory» non danno generalmente diritto a crediti. EX1550543 era classificato come Incident.

Per il numero e la località degli utenti è sufficiente una valutazione delle caselle di posta con licenza per località di utilizzo:

```powershell
Connect-MgGraph -Scopes "User.Read.All"
$abfrage = @{
    All              = $true
    Filter           = 'assignedLicenses/$count ne 0'
    ConsistencyLevel = 'eventual'
    CountVariable    = 'anzahl'
    Property         = 'UsageLocation'
}
Get-MgUser @abfrage |
    Group-Object UsageLocation |
    Select-Object Name, Count
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `$abfrage = @{ … }` | raccoglie i parametri per `Get-MgUser` in una hashtable (splatting), per mantenere breve la riga |
| `All` | legge tutti gli utenti, non solo la prima pagina |
| `Filter 'assignedLicenses/$count ne 0'` | solo utenti con almeno una licenza assegnata |
| `ConsistencyLevel eventual` | necessario per il filtro `$count` (query avanzata) |
| `CountVariable anzahl` | memorizza il numero totale nella variabile `$anzahl` |
| `Property UsageLocation` | carica anche la località di utilizzo |
| `Group-Object UsageLocation` | conta gli utenti per Paese |

</details>

Questo numero è il limite massimo. Il numero di utenti che non hanno effettivamente avuto accesso va documentato con dati propri: ticket dell'helpdesk con marca temporale, messaggi di errore da Outlook e Outlook sul Web (screenshot), risultati dei test di disponibilità del monitoraggio, nonché log di accesso e autenticazione. Più precisi sono questi dati, meno il risultato dipende esclusivamente dalla telemetria di Microsoft.

## Piccole aziende: di norma tramite il partner

Le piccole e medie aziende in Svizzera acquistano Microsoft 365 per lo più tramite un partner nel Cloud Solution Provider Program (CSP) oppure direttamente con carta di credito nell'Admin Center. Potete verificare quale percorso si applica nel Microsoft 365 Admin Center in **Fatturazione > I tuoi prodotti**: se come fonte di acquisto compare un partner, il contratto è un contratto CSP.

### Acquistato tramite un partner CSP

In questo caso Microsoft non accetta la richiesta direttamente da voi. Secondo il Partner Center, solo i fornitori diretti CSP e i fornitori indiretti (distributori) possono richiedere accrediti, non i rivenditori indiretti. Se il vostro fornitore IT è un rivenditore indiretto, la richiesta passa quindi da lui al suo distributore e da quest'ultimo a Microsoft. Ogni livello ha termini propri, spesso più anticipati. Contattateli quindi subito e non solo alla fine di dicembre.

Il vostro partner ha bisogno di:

1.  l'ID tenant,
2.  l'identificativo dell'incidente (EX1550543),
3.  periodo, numero e località degli utenti interessati, nonché gli errori osservati,
4.  un'e-mail con cui richiedete l'accredito SLA. Deve provenire da un indirizzo del dominio interessato; Microsoft non accetta e-mail da un indirizzo privato.

Microsoft conosce l'incidente stesso. La richiesta deve dimostrare che la vostra organizzazione è stata interessata: secondo Microsoft, nome, ID tenant e numero di ticket da soli non sono sufficienti. Se l'accredito viene concesso, Microsoft lo accredita al partner, che lo trasferisce a voi. Se e in che modo ciò avvenga è disciplinato dal vostro contratto con il partner. Chiedete informazioni se non compare nulla sulla fattura successiva.

### Acquistato direttamente da Microsoft

Se avete pagato Microsoft 365 direttamente a Microsoft, inviate voi stessi la richiesta nell'Admin Center. È necessario un ruolo di amministratore:

1.  In `admin.cloud.microsoft` aprite in basso a destra **Guida e supporto**.
2.  Descrivete la richiesta, ad esempio «Request an SLA service credit for a Microsoft 365 outage», e selezionate **Contatta il supporto**.
3.  Scegliete l'e-mail come canale di contatto. Il titolo deve contenere «SLA service credit request» e l'identificativo EX1550543.
4.  Nella descrizione indicate le cinque informazioni obbligatorie della sezione precedente e allegate le prove.

È necessaria una richiesta separata per ogni servizio e incidente. Secondo lo SLA, Microsoft elabora di norma le richieste entro 45 giorni.

### Vale la pena per le piccole aziende?

Con dieci utenti l'accredito è modesto: per una suite conta solo la quota di Exchange, di cui il 25 o il 50 per cento per un mese. Anche l'impegno è ridotto se raccogliete ora le informazioni. La richiesta ha un secondo vantaggio: documenta per il partner e la direzione quanto la vostra organizzazione sia stata colpita. Ciò aiuta in seguito a valutare se sia necessario un fallback per l'e-mail.

## Grandi aziende: direttamente con Microsoft, con dati di misurazione propri

Le grandi organizzazioni ottengono licenze Microsoft 365 per lo più tramite un Enterprise Agreement (EA) oppure il Microsoft Customer Agreement for Enterprise (MCA-E). In questo caso la richiesta va direttamente a Microsoft, tramite l'Admin Center come descritto sopra oppure tramite il contratto Unified Support. Se avete un Customer Success Account Manager (CSAM), informatelo della richiesta; potrà seguirne l'elaborazione. In caso di EA tramite un Licensing Solution Partner (LSP), chiarite inoltre su quale fattura comparirà l'accredito.

Con diverse migliaia di utenti sono in gioco importi significativi e Microsoft verifica quindi le informazioni con particolare attenzione. La differenza rispetto a un tenant piccolo sta nelle prove:

1.  **Determinare con precisione gli utenti interessati.** I tenant grandi sono distribuiti su più database e regioni nei data center Microsoft; non tutte le caselle di posta hanno subito disservizi per lo stesso tempo. Dichiarare forfettariamente tutti gli utenti come interessati è contestabile per i tenant grandi. Sono più solidi i dati di misurazione: test sintetici contro Outlook sul Web per sede, dati di disponibilità del monitoraggio interno, analisi dei ticket helpdesk per sede e orario.

2.  **Indicare le sedi.** Lo SLA richiede le località degli utenti interessati. Per le organizzazioni internazionali conviene suddividere per Paese, poiché l'interruzione ha colpito prima il Nord e il Sud America e solo successivamente altre regioni.

3.  **Delimitare correttamente l'ambiente ibrido.** Lo SLA esclude ritardi nel proprio data center, quali e-mail nella coda dei server Exchange locali dirette al tenant. Documentate che l'errore era interno a Microsoft 365. Le code dei server locali sono una buona prova, perché mostrano che Exchange Online non ha accettato le e-mail o le ha accettate solo con ritardo.

4.  **Scegliere il livello di servizio.** Se per voi erano interessati soprattutto i tempi di recapito e meno l'accesso alle caselle di posta, il livello di servizio per il tempo di recapito può essere una base migliore. È consentita una sola richiesta per incidente e servizio, quindi la scelta va fatta prima della richiesta.

5.  **Coinvolgere acquisti e gestione delle licenze.** L'accredito compare in fattura, non nel centro di costo IT. Coordinatevi con gli acquisti su chi presenta la richiesta, chi controlla l'accredito e come viene contabilizzato.

Le grandi organizzazioni dovrebbero inoltre preparare il processo per la prossima interruzione: un modello con le cinque informazioni obbligatorie, un monitoraggio che misuri la disponibilità di Outlook sul Web per sede e una responsabilità chiara per le richieste SLA.

## Modello per la richiesta

Il seguente modello contiene le informazioni obbligatorie previste dallo SLA. Microsoft elabora più rapidamente le richieste in inglese:

```text
Subject: SLA service credit request: EX1550543, Exchange Online

Tenant ID: <Tenant-ID>
Service: Exchange Online
Incident ID: EX1550543 (related: MO1550821)
Service level: Exchange Online availability (Outlook on the web)

Description of the incident:
Users were unable to send or receive email using Outlook on the web,
Outlook for Windows and mobile clients.

Time and duration of downtime:
From 2026-11-12 <hh:mm> UTC to 2026-11-13 <hh:mm> UTC (<n> minutes).

Affected users:
<n> of <total> licensed users, locations: <Land: Anzahl, ...>

Errors observed:
<Fehlermeldungen, z. B. Verbindungsfehler in Outlook, HTTP 5xx in Outlook on the web>

Uptime calculation (November 2026):
(<total> x 43,200 - <Ausfallminuten>) / (<total> x 43,200) x 100 = <xx.xx> %

Attachments: helpdesk ticket export, monitoring data, screenshots.
```

## Fonti

1.  [Service Level Agreement for Microsoft Online Services (panoramica delle versioni)](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services): versione del 17 novembre 2026, definizione del tempo di inattività per Exchange Online, livelli di accredito, informazioni obbligatorie, termini ed esclusioni.

2.  [Microsoft Learn: How and when to request a credit from Microsoft](https://learn.microsoft.com/en-us/partner-center/billing/request-credit): procedura di richiesta per partner CSP, esclusione dei rivenditori indiretti, prove richieste, classificazione degli Advisories.

3.  [Microsoft Learn: Get-MgServiceAnnouncementIssue](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.devices.serviceannouncement/get-mgserviceannouncementissue): cmdlet per recuperare un incidente di integrità del servizio tramite Microsoft Graph.

4.  [Microsoft 365 Status (status.cloud.microsoft)](https://status.cloud.microsoft/): comunicazioni di stato pubbliche su EX1550543 e MO1550821.

5.  [Windows Latest: Microsoft 365 outage, you may get money off your bill](https://www.windowslatest.com/2026/11/13/microsoft-365-outage-you-may-get-money-off-your-bill-if-you-use-it-for-work-but-youll-have-to-ask-microsoft-for-it-as-a-server-failure-takes-down-outlook-teams-and-exchange/): procedura di richiesta nell'Admin Center per l'interruzione di novembre, indicazione sui clienti privati.

6.  [Office Watch: Microsoft 365 went down? How to claim your SLA credit](https://office-watch.com/?p=78178): esclusione di licenze private, gratuite e non profit, una richiesta per servizio e incidente.

7.  [Live ticker: Exchange Online, interruzione mondiale dell'accesso alle caselle di posta](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11): andamento dell'interruzione del 12 e 13 novembre 2026 con cronologia Microsoft.
