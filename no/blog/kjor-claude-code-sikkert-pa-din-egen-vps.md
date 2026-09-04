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
translationId: article-f932e9e537d7704a
translationReview: automatic
translationSourceHash: 011d5e16cec877d14e68e11ff48caee9b6ee849ee6235c889676cfe64ae81628
translatedAt: 2026-09-04T08:48:30.739Z
url: https://rafaelpfister.ch/no/blog/kjor-claude-code-sikkert-pa-din-egen-vps
translationModel: gpt-5.6-terra
---

På din egen datamaskin avsluttes en Claude Code-økt ufrivillig senest når den bærbare datamaskinen går i dvale eller nettverksforbindelsen brytes. En VPS fortsetter å kjøre og er tilgjengelig fra flere enheter. Samtidig er den permanent koblet til det offentlige internettet og blir automatisk skannet kort tid etter oppstart.

Denne veiledningen kombinerer begge kravene: Claude Code forblir tilgjengelig i en `tmux`-økt, mens Debian-serveren kun tilbyr en nøkkelbeskyttet SSH-forbindelse utad. Herdingen er ikke spesifikk for Claude og egner seg også for andre offentlig tilgjengelige Linux-servere.

## Hvorfor en VPS kan være fornuftig

Sammenlignet med en rent lokal installasjon gir serveren tre praktiske fordeler:

- **Vedvarende drift.** I en `tmux`-økt fortsetter Claude å kjøre selv om SSH-forbindelsen brytes. En oppgave som tar ti minutter eller en time, fullføres uten at den bærbare datamaskinen må stå åpen.
- **Tilgjengelighet.** Den samme økten er tilgjengelig fra skrivebordsmaskinen, den bærbare datamaskinen og iPhone. Du starter en oppgave ved skrivebordet og sjekker resultatet mens du er på farten.
- **Datakontroll.** Du bestemmer selv hva som ligger på serveren. Ingen synkroniseringstjeneste, ingen utilsiktet sikkerhetskopiering av påloggingsopplysninger, forutsatt at du går nøye frem ved migreringen (se nedenfor).

`tmux` er kun en funksjon for tilgjengelighet og komfort, ikke et sikkerhetstiltak. Det egentlige arbeidet ligger i sikringen.

## Utgangspunkt

Grunnlaget er Debian 13 (Trixie), minimalt installert, uten skrivebordsmiljø og uten ekstra nettverkstjenester. Leverandøren tilbyr en foranstilt brannmur som virker uavhengig av operativsystemet. Målet er en server der kun SSH er tilgjengelig utenfra, og også det bare med passfrasebeskyttede nøkler.

## 1. Oppdater systemet

Oppdater alle pakker umiddelbart etter installasjonen:

```bash
sudo apt update
sudo apt full-upgrade
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `update` | Leser inn pakkelistene fra alle konfigurerte kilder på nytt |
| `full-upgrade` | Oppdaterer alle pakker og kan også installere nye pakker eller fjerne eksisterende pakker |

</details>

I motsetning til `upgrade` løser `full-upgrade` også avhengigheter som krever nye eller fjernede pakker. På et nytt system er dette riktig fremgangsmåte for å installere alle tilgjengelige sikkerhetsoppdateringer. Start på nytt én gang etter kjerneoppdateringer.

## 2. Egen bruker i stedet for root

Å arbeide som root er unødvendig risikabelt: Hver skrivefeil påvirker hele systemet, og direkte root-pålogging er det første automatiserte angrep forsøker. Opprett derfor en egen bruker (her `claude`) med sudo-rettigheter for tilfellene der de trengs:

```bash
sudo adduser claude
sudo usermod -aG sudo claude
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-a` | Legger til: supplerer brukerens gruppeliste i stedet for å erstatte den; bare gyldig sammen med `-G` |
| `-G sudo` | Tilleggsgruppe(r) brukeren skal tas opp i |
| `claude` | Den aktuelle brukeren; ved `adduser` navnet på kontoen som skal opprettes |

</details>

Fra nå av utføres all administrasjon via `claude` og `sudo`, ikke lenger via direkte root-tilgang.

## 3. Ed25519-nøkler med passfrase, én per enhet

Påloggingen skal utelukkende skje med SSH-nøkler, ikke passord. Ed25519 er den gjeldende standarden: kort, rask og kryptografisk solid. Det avgjørende er at nøkkelen opprettes på klienten, altså på PC-en, ikke på serveren, og beskyttes med en passfrase. Passfrasen er den andre forsvarslinjen dersom den private nøkkelen noen gang havner i feil hender.

På PC-en:

```bash
ssh-keygen -t ed25519 -C "pc-thinkpad"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-t ed25519` | Nøkkeltype, her den elliptiske metoden Ed25519 |
| `-C "pc-thinkpad"` | Kommentar som legges til den offentlige nøkkelen |

</details>

Kommentaren (`-C`) navngir enheten. Det lønner seg senere: Det opprettes en egen nøkkel for hver enhet: én for PC-en, én separat for iPhone. Hvis en enhet mistes, fjerner du målrettet dens offentlige nøkkel fra `~/.ssh/authorized_keys`, uten å måtte rulle ut alle andre tilganger på nytt.

Kun den offentlige nøkkelen hører hjemme på serveren. Den private nøkkelen forlater aldri enheten. I `authorized_keys` finnes det til slutt kun offentlige nøkler, hver med sin enhetskommentar:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...pc  pc-thinkpad
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...ios iphone-15
```

Overfør den offentlige PC-nøkkelen innledningsvis. Så lenge passordpålogging fortsatt er aktiv, gjøres dette enklest med:

```bash
ssh-copy-id claude@SERVER
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `claude@SERVER` | Bruker og målvert; den offentlige standardnøkkelen legges der til i `~/.ssh/authorized_keys` |

</details>

Test deretter at pålogging med nøkkel fungerer før passordpålogging deaktiveres i neste trinn. Filrettighetene må være riktige, ellers ignorerer sshd filen: `~/.ssh` til `700`, `authorized_keys` til `600`.

## 4. Herd SSH: ingen root, ingen passord

Serverkonfigurasjonen ligger i `/etc/ssh/sshd_config` og (i Debian 13) i drop-in-filer under `/etc/ssh/sshd_config.d/`. Endringer hører hjemme i en egen drop-in-fil; dermed forblir hovedfilen urørt og pakkeoppdateringer overskriver ingenting. Opprett filen `/etc/ssh/sshd_config.d/99-haertung.conf`:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

Dette deaktiverer direkte root-pålogging og passordpålogging. Fra nå av slipper kun de inn som har en passende privat nøkkel. Kontroller syntaksen i konfigurasjonen før den lastes på nytt:

```bash
sudo sshd -t
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-t` | Testmodus: kontrollerer konfigurasjonsfilen og nøklene for gyldighet uten å starte tjenesten |

</details>

Hvis `sshd -t` ikke rapporterer noe, er filen gyldig. Last den først da på nytt:

```bash
sudo systemctl reload ssh
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `reload` | Ber tjenesten laste inn konfigurasjonen på nytt uten å bryte eksisterende forbindelser |
| `ssh` | Målenheten, her OpenSSH-tjenesten |

</details>

**Viktig:** La den eksisterende SSH-økten være åpen og test den nye tilgangen i en annen terminal. Først når nøkkelinnloggingen der beviselig fungerer, kan den gamle økten lukkes. Dette forholdsreglene reduserer risikoen for å bli utestengt til praktisk talt null. En feil i konfigurasjonen kan ellers koste deg all tilgang.

## 5. Legg SSH på en uvanlig port

Standardporten 22 blir forsøkt av roboter døgnet rundt. Å bytte til en høy, fritt valgt port (i eksempelet `61417`) lar størstedelen av denne automatiserte støyen bomme. Dette er uttrykkelig ingen sikkerhetsgevinst i egentlig forstand: Et portbytte erstatter ikke sterk autentisering, det reduserer bare loggmengden og skannebelastningen. Nøkkelkravet fra trinn 4 forblir den egentlige sikringen.

Hvilken port det blir, er ikke vilkårlig. IANA skiller mellom tre soner: **0–1023 (well-known ports)** er forbeholdt standardtjenester (SSH selv på 22, HTTP på 80, HTTPS på 443), krever root for binding og hører ikke hjemme på en selvvalgt SSH-port. Skannere forventer disse portene, det samme gjør standardtjenester som installeres senere. **1024–49151 (registrerte porter)** er tildelt enkeltapplikasjoner på forespørsel, for eksempel 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis) eller 8080/8443 som utbredte HTTP-alternativer; en tilfeldig valgt port i dette området kolliderer lett senere med programvare som forventer nettopp sin registrerte port. **49152–65535 (dynamiske/private porter)** er ifølge IANA ikke tildelt noen tjeneste og er beregnet på midlertidige, private formål, altså riktig område for en permanent, selvvalgt port.

Det gjenstår et forbehold: Mange Linux-systemer, også Debian, bruker deler av det samme området som kildeport for egne utgående forbindelser (`net.ipv4.ip_local_port_range`, som standard rundt 32768–60999). En tjeneste som lytter permanent, kolliderer ikke egentlig med dette, siden kjernen ikke tildeler en allerede bundet port, men en port over 60999 unngår også denne teoretiske uklarheten. Eksempelet i denne artikkelen (`61417`) ligger derfor bevisst der. Før omleggingen bør du i tillegg kontrollere med `ss -lntup` (se trinn 7) at den valgte porten ikke allerede er opptatt på din egen server.

I Debian 13 finnes det en særhet: SSH kan startes via systemd-socket-aktivering. Hvis dette er tilfellet, ignoreres `Port`-angivelsen i `sshd_config` helt enkelt; porten må da angis på socketen. Kontroller først hvilket tilfelle som gjelder:

```bash
systemctl is-enabled ssh.socket
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `is-enabled` | Viser om enheten er aktivert for systemoppstart |
| `ssh.socket` | Socket-enheten for SSH-tjenesten |

</details>

Hvis kommandoen svarer med `enabled`, kjører SSH via socketen. Endre da porten der:

```bash
sudo systemctl edit ssh.socket
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `edit` | Oppretter en drop-in-overstyringsfil for enheten og åpner den i redigeringsprogrammet |
| `ssh.socket` | Socket-enheten som skal overstyres |

</details>

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

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `daemon-reload` | Leser inn alle enhetsfiler på nytt, inkludert den nettopp opprettede overstyringsfilen |
| `restart ssh.socket` | Starter socket-enheten på nytt slik at den lytter på den nye porten |

</details>

Hvis socket-aktivering ikke er aktiv (`disabled`), skal i stedet `Port 61417` legges i drop-in-filen fra trinn 4, etterfulgt av `sudo sshd -t` og `sudo systemctl restart ssh`.

Også her gjelder: Åpne først den nye porten i brannmuren (neste trinn), koble deretter til og test, og la den gamle økten være åpen til tilgang via den nye porten er bekreftet.

## 6. Brannmur: stengt som standard

Den foranstilte brannmuren hos leverandøren er den mest effektive grensen fordi den fanger opp pakker før de i det hele tatt når operativsystemet. To grunnregler:

- **Standardhandling for innkommende trafikk er DROP.** Alt som ikke uttrykkelig er tillatt, forkastes uten kommentar og uten tilbakemelding til avsenderen.
- **Ett eneste unntak:** innkommende TCP på målporten `61417`. Ikke noe mer trenger å være tilgjengelig utenfra.

Utgående trafikk forblir tillatt. Dette er bevisst: Serveren må kunne laste ned pakker, synkronisere tid og nå API-et for Claude Code. Restriktiv filtrering av utgående trafikk gir lite ekstra beskyttelse på en enkeltserver, men gjør driften merkbart mer tungvint.

De som ønsker ekstra dybdeforsvar, kan duplisere de samme reglene på verten med `nftables` eller `ufw`. For oppsettet som beskrives, er leverandørens brannmur tilstrekkelig.

## 7. Kontroller angrepsflaten

Etter herdingen kontrollerer du hva serveren faktisk tilbyr utad. To kommandoer er nok. Først: Hvilke tjenester lytter på hvilke adresser?

```bash
sudo ss -lntup
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-l` | Vis kun lyttende sockets |
| `-n` | Numerisk utdata: porter og adresser løses ikke opp til navn |
| `-t` | Inkluder TCP-sockets |
| `-u` | Inkluder UDP-sockets |
| `-p` | Viser prosessen bak hver socket; dette krever `sudo` |

</details>

Adressekolonnen er avgjørende: En tjeneste på `0.0.0.0` eller `[::]` er tilgjengelig utenfra, mens en på `127.0.0.1` eller `[::1]` kun er lokal. I sikret tilstand skal kun SSH vises offentlig. Tjenester som `chronyd` (tidssynkronisering) kan vises, men bare bundet til lokale adresser. Hvis `chronyd` kun lytter på `127.0.0.1` og `::1`, kan den ikke nås utenfra og er dermed uproblematisk.

For det andre: Finnes det mislykkede systemtjenester som tyder på et konfigurasjonsproblem?

```bash
systemctl --failed
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `--failed` | Lister utelukkende enheter i feiltilstand |

</details>

Svaret bør være `0 loaded units listed`, ikke én eneste mislykket tjeneste. Feilende enheter er ikke bare et driftsproblem, men potensielt også et sikkerhetsproblem hvis det ligger en halvstartet, feilkonfigurert nettverkstjeneste bak.

## 8. Installer og drift Claude Code

Claude Code trenger et aktuelt Node.js-kjøremiljø. Etter installasjonen av dette setter du opp CLI-en i henhold til den offisielle veiledningen og autentiserer deg på nytt på serveren; ikke last opp lokale påloggingsopplysninger (mer om dette straks).

For varig drift med `tmux`:

```bash
tmux new -s claude
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `new` | Oppretter en ny økt |
| `-s claude` | Angir øktnavnet den senere kan gjenopptas med |

</details>

Start Claude inne i økten. Med `Ctrl-b`, deretter `d`, kobler du fra økten uten å avslutte den; Claude fortsetter å kjøre. Du kommer tilbake med:

```bash
tmux attach -t claude
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `attach` | Kobler terminalen til en kjørende økt igjen |
| `-t claude` | Velger måløkten ut fra navnet |

</details>

Slik overlever en aktiv oppgave brutte forbindelser, enhetsbytter og den bærbare datamaskinens nattero.

## 9. Datahygiene ved migrering

Den mest følsomme delen ved å flytte til serveren er ikke teknologien, men spørsmålet om hva du tar med deg. Tre regler:

- **Ingen private nøkler på serveren.** I `authorized_keys` ligger det kun offentlige nøkler. Private nøkler blir værende på endeenhetene.
- **Ikke kopier påloggingsopplysninger ukritisk.** Sensitive lokale filer som en `.credentials.json` hører ikke hjemme på VPS-en uten kontroll. Autentiser deg i stedet på nytt på serveren.
- **Legg først konfigurasjon i en migreringsmappe.** Ikke skriv eksisterende Claude-minner og -konfigurasjon direkte inn i de aktive konfigurasjonsbanene, men overfør dem først til en separat migreringsmappe og vurder der hva som faktisk skal tas med. Det du ikke lenger trenger, som gamle MCP-oppføringer eller foreldreløse innstillinger, blir bevisst igjen i stedet for å følge med ukritisk.

## 10. Forhåndsvisninger på nett via en SSH-tunnel

For forhåndsvisninger på nett, for eksempel en lokal utviklingsserver som Claude starter, er fristelsen stor til bare å åpne en ekstra port. Det bør du ikke gjøre. Hver ekstra åpne port er ekstra angrepsflate. I stedet kjører forhåndsvisningen gjennom en kryptert SSH-porttunnel: Tjenesten lytter kun lokalt på serveren, og SSH videresender den til klienten.

Gjør en tjeneste som kjører lokalt på port 4321 tilgjengelig fra PC-en:

```bash
ssh -p 61417 -L 4321:localhost:4321 claude@SERVER
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-p 61417` | Porten SSH-serveren lytter på (den som ble valgt i trinn 5) |
| `-L 4321:localhost:4321` | Lokal portvideresending: Forbindelser til lokal port 4321 videresendes gjennom tunnelen til `localhost:4321` sett fra serveren |
| `claude@SERVER` | Bruker og målvert for SSH-forbindelsen |

</details>

Deretter åpner du `http://localhost:4321` i den lokale nettleseren. Trafikken går utelukkende gjennom den eksisterende, autentiserte SSH-forbindelsen, uten at én eneste ekstra port må åpnes i brannmuren.

## Tilgang fra iPhone

Tilgang på farten fungerer med samme sikkerhetsmodell som fra PC-en. Du trenger bare en SSH-klient med nøkkeladministrasjon. **Termius**, **Blink Shell** og **Secure ShellFish** er utbredte; alle kan opprette Ed25519-nøkler og lagre dem i iOS-nøkkelringen, delvis sikret med Face ID.

Fremgangsmåten tilsvarer trinn 3, bare på iPhone:

1. Opprett en egen Ed25519-nøkkel for iPhone i SSH-klienten, ikke kopier PC-nøkkelen. Den private nøkkelen blir værende i enhetens nøkkelring.
2. Legg iPhones offentlige nøkkel inn som en ekstra linje i `~/.ssh/authorized_keys` på serveren, med en beskrivende kommentar (`iphone-15`).
3. Opprett forbindelsen i klienten: serveradresse, bruker `claude`, port `61417`, og iPhone-nøkkelen for autentisering.

Det er nettopp derfor en separat nøkkel per enhet lønner seg: Hvis iPhone mistes, sletter du den ene `iphone-15`-linjen fra `authorized_keys` på serveren, og enheten er utestengt, mens PC-tilgang og alle andre nøkler fortsetter uendret.

Etter tilkobling henter du tilbake den kjørende Claude-økten med `tmux attach -t claude` og fortsetter der du slapp ved skrivebordet. Porttunnelen fra trinn 10 fungerer også fra iOS; Termius og Secure ShellFish støtter portvideresending.

## Sjekkliste

Hele prosessen oppsummert:

1. Debian 13 installert og fullstendig oppdatert med `apt full-upgrade`.
2. Egen bruker `claude` med sudo-rettigheter; direkte root-pålogging brukes ikke lenger.
3. Passfrasebeskyttede Ed25519-nøkler, én per enhet, kun offentlige nøkler i `authorized_keys`.
4. sshd herdet: `PermitRootLogin no`, `PasswordAuthentication no`; kontrollert med `sshd -t` før ny innlasting, og eksisterende økt holdt åpen til testen var gjennomført.
5. SSH på port 61417, ved socket-aktivering angitt på `ssh.socket`, ellers i sshd-konfigurasjonen.
6. Leverandørbrannmur: innkommende standard DROP, eneste unntak TCP 61417; utgående tillatt.
7. Angrepsflate kontrollert med `ss -lntup` (kun SSH offentlig, `chronyd` lokalt) og `systemctl --failed` (ingen feil).
8. Claude Code autentisert på nytt på serveren, drift i en `tmux`-økt.
9. Datahygiene: ingen private nøkler eller påloggingsopplysninger på serveren, konfigurasjon først kontrollert via en migreringsmappe.
10. Ingen ekstra porter; forhåndsvisninger på nett kjører gjennom en SSH-tunnel.

Etter dette oppsettet er kun SSH på den angitte porten tilgjengelig utenfra, og også der utelukkende med en passfrasebeskyttet nøkkel. Claude Code kjører uavhengig av endeenheten i `tmux`; forhåndsvisninger på nett forblir tilgjengelige via SSH-tunneler uten å åpne en ekstra port.

## Kilder

1.  [OpenSSH Manual – sshd_config(5)](https://man.openbsd.org/sshd_config): Referanse for alle sshd-direktiver, blant annet `PermitRootLogin`, `PasswordAuthentication` og `PubkeyAuthentication`.

2.  [Debian Wiki – SSH](https://wiki.debian.org/SSH): Debian-spesifikke merknader om SSH-konfigurasjon, inkludert drop-in-filene under `/etc/ssh/sshd_config.d/`.

3.  [systemd.socket(5) – freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html): Hvordan socket-aktivering og `ListenStream=`-direktivet fungerer, relevant for SSH-portbyttet i Debian 13.

4.  [ss(8) – iproute2 Manpage](https://man7.org/linux/man-pages/man8/ss.8.html): Alternativer for `ss` for å liste lyttende sockets med prosess og bindingsadresse.

5.  [Claude Code – Offisiell dokumentasjon](https://docs.claude.com/en/docs/claude-code/overview): Installasjon, autentisering og drift av Claude Code.
