---
title: "Hva vi kan lære av naturvitenskapen for feilsøking i IT"
navTitle: "Kontrollerte eksperimenter"
description: "Falsifiserbarhet, kontrollgruppe, forstyrrende variabler og utvalgsskjevhet: Metoden naturvitenskapene har brukt i århundrer, løser nettopp problemene IT-feilsøking regelmessig mislykkes med, illustrert med eksempler fra e-postflyt."
date: "2026-08-11"
kategorie: "SMTP / e-postflyt"
timeToRead: "15 min lesetid"
themen:
  - smtp-mailflow
  - testing
  - exchange-onprem-hybrid
hauptthema: "smtp-mailflow"
produkte:
  - "uebergreifend"
  - "exchange"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - einliefernde-ip-adressen-aggregieren
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
slug: "hva-vi-kan-laere-av-naturvitenskapen-for-feilsoking-i-it"
translationId: "article-098ed40e6d027b8b"
draft: false
translationOf: mailflow-fehlersuche-kontrollierte-experimente
translationSourceHash: e3fff70bc1386c28d78713ec89a35b4d6c29b7f16e809e8a84bd9850a40a261c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:18:18.027Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/hva-vi-kan-laere-av-naturvitenskapen-for-feilsoking-i-it
---

# Hva vi kan lære av naturvitenskapen for feilsøking i IT

En melding kommer ikke frem. Protokollen gir en feilmelding som umiddelbart antyder en forklaring. Du undersøker denne forklaringen, finner bevis, og etter to timer viser det seg at forklaringen var feil og bevisene tilfeldige.

Dette er ikke en nybegynnerfeil, men regelen. Og det er bemerkelsesverdig at vår bransje sjelden har en metode for dette problemet, selv om én har eksistert i århundrer og fungerer svært godt. Naturvitenskapene har nøyaktig den samme oppgaven: å slutte fra observasjoner til årsaker, i systemer man ikke har full oversikt over.

Denne artikkelen overfører fem prinsipper fra den vitenskapelige metoden til feilsøking i e-postflyt. Eksemplene kommer fra praksis, men fremgangsmåten er ikke spesifikk for e-post.

## Hvorfor IT-feilsøking systematisk er sårbar

E-postflyten er en kjede av systemer som hver har sitt eget perspektiv på den samme meldingen: gatewayen, filterlaget, den lokale transportserveren, skytjenesten, målpostboksen. Hver melding er skrevet fra perspektivet til nøyaktig ett lag.

I tillegg er feilmeldinger samlebetegnelser. Den samme ordlyden beskriver ofte helt ulike situasjoner, fordi systemet som avviser bare kjenner et grovt skjema. De utvidede statuskodene er nettopp laget for å danne klasser, ikke for å navngi enkelttilfeller.

Et eksempel: En skytjeneste avviste en melding med beskjed om at avsenderen ikke var tillatt for utgående levering. Den samme ordlyden opptrådte i samme miljø i to grunnleggende ulike konstellasjoner. Den ene gangen forsøkte et system å levere via tjenesten til en ekstern mottaker, altså et reelt videresendingsforsøk utad. Den andre gangen var mottakeren en ordinær postboks i tjenesten, og bare avsenderdomenet ble påklaget.

Den som tar teksten bokstavelig, leter etter det samme i begge tilfellene. Og fordi ordet «utgående» forekommer i den, leter man først i feil ende.

## Prinsipp 1: En hypotese må utelukke noe

Karl Popper beriket vitenskapsteorien med en innsikt som er direkte praktisk for feilsøking: **En påstand er bare nyttig dersom den kan motbevises.** En forklaring som passer til ethvert tenkelig observasjonsresultat, forklarer ingenting.

Overført betyr det: Formuler antakelsen din slik at den inneholder en **forutsigelse** som kan være feil. Ikke «det er noe galt med avsenderdomenet», men «hvis jeg sender den samme meldingen med et annet avsenderdomene via samme vei, kommer den frem».

Den andre formuleringen har verdi fordi den kan motbevises på fem minutter. Den første kan du bruke timevis på å fore med bevis uten å bli noe klokere.

En god test: Spør deg selv før forsøket hvilket resultat som ville **motbevise** hypotesen din. Hvis du ikke kommer på noe, har du ingen hypotese, men en stemning.

## Prinsipp 2: Én variabel, ellers alt likt

Kjernen i eksperimentet er kontroll av forstyrrende variabler. I praksis skjer regelmessig det motsatte: Man sammenligner to tilfeller som tilfeldigvis foreligger. Og de skiller seg nesten alltid på flere egenskaper samtidig.

Fra et reelt tilfelle: Meldinger fra `example-test.com` ble avvist, mens meldinger fra `partner.example` kom frem. De to domenene skilte seg på minst fire egenskaper: tilhørighet til organisasjonen, hvor e-posten er driftet, om en streng autentiseringspolicy er lagret, og innleveringsveien. Fra to datapunkter med fire forskjeller kan man ikke slutte nøyaktig noe. Alle de fire forklaringene passer.

Bygg derfor sammenligningen selv. Samme innleveringspunkt, samme mottaker, samme vei, samme tidspunkt og **nøyaktig én** endret egenskap. Hvis du mistenker avsenderdomenet, endrer du bare dette.

## Prinsipp 3: Uten kontrollforsøk er resultatet verdiløst

Dette er delen man helst utelater, og den viktigste. I klinisk forskning er kontrollgruppen en selvfølge; i IT avstår man som regel fra den og undrer seg over motstridende resultater.

**Testoppsettet ditt må først reprodusere feilen.** Hvis du ikke kan frembringe feiltilfellet med dine egne midler, sier et vellykket motforsøk ingenting. Kanskje testmeldingen din bare fungerer fordi du leverer inn et annet sted enn det opprinnelige systemet, eller fordi en kontroll ikke gjelder på din vei i det hele tatt.

En brukbar test består derfor av minst to meldinger:

| | Formål | Forventning |
|---|---|---|
| Forsøk 1 | Kontroll, replikerer originaltilfellet | **må mislykkes** |
| Forsøk 2 | Hypotese, én variabel endret | skal lykkes |

Hvis forsøk 1 ikke mislykkes, er oppsettet ditt ikke representativt. Da har du ikke lært noe om originaltilfellet, men bare om testoppsettet ditt, og må levere inn nærmere originalen.

## Et gjennomgått eksempel

Tilbake til tilfellet over, anonymisert. Meldinger fra ett system nådde ikke mottakere i skyen, mens andre meldinger til de samme mottakerne kom frem uten problemer. Tre forsøk via samme vei, til samme mottaker, med få minutters mellomrom:

| Forsøk | Avsenderdomene | Hypotese som testes | Resultat |
|---|---|---|---|
| 1 (kontroll) | `example-test.com` | Oppsettet er representativt | Avvisning, identisk med originalen |
| 2 | `example.com`, målets eget domene | Det skyldes avsenderdomenet | levert |
| 3 | `other-test.com`, eksternt domene i samme organisasjon | Det skyldes organisasjonstilhørigheten | levert |

Forsøk 1 reproduserte feilen, så oppsettet var gyldig. Forsøk 2 viste at det hang sammen med avsenderdomenet og ikke med mottaker, postboks, ruting eller tillatelser. Forsøk 3 var det egentlig elegante: Det testet målrettet den mest nærliggende alternative forklaringen og **motbeviste den**, for `other-test.com` tilhørte samme organisasjon og kom likevel gjennom.

Tre meldinger, ti minutter, og årsaken var dokumentert i stedet for antatt. Før dette hadde flere timer gått med til forsøk på forklaringer, og til slutt holdt ingen av dem.

## Prinsipp 4: Å motbevise er den egentlige fremgangen

En motbevist hypotese føles som et tilbakeskritt. I virkeligheten er den det eneste du vet sikkert. Bekreftelser er svake, fordi en observasjon kan passe til flere forklaringer. En ren motbevisning fjerner en hel gren fra søkerommet, og det permanent.

Det er nettopp her bekreftelsesbiasen virker sterkest. Når du har en antakelse, finner du nesten alltid noe som passer til den. I analysen beskrevet over fantes det en korrelasjon mellom avvisningen og spørsmålet om hvor avsenderdomenet har e-posten sin driftet. Den så overbevisende ut, men bygget på to datapunkter som skilte seg på flere egenskaper. Det tredje forsøket svekket den.

Noter derfor de motbeviste forklaringene sammen med grunnen til at de ble forkastet. Dette er ikke noe annet enn en laboratoriejournal. Den har to virkninger: Den som overtar saken senere, går ikke inn i de samme blindgatene. Og du merker selv når du tenker i sirkler, fordi en allerede forkastet idé kommer tilbake under et nytt navn.

I dokumentasjonen hører de motbeviste punktene uttrykkelig hjemme ved siden av de dokumenterte. En rapport som bare inneholder det riktige svaret, fortier halve arbeidet og inviterer til at det gjentas.

## Prinsipp 5: Kjenn utvalget ditt

Den mest subtile feilkilden er utvalgsskjevhet, og i IT rammer den særlig spørringer som leverer side for side.

Du spør etter meldingssporing for syv dager, filtrerer lokalt etter en egenskap og får ingen resultater. Konklusjonen ligger nær: Denne trafikken har ikke forekommet. I virkeligheten har du bare filtrert den første siden, og ved høyt volum dekker den bare noen få minutter.

Det korrekte resultatet er: ikke funnet i utsnittet. Det er ikke: finnes ikke. Forskjellen er den samme som mellom «ingen effekt kan påvises i studien vår» og «det finnes ingen effekt».

To utveier fungerer. Begrens tidsvinduet så mye at én side dekker det fullstendig, noe du ser ved at indikasjonen på flere resultater uteblir. Eller bla gjennom alle sidene og vurder deretter.

Og en tredje, som ofte overses: For spørsmålet om noe **aldri** forekommer, er en konfigurasjonskontroll bedre enn enhver observasjon. Hvis et system ikke har noen rute til et mål, kan det ikke levere dit, uavhengig av ethvert observasjonsvindu. Det er forskjellen mellom et empirisk og et strukturelt argument, og der du kan ha det strukturelle, velger du det.

## Overføringen: Knytt bevisbyrden til reverserbarheten

Her slutter analogien til vitenskapen, og ingeniørperspektivet tar over. Forskning søker sannhet, drift søker et fungerende anlegg. Derav følger en målestokk som vitenskapen ikke kjenner: **Arbeidet med dokumentasjon bestemmes av hvor reversibelt inngrepet er.**

Å deaktivere en Connector er én kommando, og det samme gjelder å reversere det. Da er begrunnede indikasjoner nok, for en feil er rettet på ett minutt og blir straks synlig. Å slette den samme Connectoren er ikke reversibelt; da er den ekstra dokumentasjonen gjennom konfigurasjonen på motparten eller en bruksrapport på serversiden verdt innsatsen.

Det samme gjelder endringer i regelsett. Et rent observasjonstrinn, som logger og ikke omdirigerer noe, kan du innføre på et tynt faktagrunnlag. Det er uten konsekvenser og henter inn nettopp dataene som mangler for det skarpe steget. Først omleggingen som kan holde tilbake meldinger, krever solide bevis.

Den som ikke bruker denne målestokken, gjør regelmessig begge feilene samtidig: Krever ukevis med bevis for en endring som kan reverseres på sekunder, og setter uten sikring noe i produksjon som kan stanse e-posttrafikken.

## Når du kan slutte

Det finnes et punkt der videre graving ikke lenger skaper verdi: når løsningen er klar, men mekanismen fortsatt er uklar.

I eksemplet over var det etter tre forsøk dokumentert at avsenderdomenet var utløsende, at alt annet i e-postveien fungerte og at det ikke forelå et bredere problem. Hvorfor skytjenesten internt treffer akkurat denne beslutningen, forble åpent. Det var uten betydning for korrigeringen, fordi den lå hos den sendende applikasjonen.

Skill derfor bevisst mellom to spørsmål. Hva må jeg endre for at det skal fungere? Og hvorfor oppfører systemet seg slik? Det første må du besvare, det andre kan du overlate til produsenten. En supporthenvendelse med tre kontrollerte forsøk, tidsstempler, meldingsidentifikatorer og et fungerende moteksempel er uansett langt mer verdifull enn en beskrivelse av symptomet.

For øvrig er dette også punktet der vitenskap og drift kan skilles tydelig. Vitenskapen kan ikke gi opp spørsmålet om mekanismen. Driften må prioritere det.

## Kortversjonen

Formuler hypoteser slik at de kan mislykkes, og spør deg på forhånd hvilket resultat som ville motbevise dem. Sammenlign aldri to tilfeller som tilfeldigvis foreligger, men bygg sammenligningen med nøyaktig én endret variabel. Reproduser feilen i kontrollforsøket før du tror på motforsøket. Behandle motbevisninger som fremgang og dokumenter dem skriftlig. Kontroller ved hver spørring om du ser helheten eller et utvalg. Og tilpass den krevde bevisdybden etter hvor lett det planlagte inngrepet kan reverseres.

De konkrete spørringene finner du i [Analyser e-postflyt i Exchange: Message Tracking, SMTP-protokoller og Receive-Connectors](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren). Hvis du heller vil klikke sammen kommandoene enn å skrive dem inn, finner du dem i [Kommandogeneratoren](https://rafaelpfister.ch/tools/command-builder).

## Kilder

1.  [Karl Popper: Logikk der Forschung](https://www.mohrsiebeck.com/buch/logik-der-forschung-9783161584350): Opphavet til falsifikasjonsprinsippet, som sier at en påstand bare er vitenskapelig dersom den forblir motbevisbar.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): forklarer hvorfor utvidede statuskoder bevisst er grove klasser og tillater samme kode for ulike årsaker.

3.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Hendelsestyper og felt, grunnlag for å bestemme siste behandlingstrinn.

4.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): Sidelogikken i meldingssporing, som fremmer utvalgsfeil.
