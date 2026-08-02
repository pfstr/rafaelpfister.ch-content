---
title: "HIN-plattformfornyelse 2026: Access Gateway, Client og fristene frem til 14. september"
navTitle: "Access Gateway 2026"
description: "Brannmuråpning innen 14. august, Access Gateway versjon 4 fra 17. august, SAML-endepunkter, maskinvaretoken og HIN Client innen 14. september. Mailgatewayet er ikke berørt og erstattes separat."
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
url: https://rafaelpfister.ch/no/blog/hin-plattformfornyelse-2026-access-gateway-client-og-fristene-frem-til-14-september
translationSourceHash: 1a174bd131b8bb29f9b1e1e793d4cf19b3f732e3c6fd779a25193f151ec8c109
translationModel: gpt-5.6-terra
translatedAt: 2026-08-02T06:17:25.671Z
translationReview: automatic
---

# HIN-plattformfornyelse 2026: Access Gateway, Client og fristene frem til 14. september

HIN fornyer plattformen for identitet og tilgang i 2026. Den første fristen utløper 14. august 2026, den store omstillingen følger 14. september 2026.

**Berørt er HIN Access Gateway (AGW), HIN Client og autentiseringsmidlene. HIN Mailgateway er ikke berørt.** Det erstattes også, men i et separat prosjekt med egen tidsplan.

<div class="choice-row">
  <a class="choice" href="#die-fristen">
    <span class="choice__label">Din situasjon</span>
    <span class="choice__title">Bare AGW i drift</span>
    <span class="choice__hint">Fristene nedenfor utgjør hele handlingsbehovet ditt. →</span>
  </a>
  <a class="choice" href="/stargate">
    <span class="choice__label">Din situasjon</span>
    <span class="choice__title">I tillegg migreringsbehov for Mailgateway</span>
    <span class="choice__hint">Da står også erstatningen med «Stargate» for tur, med bred utrulling fra tredje kvartal 2026. Gratis gjennomgang av miljøet ditt. →</span>
  </a>
</div>

## Fristene

| Dato | Tiltak | Gjelder |
|---|---|---|
| 14.08.2026 | Brannmuråpning for `idp.id.hin.ch` (`185.154.38.46`, `193.168.215.45`) | AGW-operatører |
| 17.08.2026 | Automatisk installasjon av AGW versjon 4 | AGW-operatører |
| fra midten av august | Manuell installasjon av HIN Client 4.0 anbefales | Alle Client-brukere |
| 14.09.2026 | SAML-endepunkter migrert | Føderasjoner, EPD-tilkoblinger |
| 14.09.2026 | Maskinvaretoken og testidentiteter utløper | Token-brukere, integrasjon |
| 14.09.2026 | Konfigurer Authenticator App på nytt | App-brukere |
| 14.09.2026 | Tvungen oppdatering til HIN Client 4.0 | Alle Client-brukere |

## Access Gateway er ikke Mailgateway

Begge har Gateway i navnet og forveksles jevnlig. Access Gateway styrer tilgang til HIN-beskyttede applikasjoner og berører ikke e-posttrafikken. Mailgateway står i e-postløypen og krypterer meldinger.

## Access Gateway: brannmur og versjon 4

Innen 14. august må AGW kunne nå `idp.id.hin.ch`. Dette er en brannmurendring, ikke en innstilling i gatewayet, og ligger derfor ofte hos den nettverksansvarlige i stedet for gateway-administratoren.

Fra 17. august installeres versjon 4 automatisk. Forutsetning: AGW på versjon 3.1.50 eller nyere, og Kerberos aktivert som autentiseringsmetode. For tilkoblingen til Active Directory kreves en LDAP-konto med leserettigheter.

De som ikke oppfyller forutsetningene, blir ikke oppdatert, og erfaringen viser at dette først oppdages når ingen lenger kan logge inn. Det er derfor bedre å kontrollere versjonen nå enn i september.

## SAML: nye endepunkter, færre attributter

```text
Föderationsdienst
  broker.hin.ch/realms/HINBroker/protocol/saml/descriptor

EPD-Zugang
  idp.id.hin.ch/auth/realms/hinid/protocol/saml/descriptor
```

Med overgangen endres attributtformatene og bindingene. Attributtmengden reduseres til GLN, navn, fødselsdato og kjønn.

Dette er punktet som bryter integrasjoner. Alle applikasjoner som bruker ytterligere attributter for roller eller klientseparasjon, får dem ikke lenger etter 14. september. Feilen viser seg ikke som en innloggingsfeil, men som manglende tilgang i målsystemet.

Testidentiteter utløper samme dato, så de som ønsker å teste omstillingen i et integrasjonsmiljø, bør gjøre det før dette.

De som driver en føderasjon, driver nesten alltid også egen e-postinfrastruktur. For disse organisasjonene faller plattformfornyelsen og [erstatningen av Mailgateway med «Stargate»](/stargate) i samme år: teknisk uavhengige, men konkurrerende om de samme personene og vedlikeholdsvinduene.

## Token, app og HIN Client 4.0

Maskinvaretoken utstedes ikke lenger og utløper 14. september. Alternativer: HIN Client, SMS-kode eller Authenticator App. Selve appen er gyldig frem til 14. september og må deretter konfigureres på nytt via selvbetjeningsportalen.

HIN Client oppdateres automatisk til versjon 4.0 senest 14. september, med manuell installasjon fra midten av august via `download.hin.ch`. Innloggingen skjer nå via nettleseren.

Det kritiske punktet er systemkravene: **Versjon 4.0 krever Windows 11 eller macOS 14.** Eldre enheter må oppdateres eller erstattes på forhånd. For en del praksiser er fristen dermed ikke en programvareoppgave, men en anskaffelsesoppgave. De som først oppdager dette i september, må håndtere leveringstider og nyinstallasjon av praksisprogramvaren.

## Fem spørsmål for å kartlegge situasjonen

1. Hvilken AGW-versjon kjører, og er Kerberos aktiv?
2. Tillater brannmuren utgående tilgang til `idp.id.hin.ch`?
3. Hvor mange arbeidsstasjoner kjører fortsatt Windows 10 eller macOS 13 og eldre?
4. Hvor mange maskinvaretoken er i bruk, og hva skal de berørte bytte til?
5. Bruker en applikasjon HIN-attributter som vil falle bort?

Svarene på 3 og 5 bestemmer omfanget. Resten kan gjøres på noen få timer og er dokumentert av HIN.

## Det andre prosjektet: «Stargate»

Uavhengig av dette erstatter HIN Mailgateway med det nye HIN Gateway, internt kalt prosjekt «Stargate», teknisk sett en Data-Mesh-tilnærming med ende-til-ende-kryptering og desentralisert nøkkelhåndtering. Det er ikke snakk om å bytte ut appliance-en, men om et arkitekturskifte.

Arbeidsomfanget ligger dermed på et helt annet nivå. Plattformfornyelsen krever først og fremst at frister overholdes for en brannmurregel, en versjon og utskifting av en enhet, mens selve e-postløypen i produksjon står til vurdering med Stargate: det opparbeidede regelverket, nøkkelmaterialet, håndteringen av mottakere uten HIN-identitet og spørsmålet om hva man faller tilbake på dersom noe ikke går som forventet. Siden migreringen skjer i bestilte firetimersvinduer og HIN anbefaler én måneds forberedelse, tåler en slik avtale ingen uavklarte punkter.

<aside class="offer-box">
  <span class="offer-box__tag">Gratis gjennomgang</span>
  <p><strong>Du trenger ikke å vite hvor du står. Det er nettopp derfor gjennomgangen finnes.</strong> Jeg ser på det eksisterende gateway-miljøet ditt og forteller deg hva som må gjøres før migreringsvinduet, uavhengig av om du migrerer selv etterpå eller får bistand.</p>
  <a class="offer-box__cta" href="/stargate">Registrer deg nå</a>
</aside>

## Kilder

1.  [HIN-plattformfornyelse: Disse tekniske tilpasningene er nødvendige for HIN-medlemmer](https://www.hin.ch/de/blog/2026/technische-anpassungen.cfm): Frister i august og september, SAML-endepunkter, redusert attributtmengde, brannmuråpninger.

2.  [Den nye HIN Client er her: Dette endres for HIN-medlemmer](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): Versjon 4.0, operativsystemkrav, nettleserbasert innlogging.

3.  [HIN Gateway: Sikker kommunikasjon innenfor HIN-fellesskapet](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm): Erstatning av Mailgateway, arkitektur, driftsmodeller, migrering i bestilte tidsvinduer.

4.  [Konfigurasjon av HIN Access Gateway](https://cdn.hin.ch/agw/manual/DE/4-konfiguration-des-hin-access-gateway.html): AGWs rolle i tilgangsadministrasjon.

5.  [Tilkobling av Active Directory](https://cdn.hin.ch/agw/manual/DE/5-anbindung-active-directory.html): Kerberos og LDAP-kontoen med leserettigheter.

6.  [HIN AG: «Fra Mailgateway til Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): Bakgrunn for «Stargate», desentraliserte noder, tidsplan.
