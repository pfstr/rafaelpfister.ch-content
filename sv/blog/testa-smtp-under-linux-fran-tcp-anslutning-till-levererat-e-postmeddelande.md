---
title: "Testa SMTP under Linux: från TCP-anslutning till levererat mejl"
navTitle: "Testa SMTP"
description: "När en appliance inte längre levererar mejl hjälper ett manuellt SMTP-test mer än någon logg. Så kontrollerar du lager för lager med standardverktyg, vad olika felbilder betyder och varför en lastbalanserare kan förvanska diagnosen."
date: "2026-07-31"
kategorie: "SMTP och e-postflöde"
timeToRead: "10 min läsning"
themen:
  - smtp-mailflow
  - testing
  - e-mail-verschluesselung
slug: "testa-smtp-under-linux-fran-tcp-anslutning-till-levererat-e-postmeddelande"
translationId: "article-cb44a92c03a47bc0"
translationOf: smtp-verbindung-testen-linux
translationSourceHash: af2a802f67ec6d294b1507eaf26e25704b938e8760ac6751104ce7258cc2a4b3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:18:28.312Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/testa-smtp-under-linux-fran-tcp-anslutning-till-levererat-e-postmeddelande
---

# Testa SMTP under Linux: från TCP-anslutning till levererat mejl

När en e-postgateway plötsligt inte längre levererar något visar appliance-loggarna ofta bara slutresultatet: en leverans misslyckas, kön växer och ett felmeddelande anger en timeout. Vad det faktiskt beror på framgår först av ett manuellt test från kommandoraden. SMTP är ett klartextprotokoll som kan hanteras helt manuellt, och just det gör det till ett diagnostiskt verktyg som är tillgängligt överallt utan extra installation.

Det andra skälet till det manuella testet: På appliances går det oftast inte att installera något. Ingen pakethanterare, inga root-rättigheter, ingen `swaks`. Alla följande steg fungerar därför med det som redan finns på i stort sett alla Linux-system.

## Separera lagren

Ett misslyckat mejlutskick kan fallera på fem olika nivåer, och var och en ger en annan felbild:

1. **Namnupplösning:** Målhosten kan inte översättas till en IP-adress.
2. **TCP-anslutning:** Anslutningen till porten upprättas inte eller återställs.
3. **SMTP-dialog:** Anslutningen finns, men servern avvisar avsändare, mottagare eller innehåll.
4. **Transportkryptering:** STARTTLS saknas, certifikatet är ogiltigt eller TLS-versionen passar inte.
5. **Avsändarkontroll:** Mejlet accepteras men avvisas hos mottagaren på grund av SPF, DKIM eller DMARC.

Diagnosen blir betydligt bättre när du kontrollerar dessa nivåer i följd och var för sig, i stället för att direkt skicka ett fullständigt testmejl. Ett misslyckat totalförsök säger bara att något inte fungerar. Lagerkontrollen berättar vad.

## Steg 1: Namnupplösning

```bash
getent hosts relay.example.com
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `hosts` | NSS-databas att fråga; använder samma källor och samma ordning som systemet självt, enligt `nsswitch.conf` |
| `relay.example.com` | Värdnamn som ska lösas upp |

</details>

Om utdata förblir tom finns ingen namnserver tillgänglig på denna host, eller så besvarar den inte externa namn. Det förekommer regelbundet i praktiken: appliances i isolerade zoner får ofta bara en intern resolver som enbart känner till egna zoner.

```bash
cat /etc/resolv.conf; grep hosts: /etc/nsswitch.conf
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `/etc/resolv.conf` | Fil som `cat` skriver ut, med de konfigurerade namnservrarna |
| `hosts:` | Sökuttryck för `grep`: raden som fastställer ordningen för upplösningskällorna (filer, DNS) |
| `/etc/nsswitch.conf` | Fil med NSS-konfigurationen som genomsöks av `grep` |

</details>

Om upplösning saknas testar du direkt mot IP-adressen i det följande. Det är helt tillräckligt för diagnosen och skiljer tydligt DNS-problemet från transportproblemet. I produktionsdrift är den saknade upplösningen naturligtvis ett separat fynd som måste åtgärdas.

## Steg 2: Portens nåbarhet

För en ren TCP-kontroll räcker bash. Pseudoenheten `/dev/tcp` öppnar en anslutning utan att `nc` eller `telnet` behöver vara installerade:

```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/192.0.2.25/25' ; echo "exit=$?"
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `timeout 10` | Avbryter följande kommando efter 10 sekunder och returnerar sedan avslutskod 124 |
| `bash -c '…'` | Kör kommandokedjan i en bash; krävs eftersom `/dev/tcp` är en bash-funktion |
| `exec 3<>/dev/tcp/192.0.2.25/25` | Öppnar filbeskrivare 3 för läsning och skrivning som en TCP-anslutning till 192.0.2.25, port 25 |
| `echo "exit=$?"` | Skriver ut avslutskoden för föregående kommando |

</details>

Avslutskoden är själva informationen här:

| exit | Betydelse |
|---|---|
| `0` | Anslutningen är upprättad, porten är öppen |
| `124` | Timeout: paket kasseras, typiskt för en brandvägg med DROP-regel |
| `1` | Omedelbart avslag (RST) eller saknad route |

Skillnaden mellan 124 och 1 är i praktiken den viktigaste ledtråden. En timeout betyder att någon längs vägen tyst kasserar trafiken, och det är nästan alltid en brandväggsregel. Ett omedelbart RST kommer däremot från ett system som svarar men inte erbjuder tjänsten.

Kontrollera direkt båda relevanta portarna och dessutom ett valfritt annat mål för att se om hosten över huvud taget får upprätta utgående anslutningar:

```bash
for t in "192.0.2.25 25" "192.0.2.25 587" "1.1.1.1 443"; do
  set -- $t
  timeout 8 bash -c "exec 3<>/dev/tcp/$1/$2" 2>/dev/null
  echo "$1:$2 -> exit=$?"
done
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `set -- $t` | Delar värdeparet vid mellanslaget i positionsparametrarna `$1` (IP-adress) och `$2` (port) |
| `timeout 8` | Avbryter anslutningsförsöket efter 8 sekunder (avslutskod 124) |
| `bash -c "…"` | Kör anslutningsupprättandet via `/dev/tcp` i en bash |
| `2>/dev/null` | Undertrycker felmeddelanden så att exakt en resultatrad visas per mål |

</details>

Om även kontrolltestet misslyckas har systemet generellt ingen direkt utgående anslutning, och trafiken ska gå via ett internt relä eller en proxy. Mer om varför detta fall är särskilt lurigt längre ned.

Om `/dev/tcp` saknas är skalet inte bash. Under `sh`, `ash` eller `ksh` finns funktionen inte, vilket ofta misstolkas som ett nätverksproblem:

```bash
ps -p $$ -o comm= ; echo "BASH_VERSION=${BASH_VERSION:-keine bash}"
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-p $$` | Begränsar utdata till processen med den aktuella skalets PID (`$$`) |
| `-o comm=` | Skriver bara ut kommandonamnet; den tomma etiketten efter `=` undertrycker rubrikraden |
| `${BASH_VERSION:-keine bash}` | Skriver ut bash-versionen eller ersättningstexten om variabeln inte är satt |

</details>

## Steg 3: Lyssna först, skicka inte

En SMTP-server hälsar själv med en `220`-banner. Det mest informativa enskilda testet är därför att öppna en anslutning och inte göra något:

```bash
{ exec 3<>/dev/tcp/192.0.2.25/25 && timeout 15 cat <&3; echo "[ende exit=$?]"; }
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `exec 3<>/dev/tcp/192.0.2.25/25` | Öppnar filbeskrivare 3 som en TCP-anslutning till målet |
| `timeout 15 cat <&3` | Läser i 15 sekunder allt som servern skickar på eget initiativ och skriver ut det |
| `echo "[ende exit=$?]"` | Visar avslutskoden efteråt; 124 betyder att inget mer kom under 15 sekunder |

</details>

Dessa få tecken skiljer två helt olika situationer. Om en `220 mail.example.com ESMTP` kommer talar motparten och alla följande fel ligger i dialogen. Om inget kommer beror det inte på ett felaktigt formulerat kommando från din sida, eftersom du inte har skickat något.

Filbeskrivaren förblir sedan öppen i skalet. Stäng den innan du startar nästa test, annars kan du fortsätta arbeta med en gammal anslutning som inte längre är intakt:

```bash
exec 3<&- 3>&-
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `3<&-` | Stänger lässidan av filbeskrivare 3 |
| `3>&-` | Stänger skrivsidan av filbeskrivare 3 |

</details>

## Steg 4: SMTP-dialogen för hand

När bannern visas genomför du hela dialogen. Det är viktigt att en läsprocess körs parallellt, så att du ser varje svar när det kommer. Ett skript som först skickar allt och sedan läser visar ingenting om ett avbrott sker mitt i dialogen:

```bash
{
exec 3<>/dev/tcp/192.0.2.25/25
cat <&3 & R=$!
sleep 1; printf 'EHLO host.example.com\r\n' >&3
sleep 2; printf 'MAIL FROM:<absender@example.com>\r\n' >&3
sleep 2; printf 'RCPT TO:<empfaenger@example.net>\r\n' >&3
sleep 2; printf 'DATA\r\n' >&3
sleep 2; printf 'From: absender@example.com\r\nTo: empfaenger@example.net\r\nSubject: Relay-Test\r\n' >&3
printf 'Date: %s\r\nMessage-ID: <%s@example.com>\r\n\r\nTestnachricht\r\n.\r\n' "$(date -R)" "$(date +%s).$" >&3
sleep 3; printf 'QUIT\r\n' >&3
sleep 2; kill $R 2>/dev/null
}
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `exec 3<>/dev/tcp/192.0.2.25/25` | Öppnar filbeskrivare 3 som en TCP-anslutning till målet |
| `cat <&3 & R=$!` | Startar en bakgrundsläsare för filbeskrivare 3 och sparar dess PID i `R` |
| `printf '…\r\n' >&3` | Skickar ett SMTP-kommando med det obligatoriska CRLF-radslutet över anslutningen |
| `sleep n` | Väntar angivet antal sekunder på serversvaret innan nästa kommando följer |
| `date -R` | Ger datumet i RFC-kompatibelt format för `Date:`-huvudet |
| `date +%s` | Ger Unix-tiden som en enkel unik grund för Message-ID |
| `kill $R 2>/dev/null` | Avslutar bakgrundsläsaren; felmeddelandet uteblir om den redan har avslutats |

</details>

Två detaljer avgör om det lyckas eller misslyckas. SMTP kräver CRLF som radslut, därför `printf` med `\r\n` och inte `echo`. Och punkten på en egen rad avslutar meddelandedelen; den måste skickas som `\r\n.\r\n`.

Det förväntade förloppet: `220` vid anslutningsupprättandet, `250` på EHLO, `250 2.1.0` på MAIL FROM, `250 2.1.5` på RCPT TO, `354` på DATA och slutligen `250 2.0.0 Ok: queued as <id>`. Notera kö-ID:t. Med det kan meddelandet spåras hos den opererande leverantören om det aldrig kommer fram till mottagaren.

EHLO-namnet förtjänar uppmärksamhet: Vissa reläer kontrollerar det mot forward- och reverse-DNS och svarar annars med `501` eller `504`. Använd det sändande systemets faktiska FQDN, inte kortnamnet.

## Steg 5: STARTTLS och certifikat

För den krypterade anslutningen hanterar `openssl s_client` själv STARTTLS-förhandlingen och överlämnar sedan kanalen till standardinmatningen:

```bash
openssl s_client -connect 192.0.2.25:25 -starttls smtp -tls1_2 -brief </dev/null
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-connect 192.0.2.25:25` | Målhost och port för anslutningen |
| `-starttls smtp` | Genomför först SMTP-dialogen i klartext och växlar sedan till TLS via STARTTLS |
| `-tls1_2` | Förhandlar enbart TLS 1.2 |
| `-brief` | Begränsar utdata till en kort sammanfattning av den förhandlade anslutningen |
| `</dev/null` | Stänger standardinmatningen direkt så att `s_client` inte väntar interaktivt efter handskakningen |

</details>

Om du ansluter via IP-adressen eftersom DNS saknas fungerar inte värdnamnskontrollen. Certifikatnamnet matchar då inte den numeriska adressen. SNI och kontrollnamn kan ställas in uttryckligen, helt utan DNS-fråga:

```bash
openssl s_client -connect 192.0.2.25:25 \
  -servername mail.example.com -verify_hostname mail.example.com \
  -starttls smtp -tls1_2 -brief </dev/null
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-servername mail.example.com` | Ställer in SNI-namnet i ClientHello, oberoende av anslutningsadressen |
| `-verify_hostname mail.example.com` | Kontrollerar servercertifikatet mot detta namn i stället för mot den numeriska adressen |

</details>

Två felbilder återkommer regelbundet här och tolkas ofta fel.

**«Didn't find STARTTLS in server response, trying anyway»** betyder att servern inte erbjöd STARTTLS i sitt EHLO-svar. `openssl` skickar ändå ett TLS-ClientHello, servern ser ogiltiga protokolldata i det och anslutningen avslutas med `wrong version number` eller `write:errno=32` (EPIPE). Båda meddelandena är följdfel. Den egentliga informationen är: ingen STARTTLS. Kontrollera med klartextdialogen i steg 4 vilka funktioner servern faktiskt rapporterar.

**Ingen STARTTLS på ett internt hopp** är ofta helt korrekt. Om en lastbalanserare vidarebefordrar anslutningen på lager 4 förhandlar inte den TLS, utan systemet bakom den mot det egentliga målet. Att testa i klartext på det interna segmentet är då inte en brist i säkerhet, utan helt enkelt arkitekturen.

## Steg 6: Python som alternativ

Om Python finns tillgängligt slipper du den manuella tidsstyrningen med `sleep`. Standardbiblioteket räcker, inget behöver installeras i efterhand:

```python
#!/usr/bin/env python3
import smtplib, ssl
from email.message import EmailMessage
from email.utils import formatdate, make_msgid

msg = EmailMessage()
msg["From"] = "absender@example.com"
msg["To"] = "empfaenger@example.net"
msg["Subject"] = "Relay-Test"
msg["Date"] = formatdate(localtime=True)
msg["Message-ID"] = make_msgid(domain="example.com")
msg.set_content("Testnachricht\n")

ctx = ssl.create_default_context()
ctx.minimum_version = ssl.TLSVersion.TLSv1_2

s = smtplib.SMTP("192.0.2.25", 25, timeout=30, local_hostname="host.example.com")
s.set_debuglevel(1)
s.ehlo()
if s.has_extn("starttls"):
    s.starttls(context=ctx, server_hostname="mail.example.com")
    s.ehlo()
    print("TLS:", s.sock.version(), s.sock.cipher()[0])
s.send_message(msg)
s.quit()
```

`set_debuglevel(1)` loggar hela dialogen inklusive alla svarskoder, och `smtplib` läser varje svar synkront. Ett avbrott visas som `SMTPServerDisconnected` tillsammans med den senast mottagna raden, i stället för som en tyst Broken Pipe.

Två saker går ofta fel här: `server_hostname` är obligatoriskt vid anslutning via en IP-adress, annars kontrollerar Python certifikatet mot den numeriska adressen. Och om du avsiktligt stänger av kontrollen måste `check_hostname = False` stå före `verify_mode = ssl.CERT_NONE`, annars kastar Python ett `ValueError`.

## Avsändaradress, SPF och alignment

Ett test misslyckas förvånansvärt ofta inte i transporten, utan på grund av den valda avsändaradressen. Tre saker bör kontrolleras i förväg.

Avsändardomänen måste vara en FQDN. En adress som `test@meine-testdomain` utan toppdomän avvisas av många MTA:er redan vid MAIL FROM med `501` eller `553`.

Domänen måste auktorisera den använda sändvägen. En titt på SPF-posten visar om den utgående adressen täcks:

```bash
dig +short TXT example.com | grep spf1
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `+short` | Skriver endast ut postvärdena, utan rubriker och metadata |
| `TXT` | Efterfrågad posttyp |
| `example.com` | Efterfrågat namn |
| `grep spf1` | Filtrerar ut SPF-raden bland flera TXT-poster |

</details>

Och med aktiv DMARC är det alignment som avgör. Om posten innehåller `aspf=s`, måste domänen i envelopen (MAIL FROM) och domänen i `From:`-huvudet stämma exakt överens, inte bara vara relaterade:

```bash
dig +short TXT _dmarc.example.com
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `+short` | Skriver endast ut postvärdena, utan rubriker och metadata |
| `TXT _dmarc.example.com` | Posttyp och namnet under domänen som definierats för DMARC |

</details>

Vid `p=reject` försvinner ett testmejl med olämpligt alignment hos mottagaren utan kommentar, trots att ditt relä har accepterat det med `250 queued`. Det är den vanligaste orsaken till meddelanden som betraktas som framgångsrika på sändarsidan men ändå aldrig kommer fram.

## När en lastbalanserare står emellan

I större miljöer skickar en appliance sällan direkt till internet. Vanligt är en virtuell server på en lastbalanserare, som accepterar anslutningen, skriver om den till en definierad adress via Source-NAT och först därefter vidarebefordrar den externt. För diagnosen får detta en obehaglig konsekvens.

En virtuell server som arbetar på lager 4 bekräftar TCP-handskakningen omedelbart innan den själv har upprättat en anslutning till målet. Om denna andra anslutning misslyckas ser klienten en framgångsrikt upprättad och direkt därefter återställd anslutning: `Connection reset by peer`, utan någon SMTP-banner. Felet ligger då varken hos dig eller målet, utan i poolen bakom den virtuella servern, exempelvis för att en medlem har markerats som nere eller för att det konfigurerade FQDN:et inte kan lösas upp.

Det förklarar också varför ett test direkt mot internetmålet måste misslyckas om vidarebefordringsregeln bara accepterar trafik från den redan omskrivna SNAT-adressen. Anslutningar med den ursprungliga källadressen matchar ingen regel och kasseras. Testa alltid mot den avsedda virtuella servern i sådana miljöer, inte mot det egentliga målet.

Vilken källadress systemet använder för ett visst mål besvaras av en enda rad. Värdet efter `src` är exakt den uppgift nätverksteamet behöver för att öppna trafiken:

```bash
ip route get 192.0.2.25
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `route get` | Frågar kärnan vilken route den skulle välja för ett konkret mål |
| `192.0.2.25` | Måladress för den simulerade anslutningen |

</details>

Om systemet står bakom NAT ser motparten inte denna utan perimeterns publika adress. Den kan inte fastställas inifrån så länge ingen trafik kommer igenom; den står i NAT-regeln.

## Felbilder i korthet

| Observation | Trolig orsak |
|---|---|
| `Name or service not known` | Ingen namnupplösning på hosten |
| Timeout, exit 124 | Brandväggen kasserar tyst (DROP) |
| `Connection refused` | Ingen tjänst på porten eller REJECT-regel |
| Anslutningen finns, ingen banner, sedan RST | Lastbalanseraren accepterar, backend kan inte nås |
| `Didn't find STARTTLS` | Servern erbjuder ingen transportkryptering |
| `wrong version number`, `errno=32` | Följdfel efter framtvingad TLS utan STARTTLS |
| `501` / `553` vid MAIL FROM | Avsändardomänen är ingen FQDN eller inte tillåten |
| `554 relay access denied` | Käll-IP är inte godkänd hos reläet |
| `250 queued`, men ingen leverans | SPF-, DKIM- eller DMARC-alignment hos mottagaren |

## Belastningstester och rate limits

För volymtester gäller en regel som ofta förbises i vardagen: Det är inte antalet meddelanden som är problemet, utan antalet anslutningar. Typiska reläer tillåter några hundra anslutningar per minut, men tiotusentals meddelanden. Håll därför en session öppen och skicka många envelopes genom den, i stället för att ansluta på nytt för varje meddelande.

I `smtplib` innebär det helt enkelt att använda samma anslutningsobjekt flera gånger och kontrollerat bygga upp sessionen på nytt efter ett fast antal meddelanden. Den som i stället öppnar en ny anslutning per mejl överskrider anslutningsgränsen långt före meddelandegränsen och framkallar avslag som ser ut som ett problem hos motparten.

## Slutsats

Det manuella SMTP-testet är inte en nödlösning för miljöer utan verktyg, utan den mest precisa diagnos som finns tillgänglig i e-postdrift. Det skiljer tydligt namnupplösning, nåbarhet, protokolldialog och kryptering från varandra och ger ett entydigt resultat för varje nivå. Den som först bara lyssnar, sedan genomför dialogen för hand och tar svarskoderna på allvar kan på några minuter komma fram till ett underlag för ett ärende hos nätverks- eller leverantörsteamet: med källadress, målport, observerat beteende och avslutskod.

## Källor

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Definierar SMTP-dialogen, kommandoordningen och innebörden av svarskoderna.

2.  [RFC 3207: SMTP Service Extension for Secure SMTP over TLS](https://www.rfc-editor.org/rfc/rfc3207): Beskriver STARTTLS som en utökning, inklusive beteendet när servern inte erbjuder den.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Struktur och utvärdering av SPF-posten för auktorisering av sändande system.

4.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Reglerar alignment mellan envelope- och header-avsändare samt policyutvärderingen.

5.  [OpenSSL: s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Referens för de använda alternativen, bland annat `-starttls`, `-servername` och `-verify_hostname`.

6.  [Python-dokumentation: smtplib](https://docs.python.org/3/library/smtplib.html): Standardbibliotek för SMTP-sessioner inklusive STARTTLS och debug-utdata.

7.  [Bash Reference Manual: Redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections): Dokumenterar `/dev/tcp` som en bash-specifik pseudoenhet för nätverksanslutningar.
