---
title: "Köra Claude Code säkert på en egen VPS"
navTitle: "VPS för Claude"
description: "En härdad Debian-VPS håller Claude Code-sessioner permanent tillgängliga. Guiden går från användarkonto och SSH-nycklar till brandvägg, datahygien, tmux och säker åtkomst från iPhone."
date: "2026-07-21"
kategorie: "Claude"
timeToRead: "12 min lästid"
themen:
  - "claude"
slug: "kora-claude-code-sakert-pa-en-egen-vps"
translationOf: "claude-code-vps-debian-absichern"
url: "https://rafaelpfister.ch/sv/blog/kora-claude-code-sakert-pa-en-egen-vps"
---

På den egna datorn avslutas en Claude Code-session ofrivilligt senast när laptopen försätts i viloläge eller nätverksanslutningen bryts. En VPS fortsätter att köra och är tillgänglig från flera enheter. Samtidigt är den permanent ansluten till det offentliga internet och skannas automatiskt kort efter starten.

Den här guiden kombinerar båda kraven: Claude Code förblir tillgängligt i en `tmux`-session, medan Debian-servern endast erbjuder en nyckelskyddad SSH-anslutning utåt. Härdningen är inte specifik för Claude och lämpar sig även för andra offentligt tillgängliga Linux-servrar.

## Varför en VPS kan vara meningsfull

Jämfört med en rent lokal installation erbjuder servern tre praktiska fördelar:

- **Beständighet.** I en `tmux`-session fortsätter Claude att köra även om SSH-anslutningen bryts. En uppgift som tar tio minuter eller en timme slutförs utan att laptopen behöver vara öppen.
- **Tillgänglighet.** Samma session är åtkomlig från stationär dator, laptop och iPhone. Man startar en uppgift vid skrivbordet och tittar på resultatet på språng.
- **Datakontroll.** Du bestämmer själv vad som finns på servern. Ingen synkningstjänst, inga åtkomstuppgifter som av misstag säkerhetskopieras, förutsatt att migrationen görs omsorgsfullt (se nedan).

`tmux` är enbart en funktion för tillgänglighet och bekvämlighet, inte en säkerhetsåtgärd. Det egentliga arbetet ligger i skyddet.

## Utgångsläge

Basen är Debian 13 (Trixie), minimalt installerat, utan skrivbordsmiljö och utan ytterligare nätverkstjänster. Leverantören erbjuder en brandvägg framför servern som fungerar oberoende av operativsystemet. Målet är en server där endast SSH är åtkomligt utifrån, och även det bara med lösenfras-skyddade nycklar.

## 1. Uppdatera systemet

Uppdatera hela paketbeståndet direkt efter installationen:

```bash
sudo apt update
sudo apt full-upgrade
```

`full-upgrade` löser, till skillnad från `upgrade`, även beroenden som kräver nya eller borttagna paket. På ett nytt system är detta rätt sätt att verkligen installera alla tillgängliga säkerhetsuppdateringar. Starta om en gång efter kärnuppdateringar.

## 2. Egen användare i stället för root

Att arbeta som root är onödigt riskabelt: varje skrivfel påverkar hela systemet, och direkt root-inloggning är det första automatiserade angrepp försöker med. Skapa därför en egen användare (här `claude`) med sudo-rättigheter för de tillfällen då de behövs:

```bash
sudo adduser claude
sudo usermod -aG sudo claude
```

Från och med nu sker all administration via `claude` och `sudo`, inte längre via direkt root-åtkomst.

## 3. Ed25519-nycklar med lösenfras, en per enhet

Inloggning ska endast ske med SSH-nycklar, inte lösenord. Ed25519 är den aktuella standarden: kort, snabb och kryptografiskt robust. Avgörande är att nyckeln skapas på klienten, alltså på datorn och inte på servern, och skyddas med en lösenfras. Lösenfrasen är den andra försvarslinjen om den privata nyckeln någonsin hamnar i fel händer.

På datorn:

```bash
ssh-keygen -t ed25519 -C "pc-thinkpad"
```

Kommentaren (`-C`) namnger enheten. Det lönar sig senare: en egen nyckel skapas för varje enhet – en för datorn, en separat för iPhone. Om en enhet försvinner tar du specifikt bort dess offentliga nyckel från `~/.ssh/authorized_keys`, utan att behöva distribuera om alla andra åtkomster.

Endast den offentliga nyckeln hör hemma på servern. Den privata nyckeln lämnar aldrig enheten. I `authorized_keys` finns till slut enbart offentliga nycklar, var och en med sin enhetskommentar:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...pc  pc-thinkpad
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...ios iphone-15
```

Överför först datorns offentliga nyckel. Så länge lösenordsinloggning fortfarande är aktiv är det enklast med:

```bash
ssh-copy-id claude@SERVER
```

Testa därefter att inloggning med nyckel fungerar innan lösenordsinloggning stängs av i nästa steg. Filrättigheterna måste vara korrekta, annars ignorerar sshd filen: `~/.ssh` på `700`, `authorized_keys` på `600`.

## 4. Härda SSH: ingen root, inget lösenord

Serverkonfigurationen finns i `/etc/ssh/sshd_config` och (i Debian 13) i drop-in-filer under `/etc/ssh/sshd_config.d/`. Ändringar ska göras i en egen drop-in-fil; då förblir huvudfilen orörd och paketuppdateringar skriver inte över något. Skapa filen `/etc/ssh/sshd_config.d/99-haertung.conf`:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

Detta inaktiverar direkt root-inloggning och lösenordsinloggning. Från och med nu kommer endast den in som har en passande privat nyckel. Kontrollera konfigurationens syntax före omladdning:

```bash
sudo sshd -t
```

Om `sshd -t` inte visar något är filen giltig. Ladda först därefter om:

```bash
sudo systemctl reload ssh
```

**Viktigt:** Låt den befintliga SSH-sessionen vara öppen och testa den nya åtkomsten i en andra terminal. Först när inloggning med nyckel bevisligen fungerar där får den gamla sessionen stängas. Denna försiktighetsåtgärd minskar risken att låsa ute sig själv till praktiskt taget noll. Ett fel i konfigurationen kan annars kosta hela åtkomsten.

## 5. Flytta SSH till en ovanlig port

Standardporten 22 provas dygnet runt av bottar. Att byta till en hög, fritt vald port (i exemplet `61417`) gör att merparten av detta automatiserade brus inte leder någonstans. Det är uttryckligen ingen säkerhetsvinst i egentlig mening: ett portbyte ersätter inte stark autentisering, utan minskar bara mängden loggar och skanningsbelastningen. Nyckelkravet från steg 4 är fortfarande det egentliga skyddet.

Vilken port det blir är inte godtyckligt. IANA skiljer mellan tre zoner: **0–1023 (well-known ports)** är reserverade för standardtjänster (SSH självt på 22, HTTP på 80, HTTPS på 443), kräver root för bindning och hör inte hemma som en egendefinierad SSH-port; just dessa portar förväntas av skannrar liksom av standardtjänster som installeras senare. **1024–49151 (registrerade portar)** är tilldelade enskilda applikationer på begäran, exempelvis 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis) eller 8080/8443 som vanliga HTTP-alternativ; en slumpmässigt vald port från detta område kan senare lätt krocka med programvara som förväntar sig just sin registrerade port. **49152–65535 (dynamiska/privata portar)** är enligt IANA inte tilldelade någon tjänst och avsedda för tillfälliga, privata ändamål – rätt intervall för en permanent, egendefinierad port.

Ett förbehåll kvarstår: många Linux-system, även Debian, använder en del av samma intervall som källport för egna utgående anslutningar (`net.ipv4.ip_local_port_range`, som standard omkring 32768–60999). En permanent lyssnande tjänst krockar därför inte i praktiken, kärnan tilldelar inte en redan bunden port, men en port över 60999 undviker även denna teoretiska oklarhet. Exemplet i den här artikeln (`61417`) ligger därför medvetet där. Kontrollera dessutom före ändringen med `ss -lntup` (se steg 7) att den valda porten inte redan används på din server.

I Debian 13 finns en fallgrop: SSH kan startas via systemd-socketaktivering. Om så är fallet ignoreras `Port`-angivelsen i `sshd_config`; porten måste då anges på socketen. Kontrollera först vilket fall som gäller:

```bash
systemctl is-enabled ssh.socket
```

Om kommandot svarar `enabled` körs SSH via socketen. Ändra då porten där:

```bash
sudo systemctl edit ssh.socket
```

Ange följande rader i redigeraren. Den första, tomma `ListenStream=`-raden tar bort den förinställda porten 22, den andra anger den nya:

```text
[Socket]
ListenStream=
ListenStream=61417
```

Tillämpa sedan ändringarna:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

Om socketaktivering inte är aktiv (`disabled`) ska i stället `Port 61417` placeras i drop-in-filen från steg 4, följt av `sudo sshd -t` och `sudo systemctl restart ssh`.

Även här gäller: öppna först den nya porten i brandväggen (nästa steg), anslut och testa sedan, och låt den gamla sessionen vara öppen tills åtkomst via den nya porten har bekräftats.

## 6. Brandvägg: stängd som standard

Leverantörens brandvägg framför servern är den effektivaste gränsen eftersom den fångar upp paket innan de ens når operativsystemet. Två grundregler:

- **Standardåtgärd för inkommande trafik: DROP.** Allt som inte uttryckligen tillåts kastas bort, utan kommentar och utan återkoppling till avsändaren.
- **Ett enda undantag:** inkommande TCP till målporten `61417`. Inget mer behöver vara åtkomligt utifrån.

Utgående trafik förblir tillåten. Det är avsiktligt: servern måste kunna hämta paket, synkronisera tiden och nå API:et för Claude Code. Restriktiv filtrering av utgående trafik ger litet extra skydd för en enskild server men gör driften märkbart mer omständlig.

Den som vill ha ytterligare försvar i djupet kan dubblera samma regler på värddatorn med `nftables` eller `ufw`. För den beskrivna uppsättningen räcker leverantörens brandvägg.

## 7. Kontrollera attackytan

Kontrollera efter härdningen vad servern faktiskt erbjuder utåt. Två kommandon räcker. Först: vilka tjänster lyssnar på vilka adresser?

```bash
sudo ss -lntup
```

`ss` listar alla lyssnande TCP- och UDP-socketar med tillhörande process (`sudo` behövs för att se processnamnen). Avgörande är adresskolumnen: en tjänst på `0.0.0.0` eller `[::]` är åtkomlig utifrån, medan en på `127.0.0.1` eller `[::1]` bara är lokal. I det säkrade läget bör endast SSH visas offentligt. Tjänster som `chronyd` (tidssynkronisering) får förekomma, men endast bundna till lokala adresser. Om `chronyd` endast lyssnar på `127.0.0.1` och `::1` är den inte åtkomlig utifrån och därmed okritisk.

För det andra: finns det misslyckade systemtjänster som tyder på ett konfigurationsproblem?

```bash
systemctl --failed
```

Svaret bör vara `0 loaded units listed`, inte en enda misslyckad tjänst. Felaktiga units är inte bara ett driftproblem utan potentiellt också ett säkerhetsproblem om de döljer en halvstartad, felkonfigurerad nätverkstjänst.

## 8. Installera och köra Claude Code

Claude Code behöver en aktuell Node.js-körmiljö. Efter installationen konfigurerar du CLI:t enligt den officiella guiden och autentiserar på nytt på servern – ladda inte upp de lokala åtkomstuppgifterna (mer om det strax).

För permanent drift `tmux`:

```bash
tmux new -s claude
```

Starta Claude i sessionen. Med `Ctrl-b`, sedan `d`, kopplar du från sessionen utan att avsluta den; Claude fortsätter att köra. Du återvänder med:

```bash
tmux attach -t claude
```

På så sätt överlever en pågående uppgift brutna anslutningar, enhetsbyten och laptopens nattvila.

## 9. Datahygien vid migration

Den känsligaste delen vid flytten till servern är inte tekniken utan frågan om vad som ska följa med. Tre regler:

- **Inga privata nycklar på servern.** I `authorized_keys` finns enbart offentliga nycklar. Privata nycklar stannar på slutenheterna.
- **Kopiera inte åtkomstuppgifter slentrianmässigt.** Känsliga lokala filer som en `.credentials.json` hör inte hemma på VPS:en utan granskning. Autentisera i stället på nytt på servern.
- **Lägg först konfiguration i en migrationsmapp.** Skriv inte befintliga Claude-minnen och -konfigurationer direkt till de aktiva konfigurationssökvägarna, utan överför dem först till en separat migrationsmapp och kontrollera där vad som verkligen ska tas över. Det som inte längre behövs, såsom gamla MCP-poster eller övergivna inställningar, lämnas medvetet kvar i stället för att följa med okontrollerat.

## 10. Webbförhandsvisningar via en SSH-tunnel

För webbförhandsvisningar, exempelvis en lokal utvecklingsserver som Claude startar, är frestelsen stor att helt enkelt öppna ytterligare en port. Gör inte det. Varje extra öppen port är extra attackyta. I stället körs förhandsvisningen genom en krypterad SSH-porttunnel: tjänsten lyssnar bara lokalt på servern och SSH vidarebefordrar den till klienten.

Gör en tjänst som körs lokalt på port 4321 åtkomlig från datorn:

```bash
ssh -p 61417 -L 4321:localhost:4321 claude@SERVER
```

Öppna sedan `http://localhost:4321` i den lokala webbläsaren. Trafiken går helt genom den befintliga, autentiserade SSH-anslutningen utan att en enda ytterligare port behöver öppnas i brandväggen.

## Åtkomst från iPhone

Åtkomst på resande fot fungerar med samma säkerhetsmodell som från datorn. Du behöver bara en SSH-klient med nyckelhantering. Vanliga alternativ är **Termius**, **Blink Shell** och **Secure ShellFish**; alla kan skapa Ed25519-nycklar och lagra dem i iOS-nyckelringen, delvis skyddade med Face ID.

Förfarandet motsvarar steg 3, bara på iPhone:

1. Skapa en egen Ed25519-nyckel för iPhone i SSH-klienten, kopiera inte datorns nyckel. Den privata nyckeln stannar i enhetens nyckelring.
2. Lägg till iPhones offentliga nyckel som en extra rad i `~/.ssh/authorized_keys` på servern, med en beskrivande kommentar (`iphone-15`).
3. Skapa anslutningen i klienten: serveradress, användare `claude`, port `61417`, iPhone-nyckeln som autentisering.

Det är just därför en separat nyckel per enhet lönar sig: om iPhone försvinner raderar du den enda `iphone-15`-raden från `authorized_keys` på servern, och enheten är utelåst medan datoråtkomsten och alla andra nycklar fortsätter fungera utan påverkan.

Efter anslutning hämtar du tillbaka den körande Claude-sessionen med `tmux attach -t claude` och fortsätter där du slutade vid skrivbordet. Även porttunneln från steg 10 fungerar från iOS; Termius och Secure ShellFish har stöd för portvidarebefordran.

## Checklista

Sammanfattat, hela förloppet:

1. Debian 13 installerat och fullt uppdaterat med `apt full-upgrade`.
2. Egen användare `claude` med sudo-rättigheter; direkt root-inloggning används inte längre.
3. Lösenfras-skyddade Ed25519-nycklar, en per enhet, endast offentliga nycklar i `authorized_keys`.
4. sshd härdad: `PermitRootLogin no`, `PasswordAuthentication no`; kontrollerad med `sshd -t` före omladdning, befintlig session lämnades öppen tills testet var klart.
5. SSH på port 61417, vid socketaktivering angiven i `ssh.socket`, annars i sshd-konfigurationen.
6. Leverantörens brandvägg: inkommande standardåtgärd DROP, enda undantaget TCP 61417; utgående trafik tillåten.
7. Attackytan kontrollerad med `ss -lntup` (endast SSH offentligt, `chronyd` lokalt) och `systemctl --failed` (inga fel).
8. Claude Code autentiserat på nytt på servern, drift i en `tmux`-session.
9. Datahygien: inga privata nycklar och inga åtkomstuppgifter på servern, konfigurationen kontrollerades först via en migrationsmapp.
10. Inga ytterligare portar; webbförhandsvisningar körs genom en SSH-tunnel.

Efter denna konfigurering är endast SSH på den angivna porten åtkomligt utifrån, och även där uteslutande med en lösenfras-skyddad nyckel. Claude Code kör oberoende av slutenheten i `tmux`; webbförhandsvisningar förblir åtkomliga via SSH-tunnlar utan att öppna ytterligare en port.

## Källor

1.  [OpenSSH Manual – sshd_config(5)](https://man.openbsd.org/sshd_config): Referens för alla sshd-direktiv, bland annat `PermitRootLogin`, `PasswordAuthentication` och `PubkeyAuthentication`.

2.  [Debian Wiki – SSH](https://wiki.debian.org/SSH): Debian-specifika anvisningar för SSH-konfiguration, inklusive drop-in-filerna under `/etc/ssh/sshd_config.d/`.

3.  [systemd.socket(5) – freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html): Hur socketaktivering fungerar och direktivet `ListenStream=`, relevant för byte av SSH-port i Debian 13.

4.  [ss(8) – iproute2 Manpage](https://man7.org/linux/man-pages/man8/ss.8.html): Alternativ för `ss` för att lista lyssnande socketar tillsammans med process och bindningsadress.

5.  [Claude Code – Officiell dokumentation](https://docs.claude.com/en/docs/claude-code/overview): Installation, autentisering och drift av Claude Code.
