---
title: "Facilitator in Microsoft Teams: cosa può fare e cosa non può fare l'agente per riunioni"
navTitle: "Teams Facilitator"
description: "Facilitator è un agente Copilot che, nelle riunioni Teams pianificate, redige note condivise, gestisce l'agenda, monitora il tempo e risponde alle domande in modo visibile a tutti. L'articolo illustra funzioni, requisiti di licenza e amministrativi, scenari d'uso sensati e limiti relativi a tipi di riunione, lingua e conformità."
date: "2026-11-27"
kategorie: "Microsoft Teams"
timeToRead: "8 min di lettura"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
protokolle:
  - "lizenzierung"
  - "powershell"
hauptthema: "microsoft-teams"
slug: "facilitator-in-microsoft-teams-cosa-puo-fare-e-cosa-non-puo-fare-l-agente-per-riunioni"
translationId: "article-33919f700e0a6587"
translationOf: teams-facilitator
url: https://rafaelpfister.ch/it/blog/facilitator-in-microsoft-teams-cosa-puo-fare-e-cosa-non-puo-fare-l-agente-per-riunioni
translationSourceHash: 796c74d7d8547585c4342621dfa84e867cf70a2cbc477de7f8e82cd88cbc2d0c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-27T09:40:27.422Z
translationReview: automatic
---

Da quando Microsoft sta distribuendo Facilitator nelle riunioni Teams, in molti tenant compare una nuova voce nelle opzioni della riunione: «Allow Copilot and Facilitator». Il nome non chiarisce cosa vi sia dietro. Facilitator è un agente di IA che partecipa alla riunione e lavora in modo visibile a tutti. Da ciò derivano sia le possibilità di utilizzo sia i limiti.

## Cos'è Facilitator

Facilitator è un agente di Microsoft 365 Copilot che, in una riunione Teams pianificata, svolge il ruolo di assistente alla moderazione. Durante la riunione scrive note in una pagina Loop condivisa, che tutti i partecipanti interni possono leggere e modificare. Legge l'agenda dall'invito, mostra un timer dell'agenda sul palco della riunione e reagisce alle @menzioni nella chat della riunione.

La differenza rispetto alle altre funzioni di IA in Teams sta nella visibilità:

| Funzione | Chi vede il risultato | Momento |
|---|---|---|
| Copilot nelle riunioni Teams | solo la persona che pone la domanda | durante e dopo la riunione |
| Riepilogo intelligente | partecipanti con licenza Teams Premium o Copilot | dopo la riunione |
| Facilitator | tutti i partecipanti interni | durante la riunione, note anche dopo |

Una domanda a Copilot rimane privata. Una domanda a Facilitator e la relativa risposta compaiono nella chat della riunione per tutti. Facilitator lavora quindi per tutti i partecipanti congiuntamente.

## Panoramica per gli utenti

Per tutte le funzioni vale quanto segue: la riunione è una riunione Teams pianificata della propria organizzazione, senza partecipanti esterni; nelle opzioni della riunione è attivata l'impostazione «Allow Copilot and Facilitator»; al momento dell'avvio è stata selezionata la lingua parlata. La licenza Copilot è necessaria solo a chi avvia Facilitator o gli assegna compiti. Tutti gli altri partecipanti interni vedono i risultati senza una propria licenza Copilot.

| Funzione | Cosa consente di fare | Chi può utilizzarla | Ulteriori requisiti |
|---|---|---|---|
| Avviare Facilitator | Attivare l'agente prima o durante la riunione | Persone con licenza Microsoft 365 Copilot | Loop in Teams attivato, app «Facilitator» non bloccata |
| Note in tempo reale | Decisioni, questioni aperte e attività vengono registrate continuamente nella scheda «Notes»; è possibile correggerle e integrarle direttamente | Lettura e modifica: tutti i partecipanti interni | La trascrizione è in esecuzione (si avvia automaticamente); viene rilevato solo ciò che viene detto nella lingua selezionata, non i messaggi di chat |
| Agenda e spunta dei punti | L'elenco dei punti all'ordine del giorno dall'invito compare nella chat, i punti discussi vengono spuntati | Visualizzazione: tutti i partecipanti interni | L'agenda è presente nell'invito oppure viene fornita tramite @Facilitator nella chat; invito non crittografato; oltre alla persona organizzatrice, nella riunione è presente almeno un'altra persona |
| Timer e promemoria | Conto alla rovescia per un punto, avviso a metà riunione, prima della fine e in caso di sforamento | Richiedere il timer: licenza Copilot; visualizzare: tutti i partecipanti interni | nessuno |
| Domande sulla riunione | Porre domande tramite @Facilitator nella chat, ad esempio «Cosa abbiamo deciso al punto 2?»; domanda e risposta sono visibili a tutti | Porre domande: licenza Copilot; leggere le risposte: tutti i partecipanti interni | Domande dopo la fine della riunione solo con l'opzione «Durante e dopo la riunione» |
| Domande con ricerca sul Web | Breve ricerca durante la riunione, ad esempio su una versione del prodotto | Licenza Copilot | Ricerca sul Web per Copilot non disattivata nel tenant |
| Creare un documento Word | Generare dalle note un documento, ad esempio come bozza di verbale | Licenza Copilot | Salvataggio in OneDrive della persona richiedente |
| Creare una presentazione PowerPoint | Generare una presentazione dalla riunione | Licenza Copilot | Public Preview: partecipazione a Teams Public Preview |
| Trasferire attività in Planner | Trasferire le attività rilevate in Planner con «Accept to sync» | Licenza Copilot | Public Preview: partecipazione a Teams Public Preview |
| Note in riunioni in presenza (smartphone) | Documentare una conversazione in presenza con l'app Teams sullo smartphone | Licenza Copilot | App Teams sullo smartphone |
| Note in Teams Room | Visualizzare chat, note e timer sullo schermo della sala riunioni | Licenza Copilot solo per la persona organizzatrice | Teams Rooms on Windows; per riunioni in presenza non pianificate Teams Rooms Pro (Public Preview) |

## Funzioni e stato

Microsoft elenca le funzioni nella documentazione amministrativa separatamente in base alla disponibilità generale e alla Public Preview (stato: novembre 2026):

| Funzione | Stato |
|---|---|
| Note in tempo reale nella scheda «Notes» (pagina Loop, modificabile congiuntamente) | disponibilità generale |
| Rispondere a domande in base al contenuto della riunione e tramite ricerca sul Web | disponibilità generale |
| Timer dell'agenda con avvisi a metà riunione e prima della fine | disponibilità generale |
| Creare un documento Word dalla riunione | disponibilità generale |
| @menzione di Facilitator nella chat della riunione | disponibilità generale |
| Note sullo smartphone per riunioni in presenza | disponibilità generale |
| Teams Rooms: chat, note e timer nella sala | disponibilità generale |
| Rilevare attività e sincronizzarle con Planner | Public Preview |
| Creare una presentazione PowerPoint dalla riunione | Public Preview |
| Note per riunioni in presenza non pianificate in Teams Rooms | Public Preview |

Per quanto riguarda l'agenda: se nell'invito è presente un elenco di punti all'ordine del giorno, Facilitator lo acquisisce, lo pubblica nella chat e spunta i punti non appena vengono discussi. La spunta avviene a intervalli, non immediatamente. Se la riunione supera l'orario di fine previsto, lo segnala. Su richiesta, imposta anche un conto alla rovescia per un singolo punto.

Per quanto riguarda le note: Facilitator riassume decisioni, questioni aperte e attività. La base è esclusivamente quanto pronunciato tramite la trascrizione. Ciò che compare solo nella chat della riunione non confluisce nelle note. Le note possono essere corrette e integrate manualmente in qualsiasi momento; Facilitator continua a scrivere nella stessa pagina.

## Requisiti

Per il funzionamento devono essere soddisfatte contemporaneamente più condizioni:

- **Licenza:** chi aggiunge Facilitator a una riunione o lo attiva durante la riunione necessita di una licenza Microsoft 365 Copilot, oltre a una licenza Microsoft 365 e Teams idonea. I partecipanti senza licenza Copilot non possono porre domande a Facilitator, ma vedono le note, le domande degli altri e le sue risposte.
- **Loop:** le esperienze Loop in Teams devono essere attivate nel tenant. Senza Loop non ci sono note.
- **Trascrizione:** attivando Facilitator, la trascrizione si avvia automaticamente. Se l'opzione della riunione «Allow Copilot and Facilitator» è impostata su «Only during the meeting», Facilitator opera senza una trascrizione salvata; in tal caso non sono possibili domande dopo la fine della riunione.
- **Abilitazione dell'app:** l'app «Facilitator» è consentita per impostazione predefinita, a meno che nel Teams Admin Center non siano bloccate tutte le app.

Nell'invito, attivate Facilitator in **Options** > **Copilot and other AI** con l'impostazione «Allow Copilot and Facilitator». In alternativa, può essere attivato durante la riunione dal menu Copilot.

## Facilitator è disattivato

Nella riunione, Teams mostra nel menu «IA della riunione» tre interruttori: Copilot, Facilitator e riepilogo intelligente. Se tutti e tre sono disattivati e compare l'avviso «Non hai accesso all'IA in questa riunione», la causa è la licenza o la riunione stessa. Se è disattivato solo Facilitator, la causa è l'app o Loop.

| Sintomo | Possibile causa | Verifica |
|---|---|---|
| Tutti e tre gli interruttori sono grigi | L'account non dispone né di una licenza Microsoft 365 Copilot né di Teams Premium | Microsoft 365 Admin Center > Utenti > Licenze. Una licenza assegnata di recente spesso diventa effettiva in Teams solo dopo alcune ore o un nuovo accesso |
| Tutti e tre gli interruttori sono grigi | Riunione immediata («Meet now»), riunione avviata dalla chat, riunione di canale o chiamata | Creare la riunione dal calendario come riunione Teams pianificata |
| Tutti e tre gli interruttori sono grigi | Riunione di un'altra organizzazione, partecipazione come persona esterna o ospite | Facilitator non è disponibile nelle riunioni esterne |
| Tutti e tre gli interruttori sono grigi | L'opzione della riunione «Consenti Copilot e Facilitator» è impostata su «Disattivato», impostata dalla persona organizzatrice, da un modello di riunione o da un'etichetta di riservatezza | Opzioni della riunione > Copilot e altra IA |
| Tutti e tre gli interruttori sono grigi | Il criterio per le riunioni Teams blocca Copilot o la trascrizione | Teams Admin Center > Riunioni > Criteri per le riunioni |
| Solo Facilitator è grigio | App «Facilitator» bloccata, ad esempio perché tutte le app sono bloccate | Teams Admin Center > App Teams > Gestisci app |
| Solo Facilitator è grigio o non ci sono note | Loop per le note della riunione disattivato | `Get-SPOTenant` mostra `IsCollabMeetingNotesFluidEnabled` |

La licenza è la causa più frequente. Senza una licenza Microsoft 365 Copilot, l'interruttore Facilitator rimane grigio in ogni riunione, indipendentemente dalle opzioni della riunione e dai criteri.

## Per cosa è adatto Facilitator

Facilitator offre il massimo vantaggio nelle riunioni in cui altrimenti qualcuno redigerebbe il verbale e non potrebbe quindi partecipare alla discussione. Esempi tipici:

- **Riunioni di progetto e di stato con un elenco fisso di punti all'ordine del giorno:** il timer dell'agenda e gli avvisi a metà riunione mantengono il rispetto del programma, mentre le note contengono decisioni e punti aperti senza un verbale separato.
- **Workshop con decisioni:** la pagina Loop condivisa è visibile durante la riunione. I partecipanti correggono direttamente le formulazioni, anziché concordare un verbale successivamente via e-mail.
- **Domande durante la riunione:** «Cosa abbiamo deciso al punto 2?» oppure una breve ricerca sul Web su una versione del prodotto ricevono risposta da Facilitator nella chat, visibile a tutti.
- **Riepilogo successivo:** su richiesta, Facilitator crea dalle note un documento Word, salvato nel OneDrive della persona richiedente.
- **Riunioni in presenza:** con l'app Teams sullo smartphone o in una Teams Room vengono documentate anche le riunioni in loco.

## Cosa non può fare Facilitator

I limiti sono descritti chiaramente nella documentazione Microsoft e riguardano soprattutto i tipi di riunione, la lingua e la conformità.

**Tipi di riunione non supportati.** Facilitator funziona solo nelle riunioni Teams pianificate. Non sono supportate riunioni immediate («Meet now»), riunioni avviate da una chat, chiamate Teams, riunioni di canale nonché chat 1:1 e di gruppo. La documentazione non menziona il supporto per webinar e Town Hall.

**Nessun partecipante esterno.** Nelle riunioni e nelle chat con persone esterne, le note generate dall'IA non sono disponibili. I partecipanti esterni in linea di principio non vedono le note. Facilitator non è quindi utilizzabile come strumento di verbalizzazione per riunioni con clienti o fornitori.

**Una lingua per riunione.** All'attivazione viene selezionata la lingua parlata. Le note vengono create solo per i passaggi in questa lingua. In una riunione che alterna tedesco e francese, le parti in francese mancano. Se la lingua selezionata differisce da quella parlata, non vengono create note. Il tedesco svizzero non è elencato come lingua a sé stante nell'elenco delle lingue Copilot supportate.

**Nessuna registrazione.** Facilitator attiva la trascrizione, ma non la registrazione. Chi necessita di immagini e audio deve avviare separatamente la registrazione.

**Nessuna agenda da inviti protetti.** Se l'invito è crittografato con un'etichetta di riservatezza, Facilitator non può leggere l'agenda. Lo stesso vale finché nella riunione è presente solo la persona organizzatrice, poiché non legge la sua copia dell'invito.

**Nessun sostituto di un verbale formale.** Le note sono un riepilogo IA della trascrizione. Per riunioni del consiglio di amministrazione, colloqui del personale o altre riunioni con rilevanza legale, è comunque necessario un verbale verificato e approvato.

## Archiviazione dei dati e conformità

Le note sono archiviate come file `.loop` nella cartella «Meetings» di OneDrive della persona che ha avviato Facilitator. Microsoft le tratta come dati di trascrizione. Per i responsabili della conformità sono rilevanti tre limitazioni:

- **eDiscovery:** le note delle riunioni generate dall'IA non vengono acquisite automaticamente come allegati cloud in Microsoft Purview eDiscovery.
- **Conservazione:** le etichette di conservazione non sono supportate per gli allegati cloud nelle note generate dall'IA.
- **Etichette di riservatezza:** le note non ereditano l'etichetta della riunione. È possibile applicare successivamente un'etichetta al componente Loop. Impostazioni quali «Prevent copy and paste» e filigrane non si applicano alle risposte e alle note di Facilitator.

Per le organizzazioni soggette a obblighi di conservazione, ciò significa che le note che devono essere considerate documenti aziendali devono essere trasferite in un archivio con una conservazione funzionante. Microsoft descrive gli aspetti di Purview in una pagina dedicata (vedere Fonti).

## Gestire e disattivare Facilitator

Nel Teams Admin Center trovate l'app in **Teams apps** > **Manage apps** cercando «Facilitator». Da lì è possibile consentirla o bloccarla per l'intero tenant. Per singoli gruppi di utenti, gestite l'app tramite criteri di autorizzazione delle app o gestione incentrata sulle app.

Le note Loop nelle riunioni, su cui si basa Facilitator, dipendono da un'impostazione del tenant SharePoint. Questa si applica a tutti gli utenti e non può essere impostata per singola persona:

```powershell
Connect-SPOService -Url "https://contoso-admin.sharepoint.com"
Set-SPOTenant -IsCollabMeetingNotesFluidEnabled $false
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Url` | Indirizzo del SharePoint Admin Center del tenant (`<tenant>-admin.sharepoint.com`) |
| `-IsCollabMeetingNotesFluidEnabled` | `$false` disattiva le note condivise della riunione e quindi le note IA di Facilitator per tutti gli utenti, `$true` le riattiva |

</details>

Questa impostazione rimuove la funzione delle note anche per le riunioni senza Facilitator. Se deve essere bloccato solo l'agente, bloccare l'app nel Teams Admin Center è la soluzione più mirata. Per riunioni senza trascrizione salvata, impostate nel criterio della riunione l'opzione per Copilot e Facilitator su «Only during the meeting».

## Fonti

1.  [Set up Facilitator in Microsoft Teams for collaborative AI-generated notes](https://learn.microsoft.com/en-us/microsoftteams/facilitator-teams): Documentazione amministrativa con requisiti di licenza, stato delle funzioni, archiviazione dei dati, limitazioni di conformità e gestione dell'app (stato: 18.11.2026).

2.  [Facilitator in Microsoft Teams meetings](https://support.microsoft.com/en-us/teams/copilot/facilitator-in-microsoft-teams-meetings): Guida per utenti finali su opzioni della riunione, agenda, timer, tipi di riunione supportati e selezione della lingua.

3.  [Facilitator in Microsoft Teams Rooms](https://learn.microsoft.com/en-us/microsoftteams/rooms/facilitator-teams-rooms): Riunioni in presenza in Teams Rooms on Windows, licenza della persona organizzatrice, assegnazione dei partecipanti che parlano nella sala.

4.  [Keep meetings focused and productive with Facilitator in Teams](https://techcommunity.microsoft.com/blog/microsoft365insiderblog/keep-meetings-focused-and-productive-with-facilitator-in-teams/4456029): Blog Microsoft 365 Insider dell'ottobre 2025 su trascrizione, base delle note e funzionamento senza trascrizione salvata.

5.  [Microsoft Purview: Teams Facilitator](https://learn.microsoft.com/en-us/purview/ai-teams-facilitator): Prospettiva Purview sui dati di Facilitator per conservazione ed eDiscovery.

6.  [Manage Loop components in OneDrive and SharePoint](https://learn.microsoft.com/microsoft-365/loop/loop-components-configuration): Impostazioni del tenant `IsLoopEnabled` e `IsCollabMeetingNotesFluidEnabled` per componenti Loop e note delle riunioni.
