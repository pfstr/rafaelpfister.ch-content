---
title: "Hardening: baseline, confini di fiducia e controlli amministrativi"
blatt: "haertung"
description: "Hardening tecnico per amministratori di messaggistica e infrastrutture: baseline di sicurezza, funzionalità minima e privilegio minimo, livelli di identità e gestione, confini di rete e protocolli di posta, chiavi, rischi di patching e supply chain, logging, rilevamento della deriva e verifica."
fakten:
  - label: Obiettivo
    wert: Ridurre in modo controllato la superficie di attacco, la fiducia implicita e il raggio d'azione del danno
    href: https://csrc.nist.gov/pubs/sp/800/123/final
  - label: Principi di progettazione
    wert: Fail-safe Defaults · Complete Mediation · Least Privilege
    href: https://web.mit.edu/Saltzer/www/publications/protection/Basic.html
  - label: Baseline
    wert: stato target documentato, approvato e verificabile
    href: https://www.cisecurity.org/controls/secure-configuration-of-enterprise-assets-and-software
  - label: Funzionalità minima
    wert: solo funzioni, porte, protocolli, software e servizi necessari all'attività
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf
  - label: Modello di accesso
    wert: nessuna attribuzione implicita di fiducia basata esclusivamente sulla posizione in rete o sulla proprietà
    href: https://csrc.nist.gov/pubs/sp/800/207/final
  - label: Aree di controllo
    wert: Host · Identità · Gestione · Rete/protocollo · Applicazione/dati · Supply chain/telemetria
    href: https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final
  - label: Confini della posta
    wert: Internet relay · Submission/Access · consegna interna · amministrazione
    href: https://csrc.nist.gov/pubs/sp/800/177/r1/final
  - label: Accesso amministrativo
    wert: personale · con autenticazione forte · con privilegi minimi · tracciabile
    href: https://www.cisecurity.org/controls/access-control-management
  - label: Rete di gestione
    wert: zona amministrativa restrittiva separata dai flussi di dati produttivi
    href: https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure
  - label: Patching
    wert: identificare · assegnare priorità · acquisire · installare · verificare l'installazione
    href: https://csrc.nist.gov/pubs/sp/800/40/r4/final
  - label: Prova della deriva
    wert: confronto tra stato previsto ed effettivo di configurazione, servizi, account, regole e log
    href: https://www.cisecurity.org/controls/audit-log-management
  - label: Riferimenti
    wert: baseline del produttore · CIS Benchmark · BSI IT-Grundschutz · propria approvazione del rischio
    href: https://www.cisecurity.org/cis-benchmarks
werbung:
  - tools
  - newsletter
ctaThemen:
  - haertung
  - messaging
  - security
translationSourceHash: f97c0de00da2f8c1a5d89407341fb9a46b9fc213b01559d4e297feb29c0a3c83
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T11:11:54.442Z
translationReview: automatic
---

# Hardening: baseline, confini di fiducia e controlli amministrativi

L'hardening è la trasformazione controllata di un sistema in uno **stato target documentato, motivato e verificabile**. Limita funzioni, accessi e relazioni di fiducia al minimo operativo, senza compromettere in modo incontrollato il servizio previsto. Il risultato non è il più lungo elenco possibile di opzioni di sicurezza abilitate, bensì un'architettura in cui ogni interfaccia raggiungibile, ogni privilegio e ogni flusso di dati dispongono di uno scopo nominato, di un responsabile e di una prova. Di conseguenza, NIST descrive la sicurezza dei server come selezione, implementazione e manutenzione continua di controlli adeguati; CIS Control 4 richiede configurazioni sicure per asset e software ([NIST SP 800-123](https://csrc.nist.gov/pubs/sp/800/123/final), [CIS Control 4: Secure Configuration](https://www.cisecurity.org/controls/secure-configuration-of-enterprise-assets-and-software)).

Un'impostazione predefinita del prodotto non è né automaticamente insicura né automaticamente la baseline di produzione corretta. I produttori devono coprire un'ampia gamma di funzioni e compatibilità. L'operatore, invece, conosce esposizione, necessità di protezione, dipendenze, capacità di ripristino e rischi residui accettati. Una baseline combina quindi le raccomandazioni del produttore, un riferimento CIS o BSI adeguato e la propria decisione architetturale. Le deviazioni non vengono mantenute tacitamente, ma documentate con causa, rischio, controllo compensativo, responsabile e data di scadenza. I CIS Benchmarks sono raccomandazioni di configurazione basate sul consenso; il BSI separa i requisiti generali per i server da quelli relativi alla posta ([CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks), [BSI SYS.1.1 Allgemeiner Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2022/07_SYS_IT_Systeme/SYS_1_1_Allgemeiner_Server_Edition_2022.pdf?__blob=publicationFile&v=3), [BSI APP.5.3 Allgemeiner E-Mail-Client und -Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_5_3_Allgemeiner_E-Mail_Client_und_Server_Edition_2023.pdf?__blob=publicationFile&v=3)).

L'hardening inizia dal servizio reale e dai suoi percorsi amministrativi, non da un elenco arbitrario di valori di registro. Per prima cosa vengono inventariati componenti, identità, dati e percorsi di rete; da ciò derivano baseline, eccezioni e controlli verificabili.

## Dalla checklist al modello delle aree di controllo

Per gli amministratori, una verifica strutturata per aree di controllo è più solida di un'unica checklist dell'host. Il modello seguente riassume i controlli di NIST SP 800-53: gestione della configurazione e funzionalità minima, accesso e autenticazione, protezione delle comunicazioni, integrità del sistema, audit, nonché controlli di supply chain e ripristino. È un modello di verifica, non uno standard aggiuntivo ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final), [NIST SP 800-53 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)).

| Area di controllo | Oggetto protetto | Valori target tipici | Evidenza operativa |
|---|---|---|---|
| Host e runtime | Sistema operativo, container, servizi, permessi dei file, funzioni di kernel/runtime | pacchetti minimi, listener minimi, processi non privilegiati, permessi file sicuri | inventario di servizi e porte, scansione della baseline, controllo di integrità |
| Identità e autorizzazioni | Persone, account di servizio, ruoli, token, certificati | identità amministrativa personale, MFA, privilegio minimo, identità di servizio separate | revisione di account/ruoli, eventi di autenticazione e privilegi |
| Livello di gestione | GUI, API, SSH, PowerShell, SNMP, canale di backup e aggiornamento | zona amministrativa dedicata, protocolli cifrati, Default Deny, processo Break-Glass | percorsi di gestione raggiungibili, log AAA, modifiche di configurazione |
| Rete e protocolli | Listener, egress, TLS, DNS, percorsi di relay, submission e accesso | flussi espliciti per ruolo, nessun protocollo in chiaro non necessario, certificati verificati | regole firewall, test dei pacchetti/TLS, monitoraggio DNS e mailflow |
| Applicazione e dati | Coda, casella di posta, policy, parser, dati temporanei, chiavi | ruoli separati, permessi file restrittivi, impostazioni predefinite sicure, diritti limitati per parser ed egress | test negativi funzionali, log di coda/policy, revisione di segreti e chiavi |
| Supply chain e telemetria | Immagini, pacchetti, firme, dipendenze, log, tempo | artefatti supportati, origine verificata, processo di patching, log centrali non alterati | inventario, hash/firma, report di patch e deriva, test degli allarmi |

NIST Zero Trust aggiunge un confine importante: un utente, servizio o dispositivo non riceve fiducia solo perché si trova nella rete interna o appartiene all'organizzazione. Autenticazione e autorizzazione vengono valutate prima dell'accesso a una risorsa. La segmentazione resta utile, ma non sostituisce identità, policy e decisione continua ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-haertung.svg?v=20260813" title="Interaktive Infografik: Härtungsmodell für Messaging-Systeme mit externen Mailgrenzen, Managementebene, Identität, Host, Anwendung, Daten, Lieferkette, Telemetrie und Verifikationsschleife" loading="lazy">
  <a href="/images/kb-interaktiv-haertung.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

## Stack tecnologico come inventario per l'hardening

L'hardening non possiede un proprio stack di linguaggi di programmazione o prodotti. Si applica allo **stack tecnologico effettivamente utilizzato**. L'inventario comprende quindi almeno firmware e hypervisor, sistema operativo o base dei container, runtime e linguaggio di programmazione, server web e di posta, librerie e parser, database, coda e object store, componenti di identità e chiavi, protocolli di gestione e percorsi di logging e aggiornamento. NIST CM-8 richiede un inventario dei componenti di sistema; CM-7 collega tale inventario alla limitazione alle funzioni necessarie ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)).

Per ogni livello vengono registrati produttore, origine, stato di supporto, moduli attivi, privilegi, listener, destinazioni egress, fonte della configurazione, percorso di patching e oggetto di ripristino. Solo così una raccomandazione come «disattivare i servizi non necessari» può essere applicata a un processo concreto e alle sue dipendenze senza danneggiare il mailflow o la capacità di ripristino.

## Confini di fiducia di una piattaforma di messaggistica

Una piattaforma di messaggistica possiede vari percorsi di ingresso tecnicamente diversi. Non devono essere trattati con un'unica regola come «solo connessioni autenticate»:

- **Internet relay:** un MTA pubblicamente raggiungibile riceve messaggi sulla [SMTP](/kb/smtp) porta 25 da MTA non precedentemente noti. Qui la verifica dei destinatari, la policy di relay, gli stati del protocollo, i limiti di risorse, la reputazione e i controlli dei contenuti limitano il rischio; l'accesso utente non è il modello di fiducia generale.
- **Message Submission:** utenti e applicazioni consegnano nuovi messaggi come mittenti identificati. Submission separa questo ruolo dal relay; autenticazione, autorizzazione, rate limit e [TLS](/kb/tls) fanno parte della policy.
- **Accesso alla posta:** IMAP, POP o HTTP accedono ai dati esistenti delle caselle di posta. RFC 8314 considera obsoleto il testo in chiaro per submission e accesso alla posta e preferisce TLS implicito.
- **Percorsi di servizio interni:** gateway, directory, database, object store, code e scanner comunicano tra loro come servizi. La posizione in rete da sola non prova l'identità; ogni connessione necessita di un percorso minimo e direzionato di dati e autorizzazioni.
- **Gestione e aggiornamenti:** GUI amministrativa, API, SSH, remoting, backup e acquisizione del software hanno un raggio d'azione del danno maggiore di un normale percorso client e appartengono a una zona di gestione e fiducia separata.

NIST SP 800-177 tratta l'autenticazione del dominio, TLS e la crittografia dei contenuti come meccanismi di sicurezza complementari attorno al protocollo SMTP ancora utilizzato. RFC 8314 separa deliberatamente relay da submission e accesso. Ne consegue che l'hardening deve verificare per ogni ruolo **chi può avviare una connessione, quale identità essa porta, quali dati elabora e verso dove può continuare a comunicare** ([NIST SP 800-177 Rev. 1](https://csrc.nist.gov/pubs/sp/800/177/r1/final), [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

L'inventario mostra ciò che deve essere protetto. Una baseline lo traduce in impostazioni concrete, versionate, testate e modificate in modo tracciabile in caso di eccezioni motivate.

## Ciclo di vita della baseline e deviazioni controllate

Una baseline efficace attraversa un ciclo di vita:

1. **Inventariare:** registrare prodotto, ruolo, versione software, moduli, listener, account, flussi di dati, chiavi e dipendenze.
2. **Scegliere il riferimento:** mappare la baseline del produttore, il CIS Benchmark, il modulo BSI e i requisiti di legge al ruolo concreto.
3. **Personalizzazione:** rimuovere le regole non applicabili, aggiungere regole più rigorose e motivare le deviazioni in base al rischio.
4. **Sperimentare:** verificare funzione, prestazioni, mailflow, monitoraggio, backup e [Disaster Recovery](/kb/backup-dr) in un ambiente rappresentativo.
5. **Distribuire in modo dichiarativo:** utilizzare GPO, Configuration Management, image, Policy-as-Code o API del produttore anziché modifiche manuali singole.
6. **Verificare continuamente:** rilevare deriva, nuovi account, listener, pacchetti, certificati, regole e modifiche della baseline.
7. **Dismettere:** rimuovere in modo controllato accessi, DNS, certificati, chiavi, dati, backup e monitoraggio.

Microsoft Security Compliance Toolkit può salvare, analizzare, confrontare, modificare e applicare come GPO le baseline Windows consigliate. Non sostituisce la personalizzazione: una baseline viene prima verificata in un gruppo pilota per funzionalità ed effetti collaterali. Lo stesso vale per le raccomandazioni CIS e BSI ([Microsoft Security Compliance Toolkit](https://learn.microsoft.com/en-us/windows/security/operating-system-security/device-management/windows-security-configuration-framework/security-compliance-toolkit-10), [CIS Benchmarks FAQ](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq)).

## Funzionalità minima: servizi, porte e software

Il controllo NIST CM-7 richiede di configurare un sistema per le capacità necessarie all'attività e di vietare o limitare funzioni, porte, protocolli, software o servizi. La domanda tecnica non è «La porta 443 è sicura?», bensì: **Quale processo ascolta su quale indirizzo, per quale ruolo, da quale zona e con quale modello di patching e identità?** ([NIST SP 800-53 Rev. 5, CM-7](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)).

Le interfacce web non necessarie, gli endpoint di debug, i protocolli di discovery, i listener locali del database e i servizi di gestione legacy vengono disattivati. I servizi necessari si associano, per quanto possibile, solo alle interfacce previste. Un MTA può ascoltare pubblicamente su SMTP, ma non il suo database. Una porta API amministrativa può essere necessaria, ma non appartiene automaticamente a Internet. CISA raccomanda per le infrastrutture di comunicazione di disattivare servizi inutilizzati o non cifrati quali Telnet, FTP, TFTP, HTTP e versioni SNMP meno recenti e di inventariare costantemente i servizi pubblicamente raggiungibili ([CISA: Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)).

### Inventariare listener e servizi attivi

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Listener- und Dienstinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetTCPConnection -State Listen |
  Sort-Object LocalPort |
  Select-Object LocalAddress, LocalPort, OwningProcess
Get-Service | Where-Object Status -eq Running |
  Sort-Object Name
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
ss -lntup
systemctl list-units --type=service --state=running
```

  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) e [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) mostrano i listener e i processi locali. [`Get-Service`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service) e [`systemctl`](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html) mostrano i servizi attivi. Il confronto tra stato previsto ed effettivo richiede quindi una matrice approvata di porte e servizi; un listener sconosciuto è un rilievo, ma non ancora un'analisi della causa.

Dopo la rimozione delle funzioni non necessarie restano gli account e i servizi che possono effettivamente agire. I loro diritti, percorsi di accesso e segreti determinano la maggior parte della superficie di attacco amministrativa.

## Identità, account e privilegio minimo

Gli account vengono separati per ruolo: identità utente normale, identità amministrativa personale, account di servizio non interattivo e account di emergenza strettamente controllato. I login amministrativi condivisi impediscono un'attribuzione affidabile. Gli account quotidiani permanentemente altamente privilegiati ampliano il raggio d'azione del danno derivante da phishing e compromissioni di browser e client. CIS Control 5 comprende account utente, amministrativi e di servizio; CIS Control 6 l'assegnazione, la manutenzione e la revoca di credenziali e privilegi ([CIS Control 5: Account Management](https://www.cisecurity.org/controls/account-management), [CIS Control 6: Access Control Management](https://www.cisecurity.org/controls/access-control-management)).

L'identità centrale migliora i processi Joiner/Mover/Leaver, ma non sostituisce un percorso di emergenza locale. Un guasto LDAP, Kerberos o SSO non deve rendere impossibile l'accesso autorizzato per il ripristino. Gli account Break-Glass costituiscono quindi una piccola eccezione deliberata: documentati offline, fortemente protetti, non utilizzati nella quotidianità, con allarme immediato in caso di utilizzo e testati regolarmente. Le identità di servizio non ricevono accesso interattivo e hanno solo i diritti, le destinazioni di rete e i segreti necessari al proprio compito. Ove possibile, si preferiscono token a breve durata, Managed Identities o certificati alle password statiche; il loro ciclo di vita e il ripristino restano parte dell'operatività.

MFA riduce il rischio di password rubate, ma non sostituisce diritti minimi e un ripristino sicuro. NIST Zero Trust richiede una decisione di accesso per soggetto ed eventualmente dispositivo prima della sessione; la posizione in rete o il possesso da soli non sono sufficienti ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final)).

### Verificare account locali e privilegiati

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokales Konteninventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-LocalUser | Select-Object Name, Enabled, LastLogon, PasswordExpires
Get-LocalGroupMember -Group Administrators
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
getent passwd
getent group sudo wheel
```

  </div>
</div>

[`Get-LocalUser`](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localuser) e [`Get-LocalGroupMember`](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localgroupmember) leggono account Windows locali e appartenenze ai gruppi. [`getent`](https://man7.org/linux/man-pages/man1/getent.1.html) interroga i database Name Service configurati e può quindi mostrare account locali e risolti centralmente. Una revisione deve inoltre rilevare ruoli effettivi nel prodotto, token API, chiavi SSH, certificati e Cloud IAM.

## Livello di gestione e percorsi amministrativi

Il livello di gestione può modificare configurazione, chiavi, routing, aggiornamenti e log e merita un confine più rigoroso del percorso dei dati utili. CISA raccomanda una rete di gestione Out-of-Band, fisicamente o logicamente separata dal flusso di dati operativo, regole Default Deny, postazioni di lavoro amministrative dedicate e registrazione AAA centralizzata. Anche le connessioni di gestione laterali tra dispositivi dovrebbero essere limitate ([CISA: Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)).

Per i sistemi di messaggistica ciò significa:

- GUI amministrativa, API, SSH e remoting sono raggiungibili solo da zone amministrative definite o attraverso un percorso bastion controllato.
- Certificati, account e regole firewall per gestione e mailflow sono mantenuti separatamente.
- Le connessioni in uscita del livello di gestione sono limitate alle destinazioni di aggiornamento, identità, tempo, log e backup.
- Le modifiche di configurazione richiedono un'identità personale, ove possibile MFA, audit e, per rischi elevati, approvazione secondo il principio dei quattro occhi.
- Un percorso di emergenza funziona senza la normale piattaforma di identità o gestione, ma non è gestito come accesso permanente nascosto.

SSH è solo un trasporto per l'amministrazione; la sua sicurezza dipende da autenticazione, gruppi di utenti consentiti, algoritmi di chiave, forwarding, permessi dei file e diritti di destinazione. Verificare la configurazione effettiva del server anziché soltanto il file di testo rileva include e impostazioni predefinite.

### Visualizzare la configurazione effettiva del server SSH

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für effektive OpenSSH-Konfiguration">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'
& "$env:WINDIR\System32\OpenSSH\sshd.exe" -T
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
sshd -T
```

  </div>
</div>

[`Get-WindowsCapability`](https://learn.microsoft.com/powershell/module/dism/get-windowscapability) mostra il componente OpenSSH installato; Microsoft documenta percorsi e particolarità di [`sshd_config`](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration). [`sshd -T`](https://man.openbsd.org/sshd) restituisce la configurazione effettiva. Le opzioni non devono essere impostate ciecamente seguendo checklist Internet: la disponibilità dell'accesso di emergenza, i tipi di chiave utilizzati e l'automazione devono far parte del test.

## Percorsi di rete: Default Deny con direzione esplicita

Una regola firewall viene documentata come contratto direzionato: **origine, destinazione, protocollo/porta, iniziatore, identità, scopo d'uso, responsabile e data di scadenza**. «Il server di posta può accedere a Internet» non è una specifica tecnica. Un listener SMTP in ingresso necessita di destinazioni egress diverse rispetto a un worker di sandbox antimalware o all'API amministrativa. I filtri egress limitano command-and-control, esfiltrazione e download incontrollati; devono considerare consapevolmente DNS, tempo, verifica dei certificati, aggiornamenti e destinazioni di consegna.

La segmentazione riduce lo spazio di movimento dopo una compromissione. È particolarmente importante tra edge Internet, elaborazione della posta, archivi mailbox/dati, directory, gestione, backup e monitoraggio. NIST Zero Trust avverte al contempo di non utilizzare la posizione in rete come unica base di fiducia. CISA raccomanda per i percorsi di gestione Default Deny e una zona separata dal traffico dei dati dei clienti ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final), [CISA: Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)).

### Verificare il firewall host e la direzione delle regole

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale Firewall-Regeln">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetFirewallProfile |
  Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction
Get-NetFirewallRule -Enabled True |
  Select-Object DisplayName, Direction, Action, Profile
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nft list ruleset
```

  </div>
</div>

[`Get-NetFirewallProfile`](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallprofile) e [`Get-NetFirewallRule`](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallrule) mostrano profili Windows e regole attive. [`nft`](https://netfilter.org/projects/nftables/manpage.html) mostra le regole nftables, incluse chain e direzione. L'output viene verificato rispetto alla matrice dei flussi di dati approvata; una policy Default Deny senza le destinazioni DNS, tempo o certificati necessarie non è uno stato di hardening riuscito.

## Protocolli di posta e fiducia nel trasporto

Relay, submission e accesso necessitano di regole TLS e di autenticazione diverse. Per submission e accesso, RFC 8314 raccomanda TLS 1.2 o superiore e preferisce TLS implicito; l'accesso in chiaro non dovrebbe più essere offerto. Per il relay SMTP, STARTTLS descrive invece una negoziazione hop-by-hop. Senza una policy aggiuntiva, un MTA mittente può continuare a consegnare in chiaro in assenza di TLS. [DANE](/kb/tls) e MTA-STS stabiliscono policy di trasporto differenti e più esplicite ([RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314), [RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)).

[SPF, DKIM e DMARC](/kb/mail-auth) autenticano riferimenti e policy del dominio, non account utente né il contenuto in quanto tale. [S/MIME e OpenPGP](/kb/verschluesselung) proteggono parti dei messaggi, ma non cambiano nulla riguardo a un accesso amministrativo insicuro o a un archivio di chiavi compromesso. Una verifica dell'hardening mantiene separate queste proprietà di sicurezza e ne controlla le dipendenze: DNS, certificati, chiavi, tempo, report e regole di eccezione. NIST SP 800-177 colloca proprio questi meccanismi complementari attorno a SMTP e DNS ([NIST SP 800-177 Rev. 1](https://csrc.nist.gov/pubs/sp/800/177/r1/final)).

### Verificare raggiungibilità e comportamento TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Mail- und Management-TLS-Prüfung">
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
  -servername mx.example.ch -verify_return_error
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) verificano il percorso TCP. [`curl`](https://curl.se/docs/manpage.html) e [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) mostrano STARTTLS, catena di certificati ed errori. Solo la policy dell'MTA e i log rispondono se, in caso di errore, la consegna sia stata ritardata, rifiutata o ripiegata sul testo in chiaro.

## Applicazione, dati, parser e chiavi

I server di posta elaborano intenzionalmente formati complessi non attendibili. Messaggi MIME, archivi, documenti, immagini e HTML raggiungono parser, scanner, convertitori, anteprime e sandbox. L'hardening non limita quindi solo le porte di rete, ma anche diritti dei processi, accesso al filesystem, archiviazione temporanea, CPU/RAM/dimensione dei file, ricorsione, eseguibilità ed egress dei componenti di analisi. Uno scanner necessita di accesso a un oggetto di controllo, ma non automaticamente a tutte le caselle di posta, ai segreti amministrativi o all'API di gestione.

Le code e le directory temporanee contengono dati riservati. Permessi dei file, cifratura, regole di eliminazione e conservazione nonché dump di debug vengono verificati esplicitamente. I log devono rendere tracciabili stati e decisioni, ma non raccogliere password, token, chiavi private o contenuti dei messaggi non necessari. Le chiavi sono separate per scopo: TLS, DKIM, S/MIME/OpenPGP, JWT/API e cifratura dei backup hanno cicli di vita, autorizzazioni e regole di ripristino differenti. NIST SP 800-53 collega privilegio minimo, integrità del sistema, protezione delle comunicazioni e audit; BSI APP.5.3 concretizza la necessità di protezione per client e server di posta ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final), [BSI APP.5.3 Allgemeiner E-Mail-Client und -Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_5_3_Allgemeiner_E-Mail_Client_und_Server_Edition_2023.pdf?__blob=publicationFile&v=3)).

## Patch, image e supply chain

La gestione delle patch è manutenzione preventiva, non un'emergenza sporadica. NIST definisce il processo come identificazione, definizione delle priorità, acquisizione, installazione e verifica di patch, aggiornamenti e upgrade. Per componenti di posta e gestione esposti, informazioni sulle vulnerabilità, superficie di attacco raggiungibile, sfruttamento attivo, criticità dei dati e compensazioni disponibili devono influenzare la priorità ([NIST SP 800-40 Rev. 4](https://csrc.nist.gov/pubs/sp/800/40/r4/final)).

Il percorso di aggiornamento è esso stesso un confine di fiducia. Pacchetti, image, container, plugin, firme antivirus e firmware di appliance vengono acquisiti da fonti autenticate e verificati con firma del produttore o hash pubblicato. Dipendenze e cambi di repository fanno parte dell'inventario. NIST SP 800-161 tratta i rischi di prodotti e servizi il cui sviluppo, integrazione e fornitura gli operatori possono vedere o controllare solo in misura limitata ([NIST SP 800-161 Rev. 1](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)).

Un aggiornamento di hardening viene testato in uno stadio rappresentativo: avvio, mailflow, coda, TLS, directory, policy, monitoraggio, backup e rollback. «Non applicare patch perché la posta è critica» sostituisce un rischio operativo noto con un rischio di sicurezza crescente. Una progettazione migliore crea ridondanza, finestre di manutenzione, build riproducibili e percorsi di fallback testati.

### Integrità degli artefatti ed eventi rilevanti per la sicurezza

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Artefakt- und Ereignisprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-FileHash .\mail-gateway-update.bin -Algorithm SHA256
Get-WinEvent -FilterHashtable @{LogName='Security'; StartTime=(Get-Date).AddHours(-4)} |
  Select-Object TimeCreated, Id, ProviderName, Message
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
sha256sum mail-gateway-update.bin
journalctl --since '-4 hours' --priority=notice..alert
```

  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) e [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) confrontano un artefatto con un hash atteso proveniente da una fonte del produttore autenticata. [`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) e [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) leggono gli eventi; il rilevamento in produzione richiede inoltre una policy di audit corretta, raccolta centrale, sincronizzazione temporale e allarmi definiti.

Una configurazione hardenizzata resta efficace solo se modifiche, controlli falliti e deviazioni diventano visibili. Per questo logging e rilevamento della deriva fanno parte dell'operatività e non solo del controllo successivo.

## Logging, telemetria e deriva

Uno stato hardenizzato non è durevole senza osservazione. I segnali rilevanti includono, tra gli altri:

- accessi riusciti e non riusciti, utilizzo di MFA e Break-Glass;
- modifiche a account, ruoli, token, certificati e chiavi;
- modifiche alla configurazione e deviazioni dalla baseline;
- nuovi listener, servizi, pacchetti, task, container o destinazioni in uscita;
- drop del firewall, connessioni egress inattese e accessi di gestione;
- errori di policy TLS, DNS, SMTP, coda e autenticazione della posta;
- sensori disattivati, lacune nei log, carenze di memoria e deviazioni temporali.

I log vengono raccolti centralmente e protetti nell'accesso affinché un host compromesso non possa semplicemente rimuovere le proprie tracce insieme allo stato del sistema. CIS Control 8 richiede un processo di gestione dei log, memoria sufficiente, orario standardizzato, log di audit dettagliati e centralizzati nonché revisioni. Un allarme è considerato implementato solo quando un evento controllato lo attiva, il servizio responsabile lo vede e un runbook guida la risposta ([CIS Control 8: Audit Log Management](https://www.cisecurity.org/controls/audit-log-management)).

Il rilevamento della deriva confronta lo stato effettivo con la baseline versionata. Il confronto comprende più degli hash dei file: configurazione effettiva, account, gruppi, ruoli IAM, certificati, regole firewall, listener, servizi, pacchetti installati, image, job pianificati e policy del fornitore. Le modifiche di emergenza vengono registrate successivamente o annullate automaticamente; altrimenti lo stato di eccezione «temporaneo» diventa il nuovo default non documentato.

## Sviluppo tecnico

Nel 1975 Saltzer e Schroeder formularono principi fondamentali di protezione come meccanismi piccoli e semplici, impostazioni predefinite sicure, verifica completa dell'autorizzazione, separazione dei privilegi e privilegio minimo. Il loro punto di partenza non era uno specifico sistema operativo, bensì l'architettura della condivisione controllata delle informazioni nei sistemi multiutente ([Saltzer/Schroeder: Basic Principles of Information Protection](https://web.mit.edu/Saltzer/www/publications/protection/Basic.html)).

Con la diffusione dei server di rete, l'hardening si estese ulteriormente a servizi remoti, protocolli, patching, audit e manutenzione sicura della configurazione. NIST SP 800-123 ha riassunto sistematicamente questa pratica server nel 2008. CIS Benchmarks basati sul consenso, moduli BSI-Grundschutz e baseline dei produttori hanno reso le configurazioni target sicure più riproducibili e confrontabili ([NIST SP 800-123](https://csrc.nist.gov/pubs/sp/800/123/final), [CIS Benchmarks FAQ](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq)).

Le architetture cloud, SaaS, API e ibride hanno in seguito indebolito l'assunzione di un chiaro perimetro interno. Nel 2020, NIST SP 800-207 ha descritto Zero Trust come un'architettura orientata alle risorse, senza fiducia implicita derivante dalla posizione in rete o dalla proprietà. Parallelamente, supply chain software, origine delle image e deriva automatizzata della baseline sono diventate aree di controllo proprie. L'hardening moderno collega quindi la minimizzazione classica dell'host a identità, policy service-to-service, configurazione dichiarativa, prove della supply chain, telemetria e ripristino testato ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final), [NIST SP 800-161 Rev. 1](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)).

## Checklist per amministratori

L'hardening è concluso solo quando le misure scelte sono verificabili nel normale funzionamento e in caso di ripristino. La checklist collega quindi configurazione, responsabilità e prova.

- [ ] Ruolo, necessità di protezione, flussi di dati e confini di fiducia del sistema sono documentati.
- [ ] Le raccomandazioni del produttore, CIS e BSI sono state mappate a una baseline versionata.
- [ ] Ogni deviazione dispone di motivazione, controllo compensativo, responsabile e data di scadenza.
- [ ] Listener, servizi, pacchetti, moduli e destinazioni in uscita sono ridotti al minimo necessario.
- [ ] Identità utente, amministrative, di servizio e Break-Glass sono separate e verificate regolarmente.
- [ ] Gli accessi amministrativi utilizzano identità personali, MFA, diritti minimi e auditing centrale.
- [ ] Il livello di gestione e il mailflow produttivo risiedono in zone separate e restrittive.
- [ ] Relay, submission, accesso e percorsi di servizio interni dispongono di policy proprie per TLS, autenticazione e rate limit.
- [ ] Parser, scanner, dati temporanei, code, chiavi e segreti hanno diritti minimi di processo e file.
- [ ] Patch e image provengono da fonti autenticate; origine e integrità sono verificate.
- [ ] Test di baseline, mailflow, backup e rollback vengono eseguiti prima della distribuzione estesa.
- [ ] I log sono centrali, coerenti nel tempo, protetti dalle modifiche e collegati ad allarmi testati.
- [ ] La deriva di account, configurazione, regole, servizi, certificati e software viene rilevata automaticamente.
- [ ] Ripristino e accesso di emergenza sono stati testati concretamente nelle condizioni hardenizzate.

## Fonti

- [NIST – SP 800-123, Guide to General Server Security](https://csrc.nist.gov/pubs/sp/800/123/final)
- [CIS – Control 4: Secure Configuration](https://www.cisecurity.org/controls/secure-configuration-of-enterprise-assets-and-software)
- [CIS – Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [BSI – SYS.1.1 Allgemeiner Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2022/07_SYS_IT_Systeme/SYS_1_1_Allgemeiner_Server_Edition_2022.pdf?__blob=publicationFile&v=3)
- [BSI – APP.5.3 Allgemeiner E-Mail-Client und -Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_5_3_Allgemeiner_E-Mail_Client_und_Server_Edition_2023.pdf?__blob=publicationFile&v=3)
- [NIST – SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)
- [NIST – SP 800-53 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)
- [NIST – SP 800-207, Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final)
- [NIST – SP 800-177 Rev. 1, Trustworthy Email](https://csrc.nist.gov/pubs/sp/800/177/r1/final)
- [IETF RFC 8314 – TLS for Email Submission and Access](https://datatracker.ietf.org/doc/html/rfc8314)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [Microsoft Learn – Security Compliance Toolkit](https://learn.microsoft.com/en-us/windows/security/operating-system-security/device-management/windows-security-configuration-framework/security-compliance-toolkit-10)
- [CIS – Benchmarks FAQ](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq)
- [CISA – Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft Learn – Get-Service](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service)
- [systemd – systemctl](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)
- [CIS – Control 5: Account Management](https://www.cisecurity.org/controls/account-management)
- [CIS – Control 6: Access Control Management](https://www.cisecurity.org/controls/access-control-management)
- [Microsoft Learn – Get-LocalUser](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localuser)
- [Microsoft Learn – Get-LocalGroupMember](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localgroupmember)
- [Linux man-pages – getent](https://man7.org/linux/man-pages/man1/getent.1.html)
- [Microsoft Learn – Get-WindowsCapability](https://learn.microsoft.com/powershell/module/dism/get-windowscapability)
- [Microsoft Learn – OpenSSH Server Configuration](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)
- [OpenBSD – sshd manpage](https://man.openbsd.org/sshd)
- [Microsoft Learn – Get-NetFirewallProfile](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallprofile)
- [Microsoft Learn – Get-NetFirewallRule](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallrule)
- [Netfilter – nft manpage](https://netfilter.org/projects/nftables/manpage.html)
- [IETF RFC 3207 – SMTP STARTTLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [IETF RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manpage](https://man.openbsd.org/nc)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [NIST – SP 800-40 Rev. 4, Enterprise Patch Management](https://csrc.nist.gov/pubs/sp/800/40/r4/final)
- [NIST – SP 800-161 Rev. 1, Cybersecurity Supply Chain Risk Management](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – sha256sum](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [CIS – Control 8: Audit Log Management](https://www.cisecurity.org/controls/audit-log-management)
- [Saltzer/Schroeder – Basic Principles of Information Protection](https://web.mit.edu/Saltzer/www/publications/protection/Basic.html)
