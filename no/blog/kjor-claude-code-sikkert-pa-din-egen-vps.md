---
title: "Kjør Claude Code sikkert på din egen VPS"
navTitle: "VPS for Claude"
description: "En herdet Debian-VPS holder Claude Code-økter permanent tilgjengelige. Veiledningen dekker alt fra brukerkonto og SSH-nøkler til brannmur, datahygiene, tmux og sikker tilgang fra iPhone."
date: "2026-07-21"
kategorie: "Claude"
timeToRead: "12 min lesetid"
themen:
  - claude
slug: "kjor-claude-code-sikkert-pa-din-egen-vps"
translationOf: "claude-code-vps-debian-absichern"
url: "https://rafaelpfister.ch/no/blog/kjor-claude-code-sikkert-pa-din-egen-vps"
translationId: article-f932e9e537d7704a
translationReview: automatic
translationSourceHash: bd2aac7348c16dbd326ab0c10a063817d88a05cb99ab88a8cde66b885dfd7c3f
translatedAt: 2026-07-29T12:29:38.969Z
---

På din egen datamaskin avsluttes en Claude Code-økt senest ufrivillig når laptopen går i dvale eller nettverkstilkoblingen brytes. En VPS fortsetter å kjøre og er tilgjengelig fra flere enheter. Samtidig er den permanent koblet til det offentlige internettet og blir automatisk skannet kort tid etter oppstart.

Denne veiledningen kombinerer begge kravene: Claude Code forblir tilgjengelig i en `tmux`-økt, mens Debian-serveren kun tilbyr en nøkkelbeskyttet SSH-forbindelse utad. Herdingen er ikke spesifikk for Claude og egner seg også for andre offentlig tilgjengelige Linux-servere.

## Hvorfor en VPS kan være nyttig

Sammenlignet med en rent lokal installasjon gir serveren tre praktiske fordeler:

- **Persistens.** I en `tmux`-økt fortsetter Claude å kjøre selv om SSH-forbindelsen kobles fra. En oppgave som tar ti minutter eller en time, fullføres uten at laptopen må stå åpen.
- **Tilgjengelighet.** Den samme økten er tilgjengelig fra skrivebordsmaskinen, laptopen og iPhone. Du starter en oppgave ved skrivebordet og sjekker resultatet mens du er på farten.
- **Datakontroll.** Du bestemmer selv hva som ligger på serveren. Ingen synkroniseringstjeneste, ingen tilgangsdata som utilsiktet sikkerhetskopieres, forutsatt at du går nøye frem under migreringen (se nedenfor).

`tmux` er kun en funksjon for tilgjengelighet og komfort, ikke et sikkerhetstiltak. Det egentlige arbeidet ligger i sikringen.

## Utgangspunkt

Grunnlaget er Debian 13 (Trixie), installert minimalt, uten skrivebordsmiljø og uten ekstra nettverkstjenester. Leverandøren tilbyr en brannmur foran serveren som virker uavhengig av operativsystemet. Målet er en server der kun SSH er tilgjengelig utenfra, og også det bare med passfrasebeskyttede nøkler.

## 1. Oppdater systemet

Oppdater hele pakkesettet umiddelbart etter installasjonen:

```bash
sudo apt update
sudo apt full-upgrade
```

`full-upgrade` løser, i motsetning til `upgrade`, også avhengigheter som krever nye eller fjernede pakker. På et ferskt system er dette riktig måte å installere alle tilgjengelige sikkerhetsoppdateringer på. Start systemet på nytt én gang etter kjerneoppdateringer.

## 2. Egen bruker i stedet for root

Å arbeide som root er unødvendig risikabelt: Hver skrivefeil påvirker hele systemet, og direkte root-pålogging er det automatiserte angrep prøver først. Opprett derfor en egen bruker (her `claude`) med sudo-rettigheter for tilfellene der det trengs:

```bash
sudo adduser claude
sudo usermod -aG sudo claude
```

Fra nå av skjer all administrasjon via `claude` og `sudo`, ikke lenger via direkte root-tilgang.

## 3. Ed25519-nøkler med passfrase, én per enhet

Påloggingen skal kun skje via SSH-nøkler, ikke passord. Ed25519 er den nåværende standarden: kort, rask og kryptografisk robust. Det avgjørende er at nøkkelen opprettes på klienten, altså på PC-en, ikke på serveren, og beskyttes med en passfrase. Passfrasen er den andre forsvarslinjen dersom den private nøkkelen noen gang havner i feil hender.

På PC-en:

```bash
ssh-keygen -t ed25519 -C "pc-thinkpad"
```

Kommentaren (`-C`) navngir enheten. Det lønner seg senere: Opprett en egen nøkkel for hver enhet: én for PC-en, en separat for iPhone. Hvis en enhet mistes, fjerner du målrettet dens offentlige nøkkel fra `~/.ssh/authorized_keys`, uten å måtte distribuere alle andre tilganger på nytt.

Bare den offentlige nøkkelen hører hjemme på serveren. Den private nøkkelen forlater aldri enheten. I `authorized_keys` finnes det til slutt utelukkende offentlige nøkler, hver med sin enhetskommentar:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...pc  pc-thinkpad
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...ios iphone-15
```

Overfør først den offentlige PC-nøkkelen. Så lenge passordpålogging fortsatt er aktiv, gjøres dette enklest med:

```bash
ssh-copy-id claude@SERVER
```

Test deretter at pålogging med nøkkel fungerer før passordpålogging deaktiveres i neste trinn. Filrettighetene må være riktige, ellers ignorerer sshd filen: `~/.ssh` på `700`, `authorized_keys` på `600`.

## 4. Herd SSH: ingen root, intet passord

Serverkonfigurasjonen ligger i `/etc/ssh/sshd_config` og (på Debian 13) i drop-in-filer under `/etc/ssh/sshd_config.d/`. Endringer hører hjemme i en egen drop-in-fil; slik forblir hovedfilen urørt, og pakkeoppdateringer overskriver ingenting. Opprett filen `/etc/ssh/sshd_config.d/99-haertung.conf`:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

Dette deaktiverer direkte root-pålogging og passordpålogging. Fra nå av slipper bare den inn som har en passende privat nøkkel. Kontroller konfigurasjonen syntaktisk før den lastes inn på nytt:

```bash
sudo sshd -t
```

Hvis `sshd -t` ikke melder noe, er filen gyldig. Last den først da inn på nytt:

```bash
sudo systemctl reload ssh
```

**Viktig:** La den eksisterende SSH-økten stå åpen, og test den nye tilgangen i en annen terminal. Først når pålogging med nøkkel der fungerer dokumenterbart, kan den gamle økten lukkes. Dette forsiktighetstiltaket reduserer risikoen for å låse seg ute til praktisk talt null. En feil i konfigurasjonen kan ellers koste deg all tilgang.

## 5. Flytt SSH til en uvanlig port

Standardporten 22 blir prøvd av boter døgnet rundt. Å bytte til en høy, fritt valgt port (i eksempelet `61417`) gjør at mesteparten av denne automatiserte støyen ikke treffer noe. Dette er uttrykkelig ikke en sikkerhetsgevinst i egentlig forstand: Et portbytte erstatter ikke sterk autentisering, det reduserer bare loggmengden og skannebelastningen. Nøkkelkravet fra trinn 4 er fortsatt den egentlige sikringen.

Hvilken port du velger, er ikke vilkårlig. IANA skiller mellom tre soner: **0–1023 (well-known ports)** er reservert for standardtjenester (SSH selv på 22, HTTP på 80, HTTPS på 443), krever root for å binde og hører ikke hjemme på en selvvalgt SSH-port; skannere forventer disse portene, det samme gjør standardtjenester som installeres senere. **1024–49151 (registrerte porter)** er tildelt enkeltapplikasjoner på forespørsel, for eksempel 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis) eller 8080/8443 som utbredte HTTP-alternativer; en tilfeldig valgt port fra dette området kolliderer lett senere med programvare som forventer akkurat sin registrerte port. **49152–65535 (dynamiske/private porter)** er ifølge IANA ikke tildelt noen tjeneste og er ment for midlertidige, private formål, altså riktig område for en fast, selvvalgt port.

Det finnes likevel et forbehold: Mange Linux-systemer, også Debian, bruker en del av det samme området som kildeport for egne utgående forbindelser (`net.ipv4.ip_local_port_range`, som standard rundt 32768–60999). En fast lyttende tjeneste kolliderer derfor ikke egentlig, kjernen tildeler ikke en port som allerede er bundet, men en port over 60999 unngår også denne teoretiske uklarheten. Eksempelet i denne artikkelen (`61417`) ligger derfor bevisst der. Før omleggingen bør du i tillegg kontrollere med `ss -lntup` (se trinn 7) at den valgte porten ikke allerede er opptatt på din server.

I Debian 13 finnes det en fallgruve her: SSH kan startes via systemd-socket-aktivering. Hvis det er tilfellet, ignoreres `Port`-angivelsen i `sshd_config` ganske enkelt; porten må da settes på socketen. Kontroller først hvilket tilfelle som gjelder:

```bash
systemctl is-enabled ssh.socket
```

Hvis kommandoen svarer med `enabled`, kjører SSH via socketen. Endre da porten der:

```bash
sudo systemctl edit ssh.socket
```

Skriv inn følgende linjer i redigeringsprogrammet. Den første, tomme `ListenStream=`-linjen sletter den forhåndsinnstilte porten 22, den andre angir den nye:

```text
[Socket]
ListenStream=
ListenStream=61417
```

Bruk deretter endringene:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

Hvis socket-aktivering ikke er aktiv (`disabled`), skal `Port 61417` i stedet inn i drop-in-filen fra trinn 4, etterfulgt av `sudo sshd -t` og `sudo systemctl restart ssh`.

Også her gjelder: Åpne først den nye porten i brannmuren (neste trinn), koble deretter til og test, og la den gamle økten stå åpen til tilgangen via den nye porten er bekreftet.

## 6. Brannmur: stengt som standard

Brannmuren hos leverandøren er den mest effektive grensen, fordi den fanger opp pakker før de i det hele tatt når operativsystemet. To grunnregler:

- **Standardhandling for innkommende trafikk er DROP.** Alt som ikke uttrykkelig er tillatt, forkastes, uten kommentar og uten tilbakemelding til avsenderen.
- **Ett eneste unntak:** innkommende TCP til målport `61417`. Ingenting mer trenger å være tilgjengelig utenfra.

Utgående trafikk forblir tillatt. Det er bevisst: Serveren må kunne laste ned pakker, synkronisere tiden og nå API-et for Claude Code. Restriktiv filtrering av utgående trafikk gir lite ekstra beskyttelse på en enkeltserver, men gjør driften merkbart mer tungvint.

Hvis du ønsker ytterligere lagdelt beskyttelse, kan du duplisere de samme reglene på verten med `nftables` eller `ufw`. For oppsettet som beskrives her, er leverandørens brannmur tilstrekkelig.

## 7. Kontroller angrepsflaten

Etter herdingen bør du kontrollere hva serveren faktisk tilbyr utad. To kommandoer er nok. Først: Hvilke tjenester lytter på hvilke adresser?

```bash
sudo ss -lntup
```

`ss` viser alle lyttende TCP- og UDP-sockets sammen med tilhørende prosess (`sudo` kreves for å se prosessnavnene). Adressekolonnen er avgjørende: En tjeneste på `0.0.0.0` eller `[::]` er tilgjengelig utenfra, mens en på `127.0.0.1` eller `[::1]` kun er lokal. I sikret tilstand skal bare SSH være offentlig synlig. Tjenester som `chronyd` (tidssynkronisering) kan vises, men bare bundet til lokale adresser. Hvis `chronyd` utelukkende lytter på `127.0.0.1` og `::1`, kan den ikke nås utenfra og er dermed uproblematisk.

For det andre: Finnes det mislykkede systemtjenester som kan tyde på et konfigurasjonsproblem?

```bash
systemctl --failed
```

Svaret bør være `0 loaded units listed`, ikke én eneste mislykket tjeneste. Feilaktige units er ikke bare et driftsproblem, men potensielt også et sikkerhetsproblem dersom det ligger en halvstartet, feilkonfigurert nettverkstjeneste bak.

## 8. Installer og bruk Claude Code

Claude Code trenger et oppdatert Node.js-kjøremiljø. Etter installasjonen setter du opp CLI-en i henhold til den offisielle veiledningen og autentiserer på nytt på serveren, i stedet for å laste opp lokale tilgangsdata (mer om dette straks).

For varig drift `tmux`:

```bash
tmux new -s claude
```

Start Claude inne i økten. Med `Ctrl-b`, deretter `d`, kobler du fra økten uten å avslutte den; Claude fortsetter å kjøre. Du kommer tilbake med:

```bash
tmux attach -t claude
```

Slik overlever en pågående oppgave frakoblede forbindelser, bytte av enhet og laptopens nattesøvn.

## 9. Datahygiene ved migrering

Den mest følsomme delen ved flytting til serveren er ikke teknikken, men spørsmålet om hva du tar med deg. Tre regler:

- **Ingen private nøkler på serveren.** I `authorized_keys` ligger kun offentlige nøkler. Private nøkler forblir på sluttaenhetene.
- **Ikke kopier tilgangsdata ukritisk.** Sensitive lokale filer, som en `.credentials.json`, hører ikke hjemme på VPS-en uten kontroll. Autentiser deg heller på nytt på serveren.
- **Legg konfigurasjon først i en migreringsmappe.** Ikke skriv eksisterende Claude-minner og -konfigurasjon direkte til de aktive konfigurasjonsbanene, men overfør dem først til en separat migreringsmappe og kontroller der hva som faktisk skal tas med. Det du ikke lenger trenger, som gamle MCP-oppføringer eller foreldreløse innstillinger, blir bevisst liggende igjen i stedet for å flytte med ukritisk.

## 10. Nettforhåndsvisninger via en SSH-tunnel

For nettforhåndsvisninger, for eksempel en lokal utviklingsserver som Claude starter, er fristelsen stor til bare å åpne enda en port. Det bør du ikke gjøre. Hver ekstra åpne port er ytterligere angrepsflate. I stedet går forhåndsvisningen gjennom en kryptert SSH-porttunnel: Tjenesten lytter kun lokalt på serveren, og SSH videresender den til klienten.

Gjør en tjeneste som kjører lokalt på port 4321 tilgjengelig fra PC-en:

```bash
ssh -p 61417 -L 4321:localhost:4321 claude@SERVER
```

Deretter åpner du `http://localhost:4321` i den lokale nettleseren. Trafikken går helt gjennom den eksisterende, autentiserte SSH-forbindelsen, uten at én eneste ekstra port må åpnes i brannmuren.

## Tilgang fra iPhone

Tilgang på farten fungerer med samme sikkerhetsmodell som fra PC-en. Du trenger bare en SSH-klient med nøkkelhåndtering. **Termius**, **Blink Shell** og **Secure ShellFish** er utbredte; alle kan opprette Ed25519-nøkler og lagre dem i iOS-nøkkelringen, delvis beskyttet med Face ID.

Fremgangsmåten tilsvarer trinn 3, bare på iPhone:

1. Opprett en egen Ed25519-nøkkel for iPhone i SSH-klienten, ikke kopier PC-nøkkelen. Den private nøkkelen forblir i enhetens nøkkelring.
2. Legg inn den offentlige nøkkelen fra iPhone som en ekstra linje i `~/.ssh/authorized_keys` på serveren, med en beskrivende kommentar (`iphone-15`).
3. Opprett forbindelsen i klienten: serveradresse, bruker `claude`, port `61417`, og iPhone-nøkkelen for autentisering.

Nettopp derfor lønner den separate nøkkelen per enhet seg: Hvis iPhone mistes, sletter du den ene `iphone-15`-linjen fra `authorized_keys` på serveren, og enheten er utestengt mens PC-tilgangen og alle andre nøkler fortsetter upåvirket.

Etter tilkobling henter du tilbake den pågående Claude-økten med `tmux attach -t claude` og fortsetter der du sluttet ved skrivebordet. Porttunnelen fra trinn 10 fungerer også fra iOS; Termius og Secure ShellFish støtter portvideresending.

## Sjekkliste

Oppsummert hele prosessen:

1. Debian 13 installert og fullstendig oppdatert med `apt full-upgrade`.
2. Egen bruker `claude` med sudo-rettigheter; direkte root-pålogging brukes ikke lenger.
3. Passfrasebeskyttede Ed25519-nøkler, én per enhet, kun offentlige nøkler i `authorized_keys`.
4. sshd herdet: `PermitRootLogin no`, `PasswordAuthentication no`; kontrollert med `sshd -t` før ny innlasting, eksisterende økt holdt åpen til testen var gjennomført.
5. SSH på port 61417, ved socket-aktivering satt på `ssh.socket`, ellers i sshd-konfigurasjonen.
6. Leverandørbrannmur: innkommende standard DROP, eneste unntak TCP 61417; utgående tillatt.
7. Angrepsflaten kontrollert med `ss -lntup` (kun SSH offentlig, `chronyd` lokalt) og `systemctl --failed` (ingen feil).
8. Claude Code autentisert på nytt på serveren, drift i en `tmux`-økt.
9. Datahygiene: ingen private nøkler og ingen tilgangsdata på serveren, konfigurasjon først kontrollert via en migreringsmappe.
10. Ingen ekstra porter; nettforhåndsvisninger går gjennom en SSH-tunnel.

Etter dette oppsettet er kun SSH på den angitte porten tilgjengelig utenfra, og også der utelukkende med en passfrasebeskyttet nøkkel. Claude Code kjører uavhengig av sluttaenheten i `tmux`; nettforhåndsvisninger forblir tilgjengelige via SSH-tunneler uten å åpne en ekstra port.

## Kilder

1.  [OpenSSH Manual – sshd_config(5)](https://man.openbsd.org/sshd_config): Referanse for alle sshd-direktiver, inkludert `PermitRootLogin`, `PasswordAuthentication` og `PubkeyAuthentication`.

2.  [Debian Wiki – SSH](https://wiki.debian.org/SSH): Debian-spesifikke merknader om SSH-konfigurasjon, inkludert drop-in-filene under `/etc/ssh/sshd_config.d/`.

3.  [systemd.socket(5) – freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html): Hvordan socket-aktivering fungerer og `ListenStream=`-direktivet, relevant for bytte av SSH-port i Debian 13.

4.  [ss(8) – iproute2 Manpage](https://man7.org/linux/man-pages/man8/ss.8.html): Alternativer for `ss` for å liste lyttende sockets med prosess og bindingsadresse.

5.  [Claude Code – Offisiell dokumentasjon](https://docs.claude.com/en/docs/claude-code/overview): Installasjon, autentisering og bruk av Claude Code.
