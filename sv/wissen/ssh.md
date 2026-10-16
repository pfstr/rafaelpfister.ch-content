---
title: "SSH: transport, identiteter och säker administration"
blatt: "ssh"
description: "SSH för infrastruktur- och meddelandeadministratörer: transport, nyckelutbyte och värdidentitet, användar- och arbetsbelastningsautentisering, kanaler, OpenSSH-nycklar och certifikat, agenter, bastioner och vidarebefordran, SFTP/SCP, serverpolicy, härdning, loggning, Windows OpenSSH och teknisk historik."
fakten:
  - label: Systemroll
    wert: krypterat, integritetsskyddat och autentiserat transport- och kanallager för administration
    href: https://www.rfc-editor.org/rfc/rfc4251.html
  - label: Standardport
    wert: TCP 22; faktisk lyssnare och åtkomstväg styrs av konfigurationen
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml
  - label: Protokollstack
    wert: Transport Protocol · User Authentication Protocol · Connection Protocol
    href: https://www.rfc-editor.org/rfc/rfc4251.html
  - label: Serveridentitet
    wert: Host Key eller Host Certificate; kontrollera före användaruppgifter
    href: https://www.rfc-editor.org/rfc/rfc4253.html
  - label: Användarautentisering
    wert: publickey · password · hostbased · keyboard-interactive; policy kan kräva flera faktorer
    href: https://www.rfc-editor.org/rfc/rfc4252.html
  - label: Kryptografisk förhandling
    wert: KEX · Host-Key-algoritm · Cipher · MAC/AEAD · komprimering
    href: https://www.rfc-editor.org/rfc/rfc9142.html
  - label: Nyckeltyper
    wert: Ed25519 · ECDSA · RSA/SHA-2 · FIDO-Security-Key-varianter; användning enligt klient-/serverkompatibilitet
    href: https://www.rfc-editor.org/rfc/rfc8709.html
  - label: Åtkomstkanaler
    wert: Shell · Exec · Subsystem · TCP-/Streamlocal-Forwarding · Agent-/X11-vidarebefordran
    href: https://www.rfc-editor.org/rfc/rfc4254.html
  - label: Filöverföring
    wert: SFTP som SSH-subsystem; scp-CLI kan, beroende på implementation, använda SFTP- eller äldre SCP-semantik
    href: https://man.openbsd.org/sftp
  - label: Trust-distribution
    wert: known_hosts/Pinning · SSHFP med DNSSEC · Host-CA · OpenSSH User-/Host-certifikat
    href: https://www.rfc-editor.org/rfc/rfc4255.html
  - label: Driftstatus
    wert: Listener · effektiv policy · Host Keys · autentiseringsväg · sessioner/kanaler · loggar · källnät
    href: https://man.openbsd.org/sshd_config
  - label: Recoveryobjekt
    wert: värdidentitet/CA, serverpolicy, Authorized Principals/Keys, Break Glass, granskningsväg och bastionberoenden
    href: https://man.openbsd.org/ssh-keygen
werbung:
  - newsletter
ctaThemen:
  - claude
  - powershell
  - haertung
translationSourceHash: 9617920ae627ef338448276f4094696352ba3f158be192d73dd10226c8ee296f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T09:31:32.660Z
translationReview: automatic
---

# SSH: transport, identiteter och säker administration

Secure Shell är en protokollfamilj för säker fjärråtkomst över ett osäkert nätverk. SSH kombinerar tre uppgifter: det etablerar en krypterad och integritetsskyddad transport, autentiserar först servern och därefter användaren eller arbetsbelastningen och multiplexar slutligen skal, enskilda kommandon, subsystem och vidarebefordringar som separata kanaler. Denna ordning är avgörande för administratörer: en stark användarnyckel skyddar inte om klienten erbjuder den till en overifierad server.

SSH är samtidigt protokoll och kontrollplan. En åtkomst kan möjliggöra rotkommandon, filöverföring, databastunnlar, agentåtkomst och hoppvägar till interna nätverk. Säkerhetsbedömningen får därför inte stanna vid lyssnarporten eller `PasswordAuthentication no`. Värdnyckeldistribution, effektiv Match-policy, nyckelursprung, principals, vidarebefordringsrättigheter, källväg, sessionsloggning och återställning av trust anchors hör till samma system.

Förklaringen följer en SSH-anslutning från TCP och versionsutbyte via nyckelförhandling och värdkontroll till användarinloggning och kanal. Vidarebefordringar, nyckeldrift, certifikat och diagnostik bygger på detta.

SSH skapar en autentiserad, krypterad transport ur en osäker nätverksanslutning och separata kanaler för skal, kommandon, filöverföring eller tunnlar inom den. Artikeln följer exakt denna ordning: serveridentitet, nyckelutbyte, användarinloggning, kanaler och drift.

## Protokollarkitektur

RFC 4251 delar in SSH-2 i tre protokoll:

1. **Transport Layer Protocol:** Versionsutbyte, algoritmförhandling, Key Exchange, serverautentisering, kryptering och integritet.
2. **User Authentication Protocol:** Autentisering av en klientanvändare mot servern över den skyddade transporten.
3. **Connection Protocol:** Multiplexering av logiska kanaler för sessioner, port forwarding och andra tjänster.

Lagren körs över en tillförlitlig bytestream, vanligen [TCP](/kb/tcp). TCP-port 22 är registrerad hos IANA som `ssh`; reverse proxies, portvidarebefordringar och bastioner kan ändra den faktiska vägen ([RFC 4251 – SSH Protocol Architecture](https://www.rfc-editor.org/rfc/rfc4251.html), [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-ssh.svg?v=20260813" title="Interaktive Infografik: SSH-Verbindungsaufbau von TCP und Versionsaustausch über KEX, Hostidentität und Benutzerautorisierung bis zu Channels, Bastion, Schlüssel- und Zertifikatslebenszyklus sowie Audit und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-ssh.svg?v=20260813">Öppna interaktiv grafik direkt</a>.
</iframe>

## Versionsutbyte och algoritmförhandling

Klient och server skickar först identifieringssträngar som `SSH-2.0-...`. Därefter utbyter de `SSH_MSG_KEXINIT` med ordnade listor för Key Exchange, Host-Key-algoritm, kryptering i båda riktningarna, MAC, komprimering och språk. Det första gemensamma alternativet per kategori väljs. Algoritmlistor är därför policy och kompatibilitetsavtal, inte bara en implementationsdetalj ([RFC 4253 – SSH Transport Layer Protocol](https://www.rfc-editor.org/rfc/rfc4253.html)).

RFC 9142 uppdaterar rekommendationer för SSH-algoritmer. Den skiljer mellan `MUST`, `SHOULD`, `MAY`, `SHOULD NOT` och `MUST NOT` för implementationer; en produktionspolicy tar dessutom hänsyn till efterlevnad, klientbestånd, hårdvara och migrationsväg ([RFC 9142 – Key Exchange Method Recommendations for SSH](https://www.rfc-editor.org/rfc/rfc9142.html)).

Konfigurationsrader som fastlåser en egen statisk cipherlista blir föråldrade. OpenSSH kan visa sina algoritmer som stöds med `ssh -Q` och visa den effektivt upplösta klient- respektive serverkonfigurationen. Operatörer tar målmedvetet bort kända äldre metoder, men behåller en testad kompatibilitetsväg för appliances.

## Key Exchange och sessionsnycklar

Key Exchange skapar en gemensam hemlighet och en exchange hash utan att skicka sessionsnyckeln över nätverket. Hashet binder identifieringssträngar, KEXINIT-meddelanden, serverns host key och algoritmspecifika ephemeralvärden. Servern signerar exchange hashet med sin värdnyckel; klienten kontrollerar därmed serveridentitet och nyckelutbyte tillsammans ([RFC 4253](https://www.rfc-editor.org/rfc/rfc4253.html)).

Moderna metoder använder ephemerala Diffie-Hellman-varianter och erbjuder Forward Secrecy för sessionsnycklar. SSH kan göra rekey under en anslutning, till exempel efter datamängd eller tid. Rekey och Connection Channels delar samma transport; problem visar sig därför som frysningar eller frånkopplingar i till synes aktiva sessioner.

RFC 8308 definierar en generisk Extension Negotiation. OpenSSH använder bland annat utökningar för att meddela serversignaturalgoritmer eller host key-uppdateringar ([RFC 8308 – Extension Negotiation in SSH](https://www.rfc-editor.org/rfc/rfc8308.html)). Strict Key Exchange skyddar den inledande sekvensen starkare mot oväntade meddelanden och standardiserades efter Terrapin-analysen ([RFC 9519 – Strict Key Exchange in SSH](https://www.rfc-editor.org/rfc/rfc9519.html)).

Efter versions- och algoritmval måste klienten först identifiera servern. Först när Host Key har accepterats är den efterföljande användarinloggningen meningsfull.

## Host Keys: serveridentitet före användaridentitet

Host Key identifierar servern eller en kontrollerad servergrupp. Vid första kontakten har en klient utan en fördelad trust anchor ingen kryptografisk grund för att bedöma det visade fingeravtrycket. ”Accept new” automatiserar pinning för nya värdar, men löser inte säker initial distribution.

OpenSSH lagrar kända nycklar och markörer i `known_hosts`. En ändrad nyckel kan betyda legitim rotation, nyinstallation, ändrad load-balancer-pool eller en attack. Varningen ”åtgärdas” inte genom att blint radera raden; den nya nyckeln verifieras via en oberoende källa och orsaken dokumenteras ([ssh(1) – host key verification](https://man.openbsd.org/ssh), [sshd(8)](https://man.openbsd.org/sshd)).

### Trustmodeller för värdidentitet

| Modell | Styrka | Driftsgräns |
|---|---|---|
| manuellt fingeravtryck out-of-band | direkt och enkelt för få värdar | säker distribution och rotation skalar dåligt |
| `known_hosts`-pinning/TOFU | identifierar senare ändringar | första kontakt förblir obekräftad |
| SSHFP med DNSSEC | DNS publicerar fingeravtryck | klienten måste validera DNSSEC; resolvervägen är en trustgräns |
| OpenSSH Host-CA | klienter litar på CA och principals | CA-nyckel, signeringsprocess, giltighet och principalmappning |
| konfigurationshantering | centralt distribuerad och granskningsbar | pipeline- och enrollmenttrust |

RFC 4255 definierar SSHFP-poster; utan DNSSEC kan de bara vara en indikation, inte ensam trust anchor ([RFC 4255 – SSHFP Resource Records](https://www.rfc-editor.org/rfc/rfc4255.html)). OpenSSH-certifikat binder Public Key, serienummer, typ, principals, giltighet, Critical Options och Extensions genom en CA-signatur ([IETF-Draft – OpenSSH Certificate Format](https://datatracker.ietf.org/doc/html/draft-ietf-sshm-cert)).

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) läser [DNS](/kb/dns); [`ssh-keyscan`](https://man.openbsd.org/ssh-keyscan) samlar erbjudna nycklar, **men autentiserar dem inte**. [`ssh-keygen`](https://man.openbsd.org/ssh-keygen) beräknar fingeravtryck och söker i `known_hosts`. Kandidater används först efter oberoende verifiering.

## Host-Key-livscykel och rotation

En server kan ha flera Host Keys för olika algoritmer. Privata Host Keys finns endast på servrar eller kontrollerade load-balancer-komponenter; säkerhetskopior och kloner får inte leda till oavsiktlig gemensam identitet. Vid autoskalning är en Host-CA ofta bättre lämpad än kopierade privata Host Keys.

En rotation omfattar:

1. skapa och skydda ny nyckel eller nytt hostcertifikat;
2. erbjuda parallellt och distribuera fingeravtryck/CA-trust;
3. kontrollera faktisk klientkompatibilitet och host principal;
4. ta bort gammal nyckel kontrollerat;
5. uppdatera inventarium, revocation och Break-Glass-dokumentation.

OpenSSH kan göra ytterligare Host Keys kända efter lyckad autentisering via `UpdateHostKeys`. Det underlättar rotation, men förutsätter att befintlig host trust redan är korrekt ([ssh_config(5)](https://man.openbsd.org/ssh_config)).

Med skyddad transport och bekräftad server börjar användar- eller maskininloggningen. Lösenord, Public Key, certifikat och andra metoder har olika krav för drift och återställning.

## User Authentication Protocol

Efter transporten begär klienten tjänsten `ssh-userauth`. RFC 4252 definierar metoderna `publickey`, `password`, `hostbased` och `none`; Keyboard-Interactive specificeras i RFC 4256. Servern kan kräva ytterligare autentisering och meddelar tillåtna nästa metoder ([RFC 4252 – SSH Authentication Protocol](https://www.rfc-editor.org/rfc/rfc4252.html), [RFC 4256 – Keyboard Interactive Authentication](https://www.rfc-editor.org/rfc/rfc4256.html)).

**Autentiseringsmetod** och **auktorisering** är separata. En giltig Public Key kan mappas till ett konto genom `authorized_keys`, katalogtjänst, `AuthorizedKeysCommand`, certifikatprincipals eller PAM. Därefter avgör Unix-rättigheter, `sudo`, SELinux/AppArmor, Windows-token och applikationspolicy de effektiva rättigheterna.

OpenSSH `AuthenticationMethods` kan tvinga fram kombinationer som `publickey,keyboard-interactive:pam`. Huruvida dessa är två oberoende faktorer beror på credentialkällor och enhetsbindning. Två metoder som båda kommer från samma komprometterade laptopagent ger mindre oberoende än vad syntaxen antyder ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

## Public-Key-metoder och nyckeltyper

Vid Public-Key-autentisering skickar klienten en signatur över sessionsidentifieraren och autentiseringsbegäran; den privata nyckeln lämnar inte klienten. Servern kontrollerar Public Key, signatur och auktorisering. RFC 8332 definierar RSA-signaturer med SHA-2 för SSH; nyckeltypen `ssh-rsa` betecknar historiskt även SHA-1-signatursemantik och får inte likställas med varje användning av en RSA-nyckel ([RFC 8332 – RSA Keys with SHA-2 Signatures](https://www.rfc-editor.org/rfc/rfc8332.html)).

RFC 8709 standardiserar Ed25519 och Ed448 för SSH. OpenSSH stöder dessutom FIDO/U2F-baserade `sk-`-nycklar, där den privata nyckeln är bunden till en Security Key och valfritt kräver användarnärvaro eller användarverifiering ([RFC 8709 – Ed25519 and Ed448 for SSH](https://www.rfc-editor.org/rfc/rfc8709.html), [OpenSSH FIDO/U2F support](https://www.openssh.com/txt/release-8.2)).

Valet följer klient-/serverkompatibilitet och hotmodell. Enbart nyckellängd avgör inte säkerheten: privat lagring, passfras, agentåtkomst, giltighet, principal, revocation och auktoriseringsalternativ är lika viktiga.

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

[`ssh-add`](https://man.openbsd.org/ssh-add) hanterar identiteter i agenten och kan testa en Public Keys signeringsförmåga. [`Get-Service`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-service) visar Windows-agenttjänsten. Privata nyckelfiler och agentsockets får minimala ACL:er; exemplen skapar inga centrala CA- eller Host Keys.

## `authorized_keys` som policyobjekt

En rad i `authorized_keys` kan innehålla alternativ före nyckeln. `from=`, `command=`, `restrict`, `no-port-forwarding`, `no-agent-forwarding`, `no-X11-forwarding`, `no-pty`, `permitopen=` och `environment=` begränsar användningen. `restrict` anger en defensiv uppsättning alternativ; explicita tillåtelser kan följa ([sshd(8) – AUTHORIZED_KEYS FILE FORMAT](https://man.openbsd.org/sshd#AUTHORIZED_KEYS_FILE_FORMAT)).

En nyckel för backup eller deployment får ingen generell shellåtkomst. Den binds till källnät, tvingat kommando och nödvändiga vidarebefordringsmål. Wrappers validerar argument som strukturerad indata och använder ingen okontrollerad shellkonkatenering.

`authorized_keys` är distribuerad konfiguration. Inventariet omfattar fingeravtryck, ägare, syfte, målkonton, källa, utgångsdatum, senaste användning och återkallelse. Övergivna nycklar är långlivade credentials utan central överblick.

## OpenSSH-certifikat och principals

OpenSSH-certifikat är inte X.509-certifikat. En SSH-CA signerar en Public Key med serienummer, Key ID, typ, principals och giltighetsfönster. User Certificates autentiserar användare mot servrar; Host Certificates autentiserar servrar mot klienter. Critical Options måste förstås, annars nekas certifikatet; Extensions ger funktioner som PTY eller forwarding ([IETF-Draft – OpenSSH Certificate Format](https://datatracker.ietf.org/doc/html/draft-ietf-sshm-cert)).

Certifikat förkortar credentiallivslängden och centraliserar utfärdandet, men eliminerar inte auktorisering. Servrar fastställer med `TrustedUserCAKeys` och Authorized Principals vilken CA och vilka principals som gäller för respektive konto. Klienter binder en Host-CA i `known_hosts` med `@cert-authority` till namnmönster.

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

[`grep`](https://www.gnu.org/software/grep/manual/grep.html) och [`Select-String`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string) filtrerar endast den effektiva klientkonfigurationen. Debugutdata från `ssh -vvv` kan innehålla användarnamn, sökvägar, målinfrastruktur och erbjudna fingeravtryck och behandlas som ett känsligt diagnosartefakt.

## ssh-agent och Agent Forwarding

En agent håller privata nycklar eller signeringshandles upplåsta och besvarar signeringsförfrågningar via en lokal socket. Processer med åtkomst till denna socket kan missbruka agenten utan att exportera Private Key. Passfras och hårdvarunyckel begränsar därför inte varje agentkompromettering.

Agent Forwarding vidarebefordrar agentkanalen till en fjärrvärd. Root eller en komprometterad process på denna värd kan under sessionen göra signeringsförfrågningar. OpenSSH varnar uttryckligen för denna risk; `ProxyJump` transporterar anslutningen genom en bastion utan att göra agentsocketen tillgänglig där ([ssh(1) – `-A` and `-J`](https://man.openbsd.org/ssh)).

Om ett mål endast nås via en Jump Host är följande modeller att föredra:

- `ProxyJump`/`ProxyCommand` utan credentiallagring på bastionen;
- kortlivade SSH-certifikat med målrelaterade principals;
- separat nyckel per trustdomän;
- kontrollerad sessionsförmedling med audit, om interaktivitet måste förmedlas centralt.

Efter lyckad inloggning öppnar SSH logiska kanaler inom samma anslutning. Shell, enskilda kommandon, SFTP och port forwarding delar därmed transport och identitet, men förblir separata operationer.

## Connection Protocol och kanaler

Efter lyckad autentisering öppnar SSH logiska kanaler. RFC 4254 definierar windowing, kanaldata, requests och typer. En session kan begära en PTY, ett shell, ett enskilt `exec`-kommando eller ett subsystem som SFTP. TCP Forwarding öppnar `direct-tcpip`- eller `forwarded-tcpip`-kanaler ([RFC 4254 – SSH Connection Protocol](https://www.rfc-editor.org/rfc/rfc4254.html)).

Multiplexering gör att flera klientanslutningar kan dela en befintlig transport. Det minskar KEX- och autentiseringskostnader, men ökar effekten av en komprometterad Control Socket. `ControlPath`, behörigheter, beständighet och målbindning anges explicit.

Keepalives finns på olika nivåer: TCPKeepAlive kontrollerar TCP-vägen, `ServerAliveInterval`/`ClientAliveInterval` använder SSH-meddelanden. De förhindrar inte programhängningar och kan med aggressiva värden avsluta sessioner vid korta nätavbrott.

## Port Forwarding och trustgränser

Local Forwarding (`-L`) öppnar en lyssnare på klientsidan och ansluter från servernätet till målet. Remote Forwarding (`-R`) öppnar en lyssnare på server- eller motpartssidan. Dynamic Forwarding (`-D`) erbjuder en SOCKS-proxy. Unix-socket-/streamlocal-forwarding kompletterar TCP.

Forwarding kan kringgå segmentering. En användare utan shellprivilegier kan trots det nå interna administrationsgränssnitt eller databaser via en tillåten vidarebefordran. Servrar styr detta med `AllowTcpForwarding`, `PermitOpen`, `PermitListen`, `GatewayPorts`, Match-block och Authorized-Key-alternativ ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

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

[`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) kontrollerar TCP-vägen till bastionen. [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) och [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) bekräftar den lokala lyssnaren. `-N`-tunneln kör inget fjärrkommando; auktorisering och mål-[TLS](/kb/tls) förblir separata gränser.

## SFTP, SCP och filrättigheter

SFTP är ett binärt filöverföringsprotokoll som vanligtvis körs som ett SSH-subsystem. Det är inte FTP över SSH. OpenSSH kan köra `internal-sftp` utan extern SFTP-serverbinär och begränsa till filöverföring genom `ChrootDirectory` plus `ForceCommand internal-sftp` ([sftp(1)](https://man.openbsd.org/sftp), [sftp-server(8)](https://man.openbsd.org/sftp-server)).

`scp`-CLI:n använde historiskt det SCP/RCP-liknande fjärrprotokollet och använder i moderna OpenSSH-versioner SFTP som standard; `-O` tvingar fram äldre protokoll. Wildcards, fjärrskalbearbetning och serverkompatibilitet skiljer sig åt ([scp(1)](https://man.openbsd.org/scp)). Runbooks anger därför explicit CLI, protokolläge och önskade filmetadata.

Auktorisering för filöverföring omfattar chrootägarskap, målsökvägar, UID/GID respektive Windows-ACL:er, umask, symlänkar, quota och atomisk placering. Uppladdning till en e-post- eller automatiseringskatalog kräver ofta en karantän-/renamegräns i stället för direkt behandling av delvis skrivna filer.

## Serversidig policy och Match-block

`sshd_config` utvärderas uppifrån och ned; för varje nyckelord gäller i princip det första erhållna värdet. `Match` ändrar tillämpningsområdet efter användare, grupp, host, adress, LocalAddress, LocalPort eller RDomain. Include-filer, distributionsstandarder och kommandoradsalternativ kan komplettera den synliga huvudfilen ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

En robust baslinje fastställer:

- tillåtna användare/grupper och källnät;
- nödvändiga autentiseringsmetoder och Break-Glass-undantag;
- root-/administratörsåtkomst och Privilege Escalation;
- forwarding, agent, X11, PTY och subsystem;
- MaxAuthTries, LoginGraceTime, session-/startgränser och keepalives;
- Host Keys, algoritmer och rekey;
- AuthorizedKeys/Principals-källor;
- loggning, banner och miljöantaganden;
- Match-specifika avvikelser.

Generella ”härdningslistor” kan stänga ute appliances eller vara verkningslösa om ett senare accepterat Match-värde tillämpas annorlunda. Det avgörande är den effektiva konfigurationen för en konkret anslutning.

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

`sshd -t` validerar syntax och nycklar, `sshd -T -C` visar den effektiva konfigurationen för testparametrar ([sshd(8)](https://man.openbsd.org/sshd)). [`Get-WinEvent`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent) och [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) läser plattformsloggarna. En lyckad omladdning och en andra testad administratörssession föregår stängning av den befintliga sessionen.

## Brute force, rate limits och nätgränser

Internetexponerade SSH-tjänster får automatiserade inloggningsförsök. Källbrandvägg, VPN/Zero-Trust-åtkomst, bastion, `AllowUsers`/`AllowGroups`, stark autentisering och kort credentiallivslängd minskar attackytan. En annan port minskar loggbrus, men förändrar varken autentisering eller sårbarhet.

Perimeter-blockeringsverktyg kan begränsa upprepade försök; de behandlar opålitliga loggdata och behöver egna regler för allowlist, IPv6, NAT, lockout och återställning. OpenSSH `MaxStartups` begränsar sannolikhetsmässigt parallella oautentiserade anslutningar, `PerSourceMaxStartups` och `PerSourcePenalties` kan beakta källbeteende. Kapacitet och DoS-risk testas under verkliga proxy-/NAT-förhållanden ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

## Loggning, audit och sessionsbevis

SSH-loggar visar vanligen anslutningskälla, användare, autentiseringsmetod/fingeravtryck, sessionsstart/-slut och utvalda subsystemhändelser. De innehåller inte automatiskt varje kört shellkommando. Shell History kontrolleras av användaren och är ingen manipulationsskyddad audit.

För privilegierad administration korreleras:

- källidentitet, enhet och åtkomstbeslut före SSH;
- bastion-/proxyhopp och målvärd;
- klientens beslut om Host-Key/CA;
- användarprincipal, autentiseringsmetod, nyckelfingeravtryck eller certifikatserie;
- session-/kanaltyp, start, slut och exitstatus;
- `sudo`-/Windows-privilegiehändelser och målsystemaudit;
- filöverföringssökväg, hash eller change ticket vid känsliga deploymenter.

Sessionsinspelning förändrar dataskydd, hemlighetsinsamling och driftsrisk. Lösenord, tokens, e-postinnehåll och Private Keys får inte okontrollerat hamna i transkript. Lagring, åtkomst och manipulationsskydd definieras före aktivering.

## Windows OpenSSH

Microsoft tillhandahåller OpenSSH Client och Server som Windows-funktioner. Servern använder tjänsten `sshd`, konfigurationen under `%ProgramData%\ssh\sshd_config`, Windows-konton och ACL:er; loggar kan visas i kanalen `OpenSSH/Operational` ([Microsoft Learn – OpenSSH overview](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-overview), [Microsoft Learn – OpenSSH Server configuration](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)).

Windows-administratörsnycklar kan beroende på konfiguration finnas i en central `administrators_authorized_keys`-fil, vars ACL måste vara strikt. Shell, subsystem, DefaultShell-registervärde och sökvägsupplösning skiljer sig från Unix. Ett portabelt klientkommando innebär inte identisk serverauktorisering.

## Backup, recovery och Break Glass

SSH-recovery omfattar inte bara `sshd_config`:

- Host Keys eller Host-CA plus skyddade privata nycklar;
- User-CA, certifikatutgivare, principals och revocationlistor;
- Authorized Keys/Commands och deras katalog-/konfigurationshanteringskälla;
- serverpolicy, Include-filer, PAM/SSSD/AD- och MFA-beroenden;
- brandvägg, bastion, DNS/SSHFP och tidskälla;
- Break-Glass-konto med offlineverifierat credential;
- out-of-band-konsol om nätverks-SSH slutar fungera;
- auditloggar och nyckel-/certifikatinventarium.

Återställning av samma privata Host Key bevarar serveridentiteten, men kan vara farligt efter en kompromettering. Recovery avgör därför mellan **kontinuitet** och **ny enrollment med rotation**. Ett restoretest använder en isolerad måladress, kontrollerar host principal, autentiseringsmetoder, Match-policy, Privilege Escalation och loggkorrelation.

Diagnostiken läser anslutningsuppbyggnaden i ordning: TCP, versionsutbyte, Key Exchange, Host Key, användarautentisering och kanalöppning.

## Diagnostik efter protokollfas

SSH följer tydliga faser. Den som först avgör om TCP, nyckelförhandling, värdkontroll, användarinloggning eller kanal är påverkad undviker ospecifika ändringar av nycklar och serverkonfiguration.

| Fas | Typisk evidens | Vanlig felklass |
|---|---|---|
| TCP | DNS, routning, lyssnare, SYN/ACK | brandvägg, fel port, proxy/bastion |
| Version | klient `-vvv`, serverlogg | inkompatibel eller främmande implementation |
| KEX | förhandlade algoritmer | ingen gemensam KEX/Cipher/MAC, Strict-KEX-problem |
| Host Key | fingeravtryck, principal, CA/known_hosts | fel värd, rotation, DNS-/LB-mål |
| Userauth | erbjudna metoder, fingeravtryck/cert, PAM | nyckel inte auktoriserad, principal, ACL, MFA |
| Connection | kanalbegäran, PTY, subsystem, forwarding | ForceCommand, PermitOpen, SFTP/Chroot, policy |
| Fjärrprocess | exitstatus, stderr, miljö, rättigheter | shellväg, sudo, locale, applikation |

Ett `Permission denied (publickey)` inträffar efter lyckad transport och host trust. En timeout före bannern är inget nyckelproblem. Fasorienterad diagnostik förhindrar att credentials eller algoritmlistor ändras utan evidens.

## Teknisk historia

Tatu Ylönen utvecklade det ursprungliga SSH 1995 som ersättning för okrypterade verktyg för fjärråtkomst och filöverföring. SSH-1 och SSH-2 skilde sig grundläggande; IETF:s SECSH Working Group standardiserade senare SSH-2-arkitekturen i RFC 4251 till 4254 ([OpenSSH – History](https://www.openssh.com/history.html), [RFC 4251](https://www.rfc-editor.org/rfc/rfc4251.html)).

OpenSSH uppstod i OpenBSD-projektet ur fri SSH-kod och portades till andra operativsystem. Det förenade klient, server, nyckelverktyg, agent, SCP och SFTP och präglade många de facto-utökningar som det egna certifikatsformatet ([OpenSSH Project](https://www.openssh.com/)).

Kryptografin utvecklades från äldre Diffie-Hellman-, DSA-, RSA/SHA-1-, CBC- och separata MAC-kombinationer till Curve25519/ECDH, Ed25519, RSA/SHA-2 och AEAD-metoder. Samtidigt flyttades drifttillit från manuellt distribuerade enskilda nycklar till hårdvaruautentiserare, Host-/User-CA:er, kortlivade certifikat och identitetsbaserade bastioner. Protokollagren förblev igenkännbara; trust- och livscykeldriften blev mer omfattande.

## Administratörschecklista i korthet

Checklistan sammanfattar anslutningsväg, host trust, användarrättigheter, nyckeldrift och loggning. Den passar för godkännande och regelbunden granskning av en SSH-åtkomst.

| Fråga | Driftsbevis |
|---|---|
| Vilken väg? | klient/kontext, DNS, käll-IP, Bastion/ProxyJump, mål-IP, port, lyssnarprocess |
| Vilken serveridentitet? | Host-Key-fingeravtryck eller hostcertifikat, principal, CA, giltighet, rotation |
| Vilka algoritmer? | förhandlade KEX-, Host-Key-, Cipher-/AEAD- och MAC-metoder, äldre undantag |
| Vem autentiserar sig? | konto, principal, Public-Key-fingeravtryck/certifikatserie, MFA-/PAM-väg, källenhet |
| Vad är auktoriserat? | grupper, Match-block, Authorized-Key-alternativ, sudo/token, subsystem/command |
| Vilka kanaler? | Shell/Exec/SFTP, PTY, Local/Remote/Dynamic Forward, Agent/X11, PermitOpen |
| Vilken policy gäller faktiskt? | `sshd -T -C`, `ssh -G`, Includes, distributionsstandarder och Match-kontext |
| Vad loggas? | Connect/Auth/Session/Channel, Privilege Escalation, överföring och central korrelation |
| Hur roteras det? | parallell Host Key/CA-trust, korta usercertifikat, inventarium och revocation |
| Hur återställs det? | OOB-konsol, Host-/User-CA, policy, principals/keys, DNS/bastion, Break Glass och audit |

## Källor

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
