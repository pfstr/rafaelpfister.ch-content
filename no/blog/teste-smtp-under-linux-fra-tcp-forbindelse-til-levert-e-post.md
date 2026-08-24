---
title: "Teste SMTP under Linux: fra TCP-forbindelse til levert e-post"
navTitle: "Teste SMTP"
description: "Når en appliance ikke lenger leverer e-post, hjelper en manuell SMTP-test mer enn noen logg. Slik kontrollerer du lag for lag med innebygde verktøy, hva de ulike feilmønstrene betyr, og hvorfor en lastbalanserer kan forvrenge diagnosen."
date: "2026-07-31"
kategorie: "SMTP og e-postflyt"
timeToRead: "10 min lesetid"
themen:
  - smtp-mailflow
  - testing
  - e-mail-verschluesselung
slug: "teste-smtp-under-linux-fra-tcp-forbindelse-til-levert-e-post"
translationId: "article-cb44a92c03a47bc0"
translationOf: smtp-verbindung-testen-linux
url: https://rafaelpfister.ch/no/blog/teste-smtp-under-linux-fra-tcp-forbindelse-til-levert-e-post
translationSourceHash: 5c8e1b19b8002fc6dc109c5471afbe91dba9302274cef0b63eebd40e01a98fe2
translationModel: gpt-5.6-terra
translatedAt: 2026-08-01T06:14:53.224Z
translationReview: automatic
---

# Teste SMTP under Linux: fra TCP-forbindelse til levert e-post

Når en e-postgateway plutselig ikke lenger leverer noe, viser appliance-loggene ofte bare siste ledd i historien: en levering feiler, køen vokser, og en feilmelding angir en tidsavbrudd. Hva som faktisk er årsaken, viser først en manuell test fra kommandolinjen. SMTP er en klartekstprotokoll som kan snakkes helt manuelt, og nettopp derfor er den et av de mest takknemlige diagnoseverktøyene i e-postdrift.

Den andre grunnen til den manuelle testen: På appliances kan man som regel ikke installere noe. Ingen pakkebehandler, ingen root-rettigheter, ingen `swaks`. Alle følgende trinn fungerer derfor med det som allerede finnes på praktisk talt alle Linux-systemer.

## Skill lagene fra hverandre

En mislykket e-postsending kan feile på fem ulike nivåer, og hvert av dem gir et annet feilmønster:

1. **Navneoppløsning:** Målverten kan ikke oversettes til en IP-adresse.
2. **TCP-forbindelse:** Forbindelsen til porten opprettes ikke eller blir tilbakestilt.
3. **SMTP-dialog:** Forbindelsen er opprettet, men serveren avviser avsender, mottaker eller innhold.
4. **Transportkryptering:** STARTTLS mangler, sertifikatet er ugyldig eller TLS-versjonen passer ikke.
5. **Avsenderkontroll:** E-posten aksepteres og forkastes hos mottakeren på grunn av SPF, DKIM eller DMARC.

Diagnosen blir betydelig bedre når du kontrollerer disse nivåene etter tur og hver for seg, i stedet for å sende en fullstendig test-e-post med én gang. Et mislykket totalforsøk forteller deg bare at noe ikke fungerer. Lagkontrollen forteller deg hva.

## Trinn 1: Navneoppløsning

```bash
getent hosts relay.example.com
```

Hvis utdataene forblir tomme, er ingen navneserver tilgjengelig fra denne verten, eller den svarer ikke på eksterne navn. Dette er vanligere enn man skulle tro: Appliances i isolerte soner får ofte bare en intern resolver som utelukkende kjenner egne soner.

```bash
cat /etc/resolv.conf; grep hosts: /etc/nsswitch.conf
```

Hvis oppløsningen mangler, tester du videre direkte mot IP-adressen. Det er fullt tilstrekkelig for diagnostikken og skiller DNS-problemet rent fra transportproblemet. I produksjonsdrift er manglende oppløsning selvsagt fortsatt et eget funn som må rettes.

## Trinn 2: Portens tilgjengelighet

For ren TCP-kontroll er bash tilstrekkelig. Pseudoenheten `/dev/tcp` åpner en forbindelse uten at `nc` eller `telnet` må være installert:

```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/192.0.2.25/25' ; echo "exit=$?"
```

Exit-koden er den egentlige informasjonen her:

| exit | Betydning |
|---|---|
| `0` | Forbindelsen er opprettet, porten er åpen |
| `124` | Tidsavbrudd: Pakker forkastes, typisk for en brannmur med DROP-regel |
| `1` | Umiddelbar avvisning (RST) eller manglende rute |

Forskjellen mellom 124 og 1 er i praksis det viktigste sporet av alle. Et tidsavbrudd betyr at noen på veien forkaster trafikken uten å svare, og det er nesten alltid en brannmurregel. En umiddelbar RST kommer derimot fra et system som svarer, men ikke tilbyr tjenesten.

Kontroller begge relevante porter med én gang, og i tillegg et valgfritt annet mål for å se om verten i det hele tatt har lov til å opprette utgående forbindelser:

```bash
for t in "192.0.2.25 25" "192.0.2.25 587" "1.1.1.1 443"; do
  set -- $t
  timeout 8 bash -c "exec 3<>/dev/tcp/$1/$2" 2>/dev/null
  echo "$1:$2 -> exit=$?"
done
```

Hvis også motprøven ikke gir noe resultat, har systemet generelt ingen direkte utgående tilgang, og trafikken må gå via et internt relay eller en proxy. Lenger nede forklares hvorfor dette tilfellet er særlig vanskelig.

Hvis `/dev/tcp` mangler, er ikke skallet bash. I `sh`, `ash` eller `ksh` finnes ikke funksjonen, noe som ofte feiltolkes som et nettverksproblem:

```bash
ps -p $$ -o comm= ; echo "BASH_VERSION=${BASH_VERSION:-keine bash}"
```

## Trinn 3: Lytt først, ikke send

En SMTP-server hilser av seg selv med et `220`-banner. Den mest informative enkeltstående testen er derfor å åpne en forbindelse og ikke gjøre noe:

```bash
{ exec 3<>/dev/tcp/192.0.2.25/25 && timeout 15 cat <&3; echo "[ende exit=$?]"; }
```

Disse få tegnene skiller to helt ulike situasjoner. Hvis et `220 mail.example.com ESMTP` kommer, snakker motparten og alle videre feil ligger i dialogen. Hvis ingenting kommer, skyldes det ikke en feilformulert kommando fra din side, for du har jo ikke sendt noen.

Fildeskriptoren forblir deretter åpen i skallet. Lukk den før du starter neste test, ellers kan du ende opp med å arbeide videre med en gammel, halvdød forbindelse:

```bash
exec 3<&- 3>&-
```

## Trinn 4: SMTP-dialogen for hånd

Når banneret vises, gjennomfører du hele dialogen. Det er viktig med en samtidig leseprosess, slik at du ser hvert svar idet det kommer. Et skript som først sender alt og deretter leser, viser deg ingenting dersom forbindelsen brytes midt i dialogen:

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

To detaljer avgjør om det blir suksess eller frustrasjon. SMTP krever CRLF som linjeslutt, derfor `printf` med `\r\n` og ikke `echo`. Og punktumet på en egen linje avslutter meldingsdelen; det må sendes som `\r\n.\r\n`.

Forventet forløp: `220` ved opprettelse av forbindelsen, `250` på EHLO, `250 2.1.0` på MAIL FROM, `250 2.1.5` på RCPT TO, `354` på DATA og til slutt `250 2.0.0 Ok: queued as <id>`. Noter kø-ID-en. Med den kan meldingen spores hos den driftsansvarlige leverandøren dersom den aldri kommer frem hos mottakeren.

EHLO-navnet fortjener oppmerksomhet: Enkelte relayer kontrollerer det mot Forward- og Reverse-DNS og svarer ellers med `501` eller `504`. Bruk den faktiske FQDN-en til det sendende systemet, ikke kortnavnet.

## Trinn 5: STARTTLS og sertifikat

For den krypterte forbindelsen utfører `openssl s_client` selv STARTTLS-forhandlingen og overfører deretter kanalen til standardinndata:

```bash
openssl s_client -connect 192.0.2.25:25 -starttls smtp -tls1_2 -brief </dev/null
```

Hvis du kobler til via IP-adressen fordi DNS mangler, blir vertsnavnkontrollen meningsløs. Sertifikatnavnet passer da ikke til den numeriske adressen. SNI og kontrollnavn kan settes eksplisitt, helt uten DNS-oppslag:

```bash
openssl s_client -connect 192.0.2.25:25 \
  -servername mail.example.com -verify_hostname mail.example.com \
  -starttls smtp -tls1_2 -brief </dev/null
```

To feilmønstre dukker jevnlig opp her og blir ofte feiltolket.

**«Didn't find STARTTLS in server response, trying anyway»** betyr at serveren ikke tilbød STARTTLS i sitt EHLO-svar. `openssl` sender likevel et TLS-ClientHello, serveren oppfatter dette som protokollsøppel, og forbindelsen avsluttes med `wrong version number` eller `write:errno=32` (EPIPE). Begge meldingene er følgefeil. Den egentlige informasjonen er: ingen STARTTLS. Se i klartekstdialogen fra trinn 4 hvilke egenskaper serveren faktisk melder.

**Ingen STARTTLS på et internt hopp** er ofte helt korrekt. Hvis en lastbalanserer videresender forbindelsen på lag 4, forhandler ikke den TLS, men systemet bak den gjør det først mot det faktiske målet. Å teste i klartekst på det interne segmentet er da ikke en sikkerhetsmangel, men ganske enkelt arkitekturen.

## Trinn 6: Python som alternativ

Hvis Python finnes, slipper du tidsstyringsarbeidet med `sleep`. Standardbiblioteket er tilstrekkelig; ingenting må installeres i tillegg:

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

`set_debuglevel(1)` logger hele dialogen, inkludert alle svarkoder, og `smtplib` leser hvert svar synkront. Et avbrudd vises som `SMTPServerDisconnected` sammen med den siste mottatte linjen, i stedet for som en stille Broken Pipe.

To fallgruver: `server_hostname` er obligatorisk når du kobler til via en IP-adresse, ellers kontrollerer Python sertifikatet mot den numeriske adressen. Og hvis du bevisst slår av kontrollen, må `check_hostname = False` stå før `verify_mode = ssl.CERT_NONE`, ellers kaster Python en `ValueError`.

## Avsenderadresse, SPF og alignment

En test feiler overraskende ofte ikke i transporten, men på grunn av valgt avsenderadresse. Tre punkter bør kontrolleres på forhånd.

Avsenderdomenet må være en FQDN. En adresse som `test@meine-testdomain` uten toppnivådomene avvises av mange MTA-er allerede ved MAIL FROM med `501` eller `553`.

Domenet må autorisere den benyttede sendingsveien. Et blikk på SPF-posten viser om den utgående adressen er dekket:

```bash
dig +short TXT example.com | grep spf1
```

Og med aktiv DMARC avgjør alignment. Hvis posten inneholder `aspf=s`, må domenet i konvolutten (MAIL FROM) og domenet i `From:`-headeren være nøyaktig identiske, ikke bare beslektede:

```bash
dig +short TXT _dmarc.example.com
```

Ved `p=reject` forsvinner en test-e-post med upassende alignment hos mottakeren uten kommentar, selv om relayet ditt har akseptert den med `250 queued`. Dette er den vanligste årsaken til meldinger som anses som vellykket sendt på avsendersiden, men som likevel aldri kommer frem.

## Når en lastbalanserer står imellom

I større miljøer sender en appliance sjelden direkte til Internett. Vanlig er en virtuell server på en lastbalanserer som tar imot forbindelsen, omskriver den til en definert adresse via Source-NAT og først deretter videresender den utad. Dette har en ubehagelig konsekvens for diagnostikken.

En virtuell server som arbeider på lag 4, kvitterer TCP-handshaken umiddelbart før den selv har opprettet en forbindelse til målet. Hvis denne andre forbindelsen feiler, ser klienten en vellykket opprettet og umiddelbart deretter tilbakestilt forbindelse: `Connection reset by peer`, uten noe SMTP-banner. Feilen ligger da ikke hos deg eller målet, men i poolen bak den virtuelle serveren, for eksempel fordi et medlem er markert som down eller den lagrede FQDN-en ikke kan slås opp.

Dette forklarer også hvorfor en test direkte mot Internettmålet må feile dersom videresendingsregelen bare aksepterer trafikk fra den allerede omskrevne SNAT-adressen. Forbindelser med den opprinnelige kildeadressen passer ikke med noen regel og forkastes. I slike miljøer må du alltid teste mot den tiltenkte virtuelle serveren, ikke mot det faktiske målet.

Hvilken kildeadresse systemet ditt bruker for et bestemt mål, besvares med én enkelt linje. Verdien etter `src` er nøyaktig opplysningen nettverksteamet trenger for å åpne opp:

```bash
ip route get 192.0.2.25
```

Hvis systemet står bak NAT, ser motparten ikke denne, men perimeterens offentlige adresse. Den kan ikke fastslås innenfra så lenge ingen trafikk slipper gjennom; den står i NAT-regelen.

## Feilmønstre i korte trekk

| Observasjon | Sannsynlig årsak |
|---|---|
| `Name or service not known` | Ingen navneoppløsning på verten |
| Tidsavbrudd, exit 124 | Brannmur forkaster uten svar (DROP) |
| `Connection refused` | Ingen tjeneste på porten eller REJECT-regel |
| Forbindelsen er opprettet, intet banner, deretter RST | Lastbalanserer aksepterer, backend er ikke tilgjengelig |
| `Didn't find STARTTLS` | Serveren tilbyr ingen transportkryptering |
| `wrong version number`, `errno=32` | Følgefeil etter tvunget TLS uten STARTTLS |
| `501` / `553` på MAIL FROM | Avsenderdomene er ikke en FQDN eller ikke tillatt |
| `554 relay access denied` | Kilde-IP er ikke godkjent hos relayet |
| `250 queued`, men ingen levering | SPF-, DKIM- eller DMARC-alignment hos mottakeren |

## Lasttester og rate limits

For volumtester gjelder en regel som ofte overses i hverdagen: Det er ikke antallet meldinger som er problemet, men antallet forbindelser. Typiske relayer tillater noen hundre forbindelser per minutt, men titusenvis av meldinger. Hold derfor en sesjon åpen og send mange konvolutter gjennom den, i stedet for å opprette en ny forbindelse for hver melding.

I `smtplib` betyr dette ganske enkelt å bruke det samme forbindelsesobjektet flere ganger og kontrollert opprette sesjonen på nytt etter et fast antall meldinger. Den som i stedet åpner en ny forbindelse per e-post, når forbindelsesgrensen lenge før meldingsgrensen og fremprovoserer avvisninger som ser ut som et problem hos motparten.

## Konklusjon

Den manuelle SMTP-testen er ikke en nødløsning for miljøer uten verktøy, men den mest presise diagnosen som er tilgjengelig i e-postdrift. Den skiller navneoppløsning, tilgjengelighet, protokolldialog og kryptering rent fra hverandre og gir et entydig resultat for hvert nivå. Den som først bare lytter, deretter fører dialogen manuelt og tar svarkodene på alvor, kan i løpet av få minutter gi en uttalelse som dokumenterer en sak overfor nettverks- eller leverandørteamet: med kildeadresse, målport, observert atferd og exit-kode.

## Kilder

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Definerer SMTP-dialogen, kommandorekkefølgen og betydningen av svarkodene.

2.  [RFC 3207: SMTP Service Extension for Secure SMTP over TLS](https://www.rfc-editor.org/rfc/rfc3207): Beskriver STARTTLS som utvidelse, inkludert virkemåten når serveren ikke tilbyr den.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Struktur og evaluering av SPF-posten for autorisering av sendende systemer.

4.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Regulerer alignment mellom konvolutt- og headeravsender samt evaluering av policyen.

5.  [OpenSSL: s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Referanse for de benyttede alternativene, blant annet `-starttls`, `-servername` og `-verify_hostname`.

6.  [Python-dokumentasjon: smtplib](https://docs.python.org/3/library/smtplib.html): Standardbibliotek for SMTP-sesjoner, inkludert STARTTLS og feilsøkingsutdata.

7.  [Bash Reference Manual: Redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections): Dokumenterer `/dev/tcp` som en bash-spesifikk pseudoenhet for nettverksforbindelser.
