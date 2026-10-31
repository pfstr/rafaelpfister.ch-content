---
title: "Hva vi kan lære av naturvitenskapen for feilsøking i IT"
navTitle: "Kontrollerte eksperimenter"
description: "Falsifiserbarhet, kontrollgruppe, forstyrrende variabler og utvalgsskjevhet: Metoden naturvitenskapen har arbeidet med i århundrer, løser nettopp problemene IT-feilsøking regelmessig mislykkes med, illustrert med eksempler fra e-postflyt."
date: "2026-08-11"
kategorie: "SMTP / e-postflyt"
timeToRead: "15 min lesetid"
themen:
  - smtp-mailflow
  - testing
  - exchange-onprem-hybrid
hauptthema: "smtp-mailflow"
produkte:
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
translationSourceHash: 1759c709160723c161883fede66d48cd968bb6a587cb49c73781b3a39498cd66
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:31:25.877Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/hva-vi-kan-laere-av-naturvitenskapen-for-feilsoking-i-it
---

# Hva vi kan lære av naturvitenskapen for feilsøking i IT

En melding kommer ikke frem. Protokollen gir en feilmelding som umiddelbart antyder en forklaring. Du undersøker denne forklaringen, finner belegg, og etter to timer viser det seg at forklaringen var feil og at beleggene var tilfeldigheter.

Dette er ikke en nybegynnerfeil, men regelen. Og det er bemerkelsesverdig at vår bransje sjelden har en metode for dette problemet, selv om én har eksistert i århundrer og fungerer svært godt. Naturvitenskapen har nøyaktig den samme oppgaven: å slutte fra observasjoner til årsaker i systemer man ikke har full oversikt over.

Denne artikkelen overfører fem grunnprinsipper fra den vitenskapelige metoden til feilsøking i e-postflyt. Eksemplene kommer fra praksis, men fremgangsmåten er ikke spesifikk for e-post.

## Hvorfor IT-feilsøking er systematisk sårbar

E-postflyt er en kjede av systemer som hver har sitt eget syn på den samme meldingen: gatewayen, filterlaget, den lokale transportserveren, skytjenesten, målets postkasse. Hver melding er skrevet fra perspektivet til nøyaktig ett lag.

I tillegg er feilmeldinger samlebetegnelser. Den samme ordlyden beskriver ofte helt forskjellige situasjoner, fordi systemet som avviser bare kjenner et grovt rutenett. De utvidede statuskodene er laget nettopp for å danne klasser, ikke for å betegne enkelttilfeller.

Et eksempel: En skytjeneste avviste en melding med beskjed om at avsenderen ikke var tillatt for utgående levering. Den samme ordlyden forekom i samme miljø i to grunnleggende forskjellige konstellasjoner. Én gang forsøkte et system å levere via tjenesten til en ekstern mottaker, altså et reelt videresendingsforsøk utad. Den andre gangen var mottakeren en vanlig postkasse i tjenesten, og det var utelukkende avsenderdomenet som ble påklaget.

Den som tar teksten bokstavelig, leter etter det samme i begge tilfeller. Og fordi ordet «utgående» forekommer, leter man først i feil ende.

## Prinsipp 1: En hypotese må utelukke noe

Karl Popper beriket vitenskapsteorien med en innsikt som er umiddelbart praktisk for feilsøking: **En påstand er bare nyttig dersom den kan motbevises.** En forklaring som passer til ethvert tenkelig observasjonsresultat, forklarer ingenting.

Overført betyr det: Formuler antakelsen din slik at den inneholder en **forutsigelse** som kan være feil. Ikke «det er noe galt med avsenderdomenet», men «hvis jeg sender den samme meldingen med et annet avsenderdomene via samme vei, kommer den frem».

Den andre formuleringen har verdi fordi den kan motbevises på fem minutter. Den første kan du fore med belegg i timevis uten å bli noe klokere.

En god test: Spør deg selv før forsøket hvilket resultat som ville **motbevise** hypotesen din. Hvis du ikke kommer på noe, har du ikke en hypotese, men en stemning.

## Prinsipp 2: Én variabel, ellers alt likt

Kjernen i eksperimentet er kontrollen av forstyrrende variabler. I praksis skjer regelmessig det motsatte: Man sammenligner to tilfeller som tilfeldigvis foreligger. Og de skiller seg nesten alltid på flere kjennetegn samtidig.

Fra en reell sak: Meldinger fra `example-test.com` ble avvist, mens meldinger fra `partner.example` kom frem. De to domenene skilte seg på minst fire kjennetegn: tilhørighet til organisasjonen, hvor e-posten er hostet, om en streng autentiseringspolicy er lagret, og innleveringsveien. Fra to datapunkter med fire forskjeller kan man slutte nøyaktig ingenting. Alle de fire forklaringene passer.

Bygg derfor sammenligningen selv. Samme innleveringspunkt, samme mottaker, samme vei, samme tidspunkt og **nøyaktig én** endret egenskap. Hvis du mistenker avsenderdomenet, endrer du bare dette.

## Prinsipp 3: Uten kontrollforsøk er resultatet verdiløst

Dette er delen man helst utelater, og den viktigste. I klinisk forskning er kontrollgruppen en selvfølge; i IT lar man som regel være og undrer seg over motstridende resultater.

**Testoppsettet ditt må først reprodusere feilen.** Hvis du ikke kan frembringe feiltilfellet med dine egne midler, sier et vellykket motforsøk ingenting. Kanskje fungerer testmeldingen din bare fordi du leverer inn et annet sted enn det opprinnelige systemet, eller fordi en kontroll ikke gjelder på din vei i det hele tatt.

En brukbar test består derfor av minst to meldinger:

| | Formål | Forventning |
|---|---|---|
| Forsøk 1 | Kontroll, replikerer originaltilfellet | **må mislykkes** |
| Forsøk 2 | Hypotese, én variabel endret | skal lykkes |

Hvis forsøk 1 ikke mislykkes, er ikke oppsettet representativt. Da har du ikke lært noe om originaltilfellet, men bare om testoppsettet ditt, og må levere inn nærmere originalen.

## Et gjennomgått eksempel

Tilbake til tilfellet over, anonymisert. Meldinger fra ett system nådde ikke mottakere i skyen, mens andre meldinger til de samme mottakerne kom frem uten problemer. Tre forsøk via samme vei, til samme mottaker, med få minutters mellomrom:

| Forsøk | Avsenderdomene | Hypotese det tester | Resultat |
|---|---|---|---|
| 1 (kontroll) | `example-test.com` | Oppsettet er representativt | Avvisning, identisk med originalen |
| 2 | `example.com`, målets eget domene | Det skyldes avsenderdomenet | levert |
| 3 | `other-test.com`, et eksternt domene fra samme organisasjon | Det skyldes organisasjonstilhørigheten | levert |

Forsøk 1 reproduserte feilen, så oppsettet var gyldig. Forsøk 2 viste at det er knyttet til avsenderdomenet og ikke til mottaker, postkasse, ruting eller tillatelser. Forsøk 3 var det egentlig elegante: Det testet målrettet den mest nærliggende alternative forklaringen og **motbeviste den**, for `other-test.com` tilhørte samme organisasjon og kom likevel gjennom.

Tre meldinger, ti minutter, og årsaken var dokumentert i stedet for antatt. Før det hadde flere timer gått med til forsøk på forklaringer, og ingen av dem holdt til slutt.

## Prinsipp 4: Å motbevise er den egentlige fremgangen

En motbevist hypotese føles som et tilbakeskritt. I virkeligheten er den det eneste du vet sikkert. Bekreftelser er svake, fordi én observasjon kan passe til flere forklaringer. En ren motbevisning fjerner en hel gren fra søkerommet, og det permanent.

Det er nettopp her bekreftelsesskjevheten virker sterkest. Når du har en antakelse, finner du nesten alltid noe som passer til den. I analysen beskrevet ovenfor var det en korrelasjon mellom avvisningen og spørsmålet om hvor avsenderdomenet lot e-posten sin hoste. Den så overbevisende ut, men bygget på to datapunkter som skilte seg på flere kjennetegn. Det tredje forsøket avkreftet den.

Noter derfor de motbeviste forklaringene sammen med grunnen til at de ble forkastet. Det er ikke noe annet enn en laboratoriejournal. Det har to virkninger: Den som overtar saken senere, går ikke gjennom de samme blindsporene. Og du selv merker når du tenker i sirkel, fordi en idé som allerede er forkastet, kommer tilbake under et nytt navn.

I dokumentasjonen hører de motbeviste punktene uttrykkelig hjemme ved siden av de dokumenterte. En rapport som bare inneholder det riktige svaret, skjuler halvparten av arbeidet og inviterer til å gjenta det.

## Prinsipp 5: Kjenn utvalget ditt

Den mest subtile feilkilden er utvalgsskjevhet, og i IT rammer den særlig spørringer som leverer sidevis.

Du spør etter meldingssporing for sju dager, filtrerer lokalt etter en egenskap og får ingen resultater. Det er nærliggende å konkludere med at denne trafikken ikke fantes. I virkeligheten har du bare filtrert den første siden, og ved høyt volum dekker den bare noen få minutter.

Det riktige resultatet er: ikke funnet i utsnittet. Det er ikke: finnes ikke. Forskjellen er den samme som mellom «ingen effekt kan påvises i vår studie» og «det finnes ingen effekt».

To utveier fungerer. Gjør tidsvinduet så lite at én side dekker det fullstendig, noe du ser ved at hintet om flere resultater uteblir. Eller bla gjennom alle sidene og vurder deretter.

Og en tredje, som ofte overses: For spørsmålet om noe **aldri** forekommer, er en konfigurasjonskontroll bedre enn enhver observasjon. Hvis et system ikke har noen rute til et mål, kan det ikke levere dit, uavhengig av observasjonsvinduet. Det er forskjellen mellom et empirisk og et strukturelt argument, og der du kan ha det strukturelle, bruker du det.

## Overføringen: Knytt bevisbyrden til reversibiliteten

Her slutter analogien med vitenskapen, og ingeniørperspektivet overtar. Forskning søker sannhet, drift søker et fungerende anlegg. Det følger en målestokk vitenskapen ikke kjenner: **Innsatsen for dokumentasjon avhenger av hvor reversibelt inngrepet er.**

Å deaktivere en connector er én kommando, og det samme er å angre. Da er begrunnede indikasjoner nok, for en feil er rettet på ett minutt og blir umiddelbart synlig. Å slette den samme connectoren er ikke reversibelt; da er det verdt den ekstra dokumentasjonen gjennom konfigurasjonen hos motparten eller en serverbasert bruksrapport.

Det samme gjelder endringer i regelsett. Du kan innføre et rent observasjonsnivå som logger og ikke videresender noe, på et tynt faktagrunnlag. Det er uten konsekvenser og samler inn nettopp dataene som mangler for det skarpe steget. Først omleggingen som kan holde tilbake meldinger, krever solide bevis.

Den som ikke bruker denne målestokken, gjør regelmessig begge feilene samtidig: Krever ukevis med bevis for en endring som kan angres på sekunder, og aktiverer uten sikring noe som kan stanse e-posttrafikken.

## Når du kan slutte

Det finnes et punkt der videre graving ikke lenger gir verdi: når utbedringen er klar, men mekanismen fortsatt er uklar.

I eksemplet over var det etter tre forsøk dokumentert at avsenderdomenet var utløseren, at alt annet i e-postveien fungerte og at det ikke forelå et bredere problem. Hvorfor skytjenesten internt avgjør akkurat slik, forble åpent. For korrigeringen var det uten betydning, fordi den lå hos den sendende applikasjonen.

Skill derfor bevisst mellom to spørsmål. Hva må jeg endre for at det skal fungere? Og hvorfor oppfører systemet seg slik? Det første må du besvare, det andre kan du overlate til produsenten. En supportsak med tre kontrollerte forsøk, tidsstempler, meldingsidentifikatorer og et fungerende moteksempel er uansett langt mer verdifull enn en beskrivelse av symptomet.

Dette er for øvrig også punktet der vitenskap og drift kan skilles tydelig. Vitenskapen kan ikke gi opp spørsmålet om mekanismen. Driften må prioritere det.

## Kortversjonen

Formuler hypoteser slik at de kan mislykkes, og spør deg selv på forhånd hvilket resultat som ville motbevise dem. Sammenlign aldri to tilfeldige tilfeller, men bygg sammenligningen med nøyaktig én endret variabel. Reproduser feilen i kontrollforsøket før du tror på motforsøket. Behandle motbevisninger som fremgang og dokumenter dem skriftlig. Kontroller ved hver spørring om du ser helheten eller et utvalg. Og tilpass den krevde bevisdybden etter hvor enkelt det planlagte inngrepet kan reverseres.

De konkrete spørringene finner du i [Analyser Exchange-e-postflyt: Message Tracking, SMTP-protokoller og Receive Connectors](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren). Hvis du heller vil klikke sammen kommandoene enn å skrive dem inn, finner du dem i [Kommandogeneratoren](https://rafaelpfister.ch/tools/command-builder).

## Endringslogg

- **25. oktober 2026:** Kildelenker oppdatert til de nye adressene hos Mohr Siebeck og Microsoft Learn.

## Kilder

1.  [Karl Popper: Logik der Forschung](https://www.mohrsiebeck.com/buch/logik-der-forschung-9783161484100/): Opphavet til falsifikasjonsprinsippet, ifølge hvilket en påstand bare er vitenskapelig dersom den forblir mulig å motbevise.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): forklarer hvorfor utvidede statuskoder bevisst er grove klasser og tillater samme kode for ulike årsaker.

3.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Hendelsestyper og felt, grunnlag for å fastslå det siste behandlingstrinnet.

4.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): Sidelogikken i meldingssporing, som fremmer utvalgsfeil.
