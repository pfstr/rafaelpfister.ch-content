---
title: "Securing Midea PortaSplit in Home Assistant: Token, key and home network"
navTitle: "Secure PortaSplit"
description: "The PortaSplit token and key come from the Midea Cloud and never expire. Learn how to secure these values, isolate the device on your home network, and keep Home Assistant, the integration and firmware up to date in a controlled manner."
date: "2026-07-24"
kategorie: "Home Assistant and IoT"
timeToRead: "14 min read"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant
  - midea-v2-cloud-api-portasplit-home-assistant
image: "../images/midea-portasplit-home-assistant/portasplit-dashboard.png"
slug: "midea-portasplit-i-home-assistant-hvorfor-token-og-nokkel-er-avgjorende"
translationOf: "midea-portasplit-home-assistant-absichern"
translationId: article-a02e26cce22063f1
translationReview: automatic
translationSourceHash: d9e59ab5647df261814123d662df93c68c9f6cd35c20cec518de0a9cf54d6e22
translatedAt: 2026-10-31T03:40:29.861Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/midea-portasplit-i-home-assistant-hvorfor-token-og-nokkel-er-avgjorende
---

<aside class="article-update">
  <p class="article-update__label">Hva PortaSplit-eiere bør gjøre nå</p>
  <p>Home Assistant henter PortaSplit-tokenet og -nøkkelen via private skygrensesnitt under oppsettet. Midea AC LAN-prosjektet har advart om mulige endringer siden 19. mai 2025; noen dato for nedstenging fra produsenten er ikke dokumentert. For eiere betyr dette:</p>
  <ol>
    <li><strong>Sikkerhetskopier token, nøkkel og konfigurasjon kryptert.</strong> Hvis hentingen senere ikke lenger fungerer, er sikkerhetskopien den eneste veien til gjenoppretting.</li>
    <li><strong>Ikke opphev sammenkoblingen uten god grunn.</strong> Fabrikkinnstillinger, fjerning fra Midea-kontoen eller bytte av Wi-Fi-modul krever at tokenet hentes på nytt.</li>
    <li><strong>Isoler PortaSplit i hjemmenettverket.</strong> Ingen portvideresending, eget IoT-VLAN, tilgang bare fra Home Assistant.</li>
  </ol>
</aside>

Lokal styring av Midea PortaSplit bygger på to enhetsspesifikke verdier: token og nøkkel. De autentiserer forbindelsen mellom Home Assistant og enheten og kan for tiden bare hentes via Midea-skyen. Dette gir to oppgaver: å sikre verdiene slik at ny konfigurering fortsatt er mulig uten skyen, og å drifte både enheten og Home Assistant slik at verdiene gjør minst mulig skade også ved en hendelse.

Serien har tre deler: [Del 1](/blog/midea-portasplit-home-assistant) beskriver oppsettet frem til dashbordet, denne delen handler om sikring, og [del 3](/blog/midea-v2-cloud-api-portasplit-home-assistant) forklarer bakgrunnen for advarslene om sky-API-et.

![Home Assistant-dashbord for Midea PortaSplit i kjøledrift: nøkkeltall øverst, termostat på 22 °C, historikk for romtemperatur, effektforbruk, energiforbruk per dag, kompressorfrekvens, kompressordrift og viftehastighet, med tekniske verdier og status under.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## Hvor token og nøkkel kommer fra

På enheter med V3-protokollen godtar PortaSplit bare lokale kommandoer med token og nøkkel. Verdiene genereres ikke av enheten, men av Midea-skyen; også den offisielle appen henter dem derfra. Fellesskapsintegrasjonene har reimplementert dette sky-kallet: De logger inn på de samme endepunktene som appen, mottar token og nøkkel og lagrer begge lokalt. Etter dette er det ikke nødvendig med skyforbindelse for løpende drift.

Det finnes ingen dokumentert lokal sammenkoblingsmekanisme som gir ut verdiene uten skyen. Teoretisk kan de leses ut fra appen, for eksempel gjennom reverse engineering eller instrumentering under kjøring; for den enkelte brukeren er dette omfattende og ingen erstatning for skyhenting. Hvis endepunktet forsvinner, forsvinner derfor også muligheten til å hente dem.

Prosjektet `Midea AC LAN` advarer i README-filen sin om at Midea i fremtiden kan stenge token-grensesnittene gradvis. Siden revisjonen av README-filen 3. september 2026 henter integrasjonen token og nøkkel via Meiju- og SmartHome-skyen. Ifølge README fungerer dette bare med en personlig konto som enheten er knyttet til; det er for tiden ikke mulig å hente tokens via NetHome Plus. Siden 9. september 2026 peker README-filen dessuten på at en SmartHome-konto migrert fra en annen app ikke leverer tokens; det kreves en nyregistrert konto med nylig tillagte enheter. Allerede konfigurerte enheter fortsetter å fungere lokalt; nye enheter og nye oppsett ville være berørt. Dette er ikke en bindende veikart fra Midea. I juni 2026 viste det seg dessuten at det angivelig lukkede SmartHome-token-API-et fortsatt fungerte; forespørselen fra fellesskapsbiblioteket var bare ufullstendig. En vurdering av advarselen og de forskjellige «V2»-betegnelsene finnes i [del 3](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Hva token og nøkkel muliggjør

Token og nøkkel har ingen utløpsdato. Ifølge `Midea AC LAN` ble klientkommunikasjonen opprinnelig ansett som tilstrekkelig beskyttet, og derfor utstedte skyen tokens som aldri utløper. I seg selv er dette ikke en sårbarhet; det blir problematisk hvis verdiene havner i logger eller ubeskyttede sikkerhetskopier, kommer på avveie eller verken kan tilbakekalles eller roteres.

Den som har token og nøkkel og kan nå enheten på nettverket, kan autentisere seg mot PortaSplit, lese statusinformasjon, slå den av og på, bytte driftsmodus og endre innstilt temperatur. Verdiene alene muliggjør ikke et angrep fra internett; angriperen trenger også en nettverksforbindelse til enheten. Token og nøkkel bør derfor behandles som et passord, og nettverket bør så langt som mulig bare tillate denne forbindelsen for Home Assistant.

Fellesskapsintegrasjonen angriper ikke klimaanlegget. Den implementerer en proprietær protokoll som er kartlagt gjennom reverse engineering. Risikoen oppstår fordi langvarige hemmeligheter lagres utenfor den tiltenkte appen.

## Sikre token, nøkkel og konfigurasjon

Sikkerhetskopiering av token, nøkkel og konfigurasjon er det viktigste engangstiltaket: Når token-grensesnittene i skyen først er stengt, er en sikkerhetskopi den eneste veien til et nytt oppsett. `Midea AC LAN` lagrer en JSON-konfigurasjonsfil for V3-enheter etter vellykket oppsett. Den dokumenterte stien er:

```text
/config/.storage/midea_ac_lan/
```

Filen har enhets-ID-en som filnavn:

```text
<device-id>.json
```

Denne filen er ikke et vanlig tekstnotat. Den kan inneholde enhets-ID, serienummer, IP-adresse, token, nøkkel, protokollinformasjon samt sky- og enhetsparametere. Derfor gjelder følgende:

- Ikke last den opp til et offentlig GitHub-repositorium.
- Ikke legg den ut i forum.
- Ikke del den som et usensurert skjermbilde.
- Ikke send den via ukryptert e-post.

Selv et privat Git-repositorium er ikke automatisk riktig lagringssted, fordi hemmeligheter blir værende i Git-historikken selv om de senere slettes fra den gjeldende filen. Bedre alternativer er en kryptert sikkerhetskopi, en passordbehandler med filvedlegg, en kryptert NAS-sikkerhetskopi, et kryptert frakoblet medium eller et kryptert arkiv med passordet lagret separat.

For sikkerhetskopiering via Home Assistant-terminalen:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Vis filen:

```bash
cat <device-id>.json
```

Filen bør ikke overføres via en offentlig nettjeneste for kopiering. Et kryptert arkiv er bedre, og kan deretter legges i en kryptert sikkerhetskopi:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

Filene i `.storage` bør ikke redigeres manuelt. Utvikleren anbefaler uttrykkelig at JSON-filen verken slettes eller endres direkte ved problemer, men at den får nytt navn og sikkerhetskopieres før endringer.

En fullstendig Home Assistant-sikkerhetskopi inneholder også disse filene. En separat kopi er likevel fornuftig, fordi Home Assistant-sikkerhetskopier kan bli skadet, en gjenoppretting kan overskrive integrasjonen, filen kan være nødvendig spesifikt ved senere ny konfigurering, og en sikkerhetskopi aldri bør ligge bare på samme system.

### Fjerne hemmeligheter fra et publisert Git-repositorium

Hvis en JSON-fil ved en feil er publisert på GitHub, er det ikke nok å slette den normalt og gjøre en ny commit. Filen forblir tilgjengelig i Git-historikken. Minst disse trinnene er nødvendige:

1. Gjør repositoriet privat umiddelbart, dersom mulig.
2. Fjern filen fra hele Git-historikken.
3. Ta hensyn til GitHub-cacher og forks.
4. Behandle tokenet som kompromittert.
5. Fjern enheten fra Midea-kontoen og koble den til på nytt dersom dette genererer nye nøkler.
6. Konfigurer Home Assistant-integrasjonen på nytt.
7. Endre passordet for Midea-kontoen dersom påloggingsopplysningene også var berørt.

Om ny sammenkobling faktisk genererer et nytt token, varierer avhengig av enhet og skyarkitektur. Du bør ikke stole på at endring av kontopassordet automatisk gjør det lokale enhetstokenet ugyldig.

## Isoler PortaSplit i nettverket

### Ingen portvideresending til PortaSplit

Den vanligste unngåelige feilen ville være å gjøre den lokale enhetsporten direkte tilgjengelig fra internett. En regel som denne ville være farlig:

```text
Internet → TCP 6444 → PortaSplit
```

Det finnes ingen god grunn til å gjøre PortaSplit direkte tilgjengelig fra internett. Home Assistant er allerede i det lokale nettverket og fungerer som en kontrollerende instans. Ruteren bør ikke ha noen portvideresending til PortaSplit, UPnP bør om mulig begrenses eller deaktiveres, innkommende forbindelser bør blokkeres som standard, og ingen DMZ-åpning bør brukes for enheten.

### Eget IoT-VLAN

Den beste nettverksarkitekturen er et separat IoT-nettverk:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

PortaSplit befinner seg i IoT-VLAN-et. Home Assistant kan få målrettet tilgang til enheten, men PortaSplit skal ikke kunne få vilkårlig tilgang til PC-er, NAS og andre interne systemer. En mulig brannmurlogikk:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Under første gangs oppsett trenger enheten internettilgang til Midea-skyen. Etter vellykket lokalt oppsett kan det testes om utgående internettilgang kan blokkeres. Det bør ikke settes en endelig sperre umiddelbart. Kontroller først om lokal styring fortsatt fungerer, om enheten forblir tilgjengelig etter omstart, om den tåler omstart av ruteren, om den fortsatt reagerer etter flere dager, om MSmartHome-appen fortsatt trengs og om fastvareoppdateringer fortsatt tilbys. De som vil fortsette å bruke skyen og fastvareoppdateringer, kan tillate utgående internettilgang midlertidig og deretter blokkere den igjen.

### Nettverkssegmentering kan hindre enhetsoppdagelse

Automatisk enhetsoppdagelse bygger ofte på broadcast- eller multicast-trafikk, og denne rutes normalt ikke over VLAN-grenser. Home Assistant finner derfor kanskje ikke PortaSplit automatisk, selv om en vanlig IP-forbindelse ville vært tillatt.

Da kan du midlertidig sette opp PortaSplit i samme VLAN som Home Assistant, oppgi enhets-IP-en manuelt, bruke en egnet broadcast-reléfunksjon eller definere målrettede brannmurregler etter oppsettet. Manuell konfigurasjon er ofte til og med det bedre alternativet fra et sikkerhetsperspektiv, fordi det ikke krever at ekstra broadcast-trafikk tillates mellom nettverkene.

### Statisk DHCP-tilordning

PortaSplit bør få en fast DHCP-tilordning i ruteren:

```text
PortaSplit → 192.168.30.25
```

En DHCP-reservasjon er vanligvis å foretrekke fremfor en statisk IP satt på enheten. Home Assistant finner enheten pålitelig, brannmurregler kan begrenses til en fast adresse, feilsøkingen blir enklere, og tilordningen forblir stabil etter omstart av ruter eller enhet. En brannmurregel kan dermed formuleres svært restriktivt:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

Den faktisk nødvendige porten må verifiseres ut fra integrasjonen og din egen enhet.

## Sikre Home Assistant og integrasjoner

### Home Assistant som sentralt tillitsanker

Den som styrer PortaSplit lokalt, flytter deler av tilliten fra Midea-skyen til Home Assistant. Hvis Home Assistant kompromitteres, kan en angriper potensielt kontrollere ikke bare klimaanlegget, men hele smarthjemmet.

Home Assistant bør derfor oppdateres regelmessig, ikke publiseres via ubeskyttet portvideresending, beskyttes med et sterkt og unikt passord, bruke flerfaktorautentisering, opprette krypterte sikkerhetskopier, bare inneholde nødvendige tillegg og ikke tillate unødvendig SSH-tilgang fra internett. For fjerntilgang er VPN, Home Assistant Cloud eller en korrekt konfigurert reverse proxy bedre alternativer enn enkel portvideresending til port 8123.

### HACS og risikoen i forsyningskjeden

`Midea Smart AC` og `Midea AC LAN` er egendefinerte integrasjoner. De kjører i Home Assistant og får dermed omfattende tilgang til kjøretidsmiljøet. En ondsinnet eller kompromittert integrasjon kan teoretisk lese konfigurasjonsdata, hente ut hemmeligheter, opprette nettverksforbindelser, skanne enheter i det lokale nettverket, lese tilstander fra andre entiteter, overføre data til eksterne systemer og påvirke tilgjengeligheten til Home Assistant.

Dette betyr ikke at de nevnte integrasjonene er ondsinnede. Begge prosjektene er offentlig tilgjengelige, utvikles aktivt og har et synlig fellesskap. Open source er imidlertid ingen automatisk sikkerhetsgaranti. Før installasjon er det minst verdt å se på om repositoriet vedlikeholdes aktivt, om det finnes jevnlige utgivelser, hvor mange personer som bidrar til koden, om det finnes åpne sikkerhetssaker, om vedlikeholdere eller repository-eiere nylig er byttet, om HACS peker til forventet repositorium og om en oppdatering inneholder uvanlig store eller uforklarlige endringer.

Oppdateringer bør ikke installeres blindt umiddelbart etter utgivelse. Særlig for sikkerhetskritiske smarthjemsystemer er det fornuftig å vente noen dager og kontrollere versjonsnotater og rapporterte problemer.

### Feilsøkingslogger inneholder sensitive data

Ved problemer ber open source-prosjekter ofte om feilsøkingslogger. Dokumentasjonen til `Midea AC LAN` viser hvordan logging aktiveres for de to relevante komponentene:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Deretter kan loggene lastes ned via Innstillinger, System og Logger. Slike logger kan, avhengig av integrasjonen og feiltilfellet, inneholde lokale IP-adresser, enhets-ID, serienummer, modellidentifikasjon, skysvar, kontoinformasjon, token eller deler av det, nettverkspakker samt tidsstempler og bruksmønstre. Før de lastes opp til en offentlig GitHub-sak, må de derfor kontrolleres og sensitive verdier sladdes.

Når feilsøkingen er avsluttet, bør feilsøkingsloggingen fjernes igjen. Permanent aktivert feilsøkingslogging øker ikke bare lagringsforbruket, men også mengden sensitiv informasjon i sikkerhetskopiene.

## Sky og fastvare

### Sikre skykontoen

Så lenge Midea-skyen brukes til oppsett eller appstyring, forblir Midea-kontoen en del av sikkerhetsmodellen. Dette innebærer et unikt passord som ikke deles med andre tjenester, en passordbehandler, flerfaktorautentisering dersom den tilbys, fjerning av gamle smarttelefoner og økter, å unngå delte kontoer og jevnlig kontroll av hvilke enheter som er registrert på kontoen.

Hvis Home Assistant-integrasjonen ber om brukernavn og passord under oppsettet, bør det kontrolleres om påloggingsopplysningene bare brukes til engangshenting av tokenet eller lagres permanent. Utviklerne av `Midea Smart AC` skriver at enheter etter oppsettet ikke kobles til innebygde integrasjonskontoer, og at token og nøkkel også kan hentes manuelt via egen konto ved hjelp av CLI. Der det er mulig, bør egen konto foretrekkes fremfor fremmede eller integrerte felleskontoer. For `Midea AC LAN` har en personlig Meiju- eller SmartHome-konto som enheten er knyttet til, vært et krav for tokenhenting siden september 2026.

### Blokkere skyen eller ikke?

Etter vellykket oppsett oppstår spørsmålet om internettilgangen til PortaSplit bør blokkeres fullstendig. For en sperre taler mindre telemetri, mindre avhengighet av eksterne tjenester, en mindre angrepsflate via produsentens sky, at enheten ikke kan kontakte vilkårlige eksterne mål, og mindre påvirkning fra endringer på skysiden.

Mot dette taler at MSmartHome-appen kanskje ikke lenger fungerer utenfor hjemmenettverket, at fastvareoppdateringer ikke lastes ned, at klokkeslett- eller skyfunksjoner kan falle bort, at ny innlogging eller gjenoppretting blir vanskeligere, og at enkelte enheter kan reagere uventet etter lengre tid uten nett.

En pragmatisk rekkefølge: Konfigurer enheten normalt, test Home Assistant og appen, sikkerhetskopier token og konfigurasjon, blokker internettilgang, start enheten og Home Assistant på nytt, observer i flere dager, og gi ved behov internettilgang igjen bare midlertidig.

### Fastvareoppdateringer: sikkerhetsgevinst eller integrasjonsrisiko?

Fastvareoppdateringer er et dilemma for IoT-enheter. De kan lukke kjente sårbarheter, forbedre stabiliteten, modernisere sikkerhetsmekanismer og gi nye funksjoner. Men de kan også endre lokale grensesnitt, ødelegge reverse-engineering-integrasjoner, gjøre tokens ugyldige, deaktivere det lokale API-et og innføre nye skyavhengigheter.

PortaSplit-fastvaren som ble levert i januar 2026, fikk for eksempel en ny stillemodus for utedelen som reduserer støynivået med rundt 6 desibel. Fellesskapsintegrasjonene måtte først kartlegge og implementere denne, dokumentert i en egen GitHub-sak for PortaSplit.

Dette innebærer: Ikke forhindre fastvareoppdateringer som hovedregel, men undersøk før en oppdatering om andre Home Assistant-brukere rapporterer problemer, sikkerhetskopier konfigurasjon og token på forhånd, opprett en Home Assistant-sikkerhetskopi og test den lokale styringen fullstendig etter oppdateringen. Sikkerhet betyr ikke «aldri oppdater». Utdatert fastvare kan være farligere enn en midlertidig inkompatibel integrasjon.

### Hva Midea selv sier om sikkerhet

Midea markedsfører SmartHome-økosystemet sitt med orientering mot flere sikkerhets- og personvernstandarder, blant annet EN 303 645, UK PSTI, NIST, GDPR-kompatibel databehandling og kravene i EUs Radio Equipment Directive. Dette er positive signaler, men ingen uttalelse om hvordan hver enkelt PortaSplit-fastvare, hvert skyendepunkt og hvert lokale API faktisk er implementert. Sertifiserings- og markedsføringspåstander erstatter ikke teknisk kontroll av den konkrete enheten.

På samme måte ville det være feil å utlede fra advarselen fra en fellesskapsintegrasjon at PortaSplit generelt er usikker. Det beskrevne problemet gjelder arkitekturen med langvarige tokens og bruken av dem i uoffisielle klienter.

## Risiko etter scenario

| Scenario | Risiko | Begrunnelse |
| --- | --- | --- |
| Vanlig hjemmenettverk uten portvideresending | oversiktlig | En angriper må først få tilgang til Wi-Fi, Home Assistant eller en sikkerhetskopi. |
| Flatt hjemmenettverk med mange usikre IoT-enheter | middels | En kompromittert annen IoT-enhet kan nå PortaSplit eller Home Assistant i samme nettverk. |
| PortaSplit direkte tilgjengelig fra internett | høy | Enheten bør aldri publiseres via portvideresending. |
| Token og nøkkel offentlig på GitHub | høy | Hemmelighetene anses som kompromittert; det er ikke garantert at de kan tilbakekalles. |
| Separat IoT-VLAN, restriktiv brannmur, lokal styring | forholdsvis lav | Selv ved en sårbarhet i enheten er bevegelsesfriheten i nettverket sterkt begrenset. |

## Sjekkliste

```text
1. Home-Assistant-Backup anfertigen
2. Token- und Konfigurationsdaten verschlüsselt sichern
3. DHCP-Reservation für die PortaSplit einrichten
4. Keine Portweiterleitung, UPnP einschränken
5. PortaSplit in ein separates IoT-VLAN verschieben
6. Zugriff von Home Assistant zur PortaSplit erlauben
7. Zugriff der PortaSplit auf interne Netze blockieren
8. Internetzugriff testweise blockieren
9. lokale Steuerung nach Neustarts prüfen
10. Firmware- und Integrationsupdates kontrolliert durchführen
```

Ønsket kommunikasjonsretning:

```text
Home Assistant
    │
    │ gezielt erlaubt
    ▼
Midea PortaSplit
    │
    ├── kein Zugriff auf PCs
    ├── kein Zugriff auf NAS
    ├── kein Zugriff auf Management-Netz
    └── Internet nur bei Bedarf
```

Drevet på denne måten er lokal styring forsvarlig fra et sikkerhetsperspektiv: Token og nøkkel holdes hemmelige og sikret, enheten er bare tilgjengelig for Home Assistant, og oppdateringer av fastvare og integrasjon installeres kontrollert.

## Endringslogg

- **25. oktober 2026:** Lagt til: Siden README-revisjonen i september 2026 krever Midea AC LAN en personlig Meiju- eller SmartHome-konto som enheten er knyttet til, for tokenhenting; NetHome Plus og migrerte SmartHome-kontoer leverer ikke tokens.

## Kilder

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integrasjon `Midea AC LAN` med «Important Notice» (siden 19. mai 2025, revidert 3. og 9. september 2026), begrunnelsen knyttet til tokens som ikke utløper og beskrivelsen av skybasert tokenhenting via en personlig Meiju- eller SmartHome-konto.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integrasjon `Midea Smart AC`: skybasert henting av token og nøkkel på V3-enheter, lokal lagring av verdiene, standardport 6444.

3.  [midea_ac_lan: Feilsøkings- og konfigurasjonsmerknader](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): Lagring av enhetskonfigurasjon under `/config/.storage/midea_ac_lan/`, anbefaling om å sikkerhetskopiere i stedet for å slette JSON-filen og loggerkonfigurasjon for feilsøkingslogger.

4.  [Issue 779: Stillemodus for PortaSplit-utedelen](https://github.com/wuwentao/midea_ac_lan/issues/779): Forespørsel om støtte for stillemodusen for utedelen som ble innført med fastvareoppdateringen i januar 2026, og som reduserer støynivået med rundt 6 desibel.

5.  [Midea SmartHome](https://www.midea.com/global/smarthome): Produsentopplysninger om sikkerhets- og personvernstandardene EN 303 645, PSTI, NIST, GDPR og RED DA.

6.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): Installasjon og administrasjon av egendefinerte integrasjoner som ikke er en del av Home Assistant Core.
