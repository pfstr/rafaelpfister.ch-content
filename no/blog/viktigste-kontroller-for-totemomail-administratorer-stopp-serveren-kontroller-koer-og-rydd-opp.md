---
title: "Viktigste kontroller for TotemoMail-administratorer: stopp serveren, kontroller køer og rydd opp kontrollert"
navTitle: "TotemoMail-kontroller"
description: "De viktigste kontrollene for drift av en TotemoMail-gateway: stopp tjenesten via systemd og Tanuki-kontrollskriptet, tell køinnhold per repository, inspiser enkeltmeldinger, rydd opp kontrollert og start tjenesten igjen."
date: "2026-08-28"
kategorie: "TotemoMail"
timeToRead: "9 min lesetid"
themen:
  - totemomail
produkte:
  - "totemomail"
  - "apache-james"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "viktigste-kontroller-for-totemomail-administratorer-stopp-serveren-kontroller-koer-og-rydd-opp"
translationId: "article-3a0a526ab6e38a06"
translationOf: totemomail-server-stoppen-queues-bereinigen
url: https://rafaelpfister.ch/no/blog/viktigste-kontroller-for-totemomail-administratorer-stopp-serveren-kontroller-koer-og-rydd-opp
translationSourceHash: bc887dcd4aa82db7e020247f75b86528f0fa331e1643c28a215a1638587197a6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:42:19.821Z
translationReview: automatic
---

# Viktigste kontroller for TotemoMail-administratorer: stopp serveren, kontroller køer og rydd opp kontrollert

Ved drift av en TotemoMail-gateway (i dag Kiteworks Email Protection Gateway) inngår fire arbeidssteg i grunnverktøyet: å stoppe tjenesten på en ryddig måte, kartlegge køinnholdet, inspisere enkeltmeldinger og rydde opp i køene på en kontrollert måte før tjenesten startes igjen.

Disse stegene trengs både ved planlagt vedlikehold og ved driftsforstyrrelser, for eksempel når en feilaktig regel, et utilgjengelig mål eller en belastningstest har fylt køene. Denne artikkelen viser hvert steg med de konkrete kommandoene, inkludert spørsmålet om hvordan tjenesten egentlig stoppes på en ryddig måte. Behandlingsmodellen bak dette (processors, repositories, filformater) er beskrevet i artikkelen [Forstå e-postruting mellom TotemoMail og Exchange Online](/blog/totemomail-m365) beskrevet.

Alle stier gjelder for en installasjon under `/opt/totemomail` med tjenestebrukeren `totemo`. Tilpass stiene til ditt miljø.

## Hvordan TotemoMail startes og stoppes

Før du stopper en tjeneste, bør du vite hvordan den kjører. For TotemoMail er tre lag involvert:

- En **systemd-unit** `totemomail.service` som ytterste kontrollnivå.
- **Kontrollskriptet** `/opt/totemomail/bin/totemomail`, som kaller opp unit-en ved start og stopp.
- **Tanuki Java Service Wrapper**: en innebygd `wrapper`-prosess som starter og overvåker den egentlige Java-prosessen, og kan starte den på nytt ved krasj.

Du kan undersøke oppsettet på systemet ditt uten å ha lesetilgang til unit-filen. `systemctl show` spør egenskapene direkte fra systemd og fungerer også når filen under `/etc/systemd/system/` bare er lesbar for root:

```bash
systemctl show totemomail.service -p Type -p User -p ExecStart -p ExecStop \
  -p KillMode -p TimeoutStopUSec --no-pager
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `show totemomail.service` | Viser kjøretidsegenskapene til unit-en slik systemd har lastet dem |
| `-p <Property>` | Begrenses utdata til den angitte egenskapen; kan angis flere ganger |
| `--no-pager` | Skriver direkte til konsollen i stedet for å åpne en pager som `less` |

</details>

En typisk utdata ser slik ut:

```text
Type=oneshot
TimeoutStopUSec=1min 30s
ExecStart={ path=/opt/totemomail/bin/totemomail ; argv[]=/opt/totemomail/bin/totemomail start ; ... }
ExecStop={ path=/opt/totemomail/bin/totemomail ; argv[]=/opt/totemomail/bin/totemomail stop ; ... }
User=totemo
KillMode=control-group
```

Herfra kan du lese de viktige egenskapene: `systemctl stop totemomail` kaller kontrollskriptet med argumentet `stop`, venter opptil 90 sekunder på en ryddig avslutning og terminerer deretter alle gjenværende prosesser i unit-en via `KillMode=control-group`. Stopp via systemd er dermed likestilt med direkte kall av skriptet, men rydder i tillegg opp dersom skriptet henger.

Statusen `active (exited)` i `systemctl status totemomail` er normal i dette oppsettet og ingen feil: Unit-en er `Type=oneshot`, startskriptet avsluttes etter start, og wrapperen fortsetter å kjøre som en selvstendig daemon som systemd bare administrerer indirekte. Derfor er det ikke unit-statusen, men prosesslisten som viser om tjenesten faktisk kjører:

```bash
ps -ef | grep -E 'wrapper|TotemoBootStrapper' | grep -v grep
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-e` | Viser alle prosesser, ikke bare dem i egen sesjon |
| `-f` | Fullt utdataformat med komplett kommandolinje |
| `grep -E 'wrapper\|TotemoBootStrapper'` | Filtrerer på wrapper-prosessen og Java-hovedklassen |
| `grep -v grep` | Fjerner selve grep-prosessene fra trefflisten |

</details>

Ved normal drift vises to prosesser: den innebygde `wrapper` (startet med `../conf/wrapper.conf` og PID-filen `totemomail.pid`) og Java-prosessen med hovedklassen `ch.totemo.bootstrapper.TotemoBootStrapper`. Mangler én av dem, er tjenesten ikke fullstendig startet.

## Steg 1: Stopp tjenesten

Stopp tjenesten først før du arbeider med køene. Så lenge TotemoMail kjører, tar den imot meldinger, behandler køene og leverer; først stoppet fryser innholdet for analyse.

```bash
sudo systemctl stop totemomail
```

Kontroller deretter at wrapper- og Java-prosessen er avsluttet:

```bash
ps -ef | grep -E 'wrapper|TotemoBootStrapper' | grep -v grep
```

Utdataene må være tomme. I tillegg forsvinner PID-filen `/opt/totemomail/bin/totemomail.pid`. Hvis en prosess fortsatt kjører etter at stopptimeouten er utløpt, terminerer systemd den via control group; kontroller i så fall `journalctl -u totemomail` før du fortsetter.

Ikke glem det foranstilte nivået: Under stoppen hoper nye innkommende meldinger seg opp i systemet som leverer dem, for eksempel i Exchange-køen eller ved et foranstilt relay. Dette er ønsket. Seriøse avsendere leverer automatisk på nytt etter omstart.

## Steg 2: Kartlegg køinnholdet

Køene i TotemoMail er filbaserte e-post-repositories fra underliggende Apache James. De ligger under James-applikasjonskatalogen, her `/opt/totemomail/mailer/apps/james/var/mail/`. Hver underkatalog er et repository, og hver melding består av to filer: `*.FileStreamStore` inneholder den komplette MIME-meldingen, `*.FileObjectStore` det serialiserte statusobjektet med metadata.

En oversikt over innholdet får du ved å telle `FileObjectStore`-filer per katalog:

```bash
for d in /opt/totemomail/mailer/apps/james/var/mail/*/; do \
  printf '%-22s %s\n' "$(basename "$d")" \
  "$(find "$d" -maxdepth 1 -name '*.FileObjectStore' | wc -l)"; \
done
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `for d in .../*/` | Itererer over alle repository-kataloger (den avsluttende `/` begrenser til kataloger) |
| `printf '%-22s %s\n'` | Formaterer utdata i to kolonner; `%-22s` fyller ut navnet venstrejustert til 22 tegn |
| `basename "$d"` | Reduserer den fullstendige stien til katalognavnet |
| `find "$d" -maxdepth 1` | Søker bare direkte i katalogen, uten underkataloger |
| `-name '*.FileObjectStore'` | Teller én fil per melding; stream-motstykket ville doblet tallet |
| `wc -l` | Teller de funne filene |

</details>

Resultatet er én linje per kø med antall meldinger, for eksempel:

```text
DBUnavailable          0
error                  12
incoming               121
outgoing               0
spool                  0
```

Standard-repositories betyr: `spool` inneholder mottatte, ennå ubehandlede meldinger, `incoming` meldinger som skal leveres internt, `outgoing` utgående meldinger, `error` mislykkede meldinger og `DBUnavailable` meldinger som er parkert fordi en backend ikke er tilgjengelig. Avhengig av konfigurasjonen finnes det flere repositories for spesielle ruter; de følger samme filskjema.

Hvis `find` kjøres fra en katalog som tjenestebrukeren ikke har tilgang til (for eksempel hjemmekatalogen til en annen bruker etter `sudo -u totemo`), vises advarselen `Failed to restore initial working directory` per kall. Den er ufarlig og forsvinner etter en `cd ~`.

## Steg 3: Se på meldingene

Tall alene er ikke nok for å ta en beslutning. Før du sletter noe, bør du vite hva som ligger i køene: uønskede meldinger fra en driftsforstyrrelse, eller legitime e-poster som skal leveres etter omstarten?

`FileStreamStore`-filene er uendrede RFC-822-meldinger. De viktigste headerne kan derfor leses direkte:

```bash
for f in /opt/totemomail/mailer/apps/james/var/mail/incoming/*.FileStreamStore; do \
  awk 'BEGIN{IGNORECASE=1} /^(From|To|Subject|Date):/{print} /^\r?$/{exit}' "$f"; \
  echo ---; \
done | less
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `BEGIN{IGNORECASE=1}` | Sammenligner headernavn uten hensyn til store og små bokstaver (GNU awk) |
| `/^(From\|To\|Subject\|Date):/{print}` | Skriver bare ut de fire relevante headerlinjene |
| `/^\r?$/{exit}` | Stopper ved tomlinjen mellom header og brødtekst; meldingsinnholdet leses ikke |
| `echo ---` | Skillelinje mellom meldingene |
| `less` | Bla i stedet for å rulle ved mange meldinger |

</details>

Ved store mengder er fordelingen mer informativ enn enkeltvisning. De hyppigste avsenderne viser:

```bash
grep -him1 '^From:' /opt/totemomail/mailer/apps/james/var/mail/incoming/*.FileStreamStore \
  | sort | uniq -c | sort -rn | head
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-h` | Undertrykker filnavnene i utdata slik at identiske avsendere samles |
| `-i` | Ignorerer store og små bokstaver |
| `-m1` | Bare første treff per fil (headeren, ikke siterte `From:`-linjer i brødteksten) |
| `sort \| uniq -c` | Grupperer identiske avsenderlinjer og teller dem |
| `sort -rn \| head` | Sorterer synkende etter hyppighet og viser de ti hyppigste |

</details>

Hvis én enkelt avsender eller ett enkelt emne dominerer med hundrevis av kopier, tyder det på en sløyfe eller en feiladressert masseutsending; disse meldingene er kandidater for opprydding. Et blikk på filtidstemplene (`ls -lt`) avgrenser i tillegg tidsrommet og viser om eldre, legitime meldinger ligger imellom.

## Steg 4: Rydd opp kontrollert

Først nå slettes det, og også nå med et mellomsteg: Flytt først innholdet til en sikkerhetskatalog i stedet for å slette det direkte. Resultatet for e-postdriften er det samme (køen er tom), men steget kan reverseres, og enkeltvise legitime meldinger kan senere legges tilbake fra sikkerhetskopien eller brukes videre som `.eml`.

```bash
mkdir -p /opt/totemomail/queue-backup-$(date +%F)
mv /opt/totemomail/mailer/apps/james/var/mail/incoming/* \
   /opt/totemomail/queue-backup-$(date +%F)/
```

Viktig: Selve repository-katalogene blir stående, bare innholdet flyttes. Og stream- og object-filen til en melding hører sammen; den som fjerner bare én av dem, etterlater foreldreløse filer som skaper feil i loggen ved neste start.

Når sikkerhetskopien er kontrollert eller innholdet uten tvil er verdiløst (for eksempel rene belastningstestmeldinger), slett hele køinnholdet på tvers av alle repositories:

```bash
find /opt/totemomail/mailer/apps/james/var/mail/ -mindepth 2 -maxdepth 2 -type f \
  \( -name '*.FileStreamStore' -o -name '*.FileObjectStore' \) -delete
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-mindepth 2 -maxdepth 2` | Treffer bare filer direkte i repository-katalogene, ikke `var/mail` selv og ingen dypere nivåer |
| `-type f` | Bare vanlige filer; katalogene beholdes |
| `\( -name ... -o -name ... \)` | Begge filtypene til en melding, stream og statusobjekt |
| `-delete` | Sletter treffene direkte; kjør først uten dette alternativet for å kontrollere trefflisten |

</details>

Kjør deretter samme telling som i steg 2: Alle repositories må vise 0.

## Steg 5: Start tjenesten igjen

```bash
sudo systemctl start totemomail
```

Starten kaller kontrollskriptet med `start`, som daemoniserer wrapperen; wrapperen starter deretter Java-prosessen. Kontroller begge via prosesslisten fra første avsnitt og se i loggfilene under `/opt/totemomail/bin/`: `wrapper.log` logger starten av wrapperen og JVM-en, `console.log` og `console.err` logger utdataene fra selve applikasjonen.

Avslutt med en funksjonstest med én enkelt testmelding gjennom gatewayen før den ordinære e-postflyten åpnes igjen. Og dersom en regel eller en e-postsløyfe hadde fylt køene: Rett først årsaken, og tillat deretter trafikken igjen. Ellers begynner kartleggingen av køinnholdet på nytt.

## Sammendrag

| Steg | Kommando | Kontroll |
|---|---|---|
| Stopp | `sudo systemctl stop totemomail` | `ps`-filter tomt, PID-fil borte |
| Tell innhold | `find`-løkke over `var/mail/*/` | Antall per repository |
| Inspiser | `awk`-headerutdrag, `grep`-avsenderstatistikk | Skill uønskede fra legitime meldinger |
| Rydd opp | `mv` til sikkerhetskopi, deretter `find ... -delete` | Tellingen viser 0 overalt |
| Start | `sudo systemctl start totemomail` | Prosesser, `wrapper.log`, testmelding |

## Kilder

1.  [Apache James Server 2: Provided Mailets](https://james.apache.org/server/2/provided_mailets.html): Dokumentasjon av mailets og repositories som køstrukturen i TotemoMail bygger på.

2.  [Tanuki Software: Java Service Wrapper](https://wrapper.tanukisoftware.com/doc/english/introduction.html): Virkemåten til wrapperen som starter og overvåker TotemoMail-Java-prosessen, inkludert PID-fil og `wrapper.conf`.

3.  [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html): Betydningen av `Type=oneshot`, `ExecStop` og `TimeoutStopSec` for units som kaller et eksternt kontrollskript.

4.  [systemd.kill(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.kill.html): `KillMode=control-group` som sikring som terminerer gjenværende prosesser i unit-en etter stoppskriptet.

5.  [RFC 5322: Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322): Oppbyggingen av meldingsheaderne som leses ut ved inspeksjon av `FileStreamStore`-filene.
