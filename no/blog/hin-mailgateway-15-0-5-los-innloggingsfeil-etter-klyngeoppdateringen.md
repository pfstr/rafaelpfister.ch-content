---
slug: "hin-mailgateway-15-0-5-los-innloggingsfeil-etter-klyngeoppdateringen"
title: "HIN Mailgateway 15.0.5: Løs innloggingsfeil etter klyngeoppdateringen"
navTitle: "Innloggingsfeil 15.0.5"
description: "Etter oppdatering av en HIN Mailgateway-klynge til versjon 15.0.5 svikter innloggingen på begge nodene etter noen minutter. Denne fremgangsmåten setter apparatene kontrollert i drift igjen."
date: "2026-06-19"
kategorie: "HIN-gateway"
timeToRead: "3 min lesetid"
themen:
  - hin-gateway
draft: false
translationOf: "hin-update-issue-version-15.0.5"
translationId: article-bd1908eec39f9c26
translationReview: automatic
translationSourceHash: 03805a55e2acb45a3453acda621e39cc3ddce6ae00cfe85f76ca2cc322cde6fa
translatedAt: 2026-09-05T08:03:17.922Z
translationModel: gpt-5.6-terra
url: https://rafaelpfister.ch/no/blog/hin-mailgateway-15-0-5-los-innloggingsfeil-etter-klyngeoppdateringen
---

# HIN Mailgateway 15.0.5: Løs innloggingsfeil etter klyngeoppdateringen

Ved oppdatering av en HIN Mailgateway fra 14.1.4.2 til 15.0.5 kan en feil i klyngereplikeringen føre til at innloggingen svikter på begge apparatene. Enkeltstående systemer er ikke berørt. Produsenten kjenner til problemet og planlegger en rettelse i en kommende versjon.

**Oppdatering 29. juli 2026:** Den annonserte rettelsen er her. Patch-utgivelsen 15.0.6 undertrykker rehasjing av passord når klyngemedlemmer kjører ulike firmware-versjoner. Dette er nøyaktig konfigurasjonen som utløste feilen beskrevet her. Vurderingen finnes i artikkelen om [SEPPmail 15.0.6 og 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1); den følgende gjenopprettingsprosedyren forblir relevant for klynger som fortsatt oppdaterer til 15.0.5.

## Feilbilde

Umiddelbart etter oppdateringen kan webgrensesnittet fortsatt åpnes. Omtrent ti minutter senere mislykkes innloggingen på begge klyngenodene. At feilen oppstår forsinket og på begge systemene, tyder på at den replikerte klyngekonfigurasjonen er årsaken.

## Gjenoppretting

Følgende trinn endrer klyngekonfigurasjonen. Først må oppdaterte sikkerhetskopier og klyngeidentifikatoren være tilgjengelige.

1. Gjenopprett de samtidig opprettede snapshotene av begge klyngenodene.
2. La én node være avslått etter gjenopprettingen.
3. Last først ned klyngeidentifikatoren på noden som kjører, og oppløs deretter klyngen.
4. Obs: Etter oppløsningen starter apparatet umiddelbart på nytt uten ytterligere bekreftelse.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Oppdater den første noden til versjon 15.0.5, og slå den deretter av.
6. Start den andre noden og gjenta de samme trinnene der.
7. Først når begge systemene fungerer hver for seg og har samme versjon, bygger du opp klyngen igjen i henhold til produsentens dokumentasjon.

Denne fremgangsmåten hindrer at en feilaktig konfigurasjon replikeres på nytt mellom nodene under oppdateringen.

## Kilder

1. [SEPPmail-dokumentasjon – «Klynge / høy tilgjengelighet»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): Klyngetyper og replikering av konfigurasjonen over alle noder.
2. [SEPPmail-dokumentasjon – «Administrasjon»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Oppdateringsrekkefølge i klyngen (frontend før backend) og kravet om identiske versjoner.
3. [HIN Mailgateway: Backup og katastrofegjenoppretting i klyngen](/blog/hin-mailgateway-backup-disaster-recovery): En grundigere gjennomgang av klyngereplikering, sikkerhetskopiering og gjenoppretting.
