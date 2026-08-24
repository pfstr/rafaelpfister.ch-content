---
title: "Testa SMTP under Linux: från TCP-anslutning till levererat e-postmeddelande"
navTitle: "Testa SMTP"
description: "När en appliance inte längre levererar e-post hjälper ett manuellt SMTP-test mer än någon logg. Så kontrollerar du lager för lager med inbyggda verktyg, vad olika felbilder betyder och varför en lastbalanserare kan förvanska diagnosen."
date: "2026-07-31"
kategorie: "SMTP och e-postflöde"
timeToRead: "10 min. läsning"
themen:
  - smtp-mailflow
  - testing
  - e-mail-verschluesselung
slug: "testa-smtp-under-linux-fran-tcp-anslutning-till-levererat-e-postmeddelande"
translationId: "article-cb44a92c03a47bc0"
translationOf: smtp-verbindung-testen-linux
url: https://rafaelpfister.ch/sv/blog/testa-smtp-under-linux-fran-tcp-anslutning-till-levererat-e-postmeddelande
translationSourceHash: 5c8e1b19b8002fc6dc109c5471afbe91dba9302274cef0b63eebd40e01a98fe2
translationModel: gpt-5.6-terra
translatedAt: 2026-08-01T06:14:14.629Z
translationReview: automatic
---

# Testa SMTP under Linux: från TCP-anslutning till levererat e-postmeddelande

När en e-postgateway plötsligt inte längre levererar något visar appliance-loggarna ofta bara slutet på historien: en leverans misslyckas, kön växer, ett felmeddelande nämner en timeout. Vad som faktiskt orsakar det visar först ett manuellt test från kommandoraden. SMTP är ett klartextprotokoll som går att kommunicera helt manuellt, och just det gör det till ett av de mest tacksamma diagnosverktygen i e-postdrift.

Det andra skälet till det manuella testet: På appliances går det oftast inte att installera något. Ingen pakethanterare, inga root-rättigheter, inget `swaks`. Alla följande steg fungerar därför med det som redan finns på praktiskt taget varje Linux-system.

## Separera lagren

Ett misslyckat e-postutskick kan fallera på fem olika nivåer, och varje nivå ger en annan felbild:

1. **Namnupplösning:** Målhosten kan inte översättas till en IP-adress.
2. **TCP-anslutning:** Anslutningen till porten upprättas inte eller återställs.
3. **SMTP-dialog:** Anslutningen är upprättad, men servern avvisar avsändare, mottagare eller innehåll.
4. **Transportkryptering:** STARTTLS saknas, certifikatet är ogiltigt eller TLS-versionen passar inte.
5. **Avsändarkontroll:** E-postmeddelandet accepteras men förkastas hos mottagaren på grund av SPF, DKIM eller DMARC.

Diagnosen blir avsevärt bättre när du kontrollerar dessa nivåer efter varandra och var för sig, i stället för att genast skicka ett komplett testmejl. Ett misslyckat helhetsförsök säger bara att något inte fungerar. Lagerkontrollen visar vad.

## Steg 1: Namnupplösning

```bash
getent hosts relay.example.com
```

Om utdata förblir tom finns ingen nåbar namnserver på denna host, eller så svarar den inte på externa namn. Det är vanligare än man tror: Appliances i avskilda zoner får ofta bara en intern resolver som enbart känner till egna zoner.

```bash
cat /etc/resolv.conf; grep hosts: /etc/nsswitch.conf
```

Om upplösningen saknas testar du följande direkt mot IP-adressen. Det räcker helt för diagnosen och skiljer tydligt DNS-problemet från transportproblemet. I produktionsdrift är den saknade upplösningen naturligtvis ett eget fynd som måste åtgärdas.

## Steg 2: Portens nåbarhet

För en ren TCP-kontroll räcker bash. Pseudoenheten `/dev/tcp` öppnar en anslutning utan att `nc` eller `telnet` behöver vara installerade:

```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/192.0.2.25/25' ; echo "exit=$?"
```

Exit-koden är den faktiska informationen här:

| exit | Betydelse |
|---|---|
| `0` | Anslutningen är upprättad, porten är öppen |
| `124` | Timeout: paket förkastas, typiskt för en brandvägg med DROP-regel |
| `1` | Omedelbar avvisning (RST) eller saknad route |

Skillnaden mellan 124 och 1 är i praktiken den allra viktigaste ledtråden. En timeout innebär att någon på vägen förkastar tyst, och det är nästan alltid en brandväggsregel. Ett omedelbart RST kommer däremot från ett system som svarar men inte erbjuder tjänsten.

Kontrollera genast båda relevanta portarna och dessutom ett valfritt annat mål för att se om hosten över huvud taget får upprätta utgående anslutningar:

```bash
for t in "192.0.2.25 25" "192.0.2.25 587" "1.1.1.1 443"; do
  set -- $t
  timeout 8 bash -c "exec 3<>/dev/tcp/$1/$2" 2>/dev/null
  echo "$1:$2 -> exit=$?"
done
```

Om även motkontrollen inte ger något resultat saknar systemet generellt direkt utgående trafik och trafiken ska gå via ett internt relay eller en proxy. Mer om varför detta fall är särskilt knepigt längre ned.

Om `/dev/tcp` saknas är skalet inte bash. Under `sh`, `ash` eller `ksh` finns funktionen inte, vilket ofta feltolkas som ett nätverksproblem:

```bash
ps -p $$ -o comm= ; echo "BASH_VERSION=${BASH_VERSION:-keine bash}"
```

## Steg 3: Lyssna först, skicka inte

En SMTP-server hälsar själv med en `220`-banner. Det mest informativa enskilda testet är därför att öppna en anslutning och inte göra något:

```bash
{ exec 3<>/dev/tcp/192.0.2.25/25 && timeout 15 cat <&3; echo "[ende exit=$?]"; }
```

Dessa få tecken skiljer två helt olika situationer åt. Om ett `220 mail.example.com ESMTP` kommer talar motparten och alla ytterligare fel ligger i dialogen. Om inget kommer beror det inte på ett felaktigt formulerat kommando från din sida, eftersom du ju inte har skickat något.

Fildeskriptorn förblir därefter öppen i skalet. Stäng den innan du startar nästa test, annars kan du råka fortsätta arbeta med en gammal, halvdöd anslutning:

```bash
exec 3<&- 3>&-
```

## Steg 4: SMTP-dialogen manuellt

När bannern visas genomför du hela dialogen. Det är viktigt att en läsprocess kör parallellt så att du ser varje svar när det kommer. Ett skript som först skickar allt och sedan läser visar ingenting vid ett avbrott mitt i dialogen:

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

Två detaljer avgör om det blir framgång eller frustration. SMTP kräver CRLF som radslut, därför `printf` med `\r\n` och inte `echo`. Punkten på en egen rad avslutar meddelandedelen; den måste skickas som `\r\n.\r\n`.

Det förväntade förloppet: `220` vid anslutningen, `250` på EHLO, `250 2.1.0` på MAIL FROM, `250 2.1.5` på RCPT TO, `354` på DATA och slutligen `250 2.0.0 Ok: queued as <id>`. Anteckna kö-ID:t. Med det går det att spåra meddelandet hos den ansvariga leverantören om det aldrig kommer fram till mottagaren.

EHLO-namnet förtjänar uppmärksamhet: Vissa relayer kontrollerar det mot forward- och reverse-DNS och svarar annars med `501` eller `504`. Använd det sändande systemets faktiska FQDN, inte kortnamnet.

## Steg 5: STARTTLS och certifikat

För den krypterade anslutningen sköter `openssl s_client` STARTTLS-förhandlingen själv och lämnar sedan över kanalen till standardindata:

```bash
openssl s_client -connect 192.0.2.25:25 -starttls smtp -tls1_2 -brief </dev/null
```

Om du ansluter via IP-adressen eftersom DNS saknas blir värdnamnskontrollen verkningslös. Certifikatnamnet matchar då inte den numeriska adressen. SNI och kontrollnamn kan anges uttryckligen, helt utan DNS-uppslagning:

```bash
openssl s_client -connect 192.0.2.25:25 \
  -servername mail.example.com -verify_hostname mail.example.com \
  -starttls smtp -tls1_2 -brief </dev/null
```

Två felbilder uppstår regelbundet här och tolkas ofta fel.

**«Didn't find STARTTLS in server response, trying anyway»** betyder att servern inte erbjöd STARTTLS i sitt EHLO-svar. `openssl` skickar ändå ett TLS-ClientHello, servern ser protokollskräp i det och anslutningen avslutas med `wrong version number` eller `write:errno=32` (EPIPE). Båda meddelandena är följdfel. Den egentliga informationen är: ingen STARTTLS. Kontrollera med klartextdialogen från steg 4 vilka capabilities servern faktiskt rapporterar.

**Ingen STARTTLS på ett internt hopp** är ofta helt korrekt. Om en lastbalanserare vidarebefordrar anslutningen på lager 4 förhandlar den inte själv om TLS, utan först systemet bakom den mot det egentliga målet. Att testa i klartext på det interna segmentet är då inte en säkerhetsbrist, utan helt enkelt arkitekturen.

## Steg 6: Python som alternativ

Om Python finns tillgängligt slipper du tidsstyrningskrånglet med `sleep`. Standardbiblioteket räcker, inget behöver installeras i efterhand:

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

Två fallgropar: `server_hostname` är obligatoriskt vid anslutning via en IP-adress, annars kontrollerar Python certifikatet mot den numeriska adressen. Och om du medvetet stänger av kontrollen måste `check_hostname = False` stå före `verify_mode = ssl.CERT_NONE`, annars ger Python ett `ValueError`.

## Avsändaradress, SPF och alignment

Ett test misslyckas förvånansvärt ofta inte på transporten, utan på den valda avsändaradressen. Tre punkter bör kontrolleras i förväg.

Avsändardomänen måste vara ett FQDN. En adress som `test@meine-testdomain` utan toppdomän avvisas av många MTA:er redan vid MAIL FROM med `501` eller `553`.

Domänen måste auktorisera den använda sändvägen. En titt på SPF-posten visar om den utgående adressen omfattas:

```bash
dig +short TXT example.com | grep spf1
```

Och med aktiv DMARC är det alignment som avgör. Om posten innehåller `aspf=s`, måste domänen i envelopen (MAIL FROM) och domänen i `From:`-headern stämma exakt överens, inte bara vara besläktade:

```bash
dig +short TXT _dmarc.example.com
```

Vid `p=reject` försvinner ett testmejl med olämpligt alignment hos mottagaren utan kommentar, trots att ditt relay har accepterat det med `250 queued`. Det är den vanligaste orsaken till meddelanden som anses framgångsrika på sändarsidan men ändå aldrig kommer fram.

## När en lastbalanserare står emellan

I större miljöer skickar en appliance sällan direkt ut på internet. Vanligt är en virtuell server på en lastbalanserare som tar emot anslutningen, skriver om den till en definierad adress via Source-NAT och först därefter vidarebefordrar den utåt. Det får en obehaglig konsekvens för diagnosen.

En virtuell server som arbetar på lager 4 bekräftar TCP-handshaken omedelbart, innan den själv har upprättat en anslutning till målet. Om denna andra anslutning misslyckas ser du på klienten en framgångsrikt upprättad och omedelbart därefter återställd anslutning: `Connection reset by peer`, utan någon SMTP-banner. Felet ligger då varken hos dig eller vid målet, utan i poolen bakom den virtuella servern, till exempel eftersom en medlem är markerad som down eller det konfigurerade FQDN:et inte kan lösas upp.

Det förklarar också varför ett test direkt mot internetmålet måste misslyckas om vidarebefordringsregeln bara accepterar trafik från den redan omskrivna SNAT-adressen. Anslutningar med den ursprungliga källadressen matchar ingen regel och förkastas. Testa alltid mot den avsedda virtuella servern i sådana miljöer, inte mot det egentliga målet.

Vilken källadress ditt system använder för ett visst mål besvaras av en enda rad. Värdet efter `src` är exakt den uppgift som nätverksteamet behöver för att öppna åtkomst:

```bash
ip route get 192.0.2.25
```

Om systemet står bakom NAT ser motparten inte denna adress utan den publika adressen vid perimetern. Den går inte att fastställa inifrån så länge ingen trafik kommer igenom; den står i NAT-regeln.

## Felbilder i korthet

| Observation | Trolig orsak |
|---|---|
| `Name or service not known` | Ingen namnupplösning på hosten |
| Timeout, exit 124 | Brandvägg förkastar tyst (DROP) |
| `Connection refused` | Ingen tjänst på porten eller REJECT-regel |
| Anslutningen är upprättad, ingen banner, sedan RST | Lastbalanseraren accepterar, backend är inte nåbar |
| `Didn't find STARTTLS` | Servern erbjuder ingen transportkryptering |
| `wrong version number`, `errno=32` | Följdfel efter tvingad TLS utan STARTTLS |
| `501` / `553` på MAIL FROM | Avsändardomänen är inte ett FQDN eller inte tillåten |
| `554 relay access denied` | Käll-IP är inte godkänd på relayet |
| `250 queued`, men ingen leverans | SPF-, DKIM- eller DMARC-alignment hos mottagaren |

## Belastningstester och rate limits

För volymtester gäller en regel som ofta förbises i vardagen: Det är inte antalet meddelanden som är problemet, utan antalet anslutningar. Typiska relayer tillåter några hundra anslutningar per minut, men tiotusentals meddelanden. Håll därför en session öppen och skicka många envelopes genom den, i stället för att ansluta på nytt för varje meddelande.

I `smtplib` innebär det helt enkelt att använda samma anslutningsobjekt flera gånger och kontrollerat bygga upp sessionen på nytt efter ett fast antal meddelanden. Den som i stället öppnar en ny anslutning per e-postmeddelande når anslutningsgränsen långt före meddelandegränsen och framkallar avvisningar som ser ut som ett problem hos motparten.

## Slutsats

Det manuella SMTP-testet är inte en nödlösning för miljöer utan verktyg, utan den mest precisa diagnos som finns tillgänglig i e-postdrift. Det skiljer tydligt namnupplösning, nåbarhet, protokolldialog och kryptering från varandra och ger ett entydigt resultat för varje nivå. Den som först bara lyssnar, sedan genomför dialogen manuellt och tar svarskoderna på allvar får på några minuter ett underlag för ett ärende till nätverks- eller leverantörsteamet: med källadress, målport, observerat beteende och exit-kod.

## Källor

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Definierar SMTP-dialogen, kommandoordningen och innebörden av svarskoderna.

2.  [RFC 3207: SMTP Service Extension for Secure SMTP over TLS](https://www.rfc-editor.org/rfc/rfc3207): Beskriver STARTTLS som en utökning, inklusive beteendet när servern inte erbjuder den.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Struktur och utvärdering av SPF-posten för auktorisering av sändande system.

4.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Reglerar alignment mellan envelope- och header-avsändare samt policyutvärdering.

5.  [OpenSSL: s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Referens för de använda alternativen, bland annat `-starttls`, `-servername` och `-verify_hostname`.

6.  [Python-dokumentation: smtplib](https://docs.python.org/3/library/smtplib.html): Standardbibliotek för SMTP-sessioner inklusive STARTTLS och debug-utdata.

7.  [Bash Reference Manual: Redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections): Dokumenterar `/dev/tcp` som en bash-specifik pseudoenhet för nätverksanslutningar.
