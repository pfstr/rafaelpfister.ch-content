---
title: "SSH: transport, identiteter og sikker administrasjon"
blatt: "ssh"
description: "SSH for infrastruktur- og meldingsadministratorer: transport, nøkkelutveksling og vertsidentitet, bruker- og arbeidsbelastningsautentisering, kanaler, OpenSSH-nøkler og -sertifikater, agenter, bastioner og videresending, SFTP/SCP, serverpolicy, herding, logging, Windows OpenSSH og teknisk historie."
fakten:
  - label: Systemrolle
    wert: kryptert, integritetsbeskyttet og autentisert transport- og kanallag for administrasjon
    href: https://www.rfc-editor.org/rfc/rfc4251.html
  - label: Standardport
    wert: TCP 22; faktisk lytter og tilgangsvei bestemmes av konfigurasjonen
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml
  - label: Protokollstakk
    wert: Transport Protocol · User Authentication Protocol · Connection Protocol
    href: https://www.rfc-editor.org/rfc/rfc4251.html
  - label: Serveridentitet
    wert: vertsnøkkel eller vertssertifikat; kontroller før brukerlegitimasjon
    href: https://www.rfc-editor.org/rfc/rfc4253.html
  - label: Brukerautentisering
    wert: publickey · password · hostbased · keyboard-interactive; policy kan kreve flere faktorer
    href: https://www.rfc-editor.org/rfc/rfc4252.html
  - label: Kryptografiforhandling
    wert: KEX · vertsnøkkelalgoritme · chiffer · MAC/AEAD · komprimering
    href: https://www.rfc-editor.org/rfc/rfc9142.html
  - label: Nøkkeltyper
    wert: Ed25519 · ECDSA · RSA/SHA-2 · FIDO-sikkerhetsnøkkelvarianter; bruk etter klient-/serverkompatibilitet
    href: https://www.rfc-editor.org/rfc/rfc8709.html
  - label: Tilgangskanaler
    wert: Shell · Exec · Subsystem · TCP-/Streamlocal-videresending · agent-/X11-videresending
    href: https://www.rfc-editor.org/rfc/rfc4254.html
  - label: Filoverføring
    wert: SFTP som SSH-subsystem; scp-CLI kan, avhengig av implementering, bruke SFTP- eller eldre SCP-semantikk
    href: https://man.openbsd.org/sftp
  - label: Tillitsdistribusjon
    wert: known_hosts/pinning · SSHFP med DNSSEC · Host-CA · OpenSSH bruker-/vertssertifikater
    href: https://www.rfc-editor.org/rfc/rfc4255.html
  - label: Driftstilstand
    wert: lytter · effektiv policy · verts­nøkler · autentiseringsvei · sesjoner/kanaler · logger · kildenettverk
    href: https://man.openbsd.org/sshd_config
  - label: Gjenopprettingsobjekt
    wert: vertsidentitet/CA, serverpolicy, Authorized Principals/Keys, Break Glass, revisjonsvei og bastionavhengigheter
    href: https://man.openbsd.org/ssh-keygen
werbung:
  - newsletter
ctaThemen:
  - claude
  - powershell
  - haertung
translationSourceHash: 9617920ae627ef338448276f4094696352ba3f158be192d73dd10226c8ee296f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-13T11:30:39.393Z
translationReview: required
---

# SSH: transport, identiteter og sikker administrasjon

Secure Shell er en protokollfamilie for sikker fjerntilgang over et usikkert nettverk. SSH kombinerer tre oppgaver: Det etablerer en kryptert og integritetsbeskyttet transport, autentiserer først serveren og deretter brukeren eller arbeidsbelastningen, og multiplekser til slutt sheller, enkeltkommandoer, subsystemer og videresendinger som separate kanaler. Denne rekkefølgen er avgjørende for administratorer: En sterk brukernøkkel beskytter ikke dersom klienten tilbyr den til en ikke-verifisert server.

SSH er både protokoll og kontrollplan. En tilgang kan muliggjøre root-kommandoer, filoverføring, databasetunneler, agenttilgang og hoppveier inn i interne nettverk. Sikkerhetsvurderingen må derfor ikke stoppe ved lytterporten eller ved `PasswordAuthentication no`. Distribusjon av verts­nøkler, effektiv Match-policy, nøkkelopprinnelse, principals, videresendingsrettigheter, kildevei, sesjonslogging og gjenoppretting av tillitsankere hører til samme system.

Forklaringen følger en SSH-forbindelse fra TCP og versjonsutveksling via nøkkelutveksling og vertskontroll til brukerinnlogging og kanal. Videresendinger, nøkkeldrift, sertifikater og diagnostikk bygger på dette.

SSH bygger en autentisert, kryptert transport fra en usikker nettverksforbindelse, med separate kanaler for shell, kommandoer, filoverføring eller tunneler. Artikkelen følger nettopp denne rekkefølgen: serveridentitet, nøkkelutveksling, brukerinnlogging, kanaler og drift.

## Protokollarkitektur

RFC 4251 deler SSH-2 inn i tre protokoller:

1. **Transport Layer Protocol:** Versjonsutveksling, algoritmeforhandling, Key Exchange, serverautentisering, kryptering og integritet.
2. **User Authentication Protocol:** Autentisering av en klientbruker mot serveren over den beskyttede transporten.
3. **Connection Protocol:** Multipleksing av logiske kanaler for sesjoner, portvideresending og andre tjenester.

Lagene kjører over en pålitelig bytestrøm, vanligvis [TCP](/kb/tcp). TCP-port 22 er registrert hos IANA som `ssh`; reverse-proxyer, portvideresendinger og bastioner kan endre den faktiske veien ([RFC 4251 – SSH-protokollarkitektur](https://www.rfc-editor.org/rfc/rfc4251.html), [IANA-registeret for tjenestenavn og portnumre](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-ssh.svg?v=20260813" title="Interaktive Infografik: SSH-Verbindungsaufbau von TCP und Versionsaustausch über KEX, Hostidentität und Benutzerautorisierung bis zu Channels, Bastion, Schlüssel- und Zertifikatslebenszyklus sowie Audit und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-ssh.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

## Versjonsutveksling og algoritmeforhandling

Klient og server sender først identifikasjonsstrenger som `SSH-2.0-...`. Deretter utveksler de `SSH_MSG_KEXINIT` med ordnede lister for Key Exchange, verts­nøkkelalgoritme, kryptering i begge retninger, MAC, komprimering og språk. Det første felles alternativet i hver kategori velges. Algoritmelister er derfor policy og kompatibilitetsavtale, ikke bare en implementasjonsdetalj ([RFC 4253 – SSH Transport Layer Protocol](https://www.rfc-editor.org/rfc/rfc4253.html)).

RFC 9142 oppdaterer anbefalingene for SSH-algoritmer. Den skiller mellom `MUST`, `SHOULD`, `MAY`, `SHOULD NOT` og `MUST NOT` for implementasjoner; en produksjonspolicy tar i tillegg hensyn til compliance, klientpark, maskinvare og migreringsvei ([RFC 9142 – Key Exchange Method Recommendations for SSH](https://www.rfc-editor.org/rfc/rfc9142.html)).

Konfigurasjonslinjer som fastlåser en egen, rigid chifferliste, blir utdaterte. OpenSSH kan vise sine støttede algoritmer med `ssh -Q` og vise effektivt oppløst klient- eller serverkonfigurasjon. Driftsansvarlige fjerner kjente eldre metoder målrettet, men beholder en testet kompatibilitetsvei for apparater.

## Key Exchange og sesjonsnøkler

Key Exchange genererer en felles hemmelighet og en utvekslingshash uten å sende sesjonsnøkkelen over nettet. Hashen binder identifikasjonsstrenger, KEXINIT-meldinger, serverens verts­nøkkel og algoritmespesifikke efemere verdier. Serveren signerer utvekslingshashen med verts­nøkkelen; klienten verifiserer dermed serveridentitet og nøkkelutveksling samlet ([RFC 4253](https://www.rfc-editor.org/rfc/rfc4253.html)).

Moderne metoder bruker efemere Diffie-Hellman-varianter og gir Forward Secrecy for sesjonsnøkler. SSH kan rekeye under en forbindelse, for eksempel etter datamengde eller tid. Rekey og Connection Channels deler samme transport; problemer fremstår derfor som frys eller frakobling av tilsynelatende aktive sesjoner.

RFC 8308 definerer en generisk utvidelsesforhandling. OpenSSH bruker utvidelser blant annet for å kunngjøre serversignaturalgoritmer eller verts­nøkkeloppdateringer ([RFC 8308 – Extension Negotiation in SSH](https://www.rfc-editor.org/rfc/rfc8308.html)). Strict Key Exchange beskytter den innledende sekvensen bedre mot uventede meldinger og ble standardisert etter Terrapin-analysen ([RFC 9519 – Strict Key Exchange in SSH](https://www.rfc-editor.org/rfc/rfc9519.html)).

Etter valg av versjon og algoritmer må klienten først gjenkjenne serveren. Først når verts­nøkkelen er akseptert, gir den etterfølgende brukerinnloggingen mening.

## Vertsnøkler: serveridentitet før brukeridentitet

Verts­nøkkelen identifiserer serveren eller en kontrollert servergruppe. Ved første kontakt har en klient uten forhåndsdistribuert tillitsanker ikke noe kryptografisk grunnlag for å vurdere det viste fingeravtrykket. «Accept new» automatiserer pinning for nye verter, men løser ikke sikker førstegangsdistribusjon.

OpenSSH lagrer kjente nøkler og markører i `known_hosts`. En endret nøkkel kan bety legitim rotasjon, nyinstallasjon, endret lastbalanseringspool eller et angrep. Advarselen «løses» ikke ved å slette linjen blindt; den nye nøkkelen kontrolleres via en uavhengig kilde og årsaken dokumenteres ([ssh(1) – verifisering av verts­nøkkel](https://man.openbsd.org/ssh), [sshd(8)](https://man.openbsd.org/sshd)).

### Tillitsmodeller for vertsidentitet

| Modell | Styrke | Driftsgrense |
|---|---|---|
| manuelt fingeravtrykk out-of-band | direkte og enkelt for få verter | sikker distribusjon og rotasjon skalerer dårlig |
| `known_hosts`-pinning/TOFU | oppdager senere endringer | første kontakt forblir udokumentert |
| SSHFP med DNSSEC | DNS publiserer fingeravtrykk | klienten må validere DNSSEC; resolverveien er tillitsgrense |
| OpenSSH Host-CA | klienter stoler på CA og principals | CA-nøkkel, signeringsprosess, gyldighet og principaltilordning |
| konfigurasjonsstyring | sentralt distribuert og reviderbar | tillit til pipeline og enrollment |

RFC 4255 definerer SSHFP-poster; uten DNSSEC kan de bare være en indikasjon, ikke et selvstendig tillitsanker ([RFC 4255 – SSHFP Resource Records](https://www.rfc-editor.org/rfc/rfc4255.html)). OpenSSH-sertifikater binder offentlig nøkkel, serienummer, type, principals, gyldighet, Critical Options og Extensions med en CA-signatur ([IETF-utkast – OpenSSH Certificate Format](https://datatracker.ietf.org/doc/html/draft-ietf-sshm-cert)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Hostidentität">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) leser [DNS](/kb/dns); [`ssh-keyscan`](https://man.openbsd.org/ssh-keyscan) samler tilbudte nøkler, men **autentiserer dem ikke**. [`ssh-keygen`](https://man.openbsd.org/ssh-keygen) beregner fingeravtrykk og søker i `known_hosts`. Kandidater tas først i bruk etter uavhengig verifisering.

## Livssyklus og rotasjon for verts­nøkler

En server kan ha flere verts­nøkler for forskjellige algoritmer. Private verts­nøkler ligger bare på servere eller kontrollerte lastbalanseringskomponenter; sikkerhetskopier og kloner må ikke føre til utilsiktet delt identitet. Ved autoskalering passer en Host-CA ofte bedre enn kopierte private verts­nøkler.

En rotasjon omfatter:

1. å opprette og beskytte ny nøkkel eller nytt vertssertifikat;
2. å tilby den parallelt og distribuere fingeravtrykk/CA-tillit;
3. å kontrollere faktisk klientkompatibilitet og vertsprincipal;
4. å fjerne gammel nøkkel kontrollert;
5. å oppdatere inventar, tilbakekalling og Break Glass-dokumentasjon.

OpenSSH kan kunngjøre ekstra verts­nøkler etter vellykket autentisering via `UpdateHostKeys`. Dette forenkler rotasjon, men forutsetter at den eksisterende vertstilliten allerede er korrekt ([ssh_config(5)](https://man.openbsd.org/ssh_config)).

Med beskyttet transport og bekreftet server begynner bruker- eller maskininnloggingen. Passord, offentlig nøkkel, sertifikat og andre metoder har ulike drifts- og gjenopprettingskrav.

## User Authentication Protocol

Etter transporten ber klienten om tjenesten `ssh-userauth`. RFC 4252 definerer metodene `publickey`, `password`, `hostbased` og `none`; Keyboard-Interactive er spesifisert i RFC 4256. Serveren kan kreve ytterligere autentisering og oppgir tillatte neste metoder ([RFC 4252 – SSH Authentication Protocol](https://www.rfc-editor.org/rfc/rfc4252.html), [RFC 4256 – Keyboard Interactive Authentication](https://www.rfc-editor.org/rfc/rfc4256.html)).

**Autentiseringsmetode** og **autorisasjon** er adskilt. En gyldig offentlig nøkkel kan tilordnes en konto via `authorized_keys`, katalogtjeneste, `AuthorizedKeysCommand`, sertifikatprincipals eller PAM. Deretter bestemmer Unix-rettigheter, `sudo`, SELinux/AppArmor, Windows-token og applikasjonspolicy de effektive rettighetene.

OpenSSH `AuthenticationMethods` kan tvinge kombinasjoner som `publickey,keyboard-interactive:pam`. Om dette er to uavhengige faktorer, avhenger av legitimasjonskilder og enhetsbinding. To metoder som begge kommer fra samme kompromitterte laptopagent, gir mindre uavhengighet enn syntaksen antyder ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

## Public-key-metoder og nøkkeltyper

Ved autentisering med offentlig nøkkel sender klienten en signatur over sesjonsidentifikator og autentiseringsforespørsel; den private nøkkelen forlater ikke klienten. Serveren kontrollerer offentlig nøkkel, signatur og autorisasjon. RFC 8332 definerer RSA-signaturer med SHA-2 for SSH; nøkkeltypen `ssh-rsa` betegner historisk også SHA-1-signatursemantikk og må ikke likestilles med enhver bruk av en RSA-nøkkel ([RFC 8332 – RSA Keys with SHA-2 Signatures](https://www.rfc-editor.org/rfc/rfc8332.html)).

RFC 8709 standardiserer Ed25519 og Ed448 for SSH. OpenSSH støtter i tillegg FIDO/U2F-baserte `sk-`-nøkler, der den private nøkkelen er bundet til en sikkerhetsnøkkel og valgfritt krever brukertilstedeværelse eller -verifisering ([RFC 8709 – Ed25519 and Ed448 for SSH](https://www.rfc-editor.org/rfc/rfc8709.html), [OpenSSH FIDO/U2F-støtte](https://www.openssh.com/txt/release-8.2)).

Valget følger klient-/serverkompatibilitet og trusselmodell. Nøkkellengde alene avgjør ikke sikkerheten: privat lagring, passfrase, agenttilgang, gyldighet, principal, tilbakekalling og autorisasjonsalternativer er like viktige.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Schlüssel- und Agentinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`ssh-add`](https://man.openbsd.org/ssh-add) administrerer identiteter i agenten og kan teste signeringsevnen til en offentlig nøkkel. [`Get-Service`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-service) viser Windows-agenttjenesten. Private nøkkelfiler og agentsockets får minimale ACL-er; eksemplene oppretter ingen sentrale CA- eller verts­nøkler.

## `authorized_keys` som policyobjekt

En linje i `authorized_keys` kan ha alternativer foran nøkkelen. `from=`, `command=`, `restrict`, `no-port-forwarding`, `no-agent-forwarding`, `no-X11-forwarding`, `no-pty`, `permitopen=` og `environment=` begrenser bruken. `restrict` setter en defensiv pakke med alternativer; eksplisitte tillatelser kan følge ([sshd(8) – AUTHORIZED_KEYS FILE FORMAT](https://man.openbsd.org/sshd#AUTHORIZED_KEYS_FILE_FORMAT)).

En nøkkel for sikkerhetskopiering eller distribusjon får ikke generell shelltilgang. Den bindes til kildenettverk, tvunget kommando og nødvendige videresendingsmål. Wrappere validerer argumenter som strukturert inndata og bruker ikke ukontrollert shellkonkatenering.

`authorized_keys` er distribuert konfigurasjon. Inventaret omfatter fingeravtrykk, eier, formål, målkontoer, kilde, utløp, siste bruk og tilbakekalling. Foreldreløse nøkler er langlivede legitimasjoner uten sentral oversikt.

## OpenSSH-sertifikater og principals

OpenSSH-sertifikater er ikke X.509-sertifikater. En SSH-CA signerer en offentlig nøkkel med serienummer, Key ID, type, principals og gyldighetsvindu. Brukersertifikater autentiserer brukere mot servere; vertssertifikater autentiserer servere mot klienter. Critical Options må forstås, ellers avvises sertifikatet; Extensions gir funksjoner som PTY eller videresending ([IETF-utkast – OpenSSH Certificate Format](https://datatracker.ietf.org/doc/html/draft-ietf-sshm-cert)).

Sertifikater forkorter levetiden for legitimasjon og sentraliserer utstedelse, men eliminerer ikke autorisasjon. Servere angir med `TrustedUserCAKeys` og Authorized Principals hvilken CA og hvilke principals som gjelder for hvilken konto. Klienter binder en Host-CA i `known_hosts` til navnemønstre med `@cert-authority`.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Zertifikatsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`grep`](https://www.gnu.org/software/grep/manual/grep.html) og [`Select-String`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string) filtrerer bare den effektive klientkonfigurasjonen. Feilsøkingsutdata fra `ssh -vvv` kan inneholde brukernavn, stier, målinfrastruktur og tilbudte fingeravtrykk og behandles som sensitivt diagnoseartefakt.

## ssh-agent og Agent Forwarding

En agent holder private nøkler eller signeringshåndtak opplåst og svarer på signeringsforespørsler via en lokal socket. Prosesser med tilgang til denne socketen kan misbruke agenten uten å eksportere den private nøkkelen. Passfrase og maskinvarenøkkel begrenser derfor ikke enhver kompromittering av agenten.

Agent Forwarding videresender agentkanalen til en ekstern vert. Root eller en kompromittert prosess på denne verten kan sende signeringsforespørsler under sesjonen. OpenSSH advarer uttrykkelig mot denne risikoen; `ProxyJump` transporterer forbindelsen gjennom en bastion uten å gjøre agentsocketen tilgjengelig der ([ssh(1) – `-A` og `-J`](https://man.openbsd.org/ssh)).

Når et mål bare er tilgjengelig via en Jump Host, foretrekkes følgende modeller:

- `ProxyJump`/`ProxyCommand` uten lagring av legitimasjon på bastionen;
- kortlivede SSH-sertifikater med målspesifikke principals;
- separat nøkkel per tillitsdomene;
- kontrollert sesjonsmegling med revisjon dersom interaktivitet må formidles sentralt.

Etter vellykket innlogging åpner SSH logiske kanaler innenfor samme forbindelse. Shell, enkeltkommando, SFTP og portvideresending deler dermed transport og identitet, men forblir separate operasjoner.

## Connection Protocol og kanaler

Etter vellykket autentisering åpner SSH logiske kanaler. RFC 4254 definerer windowing, kanaldata, forespørsler og typer. En sesjon kan be om en PTY, et shell, en enkelt `exec`-kommando eller et subsystem som SFTP. TCP-videresending åpner `direct-tcpip`- eller `forwarded-tcpip`-kanaler ([RFC 4254 – SSH Connection Protocol](https://www.rfc-editor.org/rfc/rfc4254.html)).

Multipleksing lar flere klientforbindelser dele en eksisterende transport. Det reduserer KEX- og autentiseringskostnader, men øker virkningen av en kompromittert kontrollsocket. `ControlPath`, rettigheter, persistens og målbinding settes eksplisitt.

Keepalives har ulike nivåer: TCPKeepAlive kontrollerer TCP-veien, `ServerAliveInterval`/`ClientAliveInterval` bruker SSH-meldinger. De forhindrer ikke applikasjonsheng og kan avslutte sesjoner ved korte nettverksavbrudd dersom verdiene er aggressive.

## Portvideresending og tillitsgrenser

Lokal videresending (`-L`) åpner en lytter på klientsiden og kobler fra servernettverket til målet. Ekstern videresending (`-R`) åpner en lytter på server- eller motsatt side. Dynamisk videresending (`-D`) tilbyr en SOCKS-proxy. Unix-socket-/Streamlocal-videresending supplerer TCP.

Videresending kan omgå segmentering. En bruker uten shellprivilegier kan likevel nå interne administrasjonsgrensesnitt eller databaser via en tillatt videresending. Servere styrer dette med `AllowTcpForwarding`, `PermitOpen`, `PermitListen`, `GatewayPorts`, Match-blokker og Authorized-Key-alternativer ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Bastion und Tunnelprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) kontrollerer TCP-veien til bastionen. [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) og [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) bekrefter den lokale lytteren. `-N`-tunnelen kjører ingen ekstern kommando; autorisasjon og mål-[TLS](/kb/tls) forblir selvstendige grenser.

## SFTP, SCP og filrettigheter

SFTP er en binær filoverføringsprotokoll som vanligvis kjører som SSH-subsystem. Det er ikke FTP over SSH. OpenSSH kan kjøre `internal-sftp` uten ekstern SFTP-serverbinærfil og begrense til filoverføring med `ChrootDirectory` pluss `ForceCommand internal-sftp` ([sftp(1)](https://man.openbsd.org/sftp), [sftp-server(8)](https://man.openbsd.org/sftp-server)).

`scp`-CLI-en brukte historisk den SCP/RCP-lignende eksterne protokollen og bruker i moderne OpenSSH-linjer SFTP som standard; `-O` tvinger eldre protokoll. Wildcards, ekstern shellbehandling og serverkompatibilitet varierer ([scp(1)](https://man.openbsd.org/scp)). Runbooks angir derfor CLI, protokollmodus og ønskede filmetadata eksplisitt.

Autorisasjon for filoverføring omfatter Chroot-eierskap, målbaner, UID/GID eller Windows-ACL-er, Umask, symlenker, kvote og atomisk plassering. Opplasting til en e-post- eller automatiseringskatalog trenger ofte en karantene-/omdøpingsgrense i stedet for direkte behandling av delvis skrevne filer.

## Serversidepolicy og Match-blokker

`sshd_config` evalueres ovenfra og ned; for hvert nøkkelord gjelder i utgangspunktet den første mottatte verdien. `Match` endrer virkeområdet etter bruker, gruppe, vert, adresse, LocalAddress, LocalPort eller RDomain. Include-filer, distribusjonsstandarder og kommandolinjealternativer kan supplere den synlige hovedfilen ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

En robust basislinje fastsetter:

- tillatte brukere/grupper og kildenettverk;
- nødvendige autentiseringsmetoder og Break Glass-unntak;
- root-/administratoradgang og privilege escalation;
- videresending, agent, X11, PTY og subsystemer;
- MaxAuthTries, LoginGraceTime, sesjons-/startgrenser og keepalives;
- verts­nøkler, algoritmer og rekey;
- AuthorizedKeys-/Principals-kilder;
- logging, banner og miljøforutsetning;
- Match-spesifikke avvik.

Generelle «herdingslister» kan stenge ute apparater eller være virkningsløse dersom en senere akseptert Match-verdi gjelder annerledes. Det avgjørende er den effektive konfigurasjonen for en konkret forbindelse.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Serverpolicy und Logs">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

`sshd -t` validerer syntaks og nøkler, `sshd -T -C` viser effektiv konfigurasjon for testparametere ([sshd(8)](https://man.openbsd.org/sshd)). [`Get-WinEvent`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent) og [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) leser plattformloggene. En vellykket reload og en annen testet administratorsesjon skal gå forut for at den eksisterende sesjonen lukkes.

## Brute force, rate limits og nettverksgrenser

Internet-eksponerte SSH-tjenester mottar automatiserte innloggingsforsøk. Kildefirewall, VPN/Zero-Trust-tilgang, bastion, `AllowUsers`/`AllowGroups`, sterk autentisering og kort levetid for legitimasjon reduserer angrepsflaten. En annen port reduserer loggstøy, men endrer verken autentisering eller sårbarhet.

Perimeterblokkeringsverktøy kan bremse gjentatte forsøk; de behandler upålitelige loggdata og trenger egne regler for allowlist, IPv6, NAT, lockout og recovery. OpenSSH `MaxStartups` begrenser ikke-autentiserte parallelle forbindelser probabilistisk, `PerSourceMaxStartups` og `PerSourcePenalties` kan ta hensyn til kildeadferd. Kapasitet og DoS-risiko testes under reelle proxy-/NAT-forhold ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

## Logging, revisjon og sesjonsbevis

SSH-logger dokumenterer vanligvis forbindelseskilde, bruker, autentiseringsmetode/fingeravtrykk, sesjonsstart/-slutt og utvalgte subsystemhendelser. De inneholder ikke automatisk hver utførte shellkommando. Shellhistorikk er brukerkontrollert og ikke en manipulasjonssikret revisjon.

For privilegert administrasjon korreleres:

- kildeidentitet, enhet og tilgangsbeslutning før SSH;
- bastion-/proxyhopp og målvert;
- klientens avgjørelse for verts­nøkkel/CA;
- brukerprincipal, autentiseringsmetode, nøkkelfingeravtrykk eller sertifikatserie;
- sesjons-/kanaltype, start, slutt og exitstatus;
- `sudo`-/Windows-privilegiehendelser og revisjon av målsystemet;
- filoverføringsbane, hash eller endringssak ved sensitive distribusjoner.

Sesjonsopptak endrer personvern, hemmelighetsinnsamling og driftsrisiko. Passord, tokener, e-postinnhold og private nøkler må ikke havne ukontrollert i transkripsjoner. Oppbevaring, tilgang og manipulasjonsbeskyttelse defineres før aktivering.

## Windows OpenSSH

Microsoft tilbyr OpenSSH Client og Server som Windows-funksjoner. Serveren bruker tjenesten `sshd`, konfigurasjonen under `%ProgramData%\ssh\sshd_config`, Windows-kontoer og ACL-er; logger kan vises i kanalen `OpenSSH/Operational` ([Microsoft Learn – OpenSSH-oversikt](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-overview), [Microsoft Learn – OpenSSH Server-konfigurasjon](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)).

Administratornøkler i Windows kan, avhengig av konfigurasjonen, ligge i en sentral `administrators_authorized_keys`-fil med strenge ACL-er. Shell, subsystemer, DefaultShell-registerverdien og baneoppløsning skiller seg fra Unix. En portabel klientkommando betyr ikke identisk serverautorisasjon.

## Sikkerhetskopiering, gjenoppretting og Break Glass

SSH-recovery omfatter ikke bare `sshd_config`:

- verts­nøkler eller Host-CA med beskyttede private nøkler;
- User-CA, sertifikatutsteder, principals og tilbakekallingslister;
- Authorized Keys/Commands og deres kilde i katalog-/konfigurasjonsstyring;
- serverpolicy, Include-filer, PAM/SSSD/AD- og MFA-avhengigheter;
- firewall, bastion, DNS/SSHFP og tidskilde;
- Break Glass-konto med offline-verifisert legitimasjon;
- out-of-band-konsoll dersom nettverks-SSH svikter;
- revisjonslogger og nøkkel-/sertifikatinventar.

Gjenoppretting av den samme private verts­nøkkelen bevarer serveridentiteten, men kan være farlig etter kompromittering. Recovery velger derfor mellom **kontinuitet** og **ny enrollment med rotasjon**. En gjenopprettingstest bruker en isolert måladresse og kontrollerer vertsprincipal, autentiseringsmetoder, Match-policy, privilege escalation og loggkorrelasjon.

Diagnostikken leser forbindelsesoppsettet i rekkefølge: TCP, versjonsutveksling, Key Exchange, verts­nøkkel, brukerautentisering og kanalåpning.

## Diagnostikk etter protokollfase

SSH forløper i klare faser. Den som først fastslår om TCP, nøkkelutveksling, vertskontroll, brukerinnlogging eller kanal er berørt, unngår målrettingsløse endringer i nøkler og serverkonfigurasjon.

| Fase | Typisk bevis | Vanlig feilklasse |
|---|---|---|
| TCP | DNS, rute, lytter, SYN/ACK | firewall, feil port, proxy/bastion |
| Versjon | klient `-vvv`, serverlogg | inkompatibel eller fremmed implementering |
| KEX | forhandlede algoritmer | ingen felles KEX/Cipher/MAC, Strict-KEX-problem |
| Vertsnøkkel | fingeravtrykk, principal, CA/known_hosts | feil vert, rotasjon, DNS-/LB-mål |
| Userauth | tilbudte metoder, fingeravtrykk/sertifikat, PAM | nøkkel ikke autorisert, principal, ACL, MFA |
| Connection | kanalforespørsel, PTY, subsystem, videresending | ForceCommand, PermitOpen, SFTP/Chroot, policy |
| Ekstern prosess | exitstatus, stderr, miljø, rettigheter | shellbane, sudo, locale, applikasjon |

En `Permission denied (publickey)` ligger etter vellykket transport og vertstillit. En timeout før banneret er ikke et nøkkelproblem. Faseorientert diagnostikk hindrer at legitimasjon eller algoritmelister endres uten bevis.

## Teknisk historie

Tatu Ylönen utviklet den opprinnelige SSH i 1995 som erstatning for ukrypterte verktøy for fjerntilgang og filoverføring. SSH-1 og SSH-2 var grunnleggende forskjellige; IETF SECSH Working Group standardiserte senere SSH-2-arkitekturen i RFC 4251 til 4254 ([OpenSSH – History](https://www.openssh.com/history.html), [RFC 4251](https://www.rfc-editor.org/rfc/rfc4251.html)).

OpenSSH oppstod i OpenBSD-prosjektet fra fri SSH-kode og ble portert til andre operativsystemer. Det samlet klient, server, nøkkelverktøy, agent, SCP og SFTP og preget mange de facto-utvidelser som eget sertifikatformat ([OpenSSH Project](https://www.openssh.com/)).

Kryptografien utviklet seg fra eldre kombinasjoner av Diffie-Hellman, DSA, RSA/SHA-1, CBC og separate MAC-er til Curve25519/ECDH, Ed25519, RSA/SHA-2 og AEAD-metoder. Samtidig gikk driftstilliten fra manuelt distribuerte enkeltnøkler til maskinvareautentikatorer, Host-/User-CA-er, kortlivede sertifikater og identitetsbaserte bastioner. Protokollagene forble gjenkjennelige; tillits- og livssyklusdrift ble mer omfattende.

## Admin-sjekkliste på ett sted

Sjekklisten oppsummerer forbindelsesvei, vertstillit, brukerrettigheter, nøkkeldrift og logging. Den egner seg for godkjenning og regelmessig kontroll av SSH-tilgang.

| Spørsmål | Driftsbevis |
|---|---|
| Hvilken vei? | klient/kontekst, DNS, kilde-IP, bastion/ProxyJump, mål-IP, port, lytterprosess |
| Hvilken serveridentitet? | verts­nøkkelfingeravtrykk eller vertssertifikat, principal, CA, gyldighet, rotasjon |
| Hvilke algoritmer? | forhandlede KEX-, verts­nøkkel-, Cipher-/AEAD- og MAC-metoder, eldre unntak |
| Hvem autentiserer seg? | konto, principal, offentlig nøkkelfingeravtrykk/sertifikatserie, MFA-/PAM-vei, kildeenhet |
| Hva er autorisert? | grupper, Match-blokk, Authorized-Key-alternativer, sudo/token, subsystem/kommando |
| Hvilke kanaler? | Shell/Exec/SFTP, PTY, Local/Remote/Dynamic Forward, Agent/X11, PermitOpen |
| Hvilken policy gjelder faktisk? | `sshd -T -C`, `ssh -G`, Includes, distribusjonsstandarder og Match-kontekst |
| Hva logges? | Connect/Auth/Session/Channel, privilege escalation, overføring og sentral korrelasjon |
| Hvordan roteres det? | parallell verts­nøkkel/CA-tillit, korte brukersertifikater, inventar og tilbakekalling |
| Hvordan gjenopprettes det? | OOB-konsoll, Host-/User-CA, policy, principals/nøkler, DNS/bastion, Break Glass og revisjon |

## Kilder

- [RFC 4251 – SSH-protokollarkitektur](https://www.rfc-editor.org/rfc/rfc4251.html)
- [IANA-registeret for tjenestenavn og portnumre](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)
- [RFC 4253 – SSH Transport Layer Protocol](https://www.rfc-editor.org/rfc/rfc4253.html)
- [RFC 9142 – Key Exchange Method Recommendations for SSH](https://www.rfc-editor.org/rfc/rfc9142.html)
- [RFC 8308 – Extension Negotiation in SSH](https://www.rfc-editor.org/rfc/rfc8308.html)
- [RFC 9519 – Strict Key Exchange in SSH](https://www.rfc-editor.org/rfc/rfc9519.html)
- [ssh(1)](https://man.openbsd.org/ssh)
- [sshd(8)](https://man.openbsd.org/sshd)
- [RFC 4255 – SSHFP Resource Records](https://www.rfc-editor.org/rfc/rfc4255.html)
- [IETF-utkast – OpenSSH Certificate Format](https://datatracker.ietf.org/doc/html/draft-ietf-sshm-cert)
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
- [OpenSSH FIDO/U2F-støtte](https://www.openssh.com/txt/release-8.2)
- [ssh-add(1)](https://man.openbsd.org/ssh-add)
- [Microsoft Learn – Get-Service](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-service)
- [GNU grep-manual](https://www.gnu.org/software/grep/manual/grep.html)
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
- [Microsoft Learn – OpenSSH-oversikt](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-overview)
- [Microsoft Learn – OpenSSH Server-konfigurasjon](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)
- [OpenSSH – History](https://www.openssh.com/history.html)
- [OpenSSH Project](https://www.openssh.com/)
