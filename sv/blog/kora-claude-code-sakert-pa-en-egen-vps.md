---
title: "Kör Claude Code säkert på din egen VPS"
navTitle: "VPS för Claude"
description: "En härdad Debian-VPS håller Claude Code-sessioner permanent tillgängliga. Guiden går från användarkonto och SSH-nycklar till brandvägg, datahygien, tmux och säker åtkomst från iPhone."
date: "2026-07-21"
kategorie: "Claude"
timeToRead: "12 min läsning"
themen:
  - claude
slug: "kora-claude-code-sakert-pa-en-egen-vps"
translationOf: "claude-code-vps-debian-absichern"
translationId: article-f932e9e537d7704a
translationReview: automatic
translationSourceHash: 0c584eb684bd92b439d97b20aae84cd15468c5c30d1e277d7fde7bd3fab9a37e
translatedAt: 2026-10-31T03:44:27.695Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/kora-claude-code-sakert-pa-en-egen-vps
---

En Claude Code-session på den egna datorn avslutas ofrivilligt senast när den bärbara datorn går i viloläge eller nätverksanslutningen bryts. En VPS fortsätter att köra och kan nås från flera enheter. Samtidigt är den permanent ansluten till det offentliga internet och skannas automatiskt kort efter uppstart.

Den här guiden kombinerar båda kraven: Claude Code förblir tillgängligt i en `tmux`-session, medan Debian-servern endast erbjuder en nyckelskyddad SSH-anslutning utifrån. Härdningen är inte specifik för Claude och lämpar sig även för andra offentligt tillgängliga Linux-servrar.

## Varför en VPS kan vara meningsfull

Jämfört med en rent lokal installation erbjuder servern tre praktiska fördelar:

- **Beständighet.** I en `tmux`-session fortsätter Claude att köra även om SSH-anslutningen bryts. En uppgift som tar tio minuter eller en timme slutförs utan att den bärbara datorn behöver vara öppen.
- **Tillgänglighet.** Samma session kan nås från stationär dator, bärbar dator och iPhone. Man startar en uppgift vid skrivbordet och kontrollerar resultatet på språng.
- **Datakontroll.** Man bestämmer själv vad som finns på servern. Ingen synktjänst, inga åtkomstuppgifter som av misstag säkerhetskopieras – förutsatt att man är noggrann vid migreringen (se nedan).

`tmux` är endast en funktion för tillgänglighet och bekvämlighet, inte en säkerhetsåtgärd. Det egentliga arbetet ligger i skyddet.

## Utgångsläge

Basen är Debian 13 (Trixie), minimalt installerat, utan skrivbordsmiljö och utan ytterligare nätverkstjänster. Leverantören tillhandahåller en framförliggande brandvägg som fungerar oberoende av operativsystemet. Målet är en server där endast SSH kan nås utifrån, och även det enbart med lösenfras-skyddade nycklar.

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
| `full-upgrade` | Uppdaterar alla paket och får även installera nya paket eller ta bort befintliga för detta |

</details>

Till skillnad från `upgrade` löser `full-upgrade` även beroenden som kräver nya eller borttagna paket. På ett nytt system är detta rätt sätt att faktiskt installera alla tillgängliga säkerhetsuppdateringar. Starta om en gång efter kärnuppdateringar.

## 2. Egen användare i stället för root

Att arbeta som root är onödigt riskabelt: varje skrivfel påverkar hela systemet, och direkt root-inloggning är det första automatiserade angrepp försöker med. Skapa därför en egen användare (här `claude`) med sudo-rättigheter för de fall där de behövs:

```bash
sudo adduser claude
sudo usermod -aG sudo claude
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-a` | Lägg till: kompletterar användarens gruppista i stället för att ersätta den; giltigt endast tillsammans med `-G` |
| `-G sudo` | Kompletterande grupp(er) som användaren läggs till i |
| `claude` | Berörd användare; vid `adduser` namnet på kontot som ska skapas |

</details>

Från och med nu sker all administration via `claude` och `sudo`, inte längre genom direkt root-åtkomst.

## 3. Ed25519-nycklar med lösenfras, en per enhet

Inloggning ska ske uteslutande med SSH-nycklar, inte lösenord. Ed25519 är dagens standard: kort, snabb och kryptografiskt robust. Avgörande är att nyckeln skapas på klienten, alltså på datorn och inte på servern, samt skyddas med en lösenfras. Lösenfrasen är den andra försvarslinjen om den privata nyckeln någonsin hamnar i fel händer.

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

Kommentaren (`-C`) namnger enheten. Det lönar sig senare: en separat nyckel skapas för varje enhet – en för datorn, en separat för iPhone. Om en enhet försvinner tar man selektivt bort dess offentliga nyckel från `~/.ssh/authorized_keys`, utan att behöva distribuera om alla andra åtkomster.

Endast den offentliga nyckeln hör hemma på servern. Den privata nyckeln lämnar aldrig enheten. I `authorized_keys` finns till slut endast offentliga nycklar, var och en med sin enhetskommentar:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...pc  pc-thinkpad
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...ios iphone-15
```

Överför först den offentliga datornyckeln. Så länge lösenordsinloggning fortfarande är aktiverad går det enklast med:

```bash
ssh-copy-id claude@SERVER
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `claude@SERVER` | Användare och målvärd; den offentliga standardnyckeln läggs där till i `~/.ssh/authorized_keys` |

</details>

Testa sedan att inloggning med nyckel fungerar innan lösenordsinloggning stängs av i nästa steg. Filrättigheterna måste vara korrekta, annars ignorerar sshd filen: `~/.ssh` på `700`, `authorized_keys` på `600`.

## 4. Härda SSH: ingen root, inget lösenord

Serverkonfigurationen finns i `/etc/ssh/sshd_config` och (i Debian 13) i drop-in-filer under `/etc/ssh/sshd_config.d/`. Ändringar hör hemma i en egen drop-in-fil; då förblir huvudfilen orörd och paketuppdateringar skriver inte över något. Skapa filen `/etc/ssh/sshd_config.d/99-haertung.conf`:

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

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-t` | Testläge: kontrollerar att konfigurationsfilen och nycklarna är giltiga utan att starta tjänsten |

</details>

Om `sshd -t` inte rapporterar något är filen giltig. Ladda först därefter om:

```bash
sudo systemctl reload ssh
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `reload` | Instruerar tjänsten att läsa in sin konfiguration på nytt utan att bryta befintliga anslutningar |
| `ssh` | Målenheten, här OpenSSH-tjänsten |

</details>

**Viktigt:** Låt den befintliga SSH-sessionen vara öppen och testa den nya åtkomsten i en andra terminal. Först när inloggningen med nyckel bevisligen fungerar där får den gamla sessionen stängas. Denna försiktighetsåtgärd minskar risken att låsa ute sig till praktiskt taget noll. Ett fel i konfigurationen skulle annars kosta all åtkomst.

## 5. Flytta SSH till en ovanlig port

Standardporten 22 provas av bottar dygnet runt. Ett byte till en hög, fritt vald port (i exemplet `61417`) gör att merparten av detta automatiserade brus inte leder någonstans. Detta är uttryckligen ingen säkerhetsvinst i egentlig mening: ett portbyte ersätter inte stark autentisering, det minskar endast mängden loggar och skanningsbelastningen. Nyckelkravet från steg 4 är fortfarande det egentliga skyddet.

Vilken port som används är inte godtyckligt. IANA skiljer mellan tre zoner: **0–1023 (well-known ports)** är reserverade för standardtjänster (SSH självt på 22, HTTP på 80, HTTPS på 443), kräver root för bindning och har inget att göra med en egenvald SSH-port; just dessa portar förväntar sig både skannrar och senare installerade standardtjänster. **1024–49151 (registrerade portar)** är tilldelade enskilda applikationer på begäran, exempelvis 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis) eller 8080/8443 som vanliga HTTP-alternativ; en slumpmässigt vald port i detta område kan senare lätt kollidera med programvara som förväntar sig just sin registrerade port. **49152–65535 (dynamiska/privata portar)** är enligt IANA inte tilldelade någon tjänst och avsedda för tillfälliga, privata ändamål – rätt område för en permanent, egenvald port.

En reservation kvarstår: många Linux-system, även Debian, använder en del av samma intervall som källport för egna utgående anslutningar (`net.ipv4.ip_local_port_range`, som standard omkring 32768–60999). En permanent lyssnande tjänst kolliderar egentligen inte med detta, eftersom kärnan inte tilldelar en redan bunden port, men en port över 60999 undviker även denna teoretiska osäkerhet. Exemplet i denna artikel (`61417`) ligger därför medvetet där. Kontrollera dessutom före ändringen med `ss -lntup` (se steg 7) att den valda porten inte redan används på den egna servern.

I Debian 13 finns en särskildhet: SSH kan startas via systemd-socketaktivering. I så fall ignoreras `Port`-angivelsen i `sshd_config` helt; porten måste då sättas i socketen. Kontrollera först vilket fall som gäller:

```bash
systemctl is-enabled ssh.socket
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `is-enabled` | Visar om enheten är aktiverad för systemstart |
| `ssh.socket` | Socketenheten för SSH-tjänsten |

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

Skriv in följande rader i redigeraren. Den första tomma `ListenStream=`-raden tar bort den förinställda porten 22, den andra sätter den nya:

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
| `daemon-reload` | Läser in alla enhetsfiler på nytt, inklusive override-filen som just skapats |
| `restart ssh.socket` | Startar om socketenheten så att den lyssnar på den nya porten |

</details>

Om socketaktivering inte är aktiv (`disabled`), ska i stället `Port 61417` finnas i drop-in-filen från steg 4, följt av `sudo sshd -t` och `sudo systemctl restart ssh`.

Även här gäller: öppna först den nya porten i brandväggen (nästa steg), anslut och testa sedan, och låt den gamla sessionen vara öppen tills åtkomsten via den nya porten har bekräftats.

## 6. Brandvägg: stängd som standard

Den framförliggande brandväggen hos leverantören är den effektivaste gränsen, eftersom den fångar paket innan de ens når operativsystemet. Två grundregler:

- **Standardåtgärd för inkommande trafik är DROP.** Allt som inte uttryckligen är tillåtet kastas bort, utan kommentar och utan återkoppling till avsändaren.
- **Ett enda undantag:** inkommande TCP till målporten `61417`. Mer behöver inte vara nåbart utifrån.

Utgående trafik förblir tillåten. Det är avsiktligt: servern måste kunna hämta paket, synkronisera tiden och nå API:et för Claude Code. Restriktiv filtrering av utgående trafik ger litet ytterligare skydd på en enskild server, men gör driften märkbart krångligare.

Den som vill ha ytterligare djupförsvar kan dubblera samma regler på värden med `nftables` eller `ufw`. För den beskrivna uppsättningen räcker leverantörens brandvägg.

## 7. Kontrollera attackytan

Efter härdningen kontrollerar du vad servern faktiskt erbjuder utåt. Två kommandon räcker. Först: vilka tjänster lyssnar på vilka adresser?

```bash
sudo ss -lntup
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-l` | Visa endast lyssnande sockets |
| `-n` | Numerisk utdata: portar och adresser löses inte upp till namn |
| `-t` | Inkludera TCP-sockets |
| `-u` | Inkludera UDP-sockets |
| `-p` | Visar processen bakom varje socket; för detta krävs `sudo` |

</details>

Adresskolumnen är avgörande: en tjänst på `0.0.0.0` eller `[::]` är nåbar utifrån, en på `127.0.0.1` eller `[::1]` endast lokalt. I säkrat tillstånd bör endast SSH synas offentligt. Tjänster som `chronyd` (tidssynkronisering) får förekomma, men endast bundna till lokala adresser. Om `chronyd` enbart lyssnar på `127.0.0.1` och `::1`, kan den inte nås utifrån och är därmed okritisk.

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

Svaret bör vara `0 loaded units listed`, inte en enda misslyckad tjänst. Felaktiga enheter är inte bara ett driftsproblem utan även potentiellt ett säkerhetsproblem om en halvstartad, felkonfigurerad nätverkstjänst ligger bakom.

## 8. Installera och köra Claude Code

Claude Code behöver ingen separat Node.js-körmiljö. Anthropic rekommenderar den inbyggda installationshanteraren (`curl -fsSL https://claude.ai/install.sh | bash`), som automatiskt uppdaterar Claude Code i bakgrunden; för Debian finns alternativt ett signerat apt-arkiv som uppdateras via vanliga systemuppdateringar. Efter installationen autentiserar du CLI:t på nytt på servern, i stället för att ladda upp lokala åtkomstuppgifter (mer om det strax).

För permanent drift, använd `tmux`:

```bash
tmux new -s claude
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `new` | Skapar en ny session |
| `-s claude` | Ger sessionen ett namn som den senare kan återupptas med |

</details>

Starta Claude i sessionen. Med `Ctrl-b`, följt av `d`, kopplar du från sessionen utan att avsluta den; Claude fortsätter att köra. Återgå med:

```bash
tmux attach -t claude
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `attach` | Återansluter terminalen till en pågående session |
| `-t claude` | Väljer målsessionen med dess namn |

</details>

På så sätt överlever en pågående uppgift brutna anslutningar, enhetsbyten och den bärbara datorns nattsömn.

## 9. Datahygien vid migrering

Den känsligaste delen av flytten till servern är inte tekniken utan frågan om vad som följer med. Tre regler:

- **Inga privata nycklar på servern.** I `authorized_keys` finns endast offentliga nycklar. Privata nycklar stannar på slutenheterna.
- **Kopiera inte åtkomstuppgifter slentrianmässigt.** Känsliga lokala filer som en `.credentials.json` hör inte hemma på VPS:en utan granskning. Autentisera dig i stället på nytt på servern.
- **Lägg först konfigurationen i en migreringsmapp.** Skriv inte befintliga Claude-minnen och -konfiguration direkt till de aktiva konfigurationssökvägarna, utan överför dem först till en separat migreringsmapp och kontrollera där vad som verkligen ska tas över. Det som inte längre behövs, såsom gamla MCP-poster eller övergivna inställningar, lämnas medvetet kvar i stället för att följa med okontrollerat.

## 10. Webbförhandsvisningar via en SSH-tunnel

För webbförhandsvisningar, till exempel en lokal utvecklingsserver som Claude startar, är det lockande att bara öppna ytterligare en port. Det bör man inte göra. Varje ytterligare öppen port är ytterligare attackyta. I stället körs förhandsvisningen genom en krypterad SSH-porttunnel: tjänsten lyssnar endast lokalt på servern och SSH vidarebefordrar den till klienten.

Gör en tjänst som körs lokalt på port 4321 tillgänglig från datorn:

```bash
ssh -p 61417 -L 4321:localhost:4321 claude@SERVER
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-p 61417` | Porten där SSH-servern lyssnar (den som valdes i steg 5) |
| `-L 4321:localhost:4321` | Lokal portvidarebefordran: anslutningar till den lokala porten 4321 vidarebefordras genom tunneln till `localhost:4321` sett från servern |
| `claude@SERVER` | Användare och målvärd för SSH-anslutningen |

</details>

Öppna sedan `http://localhost:4321` i den lokala webbläsaren. Trafiken går helt genom den befintliga, autentiserade SSH-anslutningen, utan att ens en enda ytterligare port behöver öppnas i brandväggen.

## Åtkomst från iPhone

Åtkomst på språng fungerar med samma säkerhetsmodell som från datorn. Allt som behövs är en SSH-klient med nyckelhantering. Vanliga alternativ är **Termius**, **Blink Shell** och **Secure ShellFish**; alla kan skapa Ed25519-nycklar och lagra dem i iOS-nyckelringen, delvis skyddade med Face ID.

Förfarandet motsvarar steg 3, men på iPhone:

1. Skapa en egen Ed25519-nyckel för iPhone i SSH-klienten, kopiera inte datorns nyckel. Den privata nyckeln stannar i enhetens nyckelring.
2. Lägg till iPhones offentliga nyckel som ytterligare en rad i `~/.ssh/authorized_keys` på servern, med en beskrivande kommentar (`iphone-15`).
3. Skapa anslutningen i klienten: serveradress, användare `claude`, port `61417`, iPhone-nyckeln för autentisering.

Det är just därför en separat nyckel per enhet är värdefull: om iPhone försvinner raderar du den enda `iphone-15`-raden från `authorized_keys` på servern, och enheten är utestängd medan datoråtkomst och alla andra nycklar fortsätter fungera opåverkade.

Efter anslutning hämtar du tillbaka den pågående Claude-sessionen med `tmux attach -t claude` och fortsätter där du slutade vid skrivbordet. Även porttunneln från steg 10 fungerar från iOS; Termius och Secure ShellFish har stöd för portvidarebefordran.

## Checklista

Sammanfattningsvis hela förfarandet:

1. Debian 13 installerat och helt uppdaterat med `apt full-upgrade`.
2. Egen användare `claude` med sudo-rättigheter; direkt root-inloggning används inte längre.
3. Lösenfras-skyddade Ed25519-nycklar, en per enhet, endast offentliga nycklar i `authorized_keys`.
4. sshd härdad: `PermitRootLogin no`, `PasswordAuthentication no`; kontrollerad med `sshd -t` före omladdning, befintlig session lämnad öppen tills testet var klart.
5. SSH på port 61417, vid socketaktivering inställd på `ssh.socket`, annars i sshd-konfigurationen.
6. Leverantörens brandvägg: inkommande standard DROP, enda undantaget TCP 61417; utgående tillåtet.
7. Attackyta kontrollerad med `ss -lntup` (endast SSH offentligt, `chronyd` lokalt) och `systemctl --failed` (inga fel).
8. Claude Code autentiserat på nytt på servern, drift i en `tmux`-session.
9. Datahygien: inga privata nycklar och inga åtkomstuppgifter på servern, konfiguration först kontrollerad via en migreringsmapp.
10. Inga ytterligare portar; webbförhandsvisningar körs genom en SSH-tunnel.

Efter denna installation är endast SSH på den angivna porten nåbar utifrån, och även där uteslutande med en lösenfras-skyddad nyckel. Claude Code körs oberoende av slutenheten i `tmux`; webbförhandsvisningar förblir åtkomliga via SSH-tunnlar utan att öppna ytterligare en port.

## Ändringshistorik

- **25 oktober 2026:** Rättat: Claude Code behöver ingen separat Node.js-körmiljö; den rekommenderade inbyggda installationshanteraren och det signerade apt-arkivet för Debian har lagts till. Källänkar har uppdaterats till de nya adresserna för Claude Code-dokumentationen.

## Källor

1.  [OpenSSH Manual – sshd_config(5)](https://man.openbsd.org/sshd_config): Referens för alla sshd-direktiv, inklusive `PermitRootLogin`, `PasswordAuthentication` och `PubkeyAuthentication`.

2.  [Debian Wiki – SSH](https://wiki.debian.org/SSH): Debian-specifika anvisningar för SSH-konfiguration, inklusive drop-in-filerna under `/etc/ssh/sshd_config.d/`.

3.  [systemd.socket(5) – freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html): Så fungerar socketaktivering och direktivet `ListenStream=`, relevant för byte av SSH-port under Debian 13.

4.  [ss(8) – iproute2 Manpage](https://man7.org/linux/man-pages/man8/ss.8.html): Alternativ för `ss` för att lista lyssnande sockets med process och bindningsadress.

5.  [Claude Code – Officiell dokumentation](https://code.claude.com/docs/en/overview): Installation, autentisering och drift av Claude Code.

6.  [Claude Code – Advanced setup](https://code.claude.com/docs/en/setup): Systemkrav, inbyggd installationshanterare, apt-arkiv och automatiska uppdateringar.
