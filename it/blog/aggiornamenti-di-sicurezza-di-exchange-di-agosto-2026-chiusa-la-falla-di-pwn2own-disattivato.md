---
title: "Aggiornamenti di sicurezza di Exchange di agosto 2026: chiusa la falla di Pwn2Own, disattivato OWA Light"
navTitle: "Exchange SU 08/2026"
description: "L'SU di agosto risolve sette vulnerabilità, incluso l'exploit di Exchange dimostrato al Pwn2Own 2026, e disattiva definitivamente OWA Light. Microsoft spiega inoltre perché gli SU di Exchange vengono ora rilasciati ogni mese e perché Exchange SE CU1 continua a tardare."
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
url: https://rafaelpfister.ch/it/blog/aggiornamenti-di-sicurezza-di-exchange-di-agosto-2026-chiusa-la-falla-di-pwn2own-disattivato
translationSourceHash: a41c24b533c3b19bf6226ac5d16e7b9668d83d13b53588da7109f5567e79db51
translationModel: gpt-5.6-terra
translatedAt: 2026-08-20T04:04:09.900Z
translationReview: required
---

# Aggiornamenti di sicurezza di Exchange di agosto 2026: chiusa la falla di Pwn2Own, disattivato OWA Light

L'11 agosto 2026 Microsoft ha pubblicato aggiornamenti di sicurezza (SU) per Exchange Server, per il quarto mese consecutivo. Gli aggiornamenti risolvono sette vulnerabilità. Nessuna era stata resa pubblica in anticipo, nessuna risulta attivamente sfruttata allo stato attuale e Microsoft classifica lo sfruttamento di tutte e sette come «Exploitation Less Likely». Non si tratta comunque di un normale Patch Tuesday, per tre motivi: l'aggiornamento chiude la falla di Exchange dimostrata nella competizione di hacking Pwn2Own, **disattiva definitivamente OWA Light dopo quasi vent'anni** e il team di Exchange ha poi spiegato perché il ritmo mensile rimarrà per il momento la norma.

## Per quali versioni di Exchange è disponibile l'aggiornamento

Gli SU sono disponibili per le seguenti versioni:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121573, build 15.2.2562.46 — come aggiornamento pubblico regolarmente disponibile.
- **Exchange Server 2019 CU15**: KB5121574, build 15.2.1748.49 — solo tramite il **programma ESU Period 2**.
- **Exchange Server 2019 CU14**: KB5121575, build 15.2.1544.44 — solo tramite ESU Period 2.
- **Exchange Server 2016 CU23**: KB5121576, build 15.1.2507.72 — solo tramite ESU Period 2.

La situazione è la stessa di luglio: Exchange 2016 e 2019 sono fuori supporto. Gli SU da maggio a ottobre 2026 sono disponibili solo per chi è iscritto al programma ESU Period 2. Tutti gli altri restano senza patch, con ormai quattordici vulnerabilità aperte, alcune con valutazioni elevate: per loro la migrazione a Exchange SE non può più attendere. Exchange Online è già protetto; negli ambienti ibridi, tuttavia, l'SU deve essere installato su tutti i server Exchange, inclusi i server di sola gestione e le macchine su cui sono installati soltanto gli Exchange Management Tools.

Il noto problema dei *messaggi wrapper* nelle cassette postali condivise degli ambienti ibridi persiste anche con l'SU di agosto; secondo Microsoft, la correzione è prevista in un prossimo aggiornamento. C'è però una buona notizia nei commenti all'annuncio della release: chi ha impostato il SettingOverride documentato come workaround **non** deve ricrearlo dopo l'installazione dell'SU di agosto: l'aggiornamento lascia intatto l'override, come confermato dal team di Exchange.

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

Tre di queste meritano uno sguardo più approfondito.

**[CVE-2026-62913](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62913)** ha il punteggio più alto del mese, CVSS 8.8: una Remote Code Execution che un aggressore autenticato con privilegi ridotti può attivare senza alcuna interazione dell'utente. Come punto di partenza è sufficiente un qualsiasi account di cassetta postale compromesso: nell'epoca del phishing e del credential stuffing, «autenticato» non è una soglia elevata.

**[CVE-2026-62911](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62911)** è l'unica vulnerabilità del mese che Microsoft classifica come *Critical* (Elevation of Privilege, CVSS 8.0). Dietro il freddo numero c'è più storia di quanto sembri: alla domanda se l'exploit di Exchange dimostrato da Orange Tsai al **Pwn2Own 2026** fosse ormai risolto, il team di Exchange rimanda nei commenti all'annuncio della release proprio a questa CVE. La scoperta della competizione è quindi chiusa: un motivo in più per non rimandare l'SU di agosto, poiché le tecniche di Pwn2Own vengono solitamente pubblicate nel dettaglio dopo la scadenza degli embarghi.

**[CVE-2026-62914](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62914)** (Spoofing, CVSS 7.3) è il motivo diretto della disattivazione di OWA Light, di cui parleremo tra poco.

Le restanti falle: CVE-2026-62910 (EoP, 7.2) richiede già privilegi elevati, mentre CVE-2026-62912 (DoS), CVE-2026-62915 (Security Feature Bypass) e CVE-2026-65813 (EoP) hanno un CVSS di 6.5. Come di consueto, i dettagli sono disponibili nella Security Update Guide (filtro «Server Software» per Exchange SE oppure «ESU» per 2016/2019).

## OWA Light: dopo quasi vent'anni è finita

### Cosa cambia con l'aggiornamento

Con l'installazione dell'SU di agosto, **OWA Light viene disattivato definitivamente** su ogni server che riceve l'aggiornamento, o uno successivo. Chi apre l'interfaccia Light verrà ora reindirizzato al normale Outlook on the web. La disattivazione fa parte dell'aggiornamento stesso e non può essere annullata tramite un'opzione; Microsoft l'aveva annunciata alcune settimane prima in un apposito post del blog.

OWA Light risale all'epoca di Exchange 2007: un'interfaccia web volutamente essenziale, pensata come fallback per browser datati e connessioni lente, ufficialmente deprecata dall'agosto 2024. La motivazione della fine è dettata dalla sicurezza: un percorso di rendering legacy separato accanto al moderno OWA aumenta la complessità e quindi la superficie di attacco; CVE-2026-62914 ne fornisce la prova concreta. Chi ha letto l'[articolo di luglio](/blog/exchange-security-updates-juli-2026) ricorderà inoltre che la mitigazione di CVE-2026-42897 di maggio aveva già reso OWA Light inutilizzabile come effetto collaterale. L'interfaccia era quindi già a un passo dalla fine.

### Per chi non può applicare la patch: disattivare manualmente OWA Light

Importante per tutti coloro che non possono (ancora) installare l'SU di agosto, ad esempio perché manca l'abilitazione ESU: Microsoft raccomanda espressamente di **disattivare manualmente** OWA Light in questo caso, per mitigare CVE-2026-62914. È possibile farlo tramite la mailbox policy OWA e la pagina di accesso:

```powershell
Get-OwaMailboxPolicy | Set-OwaMailboxPolicy -OWALightEnabled $false
Get-OwaVirtualDirectory | Set-OwaVirtualDirectory -LogonPageLightSelectionEnabled $false
```

Il primo comando disattiva la versione Light per tutte le cassette postali della rispettiva policy, il secondo rimuove l'opzione «Usa la versione Light» dalla pagina di accesso OWA. Le modifiche alla directory virtuale OWA diventano affidabili solo dopo il riciclo dell'application pool OWA o un `iisreset`.

### Cosa dovrebbero verificare ora gli amministratori

La disattivazione è tecnicamente banale, ma non sempre dal punto di vista organizzativo: OWA Light era un silenzioso salvagente per scenari di nicchia. Vale ora la pena verificare segnalibri e istruzioni dell'help desk che hanno `?layout=light` codificato in modo fisso, dispositivi kiosk e terminali con browser datati, nonché istruzioni interne per utenti che usavano la versione Light per motivi di accessibilità. Il moderno Outlook on the web funziona in tutti i browser attuali e offre proprie funzioni di accessibilità, ma chi non informa in anticipo gli utenti interessati genererà ticket.

## Perché ora arriva un SU ogni mese e dove resta Exchange SE CU1

Due giorni dopo la release, il team di Exchange ha risposto in un post del blog notevolmente aperto («Where is Exchange SE CU1 anyway?») alla domanda che molti amministratori si pongono. In breve: Microsoft utilizza strumenti di IA a livello aziendale per individuare vulnerabilità nei propri prodotti. I team, incluso quello di Exchange, stanno attualmente gestendo le segnalazioni: validandole, riproducendole, correggendole, testandole per regressioni e distribuendole ogni mese. Da maggio 2026 è così uscito un SU di Exchange ogni mese e Microsoft afferma esplicitamente che questo ritmo elevato continuerà.

Il tanto atteso **CU1 per Exchange SE** è ritardato proprio per questo motivo. Annunciato inizialmente per la prima metà del 2026, poi rinviato alla seconda, non ha ormai più una data obiettivo. Microsoft intende pubblicare CU1 solo quando ci sarà un mese senza un rilascio di sicurezza urgente nel mezzo: un CU che viene subito superato da un SU comporterebbe per molte organizzazioni un doppio lavoro di aggiornamento. Nel frattempo, il payload di sicurezza mensile confluisce continuamente nella build interna di CU1.

Per la pratica, questo significa due cose. Primo: attendere CU1 non è una strategia, né per la migrazione a SE né per l'installazione degli SU. Secondo: una **finestra di manutenzione mensile** per Exchange deve entrare stabilmente nel calendario operativo, come avviene da tempo per i server Windows.

## Installazione e attività successive

La procedura rimane quella collaudata: prima inventariare con l'[Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) quali server si trovano a quale livello CU/SU e se sono ancora necessari passaggi manuali. Quindi installare l'SU (se il livello CU è obsoleto, l'[Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) mostra il percorso), riavviare il server e verificare che tutti i servizi Exchange siano stati avviati correttamente. Se dei servizi risultano *disabilitati*, l'installazione è stata interrotta: in questo caso aiuta il workaround documentato nell'articolo di supporto Microsoft sull'errore di versione del file, oppure lo [script SetupAssist](https://aka.ms/ExSetupAssist). Infine, eseguire nuovamente l'Health Checker.

Gli SU sono cumulativi: chi ha saltato l'SU di luglio installa direttamente quello di agosto. E per gli ambienti ibridi vale l'aggiunta nota: se il certificato di autenticazione viene cambiato dopo l'installazione dell'SU, è opportuno eseguire nuovamente Hybrid Configuration Wizard.

Rimane attuale un'attività successiva di luglio: chi ha ancora attiva la mitigazione CVE-2026-42897 (M2.1.0) dovrebbe ora rimuoverla. La procedura corretta è descritta nell'[articolo sull'SU di luglio](/blog/exchange-security-updates-juli-2026).

## Procedura consigliata

In breve: installare tempestivamente l'SU di agosto su tutti i server Exchange e sulle macchine con gli strumenti di gestione: la falla di Pwn2Own e la RCE da 8.8 sono motivi sufficienti per non aspettare il prossimo Patch Tuesday. Chi non può applicare subito la patch disattiva manualmente OWA Light come misura immediata contro CVE-2026-62914. Prima della disattivazione di OWA Light, identificare e informare i gruppi di utenti interessati (vecchi segnalibri, browser kiosk, flussi di lavoro di accessibilità). Successivamente, eseguire l'Health Checker, completare le attività aperte di luglio e pianificare una finestra di manutenzione mensile per Exchange, poiché il ritmo rimarrà tale.

## Fonti

1.  [Released: August 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-august-2026-exchange-server-security-updates/4543951): Annuncio ufficiale della release con versioni supportate, nota su OWA Light, problemi noti e FAQ; nei commenti, le conferme sulla correzione Pwn2Own (CVE-2026-62911) e sul persistente SettingOverride dei messaggi wrapper.

2.  [Upcoming retirement of OWA Light in Exchange Server – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/upcoming-retirement-of-owa-light-in-exchange-server/4534943): L'annuncio anticipato della disattivazione, incluso il consiglio di Microsoft di disattivare manualmente OWA Light in assenza della patch.

3.  [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): Il team di Exchange sulla ricerca delle vulnerabilità basata sull'IA, sul persistente ritmo mensile degli SU e sul ritardo di CU1.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Riferimento per i numeri di build degli SU di agosto.

5.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Condizioni e durata, da maggio a ottobre 2026, del programma ESU.

6.  [Wrapper messages appear in shared mailbox inbox in hybrid environments – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Il problema ibrido noto da giugno, incluso il workaround SettingOverride.

7.  [Neue Sicherheitsupdates für Exchange Server (August 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-august-2026/): Analisi in lingua tedesca delle sette CVE con valori CVSS e build.

8.  [Set-OwaMailboxPolicy (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-owamailboxpolicy): Il parametro `OWALightEnabled` per disattivare manualmente la versione Light.

9.  [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventario dei livelli CU/SU e dei passaggi manuali aperti prima e dopo l'installazione.
