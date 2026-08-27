---
title: "F5 BIG-IP som utgående proxy for masseutsendelse av e-post: persistens, SNAT, tidsavbrudd og DNS-oppløsning"
navTitle: "F5 masseutsendelse"
description: "En masseutsendelse med 1000 e-poster per minutt går via en BIG-IP som utgående proxy til leverandørens relé. Artikkelen forklarer hvorfor Sticky Sessions ikke hjelper her, hvordan leverandørens vertsnavn løses riktig med en FQDN-node, og hvilke innstillinger for SNAT, tidsavbrudd og tilkoblingsgrenser som faktisk bestemmer gjennomstrømmingen."
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
aiPrompt: |
  Du bist mein Netzwerk- und Mailflow-Assistent. Wir versenden Massenmails über eine F5 BIG-IP als ausgehenden Proxy zu einem Provider-Relay. Hilf mir, die BIG-IP-Konfiguration nach diesem Artikel zu prüfen: 1. Frage mich nach Versandrate, Anzahl paralleler Verbindungen und Nachrichten pro Verbindung. 2. Frage nach Virtual-Server-Typ, Persistenzprofil, Idle-Timeout und SNAT-Konfiguration. 3. Prüfe, ob der Provider-Hostname als FQDN-Node mit Autopopulate hinterlegt ist und ob DNS-Server auf der BIG-IP konfiguriert sind. 4. Nenne mir konkrete Abweichungen von den Empfehlungen aus dem Artikel und begründe jede Änderung.
translationOf: f5-big-ip-outbound-smtp-massenversand
url: https://rafaelpfister.ch/no/blog/f5-big-ip-som-utgaende-proxy-for-masseutsendelse-av-e-post-persistens-snat-tidsavbrudd-og-dns
translationSourceHash: 218c4d189dd18000d6db2ead4b2106f8be858169c9d7b234e4f9320ac802fd46
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:30:46.891Z
translationReview: required
---

En fakturakjøring eller utsendelse av nyhetsbrev med rundt 1000 e-poster per minutt forlater bedriftsnettet, med en F5 BIG-IP som utgående proxy til leverandørens innleveringspunkt imellom. BIG-IP-en fordeler ikke til flere mål, den videresender bare. Nettopp denne konstellasjonen avgjør hvilke innstillinger som er hensiktsmessige, og hvilke tilsynelatende optimaliseringer som ikke har noen effekt.

## Arkitekturen i én setning

Utsendelsessystemene bruker en intern Virtual Server-adresse på BIG-IP-en som smarthost, BIG-IP-en oversetter avsenderadressene via SNAT til en fast offentlig IP og videresender hver tilkobling til leverandørens vertsnavn. Lastbalansering i egentlig forstand finner ikke sted på BIG-IP-en, fordi poolen bare har ett medlem. Det høres ut som en triviell konfigurasjon, men detaljvalgene (persistens, tidsavbrudd, SNAT-type, DNS-oppløsning) avgjør om utsendelsen kjører stabilt eller får uforklarlige avbrudd under belastning.

## Er Sticky Sessions bedre? Nei, av to grunner

Spørsmålet om sesjonspersistens kommer fra HTTP-verdenen, der en bruker med handlekurv eller innloggingssesjon alltid må havne på samme backend. Overført til SMTP gir konseptet ingen mening.

For det første avsluttes SMTP tilstandsløst per tilkobling: Hver tilkobling håndterer én eller flere fullstendige transaksjoner (MAIL FROM, RCPT TO, DATA) og avsluttes med QUIT. Det finnes ingen tilstand som må ligge på samme målsystem på tvers av tilkoblinger. Hvilket system på leverandørsiden som tar imot neste tilkobling, er irrelevant for leveringen.

For det andre er det rett og slett ingenting å persistere på denne BIG-IP-en: Poolen inneholder nøyaktig ett medlem, leverandørens ene IP-adresse. En persistensprofil ville bare bruke minne på en persistens-tabell og koste et oppslag ved hver tilkobling, som alltid gir samme resultat. Riktig innstilling er derfor: Default Persistence Profile på None. Selv om leverandøren senere skulle publisere flere IP-adresser bak vertsnavnet, ville persistens være kontraproduktivt, fordi det ville hindre fordeling til disse adressene og belaste enkelte mål ensidig.

Avgjørende for gjennomstrømmingen ved masseutsendelse er avsenderens tilkoblingsprofil: få langvarige tilkoblinger med mange meldinger per tilkobling i stedet for en ny tilkobling per e-post; mer om dette nedenfor.

## Virtual Server: FastL4 i stedet for Full Proxy

For ren videresending av SMTP er en Performance-(Layer-4)-Virtual-Server med FastL4-profil det riktige valget. BIG-IP-en behandler da tilkoblingen i stor grad i maskinvare eller i den akselererte banen, uten å terminere TCP-tilkoblingen fullstendig. En standard Virtual Server i Full Proxy-modus gir bare merverdi dersom du faktisk vil gripe inn i datastrømmen på BIG-IP-en, for eksempel med en SMTP-sikkerhetsprofil eller iRules på protokollnivå. For en Outbound Proxy til egen avtalte leverandør er dette unødvendig og skaper bare flere feilkilder.

Viktig i begge tilfeller: Ikke aktiver en profil som skriver inn i tilkoblingen. STARTTLS forhandles direkte mellom utsendelsessystemene og leverandørens relé; enhver instans som endrer eller filtrerer byte, setter TLS-etableringen i fare.

## DNS-oppløsning: Leverandørens vertsnavn skal ligge som FQDN-node i poolen

Leverandøren har oppgitt et vertsnavn, ikke en IP-adresse. Den nærliggende refleksen å løse opp IP-adressen én gang og legge den statisk inn som node, er den dårligste varianten: Hvis leverandøren endrer adressen (vedlikehold, flytting, DR-tilfelle), stopper utsendelsen til noen tilpasser BIG-IP-konfigurasjonen. Det er nettopp derfor FQDN-noder finnes.

En FQDN-node lagrer vertsnavnet i stedet for adressen. BIG-IP-en løser selv opp navnet, oppretter en såkalt ephemeral node for hver returnerte adresse og oppdaterer dem automatisk når DNS-svaret endres. Som standard spør den navnet på nytt etter at DNS-TTL-en er utløpt; alternativt kan et fast spørreintervall angis. Med aktivert Autopopulate overtar poolen også automatisk flere A-records som medlemmer: Dersom leverandøren senere utvider innleveringen til flere adresser, følger BIG-IP-en dette uten konfigurasjonsendring.

To forutsetninger blir ofte glemt. For det første trenger BIG-IP-en fungerende DNS-servere i systemkonfigurasjonen (System, Configuration, Device, DNS); FQDN-noder bruker systemresolverne, ikke en DNS-cache fra en listener-profil. For det andre må disse resolverne faktisk være tilgjengelige fra management- eller TMM-konteksten, ellers forblir noden i statusen unresolved og poolen tom.

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

Utsendelsessystemene angir deretter 10.0.5.10 som smarthost. Om du bruker port 25 eller 587, bestemmes av leverandøren; BIG-IP-konfigurasjonen er identisk i begge tilfeller, bare porten endres.

## SNAT: fast adresse i stedet for Automap

For utgående e-posttrafikk skal kildeadressen være under kontroll. SNAT Automap bruker Floating Self-IP-en til det utgående VLAN-et, og den kan endres ubemerket ved nettverksendringer eller failover-ombygginger. Leverandører knytter imidlertid ofte innleveringen til IP-allowlisting, og selv uten formell allowlisting er omdømmet knyttet til kildeadressen. En dedikert SNAT-pool med en fast tildelt adresse gjør kilde-IP-en til et dokumentert, stabilt konfigurasjonsobjekt.

Når det gjelder kapasitet: Én enkelt SNAT-adresse tilbyr rundt 64'000 samtidige oversettelser mot ett enkelt mål (én IP, én port), fordi hver tilkobling får sin egen ephemeral kildeport. Med belastningsprofilen som beskrives her, med noen få dusin samtidige tilkoblinger, er dette tilstrekkelig med flere størrelsesordener. Portuttømming blir først et tema når en feilkonfigurert avsender åpner en ny tilkobling per e-post og ikke lukker disse ordentlig; da samler oversettelser seg i en TIME-WAIT-lignende tilstand. Slik oppførsel retter du hos avsenderen, ikke med en ekstra SNAT-adresse.

## Tidsavbrudd: den vanligste årsaken til tilkoblingsavbrudd under belastning

En bulk-avsender holder tilkoblinger åpne og skyver melding etter melding gjennom dem. Det kan oppstå pauser mellom to meldinger: Avsenderen genererer neste blokk, reléet forsinker mottaket (tarpitting, rester av greylisting, interne køer). Idle-timeout for FastL4-profilen er som standard satt til 300 sekunder. Hvis en pause varer lenger, rydder BIG-IP-en bort tilkoblingen, og avsenderen skriver til en tilkobling som ikke lenger finnes.

To innstillinger demper dette. For det første bør Idle-timeout settes til en verdi som er høyere enn realistiske pauser; for masseutsendelse er 600 sekunder en fornuftig startverdi. Verdien bør ikke settes vilkårlig høyt, ellers samler foreldreløse tilkoblinger seg i tilkoblingstabellen. For det andre bør Reset on Timeout være aktivert i profilen: BIG-IP-en kvitterer da oppryddingen med en TCP-reset, og den sendende MTA-en oppdager umiddelbart at tilkoblingen er borte, i stedet for å gå inn i et tidsavbrudd og først planlegge meldingen på nytt etter flere minutter.

Du har ingen innflytelse på tidsavbruddene hos motparten, men de hører med i bildet: Hvis leverandørens relé lukker tilkoblinger etter 120 sekunders inaktivitet, hjelper ikke et romslig BIG-IP-timeout. Det er den laveste timeout-verdien på hele banen som gjelder; ved tvil bør du spørre leverandøren og bruke denne verdien som planleggingsgrunnlag.

## Tilkoblingsstrategi: få tilkoblinger, mange meldinger

Uten leveringskrav fra leverandøren er det verdt å gjøre en kort beregning. 1000 e-poster per minutt er rundt 17 per sekund. En SMTP-transaksjon over en allerede etablert tilkobling tar betydelig under et halvt sekund ved normal latenstid. Med 10 til 20 parallelle tilkoblinger og for eksempel 100 meldinger per tilkobling før avsenderen fornyer dem, nås målraten enkelt. På leverandørsiden er det som regel tilgjengelig betydelig mer tilkoblingskapasitet, men den deles med alle andre kunder. Få langvarige tilkoblinger med mange transaksjoner er derfor ikke bare effektivt (TCP- og TLS-etablering faller bort per melding), men også den mest skånsomme måten å bruke fremmed infrastruktur på.

Innstillingene for dette ligger i utsendelsessystemet, ikke på BIG-IP-en: maksimalt antall meldinger per tilkobling, maksimalt antall parallelle tilkoblinger til smarthost og gjenbruk av etablerte tilkoblinger. På BIG-IP-en kan hele oppsettet sikres med et Connection Limit på pool-medlemmet, for eksempel 200 samtidige tilkoblinger: I normal drift nås verdien aldri, men en feilkonfigurert avsender som plutselig åpner én tilkobling per e-post, oversvømmer dermed ikke leverandørens relé ukontrollert. Grensen er et sikkerhetsnett, ikke et styringsverktøy.

Om den konfigurerte tilkoblingsprofilen faktisk oppnås i praksis, viser målingen: Tilkoblinger per minutt og meldinger per tilkobling kan evalueres fra Message Tracking eller Connector-loggene, slik det beskrives i artikkelen [Fastslå belastningsprofilen til en e-postserver](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln) beskrevet. For en belastningstest med realistisk bulk-belastningsprofil (få sesjoner, mange meldinger per sesjon) passer smtp-source fra Postfix-pakken bedre enn HTTP-orienterte belastningsverktøy, fordi det genererer nettopp denne tilkoblingsprofilen.

## Overvåking: Ikke belast leverandøren med helsesjekker

En monitor på pool-medlemmet er fornuftig slik at BIG-IP-en oppdager feil på leverandørsiden og rapporterer dem korrekt. Her gjelder følgende: Hver helsesjekk er en reell tilkobling til leverandøren og teller der mot de samme grensene som nyttetrafikken. En enkel TCP-monitor med moderat intervall (30 sekunder eller mer) er fullt tilstrekkelig. En fullstendig SMTP-monitor som sjekker helt frem til banneret eller EHLO, gir knapt ytterligere innsikt, men oppretter loggoppføringer hos leverandøren og kan i verste fall føre til spørsmål om hvorfor det kommer en tilkobling uten e-post hvert femte sekund.

## Sjekkliste

| Innstilling | Anbefaling |
|---|---|
| Persistensprofil | None; Sticky Sessions gir ingenting for SMTP, og enda mindre i en pool med ett medlem |
| Virtual Server-type | Performance (Layer 4) med FastL4-profil, ingen inngripen i datastrømmen |
| Målnode | FQDN-node med Autopopulate i stedet for statisk IP; DNS-servere konfigurert på BIG-IP-en |
| SNAT | dedikert SNAT-pool med fast adresse kjent for leverandøren; ingen Automap |
| Idle-timeout | over de reelle sendepausene, startverdi 600 s; Reset on Timeout aktiv |
| Connection Limit | som sikkerhetsnett på pool-medlemmet, f.eks. 200 |
| Monitor | TCP, intervall 30 s eller mer; ingen aggressiv SMTP-monitor |
| Avsenderkonfigurasjon | få parallelle tilkoblinger, mange meldinger per tilkobling; gjenbruk aktivert |

Det korte svaret på det opprinnelige spørsmålet er altså: Nei, Sticky Sessions er ikke bedre, de er virkningsløse til skadelige i denne konstellasjonen. Kvaliteten på løsningen avgjøres av DNS-oppløsningen av leverandørens vertsnavn, en stabil SNAT-adresse, tidsavbrudd som passer belastningsprofilen og at utsendelsessystemene leverer sine 1000 e-poster per minutt over få etablerte tilkoblinger i stedet for over tusen enkeltstående.

## Kilder

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Avsnitt 4.5.4 og transaksjonsmodellen viser at flere e-posttransaksjoner over én tilkobling er det tiltenkte normaltilfellet.

2.  [K7820: Overview of SNAT features](https://my.f5.com/manage/s/article/K7820): F5-grunnleggende artikkel om SNAT, SNAT-pooler og portoversettelse per mål.

3.  [tmsh-referanse: ltm node](https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_node.html): dokumenterer FQDN-alternativene (name, autopopulate, interval) for noder og dermed for pool-medlemmer.

4.  [smtp-source(1), Postfix](https://www.postfix.org/smtp-source.1.html): belastningsgenerator som etterligner bulk-avsenderens tilkoblingsprofil (få sesjoner, mange meldinger).

5.  [Fastslå belastningsprofilen til en e-postserver](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln): egen veiledning for hvordan tilkoblinger per minutt og meldinger per tilkobling evalueres fra Message Tracking.
