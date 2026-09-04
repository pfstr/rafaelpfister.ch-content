---
title: "Köra Claude Code säkert på en egen VPS"
navTitle: "VPS för Claude"
description: "En härdad Debian-VPS håller Claude Code-sessioner permanent tillgängliga. Guiden täcker allt från användarkonto och SSH-nycklar till brandvägg, datahygien, tmux och säker åtkomst från iPhone."
date: "2026-07-21"
kategorie: "Claude"
timeToRead: "12 min lästid"
themen:
  - claude
slug: "kora-claude-code-sakert-pa-en-egen-vps"
translationOf: "claude-code-vps-debian-absichern"
translationId: article-f932e9e537d7704a
translationReview: automatic
translationSourceHash: 011d5e16cec877d14e68e11ff48caee9b6ee849ee6235c889676cfe64ae81628
translatedAt: 2026-09-04T08:47:19.076Z
url: https://rafaelpfister.ch/sv/blog/kora-claude-code-sakert-pa-en-egen-vps
translationModel: gpt-5.6-terra
---

På den egna datorn avslutas en Claude Code-session oavsiktligt senast när laptopen går i vila eller nätverksanslutningen bryts. En VPS fortsätter att köra och kan nås från flera enheter. Samtidigt är den ständigt ansluten till det offentliga internet och skannas automatiskt kort efter start.

Den här guiden förenar båda kraven: Claude Code förblir tillgängligt i en `tmux`-session, medan Debian-servern endast erbjuder en nyckelskyddad SSH-anslutning utifrån. Härdningen är inte specifik för Claude och passar även för andra offentligt åtkomliga Linux-servrar.

## Varför en VPS kan vara ett bra val

Jämfört med en helt lokal installation erbjuder servern tre praktiska fördelar:

- **Beständighet.** I en `tmux`-session fortsätter Claude att köra även om SSH-anslutningen bryts. En uppgift som tar tio minuter eller en timme körs klart utan att laptopen behöver vara öppen.
- **Tillgänglighet.** Samma session är åtkomlig från stationär dator, laptop och iPhone. Man startar en uppgift vid skrivbordet och kontrollerar resultatet på vägen.
- **Datakontroll.** Man bestämmer själv vad som ligger på servern. Ingen synktjänst, inga inloggningsuppgifter som av misstag säkerhetskopieras, förutsatt att migrationen görs noggrant (se nedan).

`tmux` är enbart en funktion för tillgänglighet och bekvämlighet, inte en säkerhetsåtgärd. Det egentliga arbetet ligger i skyddet.

## Utgångsläge

Grunden är Debian 13 (Trixie), minimalt installerat, utan skrivbordsmiljö och utan ytterligare nätverkstjänster. Leverantören tillhandahåller en framförliggande brandvägg som fungerar oberoende av operativsystemet. Målet är en server där endast SSH kan nås utifrån, och även det endast med lösenfras-skyddade nycklar.

## 1. Uppdatera systemet

Uppdatera hela paketbeståndet direkt efter installationen:

```bash
sudo apt update
sudo apt full-upgrade
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `update` | Läser in paketlistorna från alla konfigurerade källor på nytt |
| `full-upgrade` | Uppdaterar alla paket och får även installera nya paket eller ta bort befintliga |

</details>

Till skillnad från `upgrade` löser `full-upgrade` även beroenden som kräver nya eller borttagna paket. På ett nytt system är detta rätt sätt att verkligen installera alla tillgängliga säkerhetsuppdateringar. Starta om en gång efter kärnuppdateringar.

## 2. Egen användare i stället för root

Att arbeta som root är onödigt riskabelt: varje skrivfel påverkar hela systemet, och direkt root-inloggning är det första automatiserade angrepp försöker med. Skapa därför en egen användare (här `claude`) med sudo-rättigheter för de tillfällen då de behövs:

```bash
sudo adduser claude
sudo usermod -aG sudo claude
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-a` | Lägger till: utökar användarens grupplista i stället för att ersätta den; gäller endast tillsammans med `-G` |
| `-G sudo` | Kompletterande grupp(er) som användaren läggs till i |
| `claude` | Berörd användare; vid `adduser` namnet på kontot som ska skapas |

</details>

Från och med nu sker all administration via `claude` och `sudo`, inte längre via direkt root-åtkomst.

## 3. Ed25519-nycklar med lösenfras, en per enhet

Inloggningen ska uteslutande ske med SSH-nycklar, inte lösenord. Ed25519 är den aktuella standarden: kort, snabb och kryptografiskt robust. Avgörande är att nyckeln skapas på klienten, alltså på datorn och inte på servern, samt skyddas med en lösenfras. Lösenfrasen är den andra försvarslinjen om den privata nyckeln någon gång hamnar i fel händer.

På datorn:

```bash
ssh-keygen -t ed25519 -C "pc-thinkpad"
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-t ed25519` | Nyckeltyp, här den elliptiska metoden Ed25519 |
| `-C "pc-thinkpad"` | Kommentar som läggs till den offentliga nyckeln |

</details>

Kommentaren (`-C`) identifierar enheten. Det lönar sig senare: en separat nyckel skapas för varje enhet, en för datorn och en separat för iPhone. Om en enhet går förlorad tar man specifikt bort dess offentliga nyckel från `~/.ssh/authorized_keys`, utan att behöva distribuera alla andra åtkomster på nytt.

Endast den offentliga nyckeln hör hemma på servern. Den privata nyckeln lämnar aldrig enheten. I `authorized_keys` finns till slut uteslutande offentliga nycklar, var och en med sin enhetskommentar:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...pc  pc-thinkpad
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...ios iphone-15
```

Överför först datorns offentliga nyckel. Så länge lösenordsinloggning fortfarande är aktivt går det enklast med:

```bash
ssh-copy-id claude@SERVER
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `claude@SERVER` | Användare och målserver; den offentliga standardnyckeln läggs där till i `~/.ssh/authorized_keys` |

</details>

Testa därefter att inloggning med nyckel fungerar innan lösenordsinloggning stängs av i nästa steg. Filrättigheterna måste vara korrekta, annars ignorerar sshd filen: `~/.ssh` på `700`, `authorized_keys` på `600`.

## 4. Härda SSH: ingen root, inget lösenord

Serverkonfigurationen finns i `/etc/ssh/sshd_config` och (i Debian 13) i drop-in-filer under `/etc/ssh/sshd_config.d/`. Ändringar hör hemma i en egen drop-in-fil; då lämnas huvudfilen orörd och paketuppdateringar skriver inte över något. Skapa filen `/etc/ssh/sshd_config.d/99-haertung.conf`:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

Detta inaktiverar direkt root-inloggning och lösenordsinloggning. Från och med nu kommer endast den in som har en passande privat nyckel. Kontrollera konfigurationens syntax innan den laddas om:

```bash
sudo sshd -t
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-t` | Testläge: kontrollerar att konfigurationsfil och nycklar är giltiga utan att starta tjänsten |

</details>

Om `sshd -t` inte rapporterar något är filen giltig. Ladda först då om:

```bash
sudo systemctl reload ssh
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `reload` | Uppmanar tjänsten att ladda om sin konfiguration utan att bryta befintliga anslutningar |
| `ssh` | Målenheten, här OpenSSH-tjänsten |

</details>

**Viktigt:** Låt den befintliga SSH-sessionen vara öppen och testa den nya åtkomsten i en andra terminal. Först när inloggningen med nyckel bevisligen fungerar där får den gamla sessionen stängas. Denna försiktighetsåtgärd minskar risken att låsa ute sig till praktiskt taget noll. Ett fel i konfigurationen kan annars kosta hela åtkomsten.

## 5. Flytta SSH till en ovanlig port

Standardporten 22 testas av bottar dygnet runt. Att byta till en hög, fritt vald port (i exemplet `61417`) gör att merparten av detta automatiserade brus missar målet. Det är uttryckligen ingen säkerhetsvinst i egentlig mening: ett portbyte ersätter inte stark autentisering, det minskar bara loggmängden och skanningsbelastningen. Nyckelkravet från steg 4 är fortfarande det egentliga skyddet.

Vilken port det blir är inte godtyckligt. IANA skiljer mellan tre zoner: **0–1023 (well-known ports)** är reserverade för standardtjänster (SSH självt på 22, HTTP på 80, HTTPS på 443), kräver root för bindning och har inget att göra på en självvald SSH-port; det är just dessa portar som skannrar förväntar sig, liksom standardtjänster som installeras senare. **1024–49151 (registrerade portar)** är på begäran tilldelade enskilda program, exempelvis 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis) eller 8080/8443 som vanliga HTTP-alternativ; en slumpmässigt vald port från detta område kolliderar lätt senare med programvara som förväntar sig just sin registrerade port. **49152–65535 (dynamiska/privata portar)** är enligt IANA inte tilldelade någon tjänst och avsedda för tillfälliga, privata ändamål, vilket är rätt område för en permanent, självvald port.

En reservation kvarstår: många Linux-system, även Debian, använder en del av samma område som källport för egna utgående anslutningar (`net.ipv4.ip_local_port_range`, som standard omkring 32768–60999). En permanent lyssnande tjänst kolliderar därför inte i praktiken, eftersom kärnan inte tilldelar en redan bunden port, men en port över 60999 undviker även denna teoretiska oklarhet. Exemplet i denna artikel (`61417`) ligger därför medvetet där. Kontrollera dessutom före ändringen med `ss -lntup` (se steg 7) att den valda porten inte redan används på den egna servern.

I Debian 13 finns en särskildhet här: SSH kan startas via systemd-socketaktivering. Om så är fallet ignoreras `Port`-angivelsen i `sshd_config` helt enkelt; porten måste då anges på socketen. Kontrollera först vilket fall som gäller:

```bash
systemctl is-enabled ssh.socket
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `is-enabled` | Visar om enheten är aktiverad för systemstart |
| `ssh.socket` | SSH-tjänstens socketenhet |

</details>

Om kommandot svarar med `enabled`, körs SSH via socketen. Ändra då porten där:

```bash
sudo systemctl edit ssh.socket
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `edit` | Skapar en drop-in-override-fil för enheten och öppnar den i redigeraren |
| `ssh.socket` | Socketenheten som ska åsidosättas |

</details>

Ange följande rader i redigeraren. Den första, tomma `ListenStream=`-raden tar bort den förinställda porten 22, den andra anger den nya:

```text
[Socket]
ListenStream=
ListenStream=61417
```

Tillämpa sedan ändringen:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `daemon-reload` | Läser in alla enhetsfiler på nytt, inklusive override-filen som just skapades |
| `restart ssh.socket` | Startar om socketenheten så att den lyssnar på den nya porten |

</details>

Om socketaktivering inte är aktiv (`disabled`), ska i stället `Port 61417` läggas till i drop-in-filen från steg 4, följt av `sudo sshd -t` och `sudo systemctl restart ssh`.

Även här gäller: öppna först den nya porten i brandväggen (nästa steg), anslut och testa därefter, och låt den gamla sessionen vara öppen tills åtkomst via den nya porten har bekräftats.

## 6. Brandvägg: stängd som standard

Leverantörens framförliggande brandvägg är den effektivaste gränsen eftersom den fångar paket innan de ens når operativsystemet. Två grundregler:

- **Inkommande standardåtgärd på DROP.** Allt som inte uttryckligen är tillåtet förkastas, utan kommentar och utan återkoppling till avsändaren.
- **Ett enda undantag:** inkommande TCP på målporten `61417`. Mer behöver inte vara åtkomligt utifrån.

Utgående trafik förblir tillåten. Det är medvetet: servern måste kunna hämta paket, synkronisera tiden och nå API:et för Claude Code. Restriktiv filtrering av utgående trafik ger lite ytterligare skydd på en enskild server, men gör driften märkbart krångligare.

Den som vill ha ytterligare försvar på djupet kan dubblera samma regler på värdsidan med `nftables` eller `ufw`. För den beskrivna uppsättningen räcker leverantörens brandvägg.

## 7. Kontrollera angreppsytan

Efter härdningen kontrollerar du vad servern faktiskt erbjuder utåt. Två kommandon räcker. Först: vilka tjänster lyssnar på vilka adresser?

```bash
sudo ss -lntup
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-l` | Visar endast lyssnande socketar |
| `-n` | Numerisk utmatning: portar och adresser löses inte upp till namn |
| `-t` | Inkluderar TCP-socketar |
| `-u` | Inkluderar UDP-socketar |
| `-p` | Visar processen bakom varje socket; detta kräver `sudo` |

</details>

Avgörande är adresskolumnen: en tjänst på `0.0.0.0` eller `[::]` är åtkomlig utifrån, medan en på `127.0.0.1` eller `[::1]` endast är lokal. I säkrat läge bör endast SSH visas offentligt. Tjänster som `chronyd` (tidssynkronisering) får förekomma, men endast bundna till lokala adresser. Om `chronyd` endast lyssnar på `127.0.0.1` och `::1` kan den inte nås utifrån och är därför okritisk.

För det andra: finns det misslyckade systemtjänster som tyder på ett konfigurationsproblem?

```bash
systemctl --failed
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `--failed` | Listar endast enheter i feltillstånd |

</details>

Svaret bör vara `0 loaded units listed`, inte en enda misslyckad tjänst. Felaktiga enheter är inte bara ett driftproblem utan även potentiellt ett säkerhetsproblem om det döljer sig en halvstartad, felkonfigurerad nätverkstjänst bakom dem.

## 8. Installera och köra Claude Code

Claude Code behöver en aktuell Node.js-körmiljö. Efter installationen konfigurerar du CLI:t enligt den officiella guiden och autentiserar dig på nytt på servern, i stället för att ladda upp lokala inloggningsuppgifter (mer om det strax).

För permanent drift med `tmux`:

```bash
tmux new -s claude
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `new` | Skapar en ny session |
| `-s claude` | Tilldelar sessionsnamnet som den senare återupptas med |

</details>

Starta Claude i sessionen. Med `Ctrl-b`, sedan `d`, kopplar du loss från sessionen utan att avsluta den; Claude fortsätter att köra. Återgå med:

```bash
tmux attach -t claude
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `attach` | Återansluter terminalen till en körande session |
| `-t claude` | Väljer målsessionen utifrån dess namn |

</details>

På så sätt överlever en körande uppgift brutna anslutningar, enhetsbyten och laptopens nattvila.

## 9. Datahygien vid migration

Den känsligaste delen vid flytten till servern är inte tekniken utan frågan om vad man tar med sig. Tre regler:

- **Inga privata nycklar till servern.** I `authorized_keys` finns endast offentliga nycklar. Privata nycklar stannar på slutenheterna.
- **Kopiera inte inloggningsuppgifter slentrianmässigt.** Känsliga lokala filer som en `.credentials.json` hör inte hemma på VPS:en utan granskning. Autentisera dig i stället på nytt på servern.
- **Lägg först konfigurationen i en migrationsmapp.** Skriv inte befintliga Claude-minnen och -konfiguration direkt till de aktiva konfigurationssökvägarna, utan överför dem först till en separat migrationsmapp och kontrollera där vad som verkligen ska tas med. Det som inte längre behövs, exempelvis gamla MCP-poster eller övergivna inställningar, lämnas medvetet kvar i stället för att följa med okontrollerat.

## 10. Webbförhandsvisningar via en SSH-tunnel

För webbförhandsvisningar, exempelvis en lokal utvecklingsserver som Claude startar, är frestelsen stor att bara öppna ytterligare en port. Det bör du inte göra. Varje ytterligare öppen port är ytterligare angreppsyta. I stället körs förhandsvisningen genom en krypterad SSH-porttunnel: tjänsten lyssnar endast lokalt på servern, och SSH vidarebefordrar den till klienten.

Gör en tjänst som kör lokalt på port 4321 tillgänglig från datorn:

```bash
ssh -p 61417 -L 4321:localhost:4321 claude@SERVER
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-p 61417` | Porten som SSH-servern lyssnar på (den som valdes i steg 5) |
| `-L 4321:localhost:4321` | Lokal portvidarebefordran: anslutningar till den lokala porten 4321 vidarebefordras genom tunneln till `localhost:4321` sett från servern |
| `claude@SERVER` | Användare och målserver för SSH-anslutningen |

</details>

Öppna sedan `http://localhost:4321` i den lokala webbläsaren. Trafiken går helt genom den befintliga, autentiserade SSH-anslutningen utan att ens en enda ytterligare port behöver öppnas i brandväggen.

## Åtkomst från iPhone

Åtkomst på språng fungerar med samma säkerhetsmodell som från datorn. Det enda som behövs är en SSH-klient med nyckelhantering. Vanliga alternativ är **Termius**, **Blink Shell** och **Secure ShellFish**; alla kan skapa Ed25519-nycklar och lagra dem i iOS-nyckelringen, delvis skyddade med Face ID.

Förfarandet motsvarar steg 3, men på iPhone:

1. Skapa en egen Ed25519-nyckel för iPhone i SSH-klienten, kopiera inte datorns nyckel. Den privata nyckeln stannar i enhetens nyckelring.
2. Lägg till iPhones offentliga nyckel som en extra rad i `~/.ssh/authorized_keys` på servern, med en tydlig kommentar (`iphone-15`).
3. Skapa anslutningen i klienten: serveradress, användare `claude`, port `61417`, iPhone-nyckeln som autentisering.

Det är just därför en separat nyckel per enhet lönar sig: om iPhone försvinner tar man bort den enda `iphone-15`-raden från `authorized_keys` på servern, och enheten är utestängd medan åtkomst från datorn och alla andra nycklar fortsätter att fungera utan påverkan.

Efter anslutning hämtar du tillbaka den körande Claude-sessionen med `tmux attach -t claude` och fortsätter där du slutade vid skrivbordet. Porttunneln från steg 10 fungerar också från iOS; Termius och Secure ShellFish stöder portvidarebefordran.

## Checklista

Sammanfattning av hela processen:

1. Debian 13 installerat och helt uppdaterat med `apt full-upgrade`.
2. Egen användare `claude` med sudo-rättigheter; direkt root-inloggning används inte längre.
3. Lösenfras-skyddade Ed25519-nycklar, en per enhet, endast offentliga nycklar i `authorized_keys`.
4. sshd härdat: `PermitRootLogin no`, `PasswordAuthentication no`; kontrollerat med `sshd -t` före omladdning, befintlig session lämnad öppen tills testet var klart.
5. SSH på port 61417, vid socketaktivering angiven i `ssh.socket`, annars i sshd-konfigurationen.
6. Leverantörens brandvägg: inkommande standard DROP, enda undantaget TCP 61417; utgående tillåtet.
7. Angreppsytan kontrollerad med `ss -lntup` (endast SSH offentligt, `chronyd` lokalt) och `systemctl --failed` (inga fel).
8. Claude Code autentiserat på nytt på servern, drift i en `tmux`-session.
9. Datahygien: inga privata nycklar eller inloggningsuppgifter på servern, konfiguration först granskad via en migrationsmapp.
10. Inga ytterligare portar; webbförhandsvisningar körs genom en SSH-tunnel.

Efter denna konfiguration är endast SSH på den angivna porten åtkomligt utifrån, och även där uteslutande med en lösenfras-skyddad nyckel. Claude Code kör oberoende av slutenheten i `tmux`; webbförhandsvisningar förblir åtkomliga via SSH-tunnlar utan att öppna ytterligare en port.

## Källor

1.  [OpenSSH Manual – sshd_config(5)](https://man.openbsd.org/sshd_config): Referens för alla sshd-direktiv, däribland `PermitRootLogin`, `PasswordAuthentication` och `PubkeyAuthentication`.

2.  [Debian Wiki – SSH](https://wiki.debian.org/SSH): Debian-specifika anvisningar för SSH-konfigurationen, inklusive drop-in-filerna under `/etc/ssh/sshd_config.d/`.

3.  [systemd.socket(5) – freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html): Hur socketaktivering fungerar och `ListenStream=`-direktivet, relevant för byte av SSH-port i Debian 13.

4.  [ss(8) – iproute2 Manpage](https://man7.org/linux/man-pages/man8/ss.8.html): Alternativ för `ss` för att lista lyssnande socketar med process och bindningsadress.

5.  [Claude Code – Officiell dokumentation](https://docs.claude.com/en/docs/claude-code/overview): Installation, autentisering och drift av Claude Code.
