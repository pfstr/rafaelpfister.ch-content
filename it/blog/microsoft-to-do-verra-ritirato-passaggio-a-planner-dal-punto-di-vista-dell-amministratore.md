---
title: "Microsoft To Do verrà ritirato: passaggio a Planner dal punto di vista dell’amministratore"
navTitle: "To Do verrà ritirato"
description: "Microsoft ritirerà le app To Do il 30 giugno 2027. Gli account aziendali passeranno a Planner, le attività esistenti non verranno migrate e Planner salverà le nuove attività private al di fuori della cassetta postale Exchange. Per i tenant svizzeri cambieranno quindi la localizzazione dei dati e la copertura della conformità; gli utenti interessati possono essere individuati tramite Microsoft Graph."
date: "2026-11-24"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min di lettura"
themen:
  - microsoft-365-exchange
produkte:
  - "planner"
  - "exchange-online"
protokolle:
  - "migration"
  - "powershell"
hauptthema: "microsoft-365-exchange"
slug: "microsoft-to-do-verra-ritirato-passaggio-a-planner-dal-punto-di-vista-dell-amministratore"
translationId: "article-7691ba2aae63c942"
translationOf: microsoft-to-do-planner-umstieg
url: https://rafaelpfister.ch/it/blog/microsoft-to-do-verra-ritirato-passaggio-a-planner-dal-punto-di-vista-dell-amministratore
translationSourceHash: ae78db5df03b2677ad2bda294efdd3b1f31ee233d2de5ea0238c2de865396fe8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T12:59:02.324Z
translationReview: automatic
---

# Microsoft To Do verrà ritirato: passaggio a Planner dal punto di vista dell’amministratore

Microsoft ritirerà Microsoft To Do. Le app per Web, Windows, Mac, iOS e Android verranno disattivate il 30 giugno 2027, sia per gli account aziendali e scolastici sia per gli account Microsoft privati. Per gli account aziendali e scolastici, Microsoft Planner prenderà in carico le attività personali; la visualizzazione rivista «Le mie attività» (My tasks) verrà distribuita a partire da dicembre 2026. Microsoft ha pubblicato la modifica nel novembre 2026 in un articolo di supporto con cronologia, differenze e indicazioni per gli amministratori.

In breve: non è prevista alcuna migrazione automatica. Le attività To Do esistenti resteranno visibili in Planner fino al 30 giugno 2027 e continueranno a essere archiviate nella cassetta postale Exchange Online. A partire dalla distribuzione, Planner salverà le nuove attività private in un piano Planner privato, quindi non più in Exchange. Per i tenant svizzeri cambieranno pertanto la localizzazione dei dati e la copertura di Purview. Entro metà 2027, verificate licenze, elenchi condivisi, integrazioni e obblighi di conservazione. Gli account Microsoft privati non riceveranno alcun sostituto.

## Cronologia

| Data | Modifica |
|---|---|
| Dicembre 2026 | Inizio della distribuzione della visualizzazione rivista «Le mie attività» in Planner per gli account aziendali e scolastici. Da quel momento Planner salverà le nuove attività private in un piano Planner privato. |
| prima del 30 giugno 2027 | Microsoft metterà a disposizione una funzione di esportazione per i dati To Do. Formato, ambito e requisiti non sono ancora stati pubblicati. |
| 30 giugno 2027 | Le app To Do per Web, Windows, Mac, iOS e Android verranno ritirate. Dopo tale data, le attività To Do esistenti non saranno più visibili neppure in Planner. |
| successivamente, senza data | Il servizio To Do, le API e la sincronizzazione verranno ritirati separatamente. Microsoft comunicherà le date a parte. |

Planner è già oggi disponibile per gli account aziendali sul Web all’indirizzo `planner.cloud.microsoft` e come app in Teams. Microsoft annuncia l’adeguamento dei punti di accesso esistenti di To Do in Microsoft 365. A Thurrott, Microsoft ha precisato che le integrazioni esistenti, tra cui Outlook e Teams, passeranno a Planner.

## Le attività esistenti non verranno migrate

Microsoft non trasferirà in Planner le attività e gli elenchi To Do esistenti. Fino al 30 giugno 2027 sarà ancora possibile visualizzarli e modificarli in Planner, ma non successivamente. Non verranno trasferiti nel nuovo piano privato.

I dati non verranno eliminati con la fine delle app. Secondo Microsoft, il ritiro delle app non prevede alcuna cancellazione; Microsoft tratta separatamente la conservazione e l’accesso. To Do salva le attività come attività Exchange nella cassetta postale Exchange Online dell’utente, nella stessa cassetta postale di e-mail, appuntamenti, contatti e note. Pertanto, il contenuto resterà per il momento nella cassetta postale anche dopo il 30 giugno 2027.

Outlook classico non è interessato dal ritiro. Microsoft afferma che la sua funzionalità attività è separata da To Do, continuerà a funzionare e si sincronizzerà con Exchange. Gli elenchi To Do vengono visualizzati in Outlook classico come cartelle delle attività in «Le mie attività»; gli utenti di Outlook classico potranno quindi continuare a leggere il contenuto lì. Non è ancora descritto cosa accadrà ai dati quando Microsoft disattiverà successivamente il servizio To Do stesso.

## Le nuove attività si trovano al di fuori di Exchange

Con la nuova visualizzazione cambia la posizione di archiviazione. Secondo le FAQ per gli amministratori di Planner, le attività da To Do e Outlook si trovano in Exchange, mentre i piani e le relative attività si trovano in Azure. Planner salva gli allegati alle attività del piano nel sito SharePoint del gruppo e i piani Premium in Dataverse. Le nuove attività private da «Le mie attività» non si troveranno quindi più nella cassetta postale dell’utente. Per gli amministratori, questo comporta due conseguenze.

### Localizzazione dei dati

Exchange Online rientra tra i servizi con impegno permanente sulla localizzazione dei dati. Se un tenant è stato creato in Svizzera, secondo i Product Terms Microsoft salva i contenuti delle cassette postali inattivi (testo delle e-mail, voci di calendario e allegati) solo in Svizzera. L’impegno non menziona espressamente le attività, ma esse si trovano nella stessa cassetta postale.

Microsoft indica Planner tra gli altri servizi Microsoft 365 senza impegno permanente. Per i tenant con paese Svizzera, la tabella indica «EUR1», la macroregione Europa, come posizione di archiviazione. Inoltre, Planner non partecipa ad Advanced Data Residency. Le attività personali che To Do ha finora salvato nella cassetta postale in Svizzera verranno salvate da Planner, dopo il passaggio, nella macroregione Europa. Le posizioni di archiviazione per servizio del vostro tenant sono elencate nel Microsoft 365 Admin Center nella scheda Data Location Card. Le organizzazioni che hanno garantito contrattualmente o nelle proprie direttive interne una localizzazione dei dati in Svizzera devono valutare questa modifica.

### Conservazione ed eDiscovery

I criteri di conservazione di Exchange in Microsoft Purview acquisiscono le attività nella cassetta postale solo se hanno una data di scadenza; le attività senza data di scadenza non sono coperte. Questa limitazione si applica già oggi alle attività To Do.

Per Planner, secondo Microsoft Purview supporta la ricerca eDiscovery e Legal Hold solo per le attività nei piani condivisi con un gruppo. Non sono espressamente supportati i piani personali (piani roster), i piani nei componenti Loop, i piani Premium e i piani nei canali Teams condivisi e privati. Microsoft distribuisce i criteri di conservazione ed eliminazione per i contenuti Planner dalla metà di luglio 2026; il completamento era annunciato per l’inizio di agosto (MC1384412). Secondo l’annuncio, possono essere applicati ai piani basati su utenti e gruppi. Dall’annuncio non risulta a quale tipo di piano corrisponda tecnicamente il nuovo piano privato. Dopo la distribuzione, verificate se i vostri criteri di conservazione e hold acquisiscono le nuove attività private.

## Differenze tra To Do e Planner

Microsoft elenca le seguenti differenze:

| Area | Differenza secondo Microsoft |
|---|---|
| Organizzazione | Planner riunisce le attività private e il lavoro assegnato in «Le mie attività», oggi con visualizzazione a griglia e bacheca. Con l’aggiornamento seguirà una visualizzazione elenco. |
| Promemoria | Le notifiche delle date di scadenza in Planner differiscono dai promemoria pianificati per singola attività in To Do. |
| Condivisione | Planner utilizza piani e assegnazioni. Membri, assegnazioni e autorizzazioni degli elenchi To Do condivisi non verranno trasferiti. |
| Desktop e offline | Planner è disponibile sul Web e in Teams. Non sono previste app native per Windows e Mac; Web e Teams non offrono una funzionalità offline equivalente. |
| Categorie | Le etichette di Planner funzionano diversamente dalle categorie di Outlook e non vengono sincronizzate con esse. |
| Allegati | Gli allegati di file alle attività non sono attualmente supportati nel nuovo piano Planner privato. |
| Stella e priorità | La stella personale serve alla pianificazione individuale e arriverà con la nuova visualizzazione. La priorità di un’attività si applica a tutti i partecipanti di un piano condiviso. |
| Gruppi di elenchi | Nessun equivalente diretto in Planner. |
| Stampa, invio dell’elenco via e-mail | Attualmente non previsti per la soluzione sostitutiva. |

Gli utenti che lavorano offline con l’app To Do in Windows o Mac non avranno un sostituto equivalente in Planner.

## Licenze e accesso a Planner

Planner è incluso, tra l’altro, in Microsoft 365 Business Basic, Business Standard e Business Premium, nonché in Office 365 E1, E3 ed E5. Se Planner manca in una licenza, Microsoft offre Planner Plan 1 o Planner and Project Plan 3 come abbonamenti separati; questi sono necessari anche per le funzionalità Premium. Secondo Microsoft, non è necessaria alcuna nuova configurazione e non esiste un’opzione specifica con cui gli amministratori possano controllare o rinviare il passaggio.

Se Planner è disattivato nell’assegnazione della licenza (piano di servizio `PROJECTWORKMANAGEMENT`), gli utenti interessati dovranno nuovamente avere accesso se Planner deve sostituire To Do. Secondo le FAQ di Planner, la rimozione della licenza Planner nasconde comunque solo il riquadro: tramite `planner.cloud.microsoft` gli utenti senza licenza possono comunque creare e modificare piani. Se finora avete bloccato Planner tramite licenza, riesaminate questa decisione alla luce del passaggio.

## Rilevare il contenuto con Microsoft Graph

È possibile leggere tramite l’API To Do di Microsoft Graph quali utenti utilizzano To Do e quali elenchi sono condivisi. Il seguente script scrive in un file CSV tutti gli elenchi per utente, con stato di condivisione e numero di attività aperte. Richiede i moduli `Microsoft.Graph.Authentication` e `Microsoft.Graph.Users`, nonché una registrazione dell’app con le autorizzazioni applicative `Tasks.Read.All` e `User.Read.All` (con consenso dell’amministratore) e un certificato per l’accesso. `Tasks.Read.All` concede l’accesso in lettura alle attività di tutti gli utenti; rimuovete l’autorizzazione o la registrazione dell’app dopo aver rilevato il contenuto.

```powershell
$verbindung = @{
    ClientId              = '<App-ID>'
    TenantId              = '<Tenant-ID>'
    CertificateThumbprint = '<Zertifikat-Fingerabdruck>'
}
Connect-MgGraph @verbindung -NoWelcome

$benutzer = Get-MgUser -All -Filter "accountEnabled eq true" -Property Id, UserPrincipalName

$bestand = foreach ($b in $benutzer) {
    try {
        $listen = Get-MgUserTodoList -UserId $b.Id -All -ErrorAction Stop
    }
    catch {
        # Nessuna cassetta postale Exchange Online o nessun accesso: ignora l'utente
        continue
    }
    foreach ($l in $listen) {
        $aufgaben = @(Get-MgUserTodoTask -UserId $b.Id -TodoTaskListId $l.Id -All)
        [pscustomobject]@{
            Benutzer = $b.UserPrincipalName
            Liste    = $l.DisplayName
            Typ      = $l.WellknownListName
            Geteilt  = $l.IsShared
            Besitzer = $l.IsOwner
            Offen    = @($aufgaben | Where-Object Status -ne 'completed').Count
            Gesamt   = $aufgaben.Count
        }
    }
}

$bestand | Export-Csv -Path .\todo-bestand.csv -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `ClientId`, `TenantId` | Registrazione dell’app e tenant presso cui lo script esegue l’accesso |
| `CertificateThumbprint` | Impronta digitale del certificato nell’archivio certificati dell’utente corrente; accesso come app senza account utente |
| `-NoWelcome` | Sopprime il messaggio di benvenuto di `Connect-MgGraph` |
| `-All` | Legge tutte le pagine di un risultato anziché solo la prima |
| `-Filter "accountEnabled eq true"` | Solo account abilitati; gli account disabilitati, come le cassette postali condivise, vengono esclusi |
| `-Property Id, UserPrincipalName` | Carica solo le due proprietà necessarie |
| `-UserId` | Utente di cui vengono interrogati elenchi o attività |
| `-ErrorAction Stop` | Rende un errore intercettabile, affinché `catch` possa ignorare l’utente |
| `-TodoTaskListId` | Elenco di cui vengono interrogate le attività |
| `WellknownListName` | `defaultList` per l’elenco standard «Attività», `flaggedEmails` per i messaggi contrassegnati, `none` per gli elenchi personalizzati |
| `IsShared` | `True` se l’elenco è condiviso con altri utenti |
| `IsOwner` | `False` se l’elenco appartiene a un altro utente ed è stato solo condiviso |
| `Status -ne 'completed'` | Conta tutte le attività non completate |
| `-NoTypeInformation` | Nessuna riga del tipo all’inizio del file CSV |
| `-Encoding UTF8` | Le dieresi nei nomi degli elenchi vengono mantenute |

</details>

Gli account senza cassetta postale Exchange Online restituiscono un errore e vengono ignorati. Nei tenant di grandi dimensioni è opportuno eseguire un test con un sottoinsieme, ad esempio con `-Top 20` anziché `-All` in `Get-MgUser`.

L’analisi del file CSV:

- **Elenchi condivisi** (`Geteilt` = `True`) mostrano dove avviene la collaborazione tramite To Do. Microsoft non trasferirà queste appartenenze; gli elenchi dovranno essere ricreati come piani Planner con le persone interessate.
- **Attività aperte** negli elenchi personali (`Typ` = `none` o `defaultList`) dovranno essere trasferite manualmente dagli utenti in Planner entro il 30 giugno 2027 oppure salvate con la funzione di esportazione annunciata.
- **Messaggi contrassegnati** (`Typ` = `flaggedEmails`) resteranno come contrassegno nell’e-mail della cassetta postale.

## Account Microsoft privati

Anche per gli account Microsoft privati, inclusi Microsoft 365 Personal e Family, le app To Do verranno ritirate il 30 giugno 2027. Planner non è disponibile per questi account. Microsoft consiglia di scegliere un’altra gestione delle attività e di salvare i dati con la funzione di esportazione annunciata. Secondo Microsoft, un’esportazione non significa che i dati possano essere importati in Planner o in un’altra applicazione. Gli utenti con account aziendale e privato devono verificare separatamente entrambi gli account.

## Preparazione per gli amministratori

1. **Accesso e licenze:** assicuratevi che tutti gli utenti To Do abbiano una licenza con Planner e che il piano di servizio `PROJECTWORKMANAGEMENT` sia attivato.

2. **Elenchi condivisi e integrazioni:** rilevate il contenuto con lo script sopra. Identificate i flussi Power Automate e le applicazioni di terze parti che utilizzano l’API To Do di Microsoft Graph. Microsoft ritirerà l’API separatamente e in seguito; secondo Microsoft, le integrazioni potrebbero comunque richiedere adeguamenti.

3. **Conservazione e localizzazione dei dati:** verificate se gli obblighi di conservazione esistenti si applicano alle attività To Do e se i criteri e gli hold di Purview acquisiscono il nuovo piano privato. Valutate se la posizione di archiviazione nella macroregione Europa è compatibile con le vostre direttive. Prima di eliminare o riorganizzare i dati delle attività, coinvolgete i responsabili della conformità.

4. **Informare gli utenti:** comunicate agli utenti che non avverrà alcuna migrazione, per quanto tempo il contenuto resterà visibile in Planner e quali funzionalità cambieranno (uso offline, promemoria, allegati, gruppi di elenchi).

5. **Monitorare il Message Center:** Microsoft comunicherà lì i dettagli della distribuzione, la funzione di esportazione e le date per il servizio e le API.

## Fonti

1.  [Moving from Microsoft To Do to Microsoft Planner](https://support.microsoft.com/en-us/planner/to-do-to-planner): annuncio con cronologia, differenze tra To Do e Planner, passaggi di preparazione e FAQ.

2.  [Review data storage and compliance in Microsoft To Do](https://support.microsoft.com/en-us/todo/review-data-storage-and-compliance-in-microsoft-to-do): archiviazione delle attività To Do come attività nella cassetta postale Exchange Online.

3.  [Frequently asked questions for admins about Microsoft Planner](https://learn.microsoft.com/en-us/planner/faq-for-planner-admins): posizioni di archiviazione delle attività (Exchange, Azure, SharePoint, Dataverse), piano di servizio `PROJECTWORKMANAGEMENT` e accesso tramite `planner.cloud.microsoft` senza licenza.

4.  [Microsoft Planner for admins](https://learn.microsoft.com/en-us/planner/planner-for-admins): abbonamenti con Planner, nonché Planner Plan 1 e Planner and Project Plan 3.

5.  [Other Microsoft 365 services data locations](https://learn.microsoft.com/en-us/microsoft-365/enterprise/m365-dr-service-other?view=o365-worldwide#planner): Planner senza impegno permanente sulla localizzazione dei dati, posizione di archiviazione «EUR1» per tenant con paese Svizzera.

6.  [Data Residency for Exchange Online](https://learn.microsoft.com/en-us/microsoft-365/enterprise/m365-dr-service-exo?view=o365-worldwide): impegno dei Product Terms per i contenuti inattivi delle cassette postali in Svizzera.

7.  [Security, privacy, and compliance in Microsoft Planner](https://learn.microsoft.com/en-us/planner/planner-security-privacy-compliance): archiviazione dei dati del piano in Azure; Planner non partecipa ad Advanced Data Residency. Stato agosto 2025, prima dell’introduzione dei criteri di conservazione per Planner.

8.  [Microsoft Planner support in Microsoft Purview](https://learn.microsoft.com/en-us/planner/planner-in-purview): eDiscovery e Legal Hold solo per attività in piani basati su gruppi, elenco dei tipi di piano non supportati.

9.  [MC1384412: Retention policies for Microsoft Planner content](https://mc.merill.net/message/MC1384412): copia del messaggio Message Center relativo ai criteri di conservazione ed eliminazione per Planner, distribuzione da luglio ad agosto 2026.

10.  [Learn about retention for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange): elementi della cassetta postale supportati; attività solo con data di scadenza.

11.  [Use the Microsoft To Do API](https://learn.microsoft.com/en-us/graph/api/resources/todo-overview?view=graph-rest-1.0): risorse dell’API To Do in Microsoft Graph (elenchi, attività, checklist, risorse collegate).

12.  [List lists](https://learn.microsoft.com/en-us/graph/api/todo-list-lists?view=graph-rest-1.0): endpoint, autorizzazioni e proprietà `isShared`, `isOwner` e `wellknownListName`.

13.  [List Todo tasks](https://learn.microsoft.com/en-us/graph/api/todotasklist-list-tasks?view=graph-rest-1.0): endpoint per le attività di un elenco e autorizzazione applicativa `Tasks.Read.All`.

14.  [Microsoft Will Retire its To Do App in June 2027](https://www.thurrott.com/cloud/344071/microsoft-will-retire-its-to-do-app-on-in-june-2027): articolo con dichiarazione successiva di Microsoft sulle integrazioni in Outlook e Teams.
