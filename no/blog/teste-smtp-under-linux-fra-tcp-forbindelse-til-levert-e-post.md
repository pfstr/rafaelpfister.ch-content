---
title: "Teste SMTP under Linux: fra TCP-forbindelsen til levert e-post"
navTitle: "Teste SMTP"
description: "Når en appliance ikke lenger leverer e-post, hjelper en manuell SMTP-test mer enn enhver logg. Slik kontrollerer du lag for lag med innebygde verktøy, hva ulike feilbilder betyr, og hvorfor en lastbalanserer kan forvrenge feilsøkingen."
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
translationSourceHash: af2a802f67ec6d294b1507eaf26e25704b938e8760ac6751104ce7258cc2a4b3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:19:46.129Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/teste-smtp-under-linux-fra-tcp-forbindelse-til-levert-e-post
---

# Teste SMTP under Linux: fra TCP-forbindelsen til levert e-post

Når en e-postgateway plutselig ikke lenger leverer noe, viser appliance-loggene ofte bare sluttresultatet: en levering mislykkes, køen vokser, og en feilmelding oppgir en timeout. Hva årsaken faktisk er, viser først en manuell test fra kommandolinjen. SMTP er en klartekstprotokoll som kan kommuniseres helt manuelt, og nettopp det gjør den til et diagnoseverktøy som er tilgjengelig overalt uten ekstra installasjon.

Den andre grunnen til den manuelle testen: På appliances kan man som regel ikke installere noe. Ingen pakkebehandler, ingen root-rettigheter, ingen `swaks`. Alle følgende trinn fungerer derfor med det som uansett finnes på praktisk talt alle Linux-systemer.

## Skill lagene fra hverandre

En mislykket e-postsending kan feile på fem forskjellige nivåer, og hvert av dem gir et annet feilbilde:

1. **Navneoppløsning:** Målverten kan ikke oversettes til en IP-adresse.
2. **TCP-forbindelse:** Forbindelsen til porten opprettes ikke eller blir tilbakestilt.
3. **SMTP-dialog:** Forbindelsen er opprettet, men serveren avviser avsender, mottaker eller innhold.
4. **Transportkryptering:** STARTTLS mangler, sertifikatet er ugyldig eller TLS-versjonen passer ikke.
5. **Avsenderkontroll:** E-posten blir godtatt og forkastet hos mottakeren på grunn av SPF, DKIM eller DMARC.

Feilsøkingen blir vesentlig bedre når du kontrollerer disse nivåene etter hverandre og enkeltvis, i stedet for å sende en fullstendig test-e-post med én gang. Et mislykket totalforsøk forteller deg bare at noe ikke fungerer. Lagkontrollen forteller deg hva.

## Trinn 1: Navneoppløsning

```bash
getent hosts relay.example.com
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `hosts` | NSS-database som skal spørres; bruker de samme kildene og samme rekkefølge som systemet selv, i henhold til `nsswitch.conf` |
| `relay.example.com` | Vertsnavn som skal løses opp |

</details>

Hvis utdataene forblir tomme, er ingen navneserver tilgjengelig på denne verten, eller den svarer ikke på eksterne navn. Dette forekommer regelmessig i praksis: Appliances i isolerte soner får ofte bare en intern resolver som utelukkende kjenner egne soner.

```bash
cat /etc/resolv.conf; grep hosts: /etc/nsswitch.conf
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `/etc/resolv.conf` | Fil med de konfigurerte navneserverne, utgitt av `cat` |
| `hosts:` | Søk mønster for `grep`: linjen som angir rekkefølgen på oppløsningskildene (filer, DNS) |
| `/etc/nsswitch.conf` | Fil med NSS-konfigurasjonen, gjennomsøkt av `grep` |

</details>

Hvis oppløsningen mangler, test videre direkte mot IP-adressen. Det er fullt tilstrekkelig for feilsøkingen og skiller DNS-problemet rent fra transportproblemet. I produksjon er den manglende oppløsningen naturligvis et separat funn som må rettes.

## Trinn 2: Portens tilgjengelighet

For en ren TCP-kontroll er bash tilstrekkelig. Pseudoenheten `/dev/tcp` åpner en forbindelse uten at `nc` eller `telnet` må være installert:

```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/192.0.2.25/25' ; echo "exit=$?"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `timeout 10` | Avbryter den påfølgende kommandoen etter 10 sekunder og returnerer deretter avslutningskode 124 |
| `bash -c '…'` | Kjører kommandokjeden i en bash; nødvendig fordi `/dev/tcp` er en bash-funksjon |
| `exec 3<>/dev/tcp/192.0.2.25/25` | Åpner filbeskrivelse 3 for lesing og skriving som TCP-forbindelse til 192.0.2.25, port 25 |
| `echo "exit=$?"` | Skriver ut avslutningskoden for den foregående kommandoen |

</details>

Avslutningskoden er den egentlige informasjonen her:

| exit | Betydning |
|---|---|
| `0` | Forbindelsen er opprettet, porten er åpen |
| `124` | Timeout: Pakker forkastes, typisk for en brannmur med DROP-regel |
| `1` | Umiddelbar avvisning (RST) eller manglende rute |

Forskjellen mellom 124 og 1 er i praksis det viktigste tegnet av alle. En timeout betyr at noen forkaster trafikken stille på veien, og det er nesten alltid en brannmurregel. Et umiddelbart RST kommer derimot fra et system som svarer, men ikke tilbyr tjenesten.

Kontroller begge relevante porter med én gang, og i tillegg et vilkårlig annet mål for å se om verten i det hele tatt har tillatelse til å opprette utgående forbindelser:

```bash
for t in "192.0.2.25 25" "192.0.2.25 587" "1.1.1.1 443"; do
  set -- $t
  timeout 8 bash -c "exec 3<>/dev/tcp/$1/$2" 2>/dev/null
  echo "$1:$2 -> exit=$?"
done
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `set -- $t` | Deler verdiparet ved mellomrommet i posisjonsparameterne `$1` (IP-adresse) og `$2` (port) |
| `timeout 8` | Avbryter forbindelsesforsøket etter 8 sekunder (avslutningskode 124) |
| `bash -c "…"` | Kjører forbindelsesopprettingen med `/dev/tcp` i en bash |
| `2>/dev/null` | Undertrykker feilmeldinger slik at nøyaktig én resultatlinje vises per mål |

</details>

Hvis også kontrolltesten feiler, har systemet generelt ingen direkte utgående tilgang, og trafikken må gå via et internt relé eller en proxy. Mer om hvorfor dette tilfellet er særlig vanskelig, lenger ned.

Hvis `/dev/tcp` mangler, er skallet ikke bash. Under `sh`, `ash` eller `ksh` finnes ikke funksjonen, noe som ofte feiltolkes som et tilsynelatende nettverksproblem:

```bash
ps -p $$ -o comm= ; echo "BASH_VERSION=${BASH_VERSION:-keine bash}"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-p $$` | Begrenses utdataene til prosessen med PID-en til gjeldende skall (`$$`) |
| `-o comm=` | Skriver bare ut kommandonavnet; den tomme etiketten etter `=` undertrykker overskriftslinjen |
| `${BASH_VERSION:-keine bash}` | Skriver ut bash-versjonen eller erstatningsteksten dersom variabelen ikke er satt |

</details>

## Trinn 3: Lytt først, ikke send

En SMTP-server hilser på eget initiativ med et `220`-banner. Den mest informative enkelttesten består derfor i å åpne en forbindelse og ikke gjøre noe:

```bash
{ exec 3<>/dev/tcp/192.0.2.25/25 && timeout 15 cat <&3; echo "[ende exit=$?]"; }
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `exec 3<>/dev/tcp/192.0.2.25/25` | Åpner filbeskrivelse 3 som TCP-forbindelse til målet |
| `timeout 15 cat <&3` | Leser i 15 sekunder alt serveren sender på eget initiativ, og skriver det ut |
| `echo "[ende exit=$?]"` | Viser avslutningskoden etter utløp; 124 betyr: Ingenting kom på 15 sekunder |

</details>

Disse få tegnene skiller to helt forskjellige situasjoner. Hvis det kommer et `220 mail.example.com ESMTP`, snakker motparten og alle videre feil ligger i dialogen. Hvis ingenting kommer, skyldes det ikke en feilformulert kommando fra din side, for du har jo ikke sendt noen.

Filbeskrivelsen forblir deretter åpen i skallet. Lukk den før du starter neste test, ellers kan du ende opp med å fortsette på en gammel forbindelse som ikke lenger er intakt:

```bash
exec 3<&- 3>&-
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `3<&-` | Lukker lesesiden til filbeskrivelse 3 |
| `3>&-` | Lukker skrivesiden til filbeskrivelse 3 |

</details>

## Trinn 4: SMTP-dialogen for hånd

Når banneret er på plass, gjennomfører du hele dialogen. Det er viktig å ha en leserprosess som kjører samtidig, slik at du ser hvert svar i det øyeblikket det kommer. Et skript som først sender alt og deretter leser, viser deg ingenting dersom dialogen avbrytes midtveis:

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
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `exec 3<>/dev/tcp/192.0.2.25/25` | Åpner filbeskrivelse 3 som TCP-forbindelse til målet |
| `cat <&3 & R=$!` | Starter en bakgrunnsleser for filbeskrivelse 3 og lagrer PID-en i `R` |
| `printf '…\r\n' >&3` | Sender en SMTP-kommando med den påkrevde CRLF-linjeavslutningen over forbindelsen |
| `sleep n` | Venter angitt antall sekunder på serversvaret før neste kommando følger |
| `date -R` | Leverer datoen i RFC-kompatibelt format for `Date:`-headeren |
| `date +%s` | Leverer Unix-tiden som et enkelt entydig grunnlag for Message-ID |
| `kill $R 2>/dev/null` | Avslutter bakgrunnsleseren; feilmeldingen utelates hvis den allerede er avsluttet |

</details>

To detaljer avgjør om det lykkes eller mislykkes. SMTP krever CRLF som linjeavslutning, derfor `printf` med `\r\n` og ikke `echo`. Og punktum på en egen linje avslutter meldingsdelen; det må sendes som `\r\n.\r\n`.

Forventet forløp: `220` ved tilkobling, `250` på EHLO, `250 2.1.0` på MAIL FROM, `250 2.1.5` på RCPT TO, `354` på DATA og til slutt `250 2.0.0 Ok: queued as <id>`. Noter deg kø-ID-en. Den kan brukes til å spore meldingen hos leverandøren som drifter tjenesten dersom den aldri kommer frem hos mottakeren.

EHLO-navnet fortjener oppmerksomhet: Enkelte reléer kontrollerer det mot forover- og revers-DNS og svarer ellers med `501` eller `504`. Bruk den faktiske FQDN-en til systemet som sender, ikke kortnavnet.

## Trinn 5: STARTTLS og sertifikat

For den krypterte forbindelsen håndterer `openssl s_client` STARTTLS-forhandlingen selv og overleverer deretter kanalen til standardinndata:

```bash
openssl s_client -connect 192.0.2.25:25 -starttls smtp -tls1_2 -brief </dev/null
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-connect 192.0.2.25:25` | Målvert og port for forbindelsen |
| `-starttls smtp` | Gjennomfører først SMTP-dialogen i klartekst og bytter deretter til TLS via STARTTLS |
| `-tls1_2` | Forhandler utelukkende TLS 1.2 |
| `-brief` | Reduserer utdataene til en kort oppsummering av den fremforhandlede forbindelsen |
| `</dev/null` | Lukker standardinndata umiddelbart slik at `s_client` ikke venter interaktivt etter handshaken |

</details>

Hvis du kobler til via IP-adressen fordi DNS mangler, fungerer ikke vertsnavnkontrollen. Sertifikatnavnet passer da ikke med den numeriske adressen. SNI og kontrollnavn kan angis eksplisitt, helt uten DNS-oppslag:

```bash
openssl s_client -connect 192.0.2.25:25 \
  -servername mail.example.com -verify_hostname mail.example.com \
  -starttls smtp -tls1_2 -brief </dev/null
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-servername mail.example.com` | Angir SNI-navnet i ClientHello, uavhengig av forbindelsesadressen |
| `-verify_hostname mail.example.com` | Kontrollerer serversertifikatet mot dette navnet i stedet for den numeriske adressen |

</details>

To feilbilder forekommer regelmessig her og tolkes ofte feil.

**«Didn't find STARTTLS in server response, trying anyway»** betyr at serveren ikke tilbød STARTTLS i sitt EHLO-svar. `openssl` sender likevel et TLS-ClientHello, serveren oppfatter dette som ugyldige protokolldata og forbindelsen avsluttes med `wrong version number` eller `write:errno=32` (EPIPE). Begge meldingene er følgefeil. Den egentlige informasjonen er: ingen STARTTLS. Se med klartekstdialogen fra trinn 4 hvilke funksjoner serveren faktisk rapporterer.

**Ingen STARTTLS på et internt hopp** er ofte helt korrekt. Hvis en lastbalanserer videresender forbindelsen på lag 4, forhandler ikke den TLS, men systemet bak den gjør det først mot det faktiske målet. Å teste i klartekst på det interne segmentet er da ikke en sikkerhetsmangel, men rett og slett arkitekturen.

## Trinn 6: Python som alternativ

Hvis Python finnes, slipper du den manuelle tidsstyringen med `sleep`. Standardbiblioteket er tilstrekkelig, ingenting må installeres i tillegg:

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

`set_debuglevel(1)` logger hele dialogen, inkludert alle svarkoder, og `smtplib` leser hvert svar synkront. Et avbrudd vises som `SMTPServerDisconnected` sammen med den sist mottatte linjen, i stedet for som en stille Broken Pipe.

To ting går ofte galt her: `server_hostname` er obligatorisk ved tilkobling via en IP-adresse, ellers kontrollerer Python sertifikatet mot den numeriske adressen. Og dersom du bevisst deaktiverer kontrollen, må `check_hostname = False` stå før `verify_mode = ssl.CERT_NONE`, ellers kaster Python en `ValueError`.

## Avsenderadresse, SPF og alignment

En test feiler overraskende ofte ikke i transporten, men på grunn av den valgte avsenderadressen. Tre punkter bør kontrolleres på forhånd.

Avsenderdomenet må være en FQDN. En adresse som `test@meine-testdomain` uten toppnivådomene avvises av mange MTA-er allerede ved MAIL FROM med `501` eller `553`.

Domenet må autorisere den aktuelle sendingsveien. En titt i SPF-posten viser om den utgående adressen er dekket:

```bash
dig +short TXT example.com | grep spf1
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `+short` | Skriver bare ut postverdiene, uten headere og metadata |
| `TXT` | Posttypen som spørres etter |
| `example.com` | Navnet som spørres etter |
| `grep spf1` | Filtrerer ut SPF-linjen blant flere TXT-poster |

</details>

Og med aktiv DMARC er det alignment som avgjør. Hvis posten inneholder `aspf=s`, må domenet i konvolutten (MAIL FROM) og domenet i `From:`-headeren være nøyaktig like, ikke bare beslektet:

```bash
dig +short TXT _dmarc.example.com
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `+short` | Skriver bare ut postverdiene, uten headere og metadata |
| `TXT _dmarc.example.com` | Posttype og navnet under domenet som er definert for DMARC |

</details>

Ved `p=reject` forsvinner en test-e-post med feil alignment lydløst hos mottakeren, selv om reléet ditt godtok den med `250 queued`. Dette er den vanligste årsaken til meldinger som anses som vellykkede på sendersiden, men likevel aldri kommer frem.

## Når en lastbalanserer står imellom

I større miljøer sender en appliance sjelden direkte til internett. Vanlig er en virtuell server på en lastbalanserer, som aksepterer forbindelsen, omskriver den til en definert adresse via source NAT og først deretter videresender den utad. Dette har en ubehagelig konsekvens for feilsøkingen.

En virtuell server som opererer på lag 4, kvitterer TCP-handshaken umiddelbart før den selv har opprettet en forbindelse til målet. Hvis denne andre forbindelsen feiler, ser du på klienten en vellykket opprettet og umiddelbart deretter tilbakestilt forbindelse: `Connection reset by peer`, uten noe SMTP-banner. Feilen ligger da ikke hos deg og heller ikke hos målet, men i poolen bak den virtuelle serveren, for eksempel fordi et medlem er markert som nede eller den konfigurerte FQDN-en ikke blir løst opp.

Dette forklarer også hvorfor en test direkte mot internettdestinasjonen må feile dersom videresendingsregelen bare godtar trafikk fra den allerede omskrevne SNAT-adressen. Forbindelser med den opprinnelige kildeadressen passer ikke med noen regel og forkastes. I slike miljøer skal du alltid teste mot den tiltenkte virtuelle serveren, ikke mot det faktiske målet.

Hvilken kildeadresse systemet ditt bruker for et bestemt mål, kan besvares med én enkelt linje. Verdien etter `src` er nøyaktig opplysningen nettverksteamet trenger for å åpne opp:

```bash
ip route get 192.0.2.25
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `route get` | Spør kjernen hvilken rute den ville valgt for et konkret mål |
| `192.0.2.25` | Måladresse for den simulerte forbindelsen |

</details>

Hvis systemet står bak NAT, ser motparten ikke denne, men den offentlige adressen til perimeteret. Den kan ikke fastslås innenfra så lenge ingen trafikk slipper gjennom; den står i NAT-regelen.

## Feilbilder på et øyeblikk

| Observasjon | Sannsynlig årsak |
|---|---|
| `Name or service not known` | Ingen navneoppløsning på verten |
| Timeout, exit 124 | Brannmur forkaster stille (DROP) |
| `Connection refused` | Ingen tjeneste på porten eller REJECT-regel |
| Forbindelsen er opprettet, ikke noe banner, deretter RST | Lastbalanserer godtar, backend er ikke tilgjengelig |
| `Didn't find STARTTLS` | Serveren tilbyr ingen transportkryptering |
| `wrong version number`, `errno=32` | Følgefeil etter tvungen TLS uten STARTTLS |
| `501` / `553` på MAIL FROM | Avsenderdomenet er ikke en FQDN eller er ikke tillatt |
| `554 relay access denied` | Kilde-IP-en er ikke åpnet for reléet |
| `250 queued`, men ingen levering | SPF, DKIM eller DMARC-alignment hos mottakeren |

## Lasttester og rate limits

For volumtester gjelder en regel som ofte overses i hverdagen: Det er ikke antallet meldinger som er problemet, men antallet forbindelser. Typiske reléer tillater noen hundre forbindelser per minutt, men titusenvis av meldinger. Hold derfor én økt åpen og send mange konvolutter gjennom den, i stedet for å opprette en ny forbindelse for hver melding.

I `smtplib` betyr dette ganske enkelt å bruke samme forbindelsesobjekt flere ganger og kontrollert bygge opp økten på nytt etter et fast antall meldinger. Den som derimot åpner en ny forbindelse per e-post, overskrider forbindelsesgrensen lenge før meldingsgrensen og fremprovoserer avvisninger som ser ut som et problem hos motparten.

## Konklusjon

Den manuelle SMTP-testen er ikke en nødløsning for miljøer uten verktøy, men den mest presise feilsøkingen som er tilgjengelig i e-postdrift. Den skiller navneoppløsning, tilgjengelighet, protokolldialog og kryptering klart fra hverandre og gir et entydig resultat for hvert nivå. Den som først bare lytter, deretter utfører dialogen for hånd og tar svarkodene på alvor, kommer i løpet av få minutter frem til en konklusjon som kan dokumentere en sak overfor nettverks- eller leverandørteamet: med kildeadresse, målport, observert atferd og avslutningskode.

## Kilder

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Definerer SMTP-dialogen, kommandorekkefølgen og betydningen av svarkodene.

2.  [RFC 3207: SMTP Service Extension for Secure SMTP over TLS](https://www.rfc-editor.org/rfc/rfc3207): Beskriver STARTTLS som en utvidelse, inkludert atferden når serveren ikke tilbyr den.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Oppbygning og evaluering av SPF-posten for autorisering av sendende systemer.

4.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Regulerer alignment mellom avsenderen i konvolutten og headeren, samt policy-evalueringen.

5.  [OpenSSL: s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Referanse for alternativene som brukes, blant annet `-starttls`, `-servername` og `-verify_hostname`.

6.  [Python-dokumentasjon: smtplib](https://docs.python.org/3/library/smtplib.html): Standardbibliotek for SMTP-økter, inkludert STARTTLS og feilsøkingsutskrift.

7.  [Bash Reference Manual: Redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections): Dokumenterer `/dev/tcp` som en bash-spesifikk pseudoenhet for nettverksforbindelser.
