---
title: "Lösenordslöst på Linux-servrar: konfigurera SSH-nyckelinloggning med PuTTY, Pageant med flera"
navTitle: "PuTTY lösenordslöst"
description: "Den som som administratör dagligen ansluter till Linux-servrar skriver användarnamn och lösenord vid varje lösenordsinloggning. Ett SSH-nyckelpar reducerar detta till ett dubbelklick: skapa nyckeln med PuTTYgen, lagra den offentliga nyckeln på servern och ladda Pageant. Samma nyckel fungerar i WinSCP, MobaXterm och OpenSSH, och den som vill hamnar direkt i skalet för ett tjänstekonto."
date: "2026-08-28"
kategorie: "Linux och SSH"
timeToRead: "9 min. läsning"
themen:
  - totemomail
  - windows-client
produkte:
  - "totemomail"
  - "uebergreifend"
protokolle:
  - "ssh"
  - "haertung"
slug: "losenordslost-pa-linux-servrar-konfigurera-ssh-nyckelinloggning-med-putty-pageant-med-flera"
translationId: "article-9f94fa6eb8b95bcf"
aiPrompt: |
  Du bist mein Linux- und SSH-Assistent. Hilf mir Schritt für Schritt, einen passwortlosen SSH-Login von Windows auf meine Linux-Server einzurichten: 1. Ein Ed25519-Schlüsselpaar mit PuTTYgen erzeugen und den Public Key in authorized_keys eintragen. 2. Die PuTTY-Session mit Schlüsseldatei und Auto-login username vervollständigen und Pageant mit Autostart einrichten. 3. Optional ein Remote command hinterlegen, das mich direkt in die Shell eines Service-Accounts bringt, samt minimaler NOPASSWD-Regel unter /etc/sudoers.d. Weise mich auf typische Fehler hin: Key im falschen Home-Verzeichnis, mehrzeiliger Public Key, falsche Berechtigungen, sudoers-Befehl stimmt nicht exakt mit dem Remote command überein.
translationOf: putty-ssh-login-service-account-shell
url: https://rafaelpfister.ch/sv/blog/losenordslost-pa-linux-servrar-konfigurera-ssh-nyckelinloggning-med-putty-pageant-med-flera
translationSourceHash: e95b27e9a86f59dfb0808afee63664493f5961b983f807f31ef9ee7a36f6fb3e
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:38:18.213Z
translationReview: automatic
---

# Lösenordslöst på Linux-servrar: konfigurera SSH-nyckelinloggning med PuTTY, Pageant med flera

Den som som administratör dagligen ansluter till Linux-servrar upprepar samma inmatningar vid varje anslutning med lösenordsinloggning: användarnamn, lösenord och för tjänstekonton dessutom ett sudo-kommando. Med ett SSH-nyckelpar försvinner allt detta. Efter konfigurationen öppnar ett dubbelklick på den sparade sessionen ett färdigt skal, och samma nyckel fungerar i PuTTY, WinSCP, MobaXterm och Windows OpenSSH-klient.

Den här guiden konfigurerar den lösenordslösa inloggningen helt från Windows: skapa nyckelpar, lagra den offentliga nyckeln på servern, slutför PuTTY-sessionen och konfigurera Pageant som nyckelagent. Dessutom ingår felsökning för det vanligaste problemet (servern frågar fortfarande efter lösenordet trots nyckeln) samt, som utvidgning, direktinloggning i skalet för ett tjänstekonto som `totemo`.

## Varför nycklar i stället för lösenord

Bekvämligheten är den tydligaste effekten, men inte den viktigaste. En Ed25519-nyckel är i praktiken immun mot brute-force-attacker, medan ett lösenord bara är så starkt som dess längd och disciplinen att aldrig återanvända det. På servrar där användarna helt har övergått till nycklar går det att helt stänga av lösenordsautentisering i sshd-konfigurationen (`PasswordAuthentication no`), vilket gör att automatiserade inloggningsförsök från internet inte leder någonstans. Stäng inte av lösenordsautentisering förrän nyckelinloggningen bevisligen fungerar och en andra åtkomstväg finns tillgänglig (konsol, andra nyckel).

Principen är enkel: den privata nyckeln stannar på din Windows-dator, medan den offentliga lagras på servern. När anslutningen upprättas kontrollerar servern om motparten har den matchande privata nyckeln, utan att den någonsin lämnar datorn.

## Steg 1: Skapa ett nyckelpar med PuTTYgen

1. Starta **PuTTYgen** (ingår i PuTTY-paketet), välj **Ed25519** som typ, klicka på **Generate** och rör musen över ytan.
2. Ange en **Passphrase** i båda fälten och spara den privata nyckeln med **Save private key** som en `.ppk`-fil.
3. Kopiera hela textfältet högst upp (raden som börjar med `ssh-ed25519 AAAA...`). Det är den offentliga nyckeln i det format som servern förväntar sig.

Spara den privata nyckeln med en lösenfras. Utan lösenfras är varje kopia av filen en färdig serveråtkomst, medan filen ensam är värdelös med lösenfras. Bekvämlighetsnackdelen elimineras nästan helt av Pageant (steg 4). En nyckel utan lösenfras är endast försvarbar för obevakad automatisering, inte för interaktiv inloggning.

## Steg 2: Lagra den offentliga nyckeln på servern

På servern, inloggad som den användare som du framöver ska ansluta med:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAA... kommentar' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod go-w ~
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Kommando / alternativ | Effekt |
|---|---|
| `mkdir -p ~/.ssh` | Skapar SSH-katalogen; `-p` undertrycker felet om den redan finns |
| `chmod 700 ~/.ssh` | Endast ägaren får läsa, skriva och gå in i katalogen |
| `echo '...' >> ~/.ssh/authorized_keys` | Lägger till den offentliga nyckeln som en ny rad i filen (`>>` i stället för `>`, annars skriver du över befintliga nycklar) |
| `chmod 600 ~/.ssh/authorized_keys` | Endast ägaren får läsa och skriva nyckelfilen |
| `chmod go-w ~` | Tar bort skrivrättigheten från grupp och andra för hemkatalogen |

</details>

Den sista raden verkar oansenlig, men hör till: en hemkatalog som kan skrivas av gruppen eller alla gör att SSH-demonen tyst ignorerar nyckeln och servern fortsätter fråga efter lösenordet utan att ange varför.

## Steg 3: Slutför PuTTY-sessionen

1. Öppna PuTTY, markera den sparade sessionen och ladda den med **Load**.
2. **Connection → SSH → Auth → Credentials**: välj `.ppk`-filen vid **Private key file for authentication**.
3. **Connection → Data**: ange användarnamnet vid **Auto-login username**, annars frågar PuTTY fortfarande efter det vid varje anslutning.
4. Gå tillbaka till kategorin **Session**, markera sessionsnamnet igen och klicka på **Save**.

Det vanligaste handhavandefelet är att utelämna **Load** före ändringen eller **Save** efteråt. Utan Load redigerar du bara standardinställningarna, utan Save är ändringen borta nästa gång PuTTY startas.

## Steg 4: Pageant, lösenfras en gång per Windows-session

Pageant är PuTTY:s nyckelagent. Den håller den dekrypterade privata nyckeln i minnet, så lösenfrasen bara behöver anges en gång per Windows-session:

1. Starta Pageant (ikonen visas i systemfältet).
2. Högerklicka på ikonen, välj **Add Key**, välj `.ppk`-filen och ange lösenfrasen.
3. Därefter fungerar alla anslutningar utan fråga fram till nästa omstart.

För att Pageant ska starta automatiskt skapar du en genväg i autostartmappen (`Win+R`, sedan `shell:startup`) och skickar med nyckeln som argument:

```text
"C:\Program Files\PuTTY\pageant.exe" "C:\Pfad\zum\schluessel.ppk"
```

Windows frågar då efter lösenfrasen en gång efter inloggningen, och resten av arbetsdagen fungerar utan.

## När servern fortsätter fråga efter lösenordet

Felsökningen börjar i PuTTY:s **Event Log** (högerklicka på namnlisten i terminalsessionen). Där framgår det om nyckeln över huvud taget erbjöds:

| Resultat i Event Log | Orsak och åtgärd |
|---|---|
| Ingen post om en offentlig nyckel | `.ppk`-filen är inte sparad i sessionen, eller så redigerades fel session. Ladda sessionen, ange nyckeln och spara. |
| `Server refused our key` | Servern hittar eller accepterar inte nyckeln: fel hemkatalog, fel format eller fel behörigheter (se nedan). |
| `Access granted`, följt av lösenordsfråga | Nyckelinloggningen fungerade; frågan kommer från ett efterföljande program, vanligtvis sudo. Se utvidgningen nedan. |

De tre vanligaste orsakerna till `Server refused our key`:

- **Nyckeln i fel hemkatalog.** Den offentliga nyckeln måste finnas i `authorized_keys` för den användare som anslutningen upprättas som. Om du redan har bytt till ett annat konto med `sudo -u` eller `su` när du lägger in den, hamnar filen i dess hemkatalog i stället för din egen. `whoami` före inmatningen visar i vems hemkatalog nyckeln hamnar.
- **Fel format.** Den offentliga nyckeln måste stå på en enda rad i `authorized_keys`, i formatet från textfältet högst upp i PuTTYgen. Filen från **Save public key** har ett annat format på flera rader (`---- BEGIN SSH2 PUBLIC KEY ----`) och fungerar inte i `authorized_keys`.
- **Behörigheter.** `~/.ssh` på `700`, `authorized_keys` på `600`, hemkatalogen får inte vara skrivbar för gruppen eller alla.

Om problemet fortfarande är oklart hjälper det att på serversidan titta i `/var/log/secure` respektive `journalctl -u sshd`; där förklarar SSH-demonen varför nyckeln avvisas.

## Samma nyckel i andra verktyg

Konfigurationen på servern är verktygsoberoende och nyckeln kan återanvändas överallt:

| Verktyg | Konfiguration |
|---|---|
| **WinSCP** | Använder `.ppk`-filer direkt (Inloggning → Avancerat → SSH → Autentisering) och använder automatiskt Pageant om nyckeln är laddad där |
| **MobaXterm** | `.ppk` under Session settings → SSH → Advanced → Use private key; förstår även OpenSSH-formatet |
| **FileZilla** | Ange `.ppk` under Inställningar → SFTP eller låt Pageant vara igång |
| **OpenSSH (Windows Terminal, `ssh`)** | Kräver OpenSSH-formatet: exportera i PuTTYgen via **Conversions → Export OpenSSH key** och lagra i `~/.ssh/` |

För OpenSSH-klienten omfattar bekvämlighetsinloggningen en post i `~/.ssh/config` på Windows-datorn:

```text
Host mailgw
    HostName server.example.com
    User mmuster
    IdentityFile ~/.ssh/id_ed25519
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Host mailgw` | Fritt valbart alias; `ssh mailgw` räcker sedan som anrop |
| `HostName` | Serverns faktiska namn eller IP-adress |
| `User` | Användarnamn, motsvarar Auto-login username i PuTTY |
| `IdentityFile` | Sökväg till den privata nyckeln i OpenSSH-format |

</details>

## Utvidgning: hamna direkt i tjänstekontots skal

Många Linux-servrar i applikations- och e-postmiljöer administreras inte med ett personligt konto utan via ett tjänstekonto: Totemomail körs under `totemo`, medan andra gateways och applikationer har egna funktionskonton. Efter inloggningen följer därför rutinmässigt samma kommando:

```bash
sudo -u totemo /bin/bash -l
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `sudo` | Kör följande kommando med andra rättigheter och loggar anropet |
| `-u totemo` | Målanvändaren är `totemo` i stället för standardanvändaren `root` |
| `/bin/bash` | Kommandot som ska köras: ett nytt Bash-skal |
| `-l` | Startar Bash som inloggningsskal; det läser in mål­användarens profil (`.bash_profile`, miljövariabler, sökvägar) |

</details>

`-l` är avgörande för tjänstekonton: utan inloggningsskal saknas miljövariablerna från funktionskontots profil, exempelvis sökvägar till applikationskataloger eller Java-installationer, och applikationsspecifika kommandon misslyckas med missvisande felmeddelanden.

Direkt SSH-inloggning som tjänstekonto vore ännu kortare, men är av goda skäl oftast inte alls möjlig: funktionskonton har ofta inget lösenord eller ett spärrat inloggningsskal, och en direktåtkomst som delas av flera personer skulle undanröja den personliga granskningsloggen. Via sudo går det fortsatt att följa vilken person som när bytte till tjänstekontots skal. Följande automatisering förändrar inget i detta, den sparar bara inmatningsarbetet.

### Remote command i PuTTY-sessionen

PuTTY kan lagra ett kommando per sparad session som körs efter inloggningen i stället för det vanliga skalet:

1. Ladda sessionen med **Load**.
2. Navigera till **Connection → SSH** i trädet till vänster (huvudnoden, inte en underpunkt).
3. Ange i fältet **Remote command**: `sudo -u totemo /bin/bash -l`
4. Spara under **Session**.

Du bör känna till tre särdrag hos Remote command:

- Ett `exit` i tjänstekontots skal avslutar anslutningen helt, i stället för att återgå till ditt personliga skal. Kommandot ersätter inloggningsskalet, det kapslar inte in det.
- Om du ibland arbetar på servern med ditt personliga konto sparar du en andra session utan Remote command (ladda sessionen, töm fältet och spara under ett nytt namn).
- Filöverföringsverktyg som WinSCP eller `pscp` påverkas inte. De upprättar egna anslutningar och ignorerar PuTTY-sessionens Remote command.

Om anslutningen stängs direkt efter upprättandet eller sudo rapporterar att en terminal saknas: kontrollera under **Connection → SSH → TTY** att **Don't allocate a pseudo-terminal** inte är ikryssat. Som standard är det inte det. Viktigt för steg 2 ovan: med aktivt Remote command är du redan tjänstekontot när du lägger in den offentliga nyckeln; nyckeln ska däremot ligga i det personliga användarkontots hemkatalog.

I OpenSSH-klienten gör två rader i `Host`-posten samma sak: `RequestTTY yes` och `RemoteCommand sudo -u totemo /bin/bash -l`.

### sudo utan lösenordsfråga

Efter nyckeln och Remote command återstår en enda inmatning: sudo:s lösenordsfråga. Den försvinner endast genom sudoers-konfigurationen på servern, och det kräver root-rättigheter. På en hanterad företagsserver är detta en begäran till serveradministratören, inte en inställning i PuTTY.

Regeln hör hemma i en egen fil under `/etc/sudoers.d/` och redigeras med `visudo`:

```bash
visudo -f /etc/sudoers.d/totemo-shell
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `visudo` | Öppnar sudoers-filen i redigeraren och kontrollerar syntaxen före sparandet |
| `-f /etc/sudoers.d/totemo-shell` | Redigerar den angivna filen i stället för den centrala `/etc/sudoers` |

</details>

Filens innehåll, med det personliga användarnamnet (här `mmuster` som exempel):

```text
mmuster ALL=(totemo) NOPASSWD: /bin/bash -l
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Del | Effekt |
|---|---|
| `mmuster` | Regeln gäller endast denna användare |
| `ALL=` | På alla värdar (relevant för centralt distribuerade sudoers-filer) |
| `(totemo)` | Endast för kommandon som målanvändaren `totemo`, inte som root |
| `NOPASSWD:` | Ingen lösenordsfråga för följande kommandon |
| `/bin/bash -l` | Exakt detta kommando med exakt detta argument, inget annat |

</details>

Två punkter avgör om regeln fungerar och om den är försvarbar:

- **Exakt överensstämmelse.** Kommandot i regeln måste motsvara Remote command, inklusive argument. Om PuTTY innehåller `sudo -u totemo /bin/bash -l`, måste regeln tillåta `/bin/bash -l`. En regel för `/bin/bash` utan `-l` täcker inte anropet med `-l`, och sudo fortsätter fråga efter lösenordet.
- **Minsta möjliga omfattning.** Regeln tillåter ett enda kommando som en enda målanvändare. Den ger varken root-rättigheter eller tillgång till godtyckliga kommandon. I denna form är den en vanlig och välmotiverad begäran även på hanterade servrar. sudo-loggningen behålls helt, varje byte till tjänstekontots skal finns fortsatt i loggen.

`visudo` är inte valfritt: det kontrollerar syntaxen före sparandet. Ett skrivfel som skrivs direkt till filen kan göra sudo obrukbart för alla systemets användare. Av samma skäl är en egen fil under `/etc/sudoers.d/` att föredra framför att redigera den centrala `/etc/sudoers`; den överlever paketuppdateringar och kan tas bort utan risk.

## Resultatet

Efter konfigurationen ser inloggningen ut så här: dubbelklicka på PuTTY-sessionen, så är skalet klart. Inget användarnamn, inget lösenord och med utvidgningen inte heller någon sudo-fråga. Säkerhetsläget har inte försämrats, utan till och med förbättrats på en punkt:

| Aspekt | Före | Efter |
|---|---|---|
| Autentisering | Lösenord vid varje inloggning | Ed25519-nyckel med lösenfras, lagrad i Pageant |
| Inloggningsidentitet | Personligt konto | Oförändrat personligt konto |
| sudo-loggning | Varje byte i loggen | Oförändrat varje byte i loggen |
| NOPASSWD-omfattning | Ingen | Ett kommando, en målanvändare, inte root |

## Källor

1.  [PuTTY User Manual, kapitel 4: Configuring PuTTY](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter4.html): Dokumentation av sessionsinställningarna, däribland Remote command (SSH panel), Auto-login username (Data panel) och Pseudo-Terminal (TTY panel).

2.  [PuTTY User Manual, kapitel 8: Using public keys for SSH authentication](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter8.html): PuTTYgen, nyckeltyper, lösenfraser, OpenSSH-export och hur den offentliga nyckeln läggs in på servern.

3.  [PuTTY User Manual, kapitel 9: Using Pageant for authentication](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter9.html): Agentens funktion, laddning av nycklar och start med nyckeln som kommandoradsargument.

4.  [ssh_config(5) Manual](https://man.openbsd.org/ssh_config.5): Klientkonfiguration för OpenSSH-klienten, inklusive värdalias, IdentityFile, RequestTTY och RemoteCommand.

5.  [sudoers(5) Manual](https://www.sudo.ws/docs/man/sudoers.man/): Syntax för sudoers-regler, Runas-specifikation och NOPASSWD-tagg.

6.  [sshd(8) Manual, avsnittet AUTHORIZED_KEYS FILE FORMAT](https://man.openbsd.org/sshd.8): Formatet för authorized_keys-filen och kraven på filbehörigheter.
