---
title: "SSH: trasporto, identità e amministrazione sicura"
blatt: "ssh"
description: "SSH per amministratori di infrastrutture e messaggistica: trasporto, scambio di chiavi e identità host, autenticazione di utenti e workload, canali, chiavi e certificati OpenSSH, agenti, bastion e forwarding, SFTP/SCP, policy del server, hardening, logging, Windows OpenSSH e storia tecnica."
fakten:
  - label: Ruolo di sistema
    wert: livello di trasporto e canali cifrato, protetto dall'integrità e autenticato per l'amministrazione
    href: https://www.rfc-editor.org/rfc/rfc4251.html
  - label: Porta standard
    wert: TCP 22; il listener effettivo e il percorso di accesso dipendono dalla configurazione
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml
  - label: Stack di protocollo
    wert: Transport Protocol · User Authentication Protocol · Connection Protocol
    href: https://www.rfc-editor.org/rfc/rfc4251.html
  - label: Identità del server
    wert: Host Key o Host Certificate; verificare prima delle credenziali utente
    href: https://www.rfc-editor.org/rfc/rfc4253.html
  - label: Autenticazione utente
    wert: publickey · password · hostbased · keyboard-interactive; la policy può richiedere più fattori
    href: https://www.rfc-editor.org/rfc/rfc4252.html
  - label: Negoziazione crittografica
    wert: KEX · algoritmo Host Key · Cipher · MAC/AEAD · compressione
    href: https://www.rfc-editor.org/rfc/rfc9142.html
  - label: Tipi di chiave
    wert: Ed25519 · ECDSA · RSA/SHA-2 · varianti FIDO Security Key; impiego in base alla compatibilità client/server
    href: https://www.rfc-editor.org/rfc/rfc8709.html
  - label: Canali di accesso
    wert: Shell · Exec · Subsystem · forwarding TCP/Streamlocal · inoltro di agent/X11
    href: https://www.rfc-editor.org/rfc/rfc4254.html
  - label: Trasferimento file
    wert: SFTP come sottosistema SSH; la CLI scp può usare semantica SFTP o SCP legacy a seconda dell'implementazione
    href: https://man.openbsd.org/sftp
  - label: Distribuzione della fiducia
    wert: known_hosts/pinning · SSHFP con DNSSEC · Host CA · certificati utente/host OpenSSH
    href: https://www.rfc-editor.org/rfc/rfc4255.html
  - label: Stato operativo
    wert: listener · policy effettiva · Host Keys · percorso di autenticazione · sessioni/canali · log · rete di origine
    href: https://man.openbsd.org/sshd_config
  - label: Oggetto di recovery
    wert: identità host/CA, policy del server, Authorized Principals/Keys, Break Glass, percorso di audit e dipendenze dalla bastion
    href: https://man.openbsd.org/ssh-keygen
werbung:
  - newsletter
ctaThemen:
  - claude
  - powershell
  - haertung
translationSourceHash: 9617920ae627ef338448276f4094696352ba3f158be192d73dd10226c8ee296f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T09:27:38.398Z
translationReview: automatic
---

# SSH: trasporto, identità e amministrazione sicura

Secure Shell è una famiglia di protocolli per l'accesso remoto sicuro attraverso una rete non sicura. SSH combina tre funzioni: stabilisce un trasporto cifrato e protetto dall'integrità, autentica prima il server e poi l'utente o il workload e infine multipla shell, singoli comandi, sottosistemi e inoltri come canali separati. Questo ordine è decisivo per gli amministratori: una chiave utente forte non protegge se il client la offre a un server non verificato.

SSH è al tempo stesso protocollo e control plane. Un accesso può consentire comandi root, trasferimento file, tunnel di database, accesso all'agente e percorsi di salto verso reti interne. La valutazione della sicurezza non deve quindi terminare alla porta del listener o a `PasswordAuthentication no`. La distribuzione delle Host Key, l'effettiva policy Match, l'origine delle chiavi, i principal, i diritti di forwarding, il percorso di origine, il logging delle sessioni e il recovery degli ancoraggi di fiducia appartengono allo stesso sistema.

La spiegazione segue una connessione SSH dal TCP e dallo scambio di versioni, passando per la negoziazione delle chiavi e la verifica dell'host, fino all'accesso utente e al canale. Su questa base si innestano inoltri, gestione delle chiavi, certificati e diagnostica.

SSH trasforma una connessione di rete non sicura in un trasporto autenticato e cifrato, con canali separati per shell, comandi, trasferimento file o tunnel. L'articolo segue esattamente questo ordine: identità del server, scambio delle chiavi, accesso utente, canali e gestione.

## Architettura del protocollo

RFC 4251 suddivide SSH-2 in tre protocolli:

1. **Transport Layer Protocol:** scambio di versioni, negoziazione degli algoritmi, Key Exchange, autenticazione del server, cifratura e integrità.
2. **User Authentication Protocol:** autenticazione di un utente client verso il server sul trasporto protetto.
3. **Connection Protocol:** multiplexing di canali logici per sessioni, port forwarding e altri servizi.

I livelli operano su un byte stream affidabile, solitamente [TCP](/kb/tcp). La porta TCP 22 è registrata presso IANA come `ssh`; reverse proxy, inoltri di porta e bastion possono modificare il percorso effettivo ([RFC 4251 – SSH Protocol Architecture](https://www.rfc-editor.org/rfc/rfc4251.html), [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-ssh.svg?v=20260813" title="Interaktive Infografik: SSH-Verbindungsaufbau von TCP und Versionsaustausch über KEX, Hostidentität und Benutzerautorisierung bis zu Channels, Bastion, Schlüssel- und Zertifikatslebenszyklus sowie Audit und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-ssh.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

## Scambio di versioni e negoziazione degli algoritmi

Client e server inviano anzitutto stringhe di identificazione come `SSH-2.0-...`. Successivamente scambiano `SSH_MSG_KEXINIT` con liste ordinate per Key Exchange, algoritmo Host Key, cifratura in entrambe le direzioni, MAC, compressione e lingua. Viene scelta la prima opzione comune di ciascuna categoria. Le liste degli algoritmi sono quindi policy e contratto di compatibilità, non solo un dettaglio di implementazione ([RFC 4253 – SSH Transport Layer Protocol](https://www.rfc-editor.org/rfc/rfc4253.html)).

RFC 9142 aggiorna le raccomandazioni per gli algoritmi SSH. Distingue `MUST`, `SHOULD`, `MAY`, `SHOULD NOT` e `MUST NOT` per le implementazioni; una policy produttiva considera inoltre compliance, parco client, hardware e percorso di migrazione ([RFC 9142 – Key Exchange Method Recommendations for SSH](https://www.rfc-editor.org/rfc/rfc9142.html)).

Le righe di configurazione che fissano una propria lista rigida di cipher diventano obsolete. OpenSSH può mostrare gli algoritmi supportati con `ssh -Q` e visualizzare la configurazione client o server effettivamente risolta. I gestori rimuovono selettivamente le procedure legacy note, mantenendo però un percorso di compatibilità testato per le appliance.

## Key Exchange e chiavi di sessione

Il Key Exchange genera un segreto condiviso e un Exchange Hash senza inviare la chiave di sessione attraverso la rete. L'hash associa le stringhe di identificazione, i messaggi KEXINIT, la Host Key del server e i valori effimeri specifici dell'algoritmo. Il server firma l'Exchange Hash con la propria chiave host; il client verifica così insieme l'identità del server e lo scambio delle chiavi ([RFC 4253](https://www.rfc-editor.org/rfc/rfc4253.html)).

Le procedure moderne utilizzano varianti effimere di Diffie-Hellman e offrono Forward Secrecy per le chiavi di sessione. SSH può effettuare un rekey durante una connessione, ad esempio dopo una certa quantità di dati o di tempo. Rekey e Connection Channels condividono lo stesso trasporto; i problemi si manifestano quindi come freeze o disconnessioni di sessioni apparentemente attive.

RFC 8308 definisce una negoziazione generica delle estensioni. OpenSSH utilizza estensioni, tra l'altro, per comunicare algoritmi di firma server o aggiornamenti delle Host Key ([RFC 8308 – Extension Negotiation in SSH](https://www.rfc-editor.org/rfc/rfc8308.html)). Strict Key Exchange protegge maggiormente la sequenza iniziale da messaggi inattesi ed è stato standardizzato dopo l'analisi Terrapin ([RFC 9519 – Strict Key Exchange in SSH](https://www.rfc-editor.org/rfc/rfc9519.html)).

Dopo la scelta di versione e algoritmi, il client deve prima riconoscere il server. Solo quando la Host Key è accettata, il successivo accesso utente ha senso.

## Host Keys: identità del server prima dell'identità utente

La Host Key identifica il server o un gruppo controllato di server. Al primo contatto, un client senza Trust Anchor distribuito in anticipo non dispone di una base crittografica per valutare il fingerprint mostrato. «Accept new» automatizza il pinning per i nuovi host, ma non risolve la distribuzione iniziale sicura.

OpenSSH memorizza chiavi e marker noti in `known_hosts`. Una chiave modificata può indicare una rotazione legittima, una reinstallazione, un pool di load balancer modificato o un attacco. L'avviso non viene «risolto» cancellando alla cieca la riga: la nuova chiave viene verificata tramite una fonte indipendente e la causa documentata ([ssh(1) – host key verification](https://man.openbsd.org/ssh), [sshd(8)](https://man.openbsd.org/sshd)).

### Modelli di fiducia per l'identità host

| Modello | Punto di forza | Limite operativo |
|---|---|---|
| fingerprint manuale out-of-band | diretto e semplice per pochi host | distribuzione e rotazione sicure scalano male |
| pinning/TOFU di `known_hosts` | rileva modifiche successive | il primo contatto resta non dimostrato |
| SSHFP con DNSSEC | DNS pubblica i fingerprint | il client deve validare DNSSEC; il percorso del resolver è un confine di fiducia |
| Host CA OpenSSH | i client si fidano della CA e dei principal | chiave CA, processo di firma, validità e associazione dei principal |
| gestione della configurazione | distribuzione centralizzata e auditabile | fiducia nella pipeline e nell'enrollment |

RFC 4255 definisce i record SSHFP; senza DNSSEC possono essere solo un'indicazione, non l'unico Trust Anchor ([RFC 4255 – SSHFP Resource Records](https://www.rfc-editor.org/rfc/rfc4255.html)). I certificati OpenSSH associano Public Key, numero seriale, tipo, principal, validità, Critical Options ed Extensions tramite una firma CA ([IETF-Draft – OpenSSH Certificate Format](https://datatracker.ietf.org/doc/html/draft-ietf-sshm-cert)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Hostidentität">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName $env:SSH_HOST -Type SSHFP -DnssecOk
ssh-keyscan -t ed25519,rsa $env:SSH_HOST | Set-Content .\\hostkeys.candidate
ssh-keygen -lf .\\hostkeys.candidate
ssh-keygen -F $env:SSH_HOST</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +dnssec "$SSH_HOST" SSHFP
ssh-keyscan -t ed25519,rsa "$SSH_HOST" &gt; hostkeys.candidate
ssh-keygen -lf hostkeys.candidate
ssh-keygen -F "$SSH_HOST"</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) interrogano [DNS](/kb/dns); [`ssh-keyscan`](https://man.openbsd.org/ssh-keyscan) raccoglie le chiavi offerte, ma **non le autentica**. [`ssh-keygen`](https://man.openbsd.org/ssh-keygen) calcola i fingerprint e cerca in `known_hosts`. I candidati vengono acquisiti solo dopo una verifica indipendente.

## Ciclo di vita e rotazione delle Host Key

Un server può possedere più Host Key per algoritmi differenti. Le Host Key private risiedono solo sui server o su componenti di load balancer controllati; backup e cloni non devono portare a un'identità condivisa involontariamente. Con l'autoscaling, una Host CA è spesso più adatta delle Host Key private copiate.

Una rotazione comprende:

1. generare e proteggere una nuova chiave o un nuovo certificato host;
2. offrirli in parallelo e distribuire fingerprint/fiducia CA;
3. verificare la reale compatibilità dei client e il principal host;
4. rimuovere in modo controllato la vecchia chiave;
5. aggiornare inventario, revoca e documentazione Break Glass.

OpenSSH può annunciare Host Key aggiuntive dopo un'autenticazione riuscita tramite `UpdateHostKeys`. Questo agevola la rotazione, ma presuppone che la fiducia host esistente sia già corretta ([ssh_config(5)](https://man.openbsd.org/ssh_config)).

Con trasporto protetto e server confermato inizia l'accesso utente o macchina. Password, Public Key, certificati e altre procedure hanno requisiti operativi e di recovery differenti.

## User Authentication Protocol

Dopo il trasporto, il client richiede il servizio `ssh-userauth`. RFC 4252 definisce i metodi `publickey`, `password`, `hostbased` e `none`; Keyboard-Interactive è specificato in RFC 4256. Il server può richiedere ulteriore autenticazione e comunica i metodi successivi consentiti ([RFC 4252 – SSH Authentication Protocol](https://www.rfc-editor.org/rfc/rfc4252.html), [RFC 4256 – Keyboard Interactive Authentication](https://www.rfc-editor.org/rfc/rfc4256.html)).

**Metodo di autenticazione** e **autorizzazione** sono distinti. Una Public Key valida può essere associata a un account tramite `authorized_keys`, servizio directory, `AuthorizedKeysCommand`, principal di certificato o PAM. Successivamente, i diritti Unix, `sudo`, SELinux/AppArmor, token Windows e policy applicative determinano i diritti effettivi.

OpenSSH `AuthenticationMethods` può imporre combinazioni come `publickey,keyboard-interactive:pam`. Che si tratti di due fattori indipendenti dipende dalle fonti delle credenziali e dal legame al dispositivo. Due metodi provenienti entrambi dallo stesso agente di laptop compromesso offrono meno indipendenza di quanto faccia supporre la loro sintassi ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

## Procedura Public Key e tipi di chiave

Con l'autenticazione Public Key, il client invia una firma sull'identificatore di sessione e sulla richiesta di autenticazione; la chiave privata non lascia il client. Il server verifica Public Key, firma e autorizzazione. RFC 8332 definisce le firme RSA con SHA-2 per SSH; il tipo di chiave `ssh-rsa` designa storicamente anche la semantica di firma SHA-1 e non deve essere equiparato a ogni utilizzo di una chiave RSA ([RFC 8332 – RSA Keys with SHA-2 Signatures](https://www.rfc-editor.org/rfc/rfc8332.html)).

RFC 8709 standardizza Ed25519 ed Ed448 per SSH. OpenSSH supporta inoltre chiavi `sk-` basate su FIDO/U2F, nelle quali la chiave privata è legata a una Security Key e richiede facoltativamente la presenza o la verifica dell'utente ([RFC 8709 – Ed25519 and Ed448 for SSH](https://www.rfc-editor.org/rfc/rfc8709.html), [OpenSSH FIDO/U2F support](https://www.openssh.com/txt/release-8.2)).

La scelta segue la compatibilità client/server e il modello di minaccia. La sola lunghezza della chiave non determina la sicurezza: memorizzazione privata, passphrase, accesso dell'agente, validità, principal, revoca e opzioni di autorizzazione sono altrettanto importanti.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Schlüssel- und Agentinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">ssh-keygen -t ed25519 -a 64 -f $env:USERPROFILE\\.ssh\\id_admin_ed25519
ssh-keygen -lf $env:USERPROFILE\\.ssh\\id_admin_ed25519.pub
Get-Service ssh-agent
ssh-add -l
ssh-add -T $env:USERPROFILE\\.ssh\\id_admin_ed25519.pub</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ssh-keygen -t ed25519 -a 64 -f "$HOME/.ssh/id_admin_ed25519"
ssh-keygen -lf "$HOME/.ssh/id_admin_ed25519.pub"
ssh-add -l
ssh-add -T "$HOME/.ssh/id_admin_ed25519.pub"
ssh-keygen -y -f "$HOME/.ssh/id_admin_ed25519" | diff - "$HOME/.ssh/id_admin_ed25519.pub"</code></pre>
  </div>
</div>

[`ssh-add`](https://man.openbsd.org/ssh-add) gestisce le identità nell'agente e può testare la capacità di firma di una Public Key. [`Get-Service`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-service) mostra il servizio agente Windows. I file di chiave privata e i socket dell'agente ricevono ACL minime; gli esempi non generano chiavi CA o host centrali.

## `authorized_keys` come oggetto di policy

Una riga in `authorized_keys` può contenere opzioni prima della chiave. `from=`, `command=`, `restrict`, `no-port-forwarding`, `no-agent-forwarding`, `no-X11-forwarding`, `no-pty`, `permitopen=` e `environment=` limitano l'utilizzo. `restrict` imposta un bundle difensivo di opzioni; possono seguire autorizzazioni esplicite ([sshd(8) – AUTHORIZED_KEYS FILE FORMAT](https://man.openbsd.org/sshd#AUTHORIZED_KEYS_FILE_FORMAT)).

Una chiave per backup o deployment non riceve un accesso shell generico. Viene vincolata alla rete di origine, a un comando forzato e alle destinazioni di forwarding necessarie. I wrapper validano gli argomenti come input strutturato e non utilizzano concatenazione shell non verificata.

`authorized_keys` è configurazione distribuita. L'inventario comprende fingerprint, proprietario, scopo, account di destinazione, origine, scadenza, ultimo utilizzo e revoca. Le chiavi orfane sono credenziali durature senza visibilità centrale.

## Certificati OpenSSH e principal

I certificati OpenSSH non sono certificati X.509. Una SSH-CA firma una Public Key con numero seriale, Key ID, tipo, principal e finestra di validità. Gli User Certificates autenticano gli utenti verso i server; gli Host Certificates autenticano i server verso i client. Le Critical Options devono essere comprese, altrimenti il certificato viene rifiutato; le Extensions concedono funzioni come PTY o forwarding ([IETF-Draft – OpenSSH Certificate Format](https://datatracker.ietf.org/doc/html/draft-ietf-sshm-cert)).

I certificati riducono la durata delle credenziali e centralizzano l'emissione, ma non eliminano l'autorizzazione. I server stabiliscono con `TrustedUserCAKeys` e Authorized Principals quale CA e quali principal si applicano a quale account. I client associano una Host CA in `known_hosts` con `@cert-authority` a modelli di nome.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Zertifikatsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">ssh-keygen -L -f $env:USERPROFILE\\.ssh\\id_admin_ed25519-cert.pub
ssh-keygen -lf $env:USERPROFILE\\.ssh\\id_admin_ed25519-cert.pub
ssh -G $env:SSH_ALIAS | Select-String 'hostname|user|identityfile|certificatefile|proxyjump'
ssh -vvv $env:SSH_ALIAS</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ssh-keygen -L -f "$HOME/.ssh/id_admin_ed25519-cert.pub"
ssh-keygen -lf "$HOME/.ssh/id_admin_ed25519-cert.pub"
ssh -G "$SSH_ALIAS" | grep -E '^(hostname|user|identityfile|certificatefile|proxyjump) '
ssh -vvv "$SSH_ALIAS"</code></pre>
  </div>
</div>

[`grep`](https://www.gnu.org/software/grep/manual/grep.html) e [`Select-String`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string) filtrano soltanto la configurazione client effettiva. L'output di debug di `ssh -vvv` può contenere nomi utente, percorsi, infrastruttura di destinazione e fingerprint offerti e viene trattato come artefatto diagnostico sensibile.

## ssh-agent e Agent Forwarding

Un agente conserva chiavi private o handle di firma sbloccati e risponde alle richieste di firma tramite un socket locale. I processi con accesso a questo socket possono abusare dell'agente senza esportare la Private Key. Passphrase e chiavi hardware non limitano pertanto ogni compromissione dell'agente.

L'Agent Forwarding inoltra il canale dell'agente a un host remoto. Root o un processo compromesso su tale host può effettuare richieste di firma durante la sessione. OpenSSH avverte espressamente di questo rischio; `ProxyJump` trasporta la connessione attraverso una bastion senza rendere disponibile il socket dell'agente ([ssh(1) – `-A` and `-J`](https://man.openbsd.org/ssh)).

Se una destinazione è raggiungibile solo tramite un Jump Host, i modelli preferiti sono:

- `ProxyJump`/`ProxyCommand` senza archiviazione di credenziali sulla bastion;
- certificati SSH di breve durata con principal specifici per la destinazione;
- una chiave separata per ogni dominio di fiducia;
- session brokering controllato con audit, se l'interattività deve essere mediata centralmente.

Dopo un accesso riuscito, SSH apre canali logici all'interno della stessa connessione. Shell, comando singolo, SFTP e port forwarding condividono quindi trasporto e identità, ma restano operazioni separate.

## Connection Protocol e canali

Dopo l'autenticazione riuscita, SSH apre canali logici. RFC 4254 definisce windowing, dati di canale, richieste e tipi. Una sessione può richiedere un PTY, una shell, un singolo comando `exec` o un sottosistema come SFTP. Il TCP Forwarding apre canali `direct-tcpip` o `forwarded-tcpip` ([RFC 4254 – SSH Connection Protocol](https://www.rfc-editor.org/rfc/rfc4254.html)).

Il multiplexing consente a più connessioni client di condividere un trasporto esistente. Ciò riduce i costi di KEX e autenticazione, ma amplia l'impatto di un Control Socket compromesso. `ControlPath`, autorizzazioni, persistenza e vincolo alla destinazione vengono impostati esplicitamente.

I keepalive hanno livelli diversi: TCPKeepAlive verifica il percorso TCP, `ServerAliveInterval`/`ClientAliveInterval` utilizzano messaggi SSH. Non prevengono i blocchi applicativi e valori aggressivi possono terminare sessioni durante brevi interruzioni di rete.

## Port Forwarding e confini di fiducia

Il Local Forwarding (`-L`) apre un listener sul lato client e si connette dalla rete del server alla destinazione. Il Remote Forwarding (`-R`) apre un listener sul lato server o sul lato opposto. Il Dynamic Forwarding (`-D`) fornisce un proxy SOCKS. Il forwarding Unix socket/streamlocal integra TCP.

Il forwarding può aggirare la segmentazione. Un utente senza privilegi shell può comunque raggiungere interfacce di amministrazione o database interni tramite un inoltro consentito. I server controllano ciò con `AllowTcpForwarding`, `PermitOpen`, `PermitListen`, `GatewayPorts`, blocchi Match e opzioni Authorized Key ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Bastion und Tunnelprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection $env:BASTION_HOST -Port $env:SSH_PORT
ssh -J $env:BASTION_ALIAS -N -L 127.0.0.1:8443:$env:INTERNAL_HOST:443 $env:TARGET_ALIAS
Get-NetTCPConnection -LocalPort 8443 -State Listen
ssh -O check $env:TARGET_ALIAS</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz "$BASTION_HOST" "$SSH_PORT"
ssh -J "$BASTION_ALIAS" -N -L "127.0.0.1:8443:$INTERNAL_HOST:443" "$TARGET_ALIAS"
ss -lntp 'sport = :8443'
ssh -O check "$TARGET_ALIAS"</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) verificano il percorso TCP verso la bastion. [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) e [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) confermano il listener locale. Il tunnel `-N` non esegue alcun comando remoto; l'autorizzazione e la [TLS](/kb/tls) di destinazione restano confini indipendenti.

## SFTP, SCP e diritti sui file

SFTP è un protocollo binario di trasferimento file che normalmente opera come sottosistema SSH. Non è FTP su SSH. OpenSSH può eseguire `internal-sftp` senza un binario esterno di server SFTP e limitare al trasferimento file tramite `ChrootDirectory` più `ForceCommand internal-sftp` ([sftp(1)](https://man.openbsd.org/sftp), [sftp-server(8)](https://man.openbsd.org/sftp-server)).

La CLI `scp` utilizzava storicamente il protocollo remoto simile a SCP/RCP e nelle linee moderne di OpenSSH usa SFTP per impostazione predefinita; `-O` forza il protocollo legacy. Wildcard, elaborazione della remote shell e compatibilità server differiscono ([scp(1)](https://man.openbsd.org/scp)). I runbook indicano quindi esplicitamente CLI, modalità di protocollo e metadati file desiderati.

L'autorizzazione al trasferimento file comprende proprietà del chroot, percorsi di destinazione, UID/GID o ACL Windows, umask, symlink, quota e deposito atomico. Il caricamento in una directory di posta o automazione richiede spesso un confine di quarantena/rinomina anziché l'elaborazione diretta di file scritti parzialmente.

## Policy lato server e blocchi Match

`sshd_config` viene valutato dall'alto verso il basso; per ciascuna parola chiave si applica in linea di principio il primo valore ottenuto. `Match` modifica l'ambito di applicazione in base a utente, gruppo, host, indirizzo, LocalAddress, LocalPort o RDomain. File Include, default della distribuzione e opzioni della riga di comando possono integrare il file principale visibile ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

Una baseline robusta definisce:

- utenti/gruppi e reti di origine consentiti;
- metodi di autenticazione necessari ed eccezione Break Glass;
- accesso root/amministratore ed escalation dei privilegi;
- forwarding, agente, X11, PTY e sottosistemi;
- MaxAuthTries, LoginGraceTime, limiti di sessione/avvio e keepalive;
- Host Keys, algoritmi e rekey;
- fonti AuthorizedKeys/Principals;
- logging, banner e assunzione dell'ambiente;
- deviazioni specifiche di Match.

Le «liste di hardening» generiche possono escludere appliance oppure risultare inefficaci se un valore Match accettato successivamente viene applicato diversamente. È decisiva la configurazione effettiva per una connessione concreta.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Serverpolicy und Logs">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">sshd.exe -t
sshd.exe -T -C user=$env:SSH_USER,host=$env:SSH_HOST,addr=$env:CLIENT_IP
Get-NetTCPConnection -LocalPort $env:SSH_PORT -State Listen
Get-WinEvent -LogName OpenSSH/Operational -MaxEvents 100</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sshd -t
sshd -T -C "user=$SSH_USER,host=$SSH_HOST,addr=$CLIENT_IP"
ss -lntp "sport = :$SSH_PORT"
journalctl --unit ssh --unit sshd --since '-60 minutes' --no-pager</code></pre>
  </div>
</div>

`sshd -t` convalida sintassi e chiavi, `sshd -T -C` restituisce la configurazione effettiva per i parametri di test ([sshd(8)](https://man.openbsd.org/sshd)). [`Get-WinEvent`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent) e [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) leggono i log della piattaforma. Un reload riuscito e una seconda sessione amministrativa testata precedono la chiusura della sessione esistente.

## Brute force, rate limit e confini di rete

I servizi SSH esposti a Internet ricevono tentativi di accesso automatizzati. Firewall di origine, accesso VPN/Zero Trust, bastion, `AllowUsers`/`AllowGroups`, autenticazione forte e breve durata delle credenziali riducono la superficie di attacco. Una porta diversa riduce il rumore nei log, ma non modifica né l'autenticazione né la vulnerabilità.

Gli strumenti di blocco perimetrale possono limitare tentativi ripetuti; elaborano dati di log non attendibili e necessitano di regole proprie per allowlist, IPv6, NAT, lockout e recovery. OpenSSH `MaxStartups` limita probabilisticamente le connessioni parallele non autenticate, `PerSourceMaxStartups` e `PerSourcePenalties` possono considerare il comportamento dell'origine. Capacità e rischio DoS vengono testati in condizioni reali di proxy/NAT ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

## Logging, audit e prova di sessione

I log SSH documentano di norma origine della connessione, utente, metodo di autenticazione/fingerprint, inizio/fine sessione ed eventi selezionati dei sottosistemi. Non contengono automaticamente ogni comando shell eseguito. La shell history è controllata dall'utente e non costituisce un audit protetto dalla manomissione.

Per l'amministrazione privilegiata vengono correlati:

- identità di origine, dispositivo e decisione di accesso prima di SSH;
- hop della bastion/del proxy e host di destinazione;
- decisione Host Key/CA del client;
- principal utente, metodo di autenticazione, fingerprint della chiave o seriale del certificato;
- tipo di sessione/canale, inizio, fine e stato di uscita;
- eventi di privilegi `sudo`/Windows e audit del sistema di destinazione;
- percorso di trasferimento file, hash o ticket di modifica per deployment sensibili.

La registrazione delle sessioni modifica privacy, acquisizione di segreti e rischio operativo. Password, token, contenuti e-mail e Private Keys non devono finire in modo incontrollato nelle trascrizioni. Conservazione, accesso e protezione dalla manomissione vengono definiti prima dell'attivazione.

## Windows OpenSSH

Microsoft fornisce OpenSSH Client e Server come funzionalità Windows. Il server utilizza il servizio `sshd`, la configurazione in `%ProgramData%\ssh\sshd_config`, account Windows e ACL; i log possono comparire nel canale `OpenSSH/Operational` ([Microsoft Learn – OpenSSH overview](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-overview), [Microsoft Learn – OpenSSH Server configuration](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)).

Le chiavi degli amministratori Windows possono trovarsi, a seconda della configurazione, in un file centrale `administrators_authorized_keys` le cui ACL devono essere rigorose. Shell, sottosistemi, valore di registro DefaultShell e risoluzione dei percorsi differiscono da Unix. Un comando client portabile non implica un'autorizzazione server identica.

## Backup, recovery e Break Glass

Il recovery SSH non comprende soltanto `sshd_config`:

- Host Keys o Host CA più chiavi private protette;
- User CA, issuer dei certificati, principal e liste di revoca;
- Authorized Keys/Commands e relativa fonte di directory/configuration management;
- policy server, file Include, dipendenze PAM/SSSD/AD e MFA;
- firewall, bastion, DNS/SSHFP e fonte temporale;
- account Break Glass con credenziale verificata offline;
- console out-of-band nel caso in cui SSH di rete non sia disponibile;
- log di audit e inventario di chiavi/certificati.

Il ripristino della stessa Host Key privata preserva l'identità del server, ma può essere pericoloso dopo una compromissione. Il recovery sceglie quindi tra **continuità** e **nuovo enrollment con rotazione**. Un test di restore utilizza un indirizzo di destinazione isolato e verifica principal host, metodi di autenticazione, policy Match, escalation dei privilegi e correlazione dei log.

La diagnostica legge la configurazione della connessione in sequenza: TCP, scambio di versioni, Key Exchange, Host Key, autenticazione utente e apertura del canale.

## Diagnostica per fase del protocollo

SSH procede in fasi chiare. Stabilire prima se sono interessati TCP, negoziazione delle chiavi, verifica host, accesso utente o canale evita modifiche indiscriminate a chiavi e configurazione del server.

| Fase | Evidenza tipica | Classe di errore frequente |
|---|---|---|
| TCP | DNS, route, listener, SYN/ACK | firewall, porta errata, proxy/bastion |
| Versione | client `-vvv`, log del server | implementazione incompatibile o estranea |
| KEX | algoritmi negoziati | nessun KEX/Cipher/MAC comune, problema Strict-KEX |
| Host Key | fingerprint, principal, CA/known_hosts | host errato, rotazione, destinazione DNS/LB |
| Userauth | metodi offerti, fingerprint/certificato, PAM | chiave non autorizzata, principal, ACL, MFA |
| Connection | richiesta canale, PTY, sottosistema, forwarding | ForceCommand, PermitOpen, SFTP/Chroot, policy |
| Processo remoto | stato di uscita, stderr, ambiente, diritti | percorso shell, sudo, locale, applicazione |

Un `Permission denied (publickey)` si verifica dopo trasporto e host trust riusciti. Un timeout prima del banner non è un problema di chiavi. La diagnostica orientata alle fasi impedisce di modificare credenziali o liste di algoritmi senza evidenza.

## Storia tecnica

Tatu Ylönen sviluppò l'SSH originale nel 1995 come sostituto degli strumenti non cifrati per accesso remoto e trasferimento file. SSH-1 e SSH-2 differivano sostanzialmente; l'IETF SECSH Working Group standardizzò successivamente l'architettura SSH-2 in RFC 4251 fino a 4254 ([OpenSSH – History](https://www.openssh.com/history.html), [RFC 4251](https://www.rfc-editor.org/rfc/rfc4251.html)).

OpenSSH nacque nel progetto OpenBSD a partire da codice SSH libero e fu portato su altri sistemi operativi. Riunì client, server, strumenti per chiavi, agente, SCP e SFTP e plasmò numerose estensioni de facto come il proprio formato di certificato ([OpenSSH Project](https://www.openssh.com/)).

La crittografia si evolse da combinazioni più vecchie di Diffie-Hellman, DSA, RSA/SHA-1, CBC e MAC separati a Curve25519/ECDH, Ed25519, RSA/SHA-2 e procedure AEAD. Al contempo, la fiducia operativa si spostò da singole chiavi distribuite manualmente verso autenticatori hardware, Host/User CA, certificati di breve durata e bastion basate sull'identità. I livelli del protocollo sono rimasti riconoscibili; la gestione di fiducia e ciclo di vita è diventata più estesa.

## Checklist dell'amministratore in sintesi

La checklist riassume percorso di connessione, fiducia host, diritti utente, gestione delle chiavi e registrazione. È adatta per l'approvazione e la revisione periodica di un accesso SSH.

| Domanda | Evidenza operativa |
|---|---|
| Quale percorso? | Client/contesto, DNS, IP di origine, Bastion/ProxyJump, IP di destinazione, porta, processo listener |
| Quale identità server? | fingerprint Host Key o certificato host, principal, CA, validità, rotazione |
| Quali algoritmi? | procedure KEX, Host Key, Cipher/AEAD e MAC negoziate, eccezioni legacy |
| Chi si autentica? | account, principal, fingerprint Public Key/seriale certificato, percorso MFA/PAM, dispositivo di origine |
| Cosa è autorizzato? | gruppi, blocco Match, opzioni Authorized Key, sudo/token, sottosistema/comando |
| Quali canali? | Shell/Exec/SFTP, PTY, forward Local/Remote/Dynamic, Agent/X11, PermitOpen |
| Quale policy vale davvero? | `sshd -T -C`, `ssh -G`, Include, default della distribuzione e contesto Match |
| Cosa viene registrato? | Connect/Auth/Session/Channel, escalation dei privilegi, trasferimento e correlazione centrale |
| Come avviene la rotazione? | Host Key/CA trust paralleli, certificati utente brevi, inventario e revoca |
| Come avviene il ripristino? | console OOB, Host/User CA, policy, principal/chiavi, DNS/bastion, Break Glass e audit |

## Fonti

- [RFC 4251 – SSH Protocol Architecture](https://www.rfc-editor.org/rfc/rfc4251.html)
- [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)
- [RFC 4253 – SSH Transport Layer Protocol](https://www.rfc-editor.org/rfc/rfc4253.html)
- [RFC 9142 – Key Exchange Method Recommendations for SSH](https://www.rfc-editor.org/rfc/rfc9142.html)
- [RFC 8308 – Extension Negotiation in SSH](https://www.rfc-editor.org/rfc/rfc8308.html)
- [RFC 9519 – Strict Key Exchange in SSH](https://www.rfc-editor.org/rfc/rfc9519.html)
- [ssh(1)](https://man.openbsd.org/ssh)
- [sshd(8)](https://man.openbsd.org/sshd)
- [RFC 4255 – SSHFP Resource Records](https://www.rfc-editor.org/rfc/rfc4255.html)
- [IETF-Draft – OpenSSH Certificate Format](https://datatracker.ietf.org/doc/html/draft-ietf-sshm-cert)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [ssh-keyscan(1)](https://man.openbsd.org/ssh-keyscan)
- [ssh-keygen(1)](https://man.openbsd.org/ssh-keygen)
- [ssh_config(5)](https://man.openbsd.org/ssh_config)
- [RFC 4252 – SSH Authentication Protocol](https://www.rfc-editor.org/rfc/rfc4252.html)
- [RFC 4256 – Keyboard Interactive Authentication](https://www.rfc-editor.org/rfc/rfc4256.html)
- [sshd_config(5)](https://man.openbsd.org/sshd_config)
- [RFC 8332 – RSA Keys with SHA-2 Signatures](https://www.rfc-editor.org/rfc/rfc8332.html)
- [RFC 8709 – Ed25519 and Ed448 for SSH](https://www.rfc-editor.org/rfc/rfc8709.html)
- [OpenSSH FIDO/U2F support](https://www.openssh.com/txt/release-8.2)
- [ssh-add(1)](https://man.openbsd.org/ssh-add)
- [Microsoft Learn – Get-Service](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-service)
- [GNU grep manual](https://www.gnu.org/software/grep/manual/grep.html)
- [Microsoft Learn – Select-String](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string)
- [RFC 4254 – SSH Connection Protocol](https://www.rfc-editor.org/rfc/rfc4254.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [sftp(1)](https://man.openbsd.org/sftp)
- [sftp-server(8)](https://man.openbsd.org/sftp-server)
- [scp(1)](https://man.openbsd.org/scp)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [Microsoft Learn – OpenSSH overview](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-overview)
- [Microsoft Learn – OpenSSH Server configuration](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)
- [OpenSSH – History](https://www.openssh.com/history.html)
- [OpenSSH Project](https://www.openssh.com/)
