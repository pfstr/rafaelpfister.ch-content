---
title: "SEPPmail 15.0.6 og 15.0.6.1: Sikkerhetsrettelser og nye admin-funksjoner"
navTitle: "SEPPmail 15.0.6"
description: "SEPPmail publiserte patchutgivelsen 15.0.6 og hurtigreparasjonen 15.0.6.1 i juli 2026. I tillegg til utbedrede sårbarheter i PDF-generering og PGP-behandling, gir utgivelsene et separat MFA-felt, LDAP-autentisering for admin-GUI-en og rettelser i RuleEngine, Webmail og REST-API-et."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "5 min lesetid"
themen:
  - seppmail
slug: "seppmail-15-0-6-og-15-0-6-1-sikkerhetsrettelser-og-nye-admin-funksjoner"
translationId: "article-3046fc35b259929b"
draft: false
translationOf: seppmail-releases-15-0-6-und-15-0-6-1
url: https://rafaelpfister.ch/no/blog/seppmail-15-0-6-og-15-0-6-1-sikkerhetsrettelser-og-nye-admin-funksjoner
translationSourceHash: 5cf19b84bb90403b0a7e2795222b8f853c29c3fe562429df8538e703e565217a
translationModel: gpt-5.6-terra
translatedAt: 2026-07-31T06:27:16.463Z
translationReview: automatic
---

# SEPPmail 15.0.6 og 15.0.6.1: Sikkerhetsrettelser og nye admin-funksjoner

SEPPmail publiserte patchutgivelsen 15.0.6 den 21. juli 2026, og hurtigreparasjonen 15.0.6.1 dagen etter. Patchutgivelsen lukker flere sårbarheter, oppdaterer OpenSSH og OpenSSL og gir merkbare forbedringer for administrasjon. Hurtigreparasjonen retter to feil i RuleEngine som ble introdusert eller synlige med 15.0.6. Endringene gjelder også appliances som drives som HIN Mailgateway, siden disse er basert på samme SEPPmail-fastvare.

## Hurtigreparasjon 15.0.6.1 av 22. juli 2026

Hurtigreparasjonen løser to punkter i RuleEngine. For det første hindret en udefinert verdi i Message-objektet at loggoppføringer ble skrevet til e-postloggen. Berørte meldinger gikk dermed gjennom systemet uten logging. For det andre gjenkjenner RuleEngine nå retningen til arkiverte e-poster, slik at leveringen behandles korrekt.

De som allerede har installert 15.0.6 eller planlegger oppdateringen, bør gå direkte til 15.0.6.1.

HIN-appliances har tilsynelatende også fått hurtigreparasjonen: En HIN Mailgateway med installert versjon 15.0.6-RC-42-g278c81f84 melder nå 15.0.6-RC-88-g916e513cc som neste versjon i 15.0-grenen. RC-betegnelsene i HIN-fastvaren kan ikke tilordnes direkte til en SEPPmail-utgivelse, men tidspunktet for tilbudet taler for hurtigreparasjonen.

## Sikkerhetsrettelser i 15.0.6

Den viktigste delen av patchutgivelsen er tre rettelser i sikkerhetsarkitekturen:

- En mulig Path-Traversal-sårbarhet i PDF-genereringen er lukket. Den ble funnet av InfoGuard.
- Alt innhold som dekrypteres med PGP, Base64-kodes nå for å forhindre MIME-Structure-Injection.
- Funksjonen hashencrypt er endret til AES-256-CBC med PBKDF2.

I tillegg kommer oppdaterte biblioteker: OpenSSH 10.4 og OpenSSL 3.0.21 retter til sammen over tjue CVE-er. Bare på grunn av disse punktene anbefales oppdateringen for produksjonssystemer.

## Nye funksjoner for administrasjon

Tre endringer i admin-GUI-en merkes i hverdagen:

- **Separat MFA-inntastingsfelt:** Den andre faktoren må ikke lenger legges til passordet, men har sitt eget felt. Dette fjerner en langvarig fallgruve ved innlogging.
- **LDAP-autentisering for admin-GUI-en:** Administratorer kan nå autentisere seg mot en ekstern LDAP-server i stedet for å vedlikeholde lokale kontoer på appliancen. Oppsettet er beskrevet i artikkelen om [tilkobling av admin-GUI-en til Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung). Om HIN Mailgateway også har fått denne funksjonen, tester jeg fortsatt og vil oppdatere artikkelen etterpå; siden HIN bruker samme fastvaregrunnlag, går jeg ut fra det.
- **AutoRenew-knapp for MPKI:** I innstillingene for MPKI-konnektoren kan automatisk sertifikatfornyelse startes manuelt via «Trigger AutoRenew... ».

I tillegg bruker appliancen nå konsekvent gyldige tidssoner (standard: Europe/Zurich), og System Object ID under System >> Advanced View valideres som en gyldig OID.

## E-postbehandling og Webmail

Fire punkter er rettet i RuleEngine. Behandlingen av emnefelt fungerer nå også ved ukjent encoding. Meldinger avvises når en signatur er eksplisitt forespurt, men ikke kan opprettes; tidligere kunne slike meldinger gå videre usignert. Arkivkopier går nå gjennom leveringsfunksjonen og får dermed ARC-hoder. Og for PGP-meldinger uten MDC-data ignoreres MDC-feil i stedet for å forstyrre behandlingen.

I Webmail (GINA) er fire feil rettet: Automatisk sletting av ikke-registrerte kontoer etter utløpt karensperiode fungerer igjen, funksjonen hashdecrypt ga i enkelte tilfeller et falskt positivt dekrypteringsresultat, tillegg av et vedlegg tømte feltene Til og CC, og tidsvisningen i SMS-loggene var feil.

## REST-API, klynge og sikkerhetskopi

REST-API-et får rettelser i flere endepunkter: /system/ifaliasconfig (håndtering av null-verdier), /system/applySysconfig (tilgangskonfigurasjon), /crypto/domain/{domainName} (opplasting av domenesertifikater) samt GET og POST /ssl/csr. Tidsavbruddet for REST-kall er økt fra 300 til 900 sekunder, noe som gjør langvarige forespørsler som større konfigurasjonsendringer mer pålitelige.

I klyngedrift blokkerte tidligere en eksisterende CARP-IP IP-innstillingene til et nylig lagt til medlem; dette er rettet. Før den daglige snapshot-opprettelsen kontrollerer sikkerhetskopieringen nå i tillegg om databasen er korrupt før snapshotet skrives.

## Sammenheng med innloggingsutfallet i 15.0.5

Ved oppdatering av en klynge til 15.0.5 kunne innloggingen svikte på begge nodene. Feilbildet og gjenopprettingen er beskrevet i artikkelen om [innloggingsutfall etter 15.0.5-oppdateringen](/blog/hin-update-issue-version-15.0.5). Produsenten kjente allerede til problemet og varslet den gang en rettelse i en kommende versjon.

I Release Notes for 15.0.6 finnes nå nettopp én oppføring som passer med dette feilbildet: «prevent password rehashing when cluster members use different firmware versions». Under en klyngeoppdatering kjører nodene nødvendigvis midlertidig med ulike fastvareversjoner. Hvis én node i denne fasen beregner passordhasher på nytt og replikerer dem i klyngen, passer hashene ikke lenger med den andre versjonen, og innloggingen feiler på begge nodene, akkurat som ved utfallet som ble observert den gang. Release Notes nevner ikke eksplisitt innloggingsutfallet, men oppføringen dekker nøyaktig konstellasjonen som utløste det. Årsaken er dermed adressert i 15.0.6; nødprosedyren med oppløsning av klyngen som var nødvendig i 15.0.5, bør ikke lenger være nødvendig ved fremtidige oppdateringer.

## Mindre rettelser

I e-postloggen er datosorteringen rettet, som tidligere sorterte alfabetisk i stedet for kronologisk, og den viste størrelsen på LFT-meldinger stemmer igjen. Tilganger til ikke-eksisterende X-hoder logges ikke lenger. CertCentral-konnektoren til MPKI håndterer inndata- og REST-feil mer robust.

## Vurdering

De to RuleEngine-feilene fra hurtigreparasjonen taler for å hoppe over 15.0.6 og bruke 15.0.6.1 direkte. Opprett snapshots av begge noder i klynger før oppdateringen, og følg oppdateringsrekkefølgen i produsentens dokumentasjon. Innloggingsutfallet i 15.0.5 viste hvorfor denne forberedelsen ikke bare er en formalitet.

## Kilder

1.  [SEPPmail-dokumentasjon – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Offisielle Release Notes for 15.0.6 og 15.0.6.1 med alle enkeltheter.

2.  [HIN Mailgateway 15.0.5: Rett innloggingsutfall etter klyngeoppdateringen](/blog/hin-update-issue-version-15.0.5): Hvorfor snapshots og riktig oppdateringsrekkefølge i klyngen er avgjørende.
