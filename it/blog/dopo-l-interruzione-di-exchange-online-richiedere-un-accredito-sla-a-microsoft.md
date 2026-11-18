---
title: "Dopo l’interruzione di Exchange Online: richiedere a Microsoft un credito SLA"
navTitle: "Credito SLA"
description: "L’interruzione di Exchange Online del 12 e 13 novembre 2026 è durata circa 22 ore e per molti tenant ha quindi portato la disponibilità al di sotto del 99,9 per cento garantito. Microsoft non accredita automaticamente il credito: occorre richiederlo entro il 31 dicembre 2026. Come calcolare la disponibilità, quali prove richiede Microsoft e perché le piccole aziende devono procedere tramite il partner mentre le grandi aziende direttamente con Microsoft."
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
translationSourceHash: e2a540fa8b0a15956e2a9f4af6b604f517d4f077ec2d566bd2dd187a1d423216
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T11:21:44.684Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/dopo-l-interruzione-di-exchange-online-richiedere-un-accredito-sla-a-microsoft
---

# Dopo l’interruzione di Exchange Online: richiedere a Microsoft un credito SLA

<div class="notfall-hinweis notfall-hinweis--gross">
<p class="notfall-hinweis__titel">Supporto gratuito per la richiesta SLA</p>
<p>Vi aiutiamo gratuitamente a richiedere a Microsoft o al vostro partner il credito per l’interruzione di Exchange Online del 12 e 13 novembre 2026: calcolo della disponibilità, raccolta delle prove, presentazione della richiesta. Il termine scade il 31 dicembre 2026. Contattateci tramite il <a href="https://rafaelpfister.ch/kontakt">modulo di contatto</a>.</p>
</div>

Dal 12 novembre 2026 alle 18:54 al 13 novembre 2026 alle 17:10 (ora svizzera), molte organizzazioni non hanno avuto accesso, o hanno avuto solo un accesso intermittente, alle proprie caselle di posta Exchange Online. Microsoft ha registrato il disservizio nell’Admin Center con EX1550543 (Exchange Online) e MO1550821 (altri servizi Microsoft 365). Ho documentato l’andamento nel [live ticker dell’interruzione](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11). Per Exchange Online, Microsoft garantisce una disponibilità mensile del 99,9 per cento. Un’interruzione di oltre 22 ore è nettamente inferiore a tale soglia. Tuttavia, riceverete un credito solo se lo richiedete e la procedura dipende da come la vostra organizzazione ha acquistato Microsoft 365.

## Cosa garantisce l’SLA

Fa fede il «Service Level Agreement for Microsoft Online Services», che Microsoft pubblica nuovamente ogni mese. Questo articolo si basa sulla versione del 17 novembre 2026. Per Exchange Online, l’SLA definisce come tempo di inattività qualsiasi periodo in cui gli utenti non possono inviare o ricevere e-mail con Outlook sul web. Per Exchange Online non sono previste finestre di manutenzione pianificate.

La disponibilità viene calcolata in minuti utente:

```text
Verfügbarkeit (%) = (Benutzerminuten - Ausfallminuten) / Benutzerminuten x 100
```

I minuti utente corrispondono ai minuti del mese moltiplicati per il numero di utenti. I minuti di inattività corrispondono alla durata di ciascun incidente moltiplicata per il numero di utenti interessati. Se la disponibilità scende sotto una soglia, viene riconosciuto un credito sulla tariffa mensile del servizio interessato:

| Disponibilità nel mese | Credito |
|---|---|
| inferiore al 99,9 % | 25 % |
| inferiore al 99 % | 50 % |
| inferiore al 95 % | 100 % |

Per Microsoft Teams, SharePoint Online e OneDrive valgono le stesse soglie, con definizioni proprie del tempo di inattività. Per Teams conta solo il caso in cui gli utenti non possano chattare o avviare riunioni. Durante EX1550543, secondo Microsoft chat e riunioni hanno funzionato in gran parte; erano invece compromessi calendari e inviti. Una richiesta per Teams avrà quindi probabilmente poche possibilità di successo.

Oltre alla disponibilità della casella di posta, Exchange Online dispone di altri livelli di servizio, come il tempo di recapito all’interno di Microsoft 365 e l’accettazione delle e-mail in entrata. Se un incidente ne coinvolge più di uno, secondo l’SLA dovete sceglierne uno. Per l’interruzione di novembre, la disponibilità della casella di posta tramite Outlook sul web è il parametro più ovvio.

## Cosa significa l’interruzione in percentuale

Novembre 2026 ha 30 giorni, ovvero 43'200 minuti per utente. Ne risultano le seguenti soglie per utente:

| Soglia | Tempo di inattività consentito a novembre |
|---|---|
| 99,9 % | 43 minuti |
| 99 % | 7 ore e 12 minuti |
| 95 % | 36 ore |

Il disservizio è durato 1'336 minuti dalla prima segnalazione alla risoluzione. Due esempi di calcolo per un’organizzazione con 200 caselle di posta:

| Caso | Minuti di inattività | Disponibilità | Credito |
|---|---|---|---|
| Tutti i 200 utenti senza accesso per l’intera durata del disservizio | 200 x 1'336 = 267'200 | 96,91 % | 50 % |
| 120 utenti senza accesso per 10 ore | 120 x 600 = 72'000 | 99,17 % | 25 % |

Il numero totale di minuti utente è in entrambi i casi 200 x 43'200 = 8'640'000. È decisivo quanti utenti siano stati effettivamente interessati e per quanto tempo. Microsoft lo verifica attraverso la propria telemetria, ma nella richiesta esige le vostre indicazioni al riguardo. Molti tenant in Europa sono stati colpiti solo dalla tarda serata del 12 novembre o in modo intermittente. Calcolate quindi usando gli orari che potete dimostrare, non l’intera durata riportata nella comunicazione di Microsoft.

## Cosa è e cosa non è il credito

L’SLA disciplina alcuni aspetti che dovrebbero essere chiari prima della richiesta:

1.  **Solo su richiesta.** Microsoft non esamina né concede crediti di propria iniziativa. Secondo l’SLA, una comunicazione di stato di Microsoft non costituisce espressamente conferma che un livello di servizio non sia stato raggiunto.

2.  **Credito, non rimborso.** Il credito viene compensato con una fattura futura. È limitato alla tariffa mensile del servizio interessato.

3.  **Unico rimedio.** Secondo l’SLA, i crediti sono l’unico rimedio per problemi di disponibilità. Microsoft non risarcisce perdite di fatturato, straordinari o altri costi conseguenti. Non potete ridurre autonomamente le fatture.

4.  **Solo il servizio interessato.** Se disponete di una suite come Microsoft 365 E3 o Business Premium, il credito riguarda la quota di Exchange Online, non l’intero prezzo della licenza. Microsoft determina tale quota durante la verifica; l’SLA non indica una formula al riguardo.

5.  **Nessun credito per offerte gratuite e di prova.** Sono escluse le versioni di prova, le preview e i servizi gratuiti, nonché le licenze donate a organizzazioni non profit. I clienti privati con Microsoft 365 Personal o Family non rientrano in questo SLA.

6.  **Contratti Open in giorni.** Chi ha licenze tramite Open, Open Value o Open Value Subscription riceve un eventuale credito sotto forma di giorni di utilizzo aggiuntivi anziché come importo monetario.

7.  **Termine.** Per i servizi Microsoft 365, la richiesta completa di tutte le informazioni deve pervenire entro la fine del mese successivo a quello dell’incidente. Per l’interruzione di novembre, il termine è il **31 dicembre 2026**. Per Azure vale un termine separato di 60 giorni.

## Quali informazioni richiede Microsoft

L’SLA indica cinque informazioni obbligatorie. Se mancano, Microsoft respinge la richiesta:

1.  una descrizione precisa dell’incidente,
2.  momento e durata del tempo di inattività,
3.  le risorse interessate (per Exchange Online: tenant e servizio),
4.  numero e sedi degli utenti interessati,
5.  una descrizione degli errori verificatisi.

L’identificativo dell’incidente (EX1550543) e l’ID tenant devono sempre essere inclusi. Entrambi possono essere recuperati con Microsoft Graph PowerShell:

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
| `-Scopes "ServiceHealth.Read.All"` | Autorizzazione a leggere le comunicazioni sull’integrità del servizio |
| `-Scopes "Organization.Read.All"` | Autorizzazione a leggere i dati dell’organizzazione, incluso l’ID tenant |
| `(Get-MgOrganization).Id` | restituisce l’ID tenant |
| `-ServiceHealthIssueId "EX1550543"` | recupera esattamente questo incidente |
| `Select-Object …` | mostra identificativo, titolo, inizio, fine e classificazione dell’incidente |

</details>

La classificazione (`Classification`) indica se Microsoft considera l’incidente come `incident` oppure come `advisory`. Secondo la documentazione per partner di Microsoft, le segnalazioni di livello «Advisory» non danno di norma diritto a un credito. EX1550543 era classificato come Incident.

Per il numero e la sede degli utenti è sufficiente una valutazione delle caselle di posta con licenza in base alla località di utilizzo:

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
| `$abfrage = @{ … }` | raccoglie i parametri per `Get-MgUser` in una hashtable (splatting), mantenendo breve la riga |
| `All` | legge tutti gli utenti, non solo la prima pagina |
| `Filter 'assignedLicenses/$count ne 0'` | solo gli utenti con almeno una licenza assegnata |
| `ConsistencyLevel eventual` | necessario per il filtro `$count` (query avanzata) |
| `CountVariable anzahl` | salva il numero totale nella variabile `$anzahl` |
| `Property UsageLocation` | carica anche la località di utilizzo |
| `Group-Object UsageLocation` | conta gli utenti per Paese |

</details>

Questo numero rappresenta il limite superiore. Dimostrate con i vostri dati quanti utenti non abbiano effettivamente avuto accesso: ticket dell’helpdesk con marca temporale, messaggi di errore di Outlook e Outlook sul web (screenshot), risultati dei test di disponibilità del vostro monitoraggio, nonché registri di accesso e autenticazione. Più queste indicazioni sono precise, meno il risultato dipende esclusivamente dalla telemetria di Microsoft.

## Piccole aziende: di solito tramite il partner

In Svizzera, le piccole e medie aziende acquistano generalmente Microsoft 365 tramite un partner del Cloud Solution Provider Program (CSP) oppure direttamente con carta di credito nell’Admin Center. Potete verificare quale procedura si applica nel Microsoft 365 Admin Center, in **Fatturazione > I tuoi prodotti**: se vi è indicato un partner come fonte di acquisto, il contratto è un contratto CSP.

### Acquistato tramite un partner CSP

In questo caso Microsoft non accetta la richiesta direttamente da voi. Secondo il Partner Center, solo i fornitori diretti CSP e i fornitori indiretti (distributori) possono richiedere crediti, non i rivenditori indiretti. Se il vostro fornitore IT è un rivenditore indiretto, la richiesta passa quindi da lui al suo distributore e da quest’ultimo a Microsoft. Ogni livello ha le proprie scadenze, spesso anteriori. Contattatelo quindi ora e non solo alla fine di dicembre.

Il vostro partner necessita di:

1.  l’ID tenant,
2.  l’identificativo dell’incidente (EX1550543),
3.  periodo, numero e sedi degli utenti interessati, nonché gli errori osservati,
4.  un’e-mail in cui richiedete il credito SLA. Deve provenire da un indirizzo del dominio interessato; Microsoft non accetta un’e-mail da un indirizzo privato.

Microsoft conosce già l’incidente. La richiesta deve dimostrare che la vostra organizzazione ne è stata interessata: secondo Microsoft, nome, ID tenant e numero di ticket da soli non sono sufficienti. Se il credito viene concesso, Microsoft lo accredita al partner, che lo trasferisce a voi. Se e come ciò avvenga è disciplinato dal vostro contratto con il partner. Chiedete informazioni se non compare nulla sulla fattura successiva.

### Acquistato direttamente da Microsoft

Se avete pagato Microsoft 365 direttamente a Microsoft, presentate voi stessi la richiesta nell’Admin Center. È necessario un ruolo di amministratore:

1.  In `admin.cloud.microsoft` aprite **Guida e supporto** in basso a destra.
2.  Descrivete la richiesta, ad esempio «Request an SLA service credit for a Microsoft 365 outage», e selezionate **Contatta il supporto**.
3.  Scegliete l’e-mail come modalità di contatto. Nel titolo devono figurare «SLA service credit request» e l’identificativo EX1550543.
4.  Nella descrizione indicate le cinque informazioni obbligatorie della sezione precedente e allegate le prove.

È necessaria una richiesta separata per ogni servizio e incidente. Secondo l’SLA, Microsoft elabora normalmente le richieste entro 45 giorni.

### Ne vale la pena per le piccole aziende

Per dieci utenti, il credito è modesto: in caso di suite conta solo la quota Exchange, di cui il 25 o il 50 per cento per un mese. Anche l’onere è ridotto, se raccogliete ora le informazioni. La richiesta offre un secondo vantaggio: documenta presso il partner e la direzione quanto la vostra organizzazione sia stata colpita. Questo aiuta in seguito a decidere se sia necessario un fallback per la posta elettronica.

## Grandi aziende: direttamente con Microsoft, con dati di misurazione propri

Le grandi organizzazioni ottengono le licenze Microsoft 365 generalmente tramite un Enterprise Agreement (EA) o il Microsoft Customer Agreement for Enterprise (MCA-E). In questo caso la richiesta va direttamente a Microsoft, tramite l’Admin Center come descritto sopra oppure tramite il contratto Unified Support. Se avete un Customer Success Account Manager (CSAM), informatelo della richiesta; può seguirne l’elaborazione. In caso di EA tramite un Licensing Solution Partner (LSP), chiarite inoltre su quale fattura comparirà il credito.

Con diverse migliaia di utenti sono in gioco importi rilevanti e Microsoft verifica quindi le indicazioni con particolare precisione. La differenza rispetto a un tenant piccolo sta nelle prove:

1.  **Determinare con precisione gli utenti interessati.** I grandi tenant sono distribuiti su più database e regioni nei data center di Microsoft; non tutte le caselle di posta hanno subito il disservizio per lo stesso tempo. Segnalare genericamente tutti gli utenti come interessati è contestabile per i grandi tenant. Sono più solidi i dati di misurazione: test sintetici su Outlook sul web per sede, dati di disponibilità del monitoraggio interno, analisi dei ticket dell’helpdesk per sede e orario.

2.  **Indicare le sedi.** L’SLA richiede le sedi degli utenti interessati. Per le organizzazioni internazionali conviene ripartire per Paese, poiché il disservizio ha colpito prima il Nord e il Sud America e solo successivamente altre regioni.

3.  **Delimitare correttamente l’ambiente ibrido.** L’SLA esclude ritardi nel proprio data center, ad esempio e-mail nella coda dei server Exchange locali verso il tenant. Dimostrate che l’errore si trovava all’interno di Microsoft 365. Le code dei server locali sono una buona prova, perché mostrano che Exchange Online non ha accettato le e-mail o le ha accettate solo con ritardo.

4.  **Scegliere il livello di servizio.** Se nel vostro caso sono stati interessati soprattutto i tempi di recapito e meno l’accesso alle caselle di posta, il livello di servizio relativo al tempo di recapito può costituire una base migliore. Per ogni incidente e servizio è ammessa una sola richiesta; la scelta dovrebbe quindi essere effettuata prima della richiesta.

5.  **Coinvolgere acquisti e gestione delle licenze.** Il credito compare sulla fattura, non nel centro di costo IT. Concordate con gli acquisti chi presenta la richiesta, chi controlla il credito e come viene contabilizzato.

Le grandi organizzazioni dovrebbero inoltre preparare il processo per la prossima interruzione: un modello con le cinque informazioni obbligatorie, un monitoraggio che misuri la disponibilità di Outlook sul web per sede e una chiara responsabilità per le richieste SLA.

## Modello per la richiesta

Il seguente modello contiene le informazioni obbligatorie dell’SLA. Microsoft elabora più rapidamente le richieste in inglese:

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

1.  [Service Level Agreement for Microsoft Online Services (panoramica delle versioni)](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services): versione del 17 novembre 2026, definizione del tempo di inattività per Exchange Online, livelli di credito, informazioni obbligatorie, termini ed esclusioni.

2.  [Microsoft Learn: How and when to request a credit from Microsoft](https://learn.microsoft.com/en-us/partner-center/billing/request-credit): procedura di richiesta per partner CSP, esclusione dei rivenditori indiretti, prove richieste, classificazione degli Advisory.

3.  [Microsoft Learn: Get-MgServiceAnnouncementIssue](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.devices.serviceannouncement/get-mgserviceannouncementissue): cmdlet per recuperare un incidente di integrità del servizio tramite Microsoft Graph.

4.  [Microsoft 365 Status (status.cloud.microsoft)](https://status.cloud.microsoft/): comunicazioni di stato pubbliche su EX1550543 e MO1550821.

5.  [Windows Latest: Microsoft 365 outage, you may get money off your bill](https://www.windowslatest.com/2026/11/13/microsoft-365-outage-you-may-get-money-off-your-bill-if-you-use-it-for-work-but-youll-have-to-ask-microsoft-for-it-as-a-server-failure-takes-down-outlook-teams-and-exchange/): procedura di richiesta nell’Admin Center per l’interruzione di novembre, nota sui clienti privati.

6.  [Office Watch: Microsoft 365 went down? How to claim your SLA credit](https://office-watch.com/?p=78178): esclusione di licenze private, gratuite e non profit; una richiesta per servizio e incidente.

7.  [Live ticker: Exchange Online, interruzione mondiale dell’accesso alle caselle di posta](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11): andamento del disservizio del 12 e 13 novembre 2026 con cronologia Microsoft.
