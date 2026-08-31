---
title: "Aggiornamenti di sicurezza di Exchange di agosto 2026: chiusa la falla Pwn2Own, disattivato OWA Light"
navTitle: "Exchange SU 08/2026"
description: "L'SU di agosto corregge sette vulnerabilità, incluso l'exploit di Exchange dimostrato al Pwn2Own 2026, e disattiva definitivamente OWA Light. Microsoft spiega inoltre perché gli SU di Exchange vengono ora rilasciati mensilmente e perché Exchange SE CU1 continua a farsi attendere."
date: "2026-08-19"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min di lettura"
themen:
  - exchange-updates
produkte:
  - "exchange"
protokolle:
  - "releases"
  - "powershell"
slug: "aggiornamenti-di-sicurezza-di-exchange-di-agosto-2026-chiusa-la-falla-di-pwn2own-disattivato"
translationId: "article-b07bfd4074212673"
draft: false
translationOf: exchange-security-updates-august-2026
translationSourceHash: 41e10101798a88902017688d719457fce48959ba3acd2b3f1c757867b1b368d7
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T09:58:57.516Z
translationReview: required
url: https://rafaelpfister.ch/it/blog/aggiornamenti-di-sicurezza-di-exchange-di-agosto-2026-chiusa-la-falla-di-pwn2own-disattivato
---

# Aggiornamenti di sicurezza di Exchange di agosto 2026: chiusa la falla Pwn2Own, disattivato OWA Light

L'11 agosto 2026 Microsoft ha pubblicato aggiornamenti di sicurezza (SU) per Exchange Server, per il quarto mese consecutivo. Gli aggiornamenti correggono sette vulnerabilità. Nessuna era nota pubblicamente in anticipo, nessuna risulta attivamente sfruttata allo stato attuale e Microsoft classifica lo sfruttamento di tutte e sette come «Exploitation Less Likely». Non si tratta comunque di un Patch Tuesday ordinario, per tre motivi: l'aggiornamento corregge la vulnerabilità di Exchange dimostrata nella competizione di hacking Pwn2Own, **disattiva definitivamente OWA Light dopo quasi vent'anni** e il team di Exchange ha successivamente spiegato perché il ritmo mensile resterà per il momento la norma.

## Per quali versioni di Exchange è disponibile l'aggiornamento

Gli SU sono disponibili per le seguenti versioni:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121573, build 15.2.2562.46; come aggiornamento pubblico regolarmente disponibile.
- **Exchange Server 2019 CU15**: KB5121574, build 15.2.1748.49; solo tramite il **programma Period 2 ESU**.
- **Exchange Server 2019 CU14**: KB5121575, build 15.2.1544.44; solo tramite Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121576, build 15.1.2507.72; solo tramite Period 2 ESU.

La situazione è la stessa di luglio: Exchange 2016 e 2019 sono fuori supporto. Gli SU da maggio a ottobre 2026 sono disponibili solo per chi è iscritto al programma Period 2 ESU. Tutti gli altri rimangono senza patch, con ormai quattordici vulnerabilità aperte, alcune con valutazioni elevate; il passaggio a Exchange SE non può più essere rimandato. Exchange Online è già protetto; negli ambienti ibridi, tuttavia, l'SU deve essere installato su tutti i server Exchange, inclusi i server dedicati esclusivamente alla gestione e le macchine su cui sono installati solo gli Exchange Management Tools.

Il problema noto dei *messaggi wrapper* nelle cassette postali condivise degli ambienti ibridi persiste anche con l'SU di agosto; secondo Microsoft, la correzione è prevista per un aggiornamento futuro. C'è almeno una rassicurazione nella sezione commenti dell'annuncio di rilascio: chi ha impostato il SettingOverride documentato come workaround non deve ricrearlo dopo l'installazione dell'SU di agosto. Come confermato dal team di Exchange, l'aggiornamento lascia invariato l'override.

## Panoramica delle sette vulnerabilità

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-62913 | Remote Code Execution | 8.8 |
| CVE-2026-62911 | Elevation of Privilege | 8.0 |
| CVE-2026-62914 | Spoofing | 7.3 |
| CVE-2026-62910 | Elevation of Privilege | 7.2 |
| CVE-2026-62912 | Denial of Service | 6.5 |
| CVE-2026-62915 | Security Feature Bypass | 6.5 |
| CVE-2026-65813 | Elevation of Privilege | 6.5 |

Tre meritano un'analisi più approfondita.

**[CVE-2026-62913](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62913)** ha, con CVSS 8.8, il valore più alto del mese: una Remote Code Execution che un aggressore autenticato con privilegi minimi può attivare senza alcuna interazione dell'utente. Come punto di partenza basta un qualsiasi account di cassetta postale compromesso; nell'era del phishing e del credential stuffing, «autenticato» non rappresenta una barriera elevata.

**[CVE-2026-62911](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62911)** è l'unica vulnerabilità del mese che Microsoft classifica come *Critical* (Elevation of Privilege, CVSS 8.0). Dietro questo numero sobrio c'è più storia di quanto appaia: alla domanda se l'exploit di Exchange dimostrato da Orange Tsai al **Pwn2Own 2026** fosse stato nel frattempo corretto, il team di Exchange rimanda nella sezione commenti dell'annuncio di rilascio proprio a questa CVE. La scoperta della competizione è quindi corretta: un motivo in più per non rimandare l'SU di agosto, poiché le tecniche Pwn2Own vengono solitamente pubblicate in dettaglio dopo la scadenza degli embarghi.

**[CVE-2026-62914](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62914)** (Spoofing, CVSS 7.3) è il motivo diretto della disattivazione di OWA Light, di cui parleremo subito.

Le altre falle: CVE-2026-62910 (EoP, 7.2) richiede già privilegi elevati, CVE-2026-62912 (DoS), CVE-2026-62915 (Security Feature Bypass) e CVE-2026-65813 (EoP) hanno un CVSS di 6.5. I dettagli sono disponibili come di consueto nella Security Update Guide (filtro «Server Software» per Exchange SE oppure «ESU» per 2016/2019).

## OWA Light: dopo quasi vent'anni è finita

### Cosa cambia con l'aggiornamento

Con l'installazione dell'SU di agosto, **OWA Light viene disattivato in modo permanente**: su ogni server su cui viene installato l'aggiornamento (o uno successivo). Chi richiama l'interfaccia Light verrà d'ora in poi indirizzato al normale Outlook on the web. La disattivazione fa parte dell'aggiornamento stesso e non può essere annullata tramite un'opzione; Microsoft l'aveva annunciata alcune settimane prima in un apposito post del blog.

OWA Light risale all'era di Exchange 2007: un'interfaccia web volutamente semplice come fallback per browser datati e connessioni lente, ufficialmente deprecata dall'agosto 2024. La motivazione della fine è legata alla sicurezza: un percorso di rendering legacy separato accanto al moderno OWA aumenta la complessità e quindi la superficie di attacco; CVE-2026-62914 ne fornisce la prova concreta. Chi ha letto l'[articolo di luglio](/blog/exchange-security-updates-juli-2026) ricorderà inoltre che la mitigazione CVE-2026-42897 di maggio aveva già reso OWA Light inutilizzabile come effetto collaterale. L'interfaccia era quindi già sul punto di essere ritirata.

### Per chi non può applicare la patch: disattivare manualmente OWA Light

Importante per tutti coloro che non possono (ancora) installare l'SU di agosto, ad esempio perché manca l'abilitazione ESU: Microsoft raccomanda esplicitamente in questo caso di **disattivare manualmente OWA Light**, per mitigare CVE-2026-62914. Si può fare tramite la policy delle cassette postali OWA e la pagina di accesso:

```powershell
Get-OwaMailboxPolicy | Set-OwaMailboxPolicy -OWALightEnabled $false
Get-OwaVirtualDirectory | Set-OwaVirtualDirectory -LogonPageLightSelectionEnabled $false
```

Il primo comando disattiva la versione Light per tutte le cassette postali della relativa policy; il secondo rimuove dalla pagina di accesso OWA l'opzione «Usa versione Light». Le modifiche alla directory virtuale OWA vengono applicate in modo affidabile solo dopo il riciclo dell'app pool OWA o un `iisreset`.

### Cosa dovrebbero verificare ora gli amministratori

La disattivazione è tecnicamente banale, ma non sempre dal punto di vista organizzativo: OWA Light era la soluzione di ripiego silenziosa per scenari di nicchia. Occorre ora verificare segnalibri e istruzioni dell'help desk che hanno `?layout=light` codificato in modo fisso, dispositivi kiosk e terminali con browser obsoleti, nonché istruzioni interne per utenti e utentesse che utilizzavano la versione Light per motivi di accessibilità. Il moderno Outlook on the web funziona in tutti i browser attuali e offre proprie funzionalità di accessibilità; ma chi non informa preventivamente gli utenti interessati genererà ticket.

## Perché ora arriva un SU ogni mese e dove resta Exchange SE CU1

Due giorni dopo il rilascio, il team di Exchange ha risposto in un post del blog notevolmente aperto («Where is Exchange SE CU1 anyway?») alla domanda che molti amministratori si pongono. In breve: Microsoft utilizza strumenti di IA in tutta l'azienda per individuare vulnerabilità nei propri prodotti. I team, incluso quello di Exchange, stanno attualmente elaborando le segnalazioni: convalidarle, riprodurle, correggerle, testarle per regressioni e distribuirle mensilmente. Da maggio 2026 è quindi uscito un SU di Exchange ogni mese, e Microsoft afferma esplicitamente che questo ritmo elevato continuerà.

Il tanto atteso **CU1 per Exchange SE** è ritardato proprio per questo motivo. Inizialmente annunciato per il primo semestre del 2026, poi rinviato al secondo, non ha ormai più una data obiettivo. Microsoft intende pubblicare CU1 solo quando ci sarà un mese senza un urgente rilascio di sicurezza nel mezzo; un CU subito superato da un SU comporterebbe per molte organizzazioni un doppio lavoro di aggiornamento. Nel frattempo, il carico di sicurezza mensile confluisce continuamente nella build interna di CU1.

Per la pratica questo significa due cose. Primo: aspettare CU1 non è una strategia, né per la migrazione a SE né per l'installazione degli SU. Secondo: una **finestra di manutenzione mensile** per Exchange entra da subito stabilmente nel calendario operativo, proprio come è ormai normale da tempo per i server Windows.

## Installazione e attività successive

La procedura resta quella collaudata: prima fare l'inventario con l'[Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) per vedere quali server si trovano a quale livello CU/SU e se sono ancora necessari passaggi manuali. Poi installare l'SU (se il livello CU è obsoleto, l'[Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) mostra il percorso), riavviare il server e verificare che tutti i servizi Exchange siano stati avviati correttamente. Se alcuni servizi risultano *disabilitati*, l'installazione è stata interrotta; in quel caso è utile il workaround documentato nell'articolo del supporto Microsoft relativo all'errore di versione del file oppure lo [script SetupAssist](https://aka.ms/ExSetupAssist). Infine, eseguire nuovamente l'Health Checker.

Gli SU sono cumulativi: chi ha saltato l'SU di luglio può installare direttamente quello di agosto. E per gli ambienti ibridi vale l'aggiunta nota: se dopo l'installazione dell'SU viene sostituito il certificato Auth, è consigliabile eseguire nuovamente l'Hybrid Configuration Wizard.

Resta attuale un'attività successiva di luglio: chi ha ancora attiva la mitigazione CVE-2026-42897 (M2.1.0) dovrebbe rimuoverla ora; la procedura corretta è descritta nell'[articolo sull'SU di luglio](/blog/exchange-security-updates-juli-2026).

## Procedura consigliata

In breve: installare tempestivamente l'SU di agosto su tutti i server Exchange e sulle macchine con Management Tools: la falla Pwn2Own e l'RCE da 8.8 sono motivi sufficienti per non attendere il prossimo Patch Tuesday. Chi non può applicare subito la patch può disattivare manualmente OWA Light come misura immediata contro CVE-2026-62914. Prima della disattivazione di OWA Light, identificare e informare i gruppi di utenti interessati (vecchi segnalibri, browser kiosk, flussi di lavoro per l'accessibilità). Quindi eseguire l'Health Checker, completare le attività aperte di luglio e pianificare una finestra di manutenzione mensile per Exchange, perché il ritmo rimane questo.

## Fonti

1.  [Released: August 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-august-2026-exchange-server-security-updates/4543951): Annuncio ufficiale del rilascio con versioni supportate, indicazione su OWA Light, problemi noti e FAQ; nei commenti, le conferme sulla correzione Pwn2Own (CVE-2026-62911) e sul SettingOverride wrapper ancora esistente.

2.  [Upcoming retirement of OWA Light in Exchange Server – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/upcoming-retirement-of-owa-light-in-exchange-server/4534943): L'annuncio preliminare della disattivazione, inclusa la raccomandazione di Microsoft di disattivare manualmente OWA Light in assenza di patch.

3.  [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): Il team di Exchange sulla ricerca di vulnerabilità supportata dall'IA, sul ritmo mensile persistente degli SU e sul ritardo di CU1.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Riferimento per i numeri di build degli SU di agosto.

5.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Condizioni e durata (da maggio a ottobre 2026) del programma ESU.

6.  [Wrapper messages appear in shared mailbox inbox in hybrid environments – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Il problema ibrido noto da giugno, incluso il workaround SettingOverride.

7.  [Neue Sicherheitsupdates für Exchange Server (August 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-august-2026/): Analisi in lingua tedesca delle sette CVE con valori CVSS e build.

8.  [Set-OwaMailboxPolicy (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-owamailboxpolicy): Il parametro `OWALightEnabled` per la disattivazione manuale della versione Light.

9.  [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventario dei livelli CU/SU e dei passaggi manuali ancora aperti prima e dopo l'installazione.
