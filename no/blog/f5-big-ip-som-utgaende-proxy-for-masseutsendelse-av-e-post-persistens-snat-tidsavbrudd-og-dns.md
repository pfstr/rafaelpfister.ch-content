---
title: "F5 BIG-IP som utgående proxy for masseutsending av e-post: persistens, SNAT, tidsavbrudd og DNS-oppløsning"
navTitle: "F5 masseutsending"
description: "En masseutsending med 1000 e-poster per minutt går via en BIG-IP som utgående proxy til leverandørens relay. Artikkelen forklarer hvorfor Sticky Sessions ikke gir noen gevinst her, hvordan leverandørens vertsnavn løses riktig opp med en FQDN-node, og hvilke innstillinger for SNAT, tidsavbrudd og tilkoblingsgrenser som faktisk bestemmer gjennomstrømningen."
date: "2026-08-26"
kategorie: "Lastbalanserer"
timeToRead: "9 min lesetid"
themen:
  - loadbalancer
  - smtp-mailflow
produkte:
  - "loadbalancer"
protokolle:
  - "smtp"
  - "tcp"
  - "dns"
hauptthema: "loadbalancer"
related:
  - massenmailing-provider-wechsel-checkliste
  - mailserver-lastprofil-ermitteln
slug: "f5-big-ip-som-utgaende-proxy-for-masseutsendelse-av-e-post-persistens-snat-tidsavbrudd-og-dns"
featured: true
translationId: "article-ee5e63e82ffd2604"
translationOf: f5-big-ip-outbound-smtp-massenversand
translationSourceHash: 5c62d1467d3a000a8c4a3255d6f925e294d76d1b65054e91b863a5166e1a22f8
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:24:44.521Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/f5-big-ip-som-utgaende-proxy-for-masseutsendelse-av-e-post-persistens-snat-tidsavbrudd-og-dns
---

En fakturakjøring eller utsending av nyhetsbrev med rundt 1000 e-poster per minutt forlater bedriftsnettverket, med en F5 BIG-IP imellom som utgående proxy til leverandørens innleveringspunkt. BIG-IP-en fordeler ikke til flere mål, den videresender bare. Nettopp denne konstellasjonen avgjør hvilke innstillinger som er fornuftige, og hvilke tilsynelatende optimaliseringer som ikke har noen effekt.

## Arkitekturen i én setning

Utsendingssystemene bruker en intern Virtual Server-adresse på BIG-IP-en som Smarthost, BIG-IP-en oversetter avsenderadressene via SNAT til en fast offentlig IP og videresender hver tilkobling til leverandørens vertsnavn. Lastbalansering i egentlig forstand skjer ikke på BIG-IP-en, fordi poolen bare har ett medlem. Det høres ut som en triviell konfigurasjon, men detaljbeslutningene (persistens, tidsavbrudd, SNAT-type, DNS-oppløsning) avgjør om utsendingen går stabilt eller får uforklarlige avbrudd under belastning.

## Er Sticky Sessions bedre? Nei, av to grunner

Spørsmålet om sesjonspersistens kommer fra HTTP-verdenen, der en bruker med handlekurv eller innloggingssesjon alltid må havne på samme backend. Overført til SMTP gir konseptet ingen mening.

For det første avsluttes SMTP tilstandsløst per tilkobling: Hver tilkobling gjennomfører én eller flere fullstendige transaksjoner (MAIL FROM, RCPT TO, DATA) og avsluttes med QUIT. Det finnes ingen tilstand som må ligge på samme målsystem på tvers av tilkoblinger. Hvilket system på leverandørsiden som aksepterer den neste tilkoblingen, er irrelevant for leveringen.

For det andre finnes det rett og slett ingenting å persistere på denne BIG-IP-en: Poolen inneholder nøyaktig ett medlem, leverandørens ene IP-adresse. En persistensprofil ville bare bruke minne på en persistens-tabell og kreve et oppslag ved hver tilkobling, som alltid gir samme resultat. Riktig innstilling er derfor: Default Persistence Profile til None. Selv om leverandøren senere skulle publisere flere IP-adresser bak vertsnavnet, ville persistens være kontraproduktivt, fordi det ville hindre fordelingen på disse adressene og belaste enkelte mål ensidig.

Avgjørende for gjennomstrømningen ved masseutsending er avsenderens tilkoblingsprofil: få langvarige tilkoblinger med mange meldinger per tilkobling, i stedet for én ny tilkobling per e-post; mer om dette nedenfor.

## Virtual Server: FastL4 i stedet for Full Proxy

For ren videresending av SMTP er en Performance-(Layer-4)-Virtual-Server med FastL4-profil det riktige valget. BIG-IP-en behandler da tilkoblingen i stor grad i maskinvare eller i den akselererte banen, uten å terminere TCP-tilkoblingen fullstendig. En standard Virtual Server i Full-Proxy-modus gir bare merverdi dersom du faktisk vil gripe inn i datastrømmen på BIG-IP-en, for eksempel med en SMTP-sikkerhetsprofil eller iRules på protokollnivå. For en utgående proxy til egen avtalepartner er dette unødvendig og skaper bare flere feilkilder.

Viktig i begge tilfeller: Ikke aktiver noen profil som skriver inn i tilkoblingen. STARTTLS forhandles direkte mellom utsendingssystemene og leverandørens relay; enhver instans som endrer eller filtrerer byte, setter TLS-etableringen i fare.

## DNS-oppløsning: leverandørens vertsnavn skal ligge som FQDN-node i poolen

Leverandøren har oppgitt et vertsnavn, ikke en IP-adresse. Den nærliggende refleksen, å løse opp IP-adressen én gang og legge den inn statisk som node, er den dårligste varianten: Hvis leverandøren endrer adresse (vedlikehold, flytting, DR-tilfelle), stopper utsendingen til noen tilpasser BIG-IP-konfigurasjonen. Nettopp derfor finnes FQDN-noder.

En FQDN-node lagrer vertsnavnet i stedet for adressen. BIG-IP-en løser selv opp navnet, oppretter en såkalt Ephemeral-node for hver returnerte adresse og oppdaterer disse automatisk når DNS-svaret endres. Som standard spør den etter navnet på nytt hvert 3600. sekund (én time); med `interval ttl` følger den i stedet TTL-en i DNS-svaret, alternativt kan et annet fast spørreintervall angis. Med Autopopulate aktivert overtar poolen også flere A-records automatisk som medlemmer: Dersom leverandøren senere utvider innleveringen til flere adresser, følger BIG-IP-en etter uten konfigurasjonsendring.

To forutsetninger blir ofte glemt. For det første trenger BIG-IP-en fungerende DNS-servere i systemkonfigurasjonen (System, Configuration, Device, DNS); FQDN-noder bruker systemresolverne, ikke en DNS-cache fra en listener-profil. For det andre bør disse resolverne faktisk være tilgjengelige fra management- eller TMM-konteksten, ellers forblir noden i status unresolved og poolen tom.

Konfigurasjonen i tmsh ser slik ut (adresser og navn er eksempler):

```bash
tmsh create ltm node relay-provider fqdn { \
  name mail-relay.provider.example autopopulate enabled }

tmsh create ltm pool pool_provider_smtp \
  members add { relay-provider:25 } monitor tcp

tmsh create ltm snatpool snat_mailout \
  members add { 198.51.100.10 }

tmsh create ltm virtual vs_mailout_smtp \
  destination 10.0.5.10:25 ip-protocol tcp \
  profiles add { fastL4 } pool pool_provider_smtp \
  source-address-translation { type snat pool snat_mailout }
```

Utsendingssystemene angir deretter 10.0.5.10 som Smarthost. Om du bruker port 25 eller 587, bestemmes av leverandøren; BIG-IP-konfigurasjonen er identisk i begge tilfeller, bare porten endres.

## SNAT: fast adresse i stedet for Automap

For utgående e-posttrafikk bør kildeadressen være under kontroll. SNAT Automap bruker Floating Self-IP-en til det utgående VLAN-et, og den kan endres umerkelig ved nettverksendringer eller failover-ombygginger. Leverandører knytter imidlertid ofte innleveringen til IP-allowlisting, og selv uten formell allowlisting er omdømme knyttet til kildeadressen. En dedikert SNAT-pool med en fast tildelt adresse gjør kilde-IP-en til et dokumentert, stabilt konfigurasjonsobjekt.

Når det gjelder kapasitet: Én enkelt SNAT-adresse gir mot ett enkelt mål (én IP, én port) rundt 64'000 samtidige oversettelser, fordi hver tilkobling får sin egen ephemeral kildeport. Med belastningsprofilen som beskrives her, med noen få dusin samtidige tilkoblinger, er dette mer enn tilstrekkelig. Portuttømming blir først et tema når en feilkonfigurert avsender åpner en ny tilkobling per e-post og ikke lukker den korrekt; da samler oversettelser seg i en TIME-WAIT-lignende tilstand. Slik atferd må du rette hos avsenderen, ikke med en ekstra SNAT-adresse.

## Tidsavbrudd: den vanligste årsaken til tilkoblingsbrudd under belastning

En bulk-avsender holder tilkoblinger åpne og sender melding etter melding gjennom dem. Det kan oppstå pauser mellom to meldinger: Avsenderen genererer neste blokk, relayet forsinker aksepten (Tarpitting, rester av Greylisting, interne køer). Idle Timeout for FastL4-profilen er som standard 300 sekunder. Varer en pause lenger, rydder BIG-IP-en bort tilkoblingen, og avsenderen skriver til en tilkobling som ikke lenger finnes.

To innstillinger avhjelper dette. For det første bør Idle Timeout settes til en verdi som overstiger realistiske pauser; for masseutsending er 600 sekunder en fornuftig startverdi. Verdien bør ikke settes vilkårlig høyt, ellers samler foreldreløse tilkoblinger seg i tilkoblingstabellen. For det andre bør Reset on Timeout forbli aktivert i profilen: BIG-IP-en kvitterer da oppryddingen med en TCP-reset, og den sendende MTA-en oppdager umiddelbart at tilkoblingen er borte, i stedet for å vente på timeout og planlegge meldingen på nytt først etter flere minutter.

Du har ingen innflytelse på tidsavbruddene hos motparten, men de må tas med i bildet: Hvis leverandørens relay lukker tilkoblinger etter 120 sekunders inaktivitet, hjelper ikke et romslig BIG-IP-timeout. Det er den laveste timeout-verdien på hele veien som gjelder; spør leverandøren ved tvil og bruk denne verdien som grunnlag for planleggingen.

## Tilkoblingsstrategi: få tilkoblinger, mange meldinger

Uten innleveringskrav fra leverandøren er en kort beregning nyttig. 1000 e-poster per minutt er rundt 17 per sekund. En SMTP-transaksjon over en allerede etablert tilkobling tar ved normal latenstid betydelig under et halvt sekund. Med 10 til 20 parallelle tilkoblinger og for eksempel 100 meldinger per tilkobling før avsenderen fornyer dem, oppnås målraten uten problemer. På leverandørsiden er det vanligvis betydelig mer tilkoblingskapasitet tilgjengelig, men den deles med alle andre kunder. Få langvarige tilkoblinger med mange transaksjoner er derfor ikke bare effektivt (TCP- og TLS-etablering bortfaller per melding), men også den mest skånsomme måten å bruke fremmed infrastruktur på.

Innstillingene for dette ligger i utsendingssystemet, ikke på BIG-IP-en: maksimalt antall meldinger per tilkobling, maksimalt antall parallelle tilkoblinger til Smarthost og gjenbruk av etablerte tilkoblinger. På BIG-IP-en kan hele oppsettet sikres med en Connection Limit på pool-medlemmet, for eksempel 200 samtidige tilkoblinger: Ved normal drift nås aldri verdien, men en feilkonfigurert avsender som plutselig åpner én tilkobling per e-post, oversvømmer ikke leverandørens relay ukontrollert. Grensen er et sikkerhetsnett, ikke et styringsverktøy.

Om den konfigurerte tilkoblingsprofilen faktisk oppnås i praksis, viser målingen: Tilkoblinger per minutt og meldinger per tilkobling kan evalueres fra Message Tracking eller Connector-loggene, som beskrevet i artikkelen [Fastslå belastningsprofilen til en e-postserver](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln). For en belastningstest med et realistisk bulk-belastningsbilde (få sesjoner, mange meldinger per sesjon) egner smtp-source fra Postfix-pakken seg bedre enn HTTP-orienterte belastningsverktøy, fordi det genererer nettopp denne tilkoblingsprofilen.

## Overvåking: ikke belast leverandøren med helsesjekker

En monitor på pool-medlemmet er fornuftig, slik at BIG-IP-en oppdager og rapporterer feil på leverandørsiden på en ryddig måte. Her gjelder følgende: Hver helsesjekk er en reell tilkobling til leverandøren og teller mot de samme grensene som nyttetrafikken. En enkel TCP-monitor med moderat intervall (30 sekunder eller mer) er fullt tilstrekkelig. En fullstendig SMTP-monitor som kontrollerer helt til banneret eller EHLO, gir knapt mer innsikt, men genererer loggoppføringer hos leverandøren og i verste fall spørsmål om hvorfor en tilkobling uten e-post kommer inn hvert femte sekund.

## Sjekkliste

| Innstilling | Anbefaling |
|---|---|
| Persistensprofil | None; Sticky Sessions gir ingen gevinst med SMTP, og enda mindre med en pool med ett medlem |
| Virtual Server-type | Performance (Layer 4) med FastL4-profil, ingen inngrep i datastrømmen |
| Målnode | FQDN-node med Autopopulate i stedet for statisk IP; DNS-servere konfigurert på BIG-IP-en |
| SNAT | dedikert SNAT-pool med fast adresse kjent for leverandøren; ikke Automap |
| Idle Timeout | over de faktiske sendepausene, startverdi 600 s; Reset on Timeout aktiv |
| Connection Limit | som sikkerhetsnett på pool-medlemmet, f.eks. 200 |
| Monitor | TCP, intervall 30 s eller mer; ingen aggressiv SMTP-monitor |
| Avsenderkonfigurasjon | få parallelle tilkoblinger, mange meldinger per tilkobling; gjenbruk aktivert |

Det korte svaret på det opprinnelige spørsmålet er altså: Nei, Sticky Sessions er ikke bedre, de er virkningsløse til skadelige i denne konstellasjonen. Kvaliteten på løsningen avgjøres av DNS-oppløsningen av leverandørens vertsnavn, av en stabil SNAT-adresse, av tidsavbrudd som passer belastningsprofilen, og av at utsendingssystemene leverer sine 1000 e-poster per minutt gjennom få etablerte tilkoblinger i stedet for gjennom tusen enkeltstående.

## Endringslogg

- **25. oktober 2026:** Korrigert: Standardintervallet for navneoppløsning av FQDN-noder er 3600 sekunder, ikke DNS-TTL; BIG-IP-en følger bare TTL-en med innstillingen interval ttl.

## Kilder

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Avsnitt 4.5.4 og transaksjonsmodellen viser at flere e-posttransaksjoner over én tilkobling er det tiltenkte normaltilfellet.

2.  [K7820: Overview of SNAT features](https://my.f5.com/manage/s/article/K7820): F5-grunnlagsartikkel om SNAT, SNAT-pooler og portoversettelse per mål.

3.  [tmsh-referanse: ltm node](https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_node.html): dokumenterer FQDN-alternativene (name, autopopulate, interval) for noder og dermed for pool-medlemmer, inkludert standardintervallet på 3600 sekunder og verdien `ttl`.

4.  [smtp-source(1), Postfix](https://www.postfix.org/smtp-source.1.html): belastningsgenerator som gjenskaper bulk-avsenderens tilkoblingsprofil (få sesjoner, mange meldinger).

5.  [Fastslå belastningsprofilen til en e-postserver](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln): egen veiledning om hvordan tilkoblinger per minutt og meldinger per tilkobling evalueres fra Message Tracking.

6.  [F5 Bug ID 746657: tmsh help for FQDN node or pool member shows incorrect default for fqdn interval](https://cdn.f5.com/product/bugtracker/ID746657.html): bekrefter 3600 sekunder som den faktiske standardverdien for FQDN-spørreintervallet; den tidligere oppgitte TTL-verdien i hjelpen var feil.
