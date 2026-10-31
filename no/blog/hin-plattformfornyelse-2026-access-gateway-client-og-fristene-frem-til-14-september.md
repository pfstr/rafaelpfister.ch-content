---
title: "HIN-plattformfornyelse 2026: Access Gateway, Client og fristene frem til 26. oktober"
navTitle: "Access Gateway 2026"
description: "Firewall-åpning innen 31. august, Access Gateway versjon 4 fra september, SAML-endepunkter innen 12. oktober, maskinvaretokener og HIN Client 4.0 innen 26. oktober. Mailgatewayet er ikke berørt og erstattes separat."
date: "2026-08-01"
kategorie: "HIN-Gateway"
timeToRead: "5 min lesetid"
themen:
  - hin-gateway
  - active-directory-entra
related:
  - hin-mailgateway-backup-disaster-recovery
  - hin-update-issue-version-15.0.5
slug: "hin-plattformfornyelse-2026-access-gateway-client-og-fristene-frem-til-14-september"
translationId: "article-106aa61d54408397"
translationOf: hin-plattformerneuerung-2026
translationSourceHash: cd34e6297afca9491bd965e40dbaf55b920296e62e57726754824474fcb27f0d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:47:23.019Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/hin-plattformfornyelse-2026-access-gateway-client-og-fristene-frem-til-14-september
---

# HIN-plattformfornyelse 2026: Access Gateway, Client og fristene frem til 26. oktober

HIN fornyer plattformen for identitet og tilgang i 2026. HIN fastsatte fristene på nytt 27. august 2026: Den første fristen utløp 31. august 2026, de gamle SAML-endepunktene har ikke vært tilgjengelige siden 12. oktober 2026, og den store omstillingen følger 26. oktober 2026.

**Berørt er HIN Access Gateway (AGW), HIN Client og autentiseringsmidlene. HIN Mailgateway er ikke berørt.** Det blir også erstattet, men i et separat prosjekt med egen fremdriftsplan.

<div class="choice-row">
  <a class="choice" href="#die-fristen">
    <span class="choice__label">Din situasjon</span>
    <span class="choice__title">Kun AGW i drift</span>
    <span class="choice__hint">Fristene nedenfor er hele handlingsbehovet ditt. →</span>
  </a>
  <a class="choice" href="/stargate">
    <span class="choice__label">Din situasjon</span>
    <span class="choice__title">I tillegg migreringsbehov for Mailgateway</span>
    <span class="choice__hint">Da venter også erstatningen med «Stargate», med bred utrulling fra tredje kvartal 2026. Gratis gjennomgang av miljøet ditt. →</span>
  </a>
</div>

## Fristene

| Dato | Tiltak | Berører |
|---|---|---|
| 31.08.2026 | Firewall-åpning for `idp.id.hin.ch` og `cdn.hin.ch` (443/tcp) | AGW-operatører |
| fra september 2026 | Automatisk installasjon av AGW versjon 4 (unntatt klyngeinstallasjoner) | AGW-operatører |
| september til 26.10.2026 | Automatisk oppdatering til HIN Client 4.0 i puljer | Alle Client-brukere |
| 12.10.2026 | Gamle SAML-endepunkter ikke lenger tilgjengelige | Føderasjoner, EPD-tilkoblinger |
| 26.10.2026 | Maskinvaretokener og testidentiteter for EPD-tilgang utløper | Token-brukere, integrasjon |
| 26.10.2026 | Tidligere Authenticator App kan brukes for siste gang, ny konfigurasjon fra 27.10.2026 | App-brukere |
| 26.10.2026 | HIN Client 4.0 kreves | Alle Client-brukere |
| fra begynnelsen av november 2026 | Innlogging via nettleseren i HIN Client | Alle Client-brukere |

## Access Gateway er ikke Mailgateway

Begge har Gateway i navnet og forveksles jevnlig. Access Gateway styrer tilgangen til HIN-beskyttede applikasjoner og berører ikke e-posttrafikken. Mailgateway står i e-postflyten og krypterer meldinger.

## Access Gateway: firewall og versjon 4

Innen 31. august 2026 måtte AGW kunne nå `idp.id.hin.ch` og `cdn.hin.ch` via 443/tcp. HIN anbefaler å sette regelen på vertsnavnene i stedet for IP-adressene, fordi IP-adressene kan endres. Dette er en firewall-endring, ikke en innstilling i gatewayet, og ligger derfor ofte hos den nettverksansvarlige fremfor gateway-administratoren.

Siden september 2026 har versjon 4 blitt installert automatisk; klyngeinstallasjoner er unntatt, og HIN tar direkte kontakt med disse kundene. Forutsetninger: AGW på versjon 3.1.50 eller nyere, og Kerberos aktivert som autentiseringsmetode. For tilkobling til Active Directory kreves en LDAP-konto med leserettigheter.

De som ikke oppfyller forutsetningene, blir ikke oppdatert, og erfaringen viser at dette først oppdages når ingen lenger kan logge inn. Det er derfor bedre å kontrollere versjonen nå enn først ved en driftsforstyrrelse.

## SAML: nye endepunkter, færre attributter

```text
Föderationsdienst
  broker.hin.ch/realms/HINBroker/protocol/saml/descriptor

EPD-Zugang
  idp.id.hin.ch/auth/realms/hinid/protocol/saml/descriptor
```

Med overgangen endres attributtformater og bindinger. Attributtmengden reduseres til GLN, navn, fødselsdato og kjønn.

Dette er punktet der integrasjoner feiler. Alle applikasjoner som bruker ytterligere attributter for roller eller klientseparasjon, mottar dem ikke lenger via de nye endepunktene; de gamle har ikke vært tilgjengelige siden 12. oktober 2026. Feilen viser seg ikke som en innloggingsfeil, men som manglende rettighet i målsystemet.

Testidentiteter for EPD-tilgang mister gyldigheten 26. oktober 2026 og må søkes om på nytt. For EPD-integrasjon krever HIN identiteter kontrollert ved videoidentifikasjon; et testmiljø er ennå ikke tilgjengelig. Fra 26. oktober 2026 kan det dessuten komme assertions uten GLN.

De som drifter en føderasjon, drifter nesten alltid også egen e-postinfrastruktur. For disse organisasjonene faller plattformfornyelsen og [erstatningen av Mailgateway med «Stargate»](/stargate) i samme år: teknisk uavhengige, men konkurrerende om de samme personene og vedlikeholdsvinduene.

## Token, app og HIN Client 4.0

Maskinvaretokener utstedes ikke lenger og utløper 26. oktober 2026. Alternativer: HIN Client, SMS-kode eller Authenticator App. Den tidligere Authenticator App kan brukes til 26. oktober 2026; den nye appen kan konfigureres som en annen faktor i selvbetjeningsportalen fra 27. oktober 2026. HIN anbefaler å sette opp innlogging med passord og SMS-kode som nødtilgang på forhånd.

HIN Client oppdateres automatisk til versjon 4.0 i puljer fra september til 26. oktober 2026. Hvis Client ikke viser versjon 4.0 den 26. oktober, må den installeres manuelt via `download.hin.ch`. Innlogging via nettleseren følger fra begynnelsen av november 2026.

Det kritiske punktet er systemkravene: **Versjon 4.0 krever Windows 11 med TPM eller macOS 14**, og på Intel-Macer i tillegg en Security Chip; uten TPM gjenstår innlogging med SMS-kode. Eldre enheter må oppdateres eller erstattes på forhånd. For en del praksiser er fristen dermed ikke en programvareoppgave, men en anskaffelsesoppgave. Den som først oppdager dette rett før 26. oktober, må håndtere leveringstider og nyinstallasjon av praksisprogramvaren.

## Fem spørsmål for å fastslå situasjonen

1. Hvilken AGW-versjon kjører, og er Kerberos aktiv?
2. Tillater firewallen utgående tilgang til `idp.id.hin.ch`?
3. Hvor mange arbeidsstasjoner kjører fortsatt Windows 10 eller macOS 13 og eldre?
4. Hvor mange maskinvaretokener er i bruk, og hva bytter de berørte til?
5. Bruker en applikasjon HIN-attributter som snart faller bort?

Svarene på 3 og 5 avgjør omfanget. Resten er unnagjort på noen få timer og dokumentert av HIN.

## Det andre prosjektet: «Stargate»

Uavhengig av dette erstatter HIN Mailgateway med det nye HIN Gateway, internt prosjekt «Stargate», teknisk sett en Data Mesh-tilnærming med ende-til-ende-kryptering og desentralisert nøkkelhåndtering. Dette er ikke et bytte av appliance, men et arkitekturskifte.

Arbeidsomfanget ligger dermed på et helt annet nivå. Plattformfornyelsen krever først og fremst at fristene overholdes for en firewall-regel, en versjon og et enhetsbytte, mens det ved Stargate er selve e-postflyten i produksjon som settes i spill: det etablerte regelverket, nøkkelmaterialet, behandlingen av mottakere uten HIN-identitet og spørsmålet om hva man faller tilbake på dersom noe ikke går som forventet. Ettersom migreringen skjer i bestilte firetimersvinduer og HIN anbefaler én måneds forberedelse, tåler en slik avtale ingen uavklarte punkter.

<aside class="offer-box">
  <span class="offer-box__tag">Gratis gjennomgang</span>
  <p><strong>Du trenger ikke vite hvor du står. Det er nettopp det gjennomgangen er til for.</strong> Jeg ser på det eksisterende gateway-miljøet ditt og forteller deg hva som må gjøres før migreringsvinduet, uavhengig av om du deretter migrerer selv eller får bistand.</p>
  <a class="offer-box__cta" href="/stargate">Registrer deg nå</a>
</aside>

## Endringslogg

- **25. oktober 2026:** Fristene oppdatert i henhold til HINs nye fastsettelse 27. august 2026: Firewall-åpning (nå også for cdn.hin.ch) innen 31. august, gamle SAML-endepunkter innen 12. oktober, maskinvaretokener, tidligere Authenticator App og HIN Client 4.0 innen 26. oktober 2026. Lagt til: TPM-krav for HIN Client 4.0 og nettleserinnlogging fra begynnelsen av november 2026.

## Kilder

1.  [HIN-plattformfornyelse: Disse tekniske tilpasningene er nødvendige for HIN-medlemmer](https://www.hin.ch/de/blog/2026/technische-anpassungen.cfm): Frister fra august til oktober 2026 (status 27. august 2026), SAML-endepunkter, redusert attributtmengde, firewall-åpninger, Authenticator App og maskinvaretokener.

2.  [Den nye HIN Client er her: Dette endrer seg for HIN-medlemmer](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): Versjon 4.0, utrulling innen 26. oktober 2026, krav til operativsystem og TPM, nettleserbasert innlogging fra november 2026.

3.  [HIN Gateway: Sikker kommunikasjon i HIN Community](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm): Erstatning av Mailgateway, arkitektur, driftsmodeller, migrering i bestilte tidsvinduer.

4.  [Konfigurasjon av HIN Access Gateway](https://cdn.hin.ch/agw/manual/DE/4-konfiguration-des-hin-access-gateway.html): AGWs rolle i tilgangsstyring.

5.  [Tilkobling til Active Directory](https://cdn.hin.ch/agw/manual/DE/5-anbindung-active-directory.html): Kerberos og LDAP-kontoen med leserettigheter.

6.  [HIN AG: «Fra Mailgateway til Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): Bakgrunn om «Stargate», desentraliserte noder, tidsplan.
