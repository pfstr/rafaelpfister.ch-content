---
title: "Hva vi kan lære av naturvitenskapen for feilsøking i IT"
navTitle: "Kontrollerte eksperimenter"
description: "Falsifiserbarhet, kontrollgruppe, forstyrrende variabler og utvalgsskjevhet: Metoden naturvitenskapen har arbeidet med i århundrer, løser nettopp problemene IT-feilsøking regelmessig mislykkes med. Med gjennomspilte eksempler fra e-postflyten."
date: "2026-08-11"
kategorie: "SMTP / e-postflyt"
timeToRead: "15 min lesetid"
themen:
  - smtp-mailflow
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
url: https://rafaelpfister.ch/no/blog/hva-vi-kan-laere-av-naturvitenskapen-for-feilsoking-i-it
translationSourceHash: d2466d0e63e5b08052fe7a47766ec2500b94c84097bfcfe91f8f6348cd6d1cc2
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:21:53.603Z
translationReview: automatic
---

# Hva vi kan lære av naturvitenskapen for feilsøking i IT

En melding kommer ikke frem. Protokollen gir en feilmelding som umiddelbart antyder en forklaring. Du undersøker denne forklaringen, finner belegg, og etter to timer viser det seg at forklaringen var feil og at beleggene var tilfeldigheter.

Dette er ikke en nybegynnerfeil, men regelen. Og det er bemerkelsesverdig at vår bransje sjelden har en metode for dette problemet, selv om én har eksistert i århundrer og fungerer svært godt. Naturvitenskapen har nøyaktig samme oppgave: å slutte fra observasjoner til årsaker, i systemer man ikke har full oversikt over.

Denne artikkelen overfører fem prinsipper fra den vitenskapelige metoden til feilsøking i e-postflyten. Eksemplene er hentet fra praksis, men fremgangsmåten er ikke spesifikk for e-post.

## Hvorfor IT-feilsøking er systematisk sårbar

E-postflyten er en kjede av systemer som hver har sitt eget syn på den samme meldingen: gatewayen, filterlaget, den lokale transportserveren, skytjenesten, målpostboksen. Hver melding er skrevet fra perspektivet til nøyaktig ett lag.

I tillegg kommer dette: Feiltekster er samlebetegnelser. Den samme ordlyden beskriver ofte helt ulike situasjoner fordi systemet som avviser bare kjenner et grovt inndelingsmønster. De utvidede statuskodene er laget nettopp for å danne klasser, ikke for å navngi enkelttilfeller.

Et eksempel: En skytjeneste avviste en melding med henvisning til at avsenderen ikke var tillatt for utgående levering. Den samme ordlyden forekom i samme miljø i to helt forskjellige konstellasjoner. Én gang forsøkte et system å levere via tjenesten til en ekstern mottaker, altså et reelt forsøk på videresending utad. Den andre gangen var mottakeren en ordinær postboks i tjenesten, og det var utelukkende avsenderdomenet som ble påtalt.

Den som tolker teksten bokstavelig, leter etter det samme i begge tilfeller. Og fordi ordet «utgående» forekommer i den, leter man først i feil ende.

## Prinsipp 1: En hypotese må utelukke noe

Karl Popper beriket vitenskapsteorien med en innsikt som er umiddelbart praktisk for feilsøking: **En påstand er bare nyttig dersom den kan motbevises.** En forklaring som passer til ethvert tenkelig observasjonsresultat, forklarer ingenting.

Overført betyr det: Formuler antakelsen din slik at den inneholder en **forutsigelse** som kan være feil. Ikke «det er noe galt med avsenderdomenet», men «hvis jeg sender den samme meldingen med et annet avsenderdomene via samme vei, kommer den frem».

Den andre formuleringen er verdifull fordi den kan knuses på fem minutter. Den første kan du fore med belegg i timevis uten å bli noe klokere.

En god test for dette: Spør deg selv før forsøket hvilket resultat som ville **motbevise** hypotesen din. Hvis du ikke kommer på noe, har du ikke en hypotese, men en stemning.

## Prinsipp 2: Én variabel, ellers alt likt

Kjernen i eksperimentet er kontrollen av forstyrrende variabler. I praksis skjer regelmessig det motsatte: Man sammenligner to tilfeller som tilfeldigvis foreligger. Og de skiller seg nesten alltid på flere kjennetegn samtidig.

Fra et reelt tilfelle: Meldinger fra `example-test.com` ble avvist, meldinger fra `partner.example` kom frem. De to domenene skilte seg på minst fire kjennetegn: tilhørighet til organisasjonen, hvor e-posten er driftet, om det er lagt inn en streng autentiseringspolicy, og innleveringsveien. Fra to datapunkter med fire forskjeller kan man ikke slutte absolutt noe. Alle de fire forklaringene passer.

Bygg derfor sammenligningen selv. Samme innleveringspunkt, samme mottaker, samme vei, samme tidspunkt og **nøyaktig ett** endret kjennetegn. Dersom du mistenker avsenderdomenet, endrer du bare dette.

## Prinsipp 3: Uten kontrollforsøk er resultatet verdiløst

Dette er delen man helst utelater, og den viktigste. I klinisk forskning er kontrollgruppen en selvfølge; i IT avstår man som regel fra den og undrer seg over motstridende resultater.

**Testoppsettet ditt må først reprodusere feilen.** Hvis du ikke kan frembringe feiltilfellet med egne midler, sier et vellykket motforsøk ingenting. Kanskje fungerer testmeldingen din bare fordi du leverer inn et annet sted enn originalsystmet, eller fordi en kontroll ikke i det hele tatt gjelder på din vei.

En brukbar test består derfor av minst to meldinger:

| | Formål | Forventning |
|---|---|---|
| Forsøk 1 | Kontroll, replikerer originaltilfellet | **må mislykkes** |
| Forsøk 2 | Hypotese, én variabel endret | skal lykkes |

Hvis forsøk 1 ikke mislykkes, er oppsettet ditt ikke representativt. Da har du ikke lært noe om originaltilfellet, men bare om testoppsettet ditt, og må levere inn nærmere originalen.

## Et gjennomspilt eksempel

Tilbake til tilfellet over, anonymisert. Meldinger fra ett system nådde ikke mottakere i skyen, mens andre meldinger til de samme mottakerne kom frem uten problemer. Tre forsøk via samme vei, til samme mottaker, med få minutters mellomrom:

| Forsøk | Avsenderdomene | Hypotese det tester | Resultat |
|---|---|---|---|
| 1 (kontroll) | `example-test.com` | Oppsettet er representativt | Avvisning, identisk med originalen |
| 2 | `example.com`, målets eget domene | Det skyldes avsenderdomenet | levert |
| 3 | `other-test.com`, eksternt domene i samme organisasjon | Det skyldes organisasjonstilhørigheten | levert |

Forsøk 1 reproduserte feilen, så oppsettet var gyldig. Forsøk 2 viste at det var knyttet til avsenderdomenet og ikke mottaker, postboks, ruting eller tillatelser. Forsøk 3 var det egentlig elegante: Det testet målrettet den mest nærliggende alternative forklaringen og **motbeviste den**, for `other-test.com` tilhørte samme organisasjon og kom likevel gjennom.

Tre meldinger, ti minutter, og årsaken var dokumentert i stedet for antatt. Før dette hadde flere timer gått med til forklaringsforsøk som til slutt ingen holdt.

## Prinsipp 4: Å motbevise er den egentlige fremgangen

En motbevist hypotese føles som et tilbakeskritt. I virkeligheten er den det eneste du vet med sikkerhet. Bekreftelser er svake, for en observasjon kan passe til flere forklaringer. En ren motbevisning fjerner en hel gren fra søkerommet, og det permanent.

Det er nettopp her bekreftelsesbias virker sterkest. Når du har en antakelse, finner du nesten alltid noe som passer til den. I analysen beskrevet over fantes det en korrelasjon mellom avvisningen og spørsmålet om hvor avsenderdomenet lot e-posten sin drifte. Den så overbevisende ut, men bygget på to datapunkter som skilte seg på flere kjennetegn. Det tredje forsøket rev den i stykker.

Noter derfor de motbeviste forklaringene sammen med grunnen til at de ble forkastet. Dette er ikke noe annet enn en laboratoriejournal. Det har to virkninger: Den som senere overtar saken, går ikke inn i de samme blindgatene. Og du selv merker når du tenker i sirkler, fordi en allerede forkastet idé kommer tilbake under et nytt navn.

I dokumentasjonen hører de motbeviste punktene uttrykkelig hjemme ved siden av de dokumenterte. En rapport som bare inneholder det riktige svaret, skjuler halve arbeidet og inviterer til å gjenta det.

## Prinsipp 5: Kjenn utvalget ditt

Den mest subtile feilkilden er utvalgsskjevhet, og i IT rammer den særlig spørringer som leverer resultater sidevis.

Du spør etter sju dager med meldingssporing, filtrerer lokalt etter et kjennetegn og får ingen resultater. Konklusjonen ligger nær: Denne trafikken har ikke eksistert. I virkeligheten har du bare filtrert den første siden, og ved høy trafikk dekker den noen få minutter.

Det riktige resultatet er: ikke funnet i utsnittet. Det er ikke: eksisterer ikke. Forskjellen er den samme som mellom «ingen effekt kan påvises i vår studie» og «det finnes ingen effekt».

To utveier fungerer. Begrens tidsvinduet til en side dekker det fullstendig, noe du ser ved at henvisningen til flere resultater uteblir. Eller bla gjennom alle sidene og analyser deretter.

Og en tredje, som ofte overses: For spørsmålet om noe **aldri** forekommer, er en konfigurasjonskontroll bedre enn enhver observasjon. Hvis et system ikke har en rute til et mål, kan det ikke levere dit, uavhengig av ethvert observasjonsvindu. Det er forskjellen mellom et empirisk og et strukturelt argument, og der du kan bruke det strukturelle, bruker du det.

## Overføringen: Knytt bevisbyrden til reverserbarheten

Her slutter analogien til vitenskapen, og ingeniørperspektivet overtar. Forskning vil ha sannhet, drift vil ha et anlegg som fungerer. Av dette følger en målestokk som vitenskapen ikke kjenner: **Innsatsen for dokumentasjon avhenger av hvor reversibelt inngrepet er.**

Å deaktivere en connector er én kommando, og det samme gjelder å angre det. Da er begrunnede indisier nok, for en feil er rettet på ett minutt og blir umiddelbart synlig. Å slette den samme connectoren er ikke reversibelt; da er den ekstra dokumentasjonen gjennom konfigurasjonen på motparten eller en bruksrapport på serversiden verdt det.

Det samme gjelder endringer i regelverk. Et rent observasjonstrinn som logger og ikke omdirigerer noe, kan du innføre på et tynt faktagrunnlag. Det er uten konsekvenser og innhenter akkurat dataene som mangler for det skarpe steget. Først omleggingen som kan holde tilbake meldinger, krever robuste belegg.

Den som ikke bruker denne målestokken, gjør regelmessig begge feilene samtidig: Krever ukelange bevis for en endring som kan angres på sekunder, og aktiverer uten sikring noe som kan stanse e-posttrafikken.

## Når du kan slutte

Det finnes et punkt der videre graving ikke lenger skaper verdi: når løsningen er fastslått, men mekanismen fortsatt er uklar.

I eksempelet over var det etter tre forsøk dokumentert at avsenderdomenet er utløsende faktor, at alt annet i e-postveien fungerer og at det ikke foreligger et bredere problem. Hvorfor skytjenesten internt avgjør nøyaktig slik, forble åpent. For korrigeringen var dette uten betydning, for den lå hos applikasjonen som sender.

Skill derfor bevisst mellom to spørsmål. Hva må jeg endre for at det skal fungere? Og hvorfor oppfører systemet seg slik? Det første må du besvare, det andre kan du overlate til produsenten. En supportsak med tre kontrollerte forsøk, tidsstempler, meldingsidentifikatorer og et fungerende moteksempel er uansett langt mer verdifull enn en beskrivelse av symptomet.

Dette er for øvrig også punktet der vitenskap og drift kan skilles tydelig. Vitenskapen kan ikke gi opp spørsmålet om mekanismen. Driften må prioritere det.

## Kortversjonen

Formuler hypoteser slik at de kan mislykkes, og spør deg selv på forhånd hvilket resultat som ville ha motbevist dem. Sammenlign aldri to tilfeldig foreliggende tilfeller, men bygg sammenligningen med nøyaktig én endret variabel. Reproduser feilen i kontrollforsøket før du tror på motforsøket. Behandle motbevisninger som fremgang og dokumenter dem skriftlig. Kontroller ved hver spørring om du ser helheten eller et utvalg. Og la den påkrevde bevisdybden avhenge av hvor lett det planlagte inngrepet kan angres.

De konkrete spørringene finner du i [Analyser Exchange-e-postflyt: Message Tracking, SMTP-protokoller og Receive Connectors](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren). Hvis du heller vil klikke sammen kommandoene enn å skrive dem inn, finner du dem i [Kommandogeneratoren](https://rafaelpfister.ch/tools/command-builder).

## Kilder

1.  [Karl Popper: Forskningens logikk](https://www.mohrsiebeck.com/buch/logik-der-forschung-9783161584350): Opprinnelsen til falsifikasjonsprinsippet, som sier at en påstand bare er vitenskapelig dersom den forblir mulig å motbevise.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): forklarer hvorfor utvidede statuskoder bevisst er grove klasser og tillater samme kode for forskjellige årsaker.

3.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): hendelsestyper og felt, grunnlaget for å fastslå det siste behandlingstrinnet.

4.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): sidelogikken i meldingssporing, som fremmer utvalgsfeil.
