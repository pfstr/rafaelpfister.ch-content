---
title: "SEPPmail 15.0.6 og 15.0.6.1: Sikkerhetsrettinger og nye administratorfunksjoner"
navTitle: "SEPPmail 15.0.6"
description: "SEPPmail lanserte patch-utgivelsen 15.0.6 og hurtigreparasjonen 15.0.6.1 i juli 2026. I tillegg til utbedrede sårbarheter i PDF-generering og PGP-behandling gir utgivelsene et eget MFA-felt, LDAP-autentisering for administrator-GUI-en og rettinger i RuleEngine, webmail og REST-API-et."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "5 min lesetid"
themen:
  - seppmail
slug: "seppmail-15-0-6-og-15-0-6-1-sikkerhetsrettelser-og-nye-admin-funksjoner"
translationId: "article-3046fc35b259929b"
draft: false
translationOf: seppmail-releases-15-0-6-und-15-0-6-1
translationSourceHash: 636a7246234584a2b5797f53239fe65129de0f4463b8f773d0a7d9ed06d61f91
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:16:22.382Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/seppmail-15-0-6-og-15-0-6-1-sikkerhetsrettelser-og-nye-admin-funksjoner
---

# SEPPmail 15.0.6 og 15.0.6.1: Sikkerhetsrettinger og nye administratorfunksjoner

SEPPmail lanserte patch-utgivelsen 15.0.6 21. juli 2026 og hurtigreparasjonen 15.0.6.1 én dag senere. Patch-utgivelsen lukker flere sårbarheter, oppdaterer OpenSSH og OpenSSL og gir merkbare forbedringer for administrasjonen. Hurtigreparasjonen retter to feil i RuleEngine som ble introdusert eller synlige med 15.0.6. Endringene gjelder også Appliances som drives som HIN Mailgateway, ettersom disse er basert på samme SEPPmail-fastvare.

## Hurtigreparasjon 15.0.6.1 fra 22. juli 2026

Hurtigreparasjonen løser to punkter i RuleEngine. For det første hindret en udefinert verdi i Message-objektet at loggoppføringer ble skrevet til e-postloggen. Berørte meldinger gikk dermed gjennom systemet uten logging. For det andre gjenkjenner RuleEngine nå retningen til arkiverte e-poster, slik at leveringen deres håndteres korrekt.

De som allerede har installert 15.0.6 eller planlegger oppdateringen, bør gå direkte til 15.0.6.1.

HIN-Appliances har tilsynelatende også mottatt hurtigreparasjonen: En HIN Mailgateway med installert versjon 15.0.6-RC-42-g278c81f84 melder nå 15.0.6-RC-88-g916e513cc som neste versjon i 15.0-grenen. RC-betegnelsene i HIN-fastvaren kan ikke tilordnes direkte til en SEPPmail-utgivelse, men tidspunktet da den ble tilbudt, taler for hurtigreparasjonen.

## Sikkerhetsrettinger i 15.0.6

Den viktigste delen av patch-utgivelsen er tre rettinger i sikkerhetsarkitekturen:

- En mulig path-traversal-sårbarhet i PDF-genereringen ble lukket. Den ble funnet av InfoGuard.
- Alt innhold dekryptert via PGP Base64-kodes nå for å forhindre MIME-structure-injection.
- Funksjonen hashencrypt ble lagt om til AES-256-CBC med PBKDF2.

I tillegg kommer oppdaterte biblioteker: OpenSSH 10.4 og OpenSSL 3.0.21 utbedrer til sammen over tjue CVE-er. Bare på grunn av disse punktene anbefales oppdateringen for produksjonssystemer.

## Nye funksjoner for administrasjon

Tre endringer i administrator-GUI-en merkes i hverdagen:

- **Eget MFA-inndatafelt:** Den andre faktoren trenger ikke lenger å legges til passordet, men har sitt eget felt. Dette fjerner en langvarig feilkilde ved innlogging.
- **LDAP-autentisering for administrator-GUI-en:** Administratorer kan nå autentisere seg mot en ekstern LDAP-server i stedet for å vedlikeholde lokale kontoer på Appliance-en. Oppsettet er beskrevet i artikkelen om [tilkobling av administrator-GUI-en til Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung). Om HIN Mailgateway også har fått denne funksjonen, tester jeg fortsatt og vil deretter oppdatere artikkelen; siden HIN bruker samme fastvarebase, går jeg ut fra det.
- **AutoRenew-knapp for MPKI:** I innstillingene for MPKI-Connector kan automatisk sertifikatfornyelse startes manuelt med «Trigger AutoRenew... ».

I tillegg bruker Appliance-en nå konsekvent gyldige tidssoner (standard: Europe/Zurich), og System Object ID under System >> Advanced View valideres som en gyldig OID.

## E-postbehandling og webmail

Fire punkter ble rettet i RuleEngine. Emnebehandlingen fungerer nå også ved ukjent encoding. Meldinger avvises når en signatur er eksplisitt forespurt, men ikke kan opprettes; tidligere kunne slike meldinger fortsette usignert. Arkivkopier går nå gjennom leveringsfunksjonen og får dermed ARC-headere. Og for PGP-meldinger uten MDC-data ignoreres MDC-feil i stedet for å forstyrre behandlingen.

I webmail (GINA) ble fire feil rettet: Automatisk sletting av ikke-registrerte kontoer etter utløpet av karensperioden fungerer igjen, funksjonen hashdecrypt ga i enkelte tilfeller et falskt positivt dekrypteringsresultat, tilføyelse av et vedlegg tømte feltene Til og CC, og tidsvisningen i SMS-loggene var feil.

## REST-API, cluster og backup

REST-API-et får rettinger på flere endepunkter: /system/ifaliasconfig (håndtering av null-verdier), /system/applySysconfig (tilgangskonfigurasjon), /crypto/domain/{domainName} (opplasting av domene-sertifikater) samt GET og POST /ssl/csr. Tidsavbruddet for REST-kall ble økt fra 300 til 900 sekunder, noe som gjør langvarige forespørsler som større konfigurasjonsendringer mer pålitelige.

I cluster-drift blokkerte tidligere en eksisterende CARP-IP IP-innstillingene til et nylig lagt til medlem; dette er rettet. Før den daglige opprettelsen av snapshot kontrollerer backupen nå også om databasen er korrupt før snapshotet skrives.

## Sammenheng med innloggingsfeilen i 15.0.5

Ved oppdatering av et cluster til 15.0.5 kunne innloggingen feile på begge noder. Feilbildet og gjenopprettingen er beskrevet i artikkelen om [innloggingsfeil etter 15.0.5-oppdateringen](/blog/hin-update-issue-version-15.0.5). Produsenten kjente allerede til problemet på det tidspunktet og varslet en retting i en påfølgende versjon.

I Release Notes for 15.0.6 finnes det nå nøyaktig én oppføring som passer med dette feilbildet: «prevent password rehashing when cluster members use different firmware versions». Under en cluster-oppdatering kjører nodene uunngåelig midlertidig med ulike fastvareversjoner. Hvis én node i denne fasen beregner passordhasher på nytt og replikerer dem i clusteret, passer ikke hashene lenger med den andre versjonen, og innloggingen mislykkes på begge noder, akkurat som ved utfallet som ble observert den gangen. Release Notes nevner ikke innloggingsfeilen uttrykkelig, men oppføringen dekker nøyaktig konstellasjonen som utløste den. Årsaken er dermed håndtert i 15.0.6; nødprosedyren med oppløsning av clusteret som var nødvendig i 15.0.5, bør være unødvendig ved fremtidige oppdateringer.

## Mindre rettinger

I e-postloggen ble datumsorteringen rettet, som tidligere sorterte alfabetisk i stedet for kronologisk, og den viste størrelsen på LFT-meldinger stemmer igjen. Tilgang til ikke-eksisterende X-headere loggføres ikke lenger. CertCentral-Connectoren for MPKI håndterer inndata- og REST-feil mer robust.

## Vurdering

De to RuleEngine-feilene fra hurtigreparasjonen taler for å hoppe over 15.0.6 og bruke 15.0.6.1 direkte. For cluster bør du opprette snapshots av begge noder før oppdateringen og følge oppdateringsrekkefølgen i produsentens dokumentasjon. Innloggingsfeilen i 15.0.5 har vist hvorfor denne forberedelsen ikke er en formalitet.

## Kilder

1.  [SEPPmail-dokumentasjon – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Offisielle Release Notes for 15.0.6 og 15.0.6.1 med alle enkeltdetaljer.

2.  [HIN Mailgateway 15.0.5: Løs innloggingsfeil etter cluster-oppdateringen](/blog/hin-update-issue-version-15.0.5): Hvorfor snapshots og riktig oppdateringsrekkefølge i clusteret er avgjørende.
