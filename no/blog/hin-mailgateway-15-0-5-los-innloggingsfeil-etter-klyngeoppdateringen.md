---
slug: "hin-mailgateway-15-0-5-los-innloggingsfeil-etter-klyngeoppdateringen"
title: "HIN Mailgateway 15.0.5: Løs påloggingsfeil etter cluster-oppdateringen"
navTitle: "Påloggingsfeil 15.0.5"
description: "Etter oppdatering av et HIN Mailgateway-cluster til versjon 15.0.5 slutter påloggingen å fungere på begge nodene etter noen minutter. Denne fremgangsmåten setter appliance-enhetene kontrollert i drift igjen."
date: "2026-06-19"
kategorie: "HIN-Gateway"
timeToRead: "3 min lesetid"
themen:
  - hin-gateway
draft: false
translationOf: "hin-update-issue-version-15.0.5"
translationId: article-bd1908eec39f9c26
translationReview: automatic
translationSourceHash: d40e2a9637cb0711ca33ed8ffde7827192362f668bebafb7b1f6862fda736d04
translatedAt: 2026-08-08T14:23:52.260Z
url: https://rafaelpfister.ch/no/blog/hin-mailgateway-15-0-5-los-innloggingsfeil-etter-klyngeoppdateringen
translationModel: gpt-5.6-terra
---

# HIN Mailgateway 15.0.5: Løs påloggingsfeil etter cluster-oppdateringen

Ved oppdatering av en HIN Mailgateway fra 14.1.4.2 til 15.0.5 kan en feil i cluster-replikeringen sette påloggingen ut av spill på begge appliance-enhetene. Enkeltstående systemer er ikke berørt. Produsenten kjenner til problemet og planlegger en korrigering i en senere versjon.

**Oppdatering 29. juli 2026:** Den varslede korrigeringen er tilgjengelig. Patch-utgivelsen 15.0.6 undertrykker passord-rehashing når cluster-medlemmer kjører ulike firmware-versjoner. Dette er nøyaktig konstellasjonen som utløste feilen beskrevet her. Vurderingen finnes i artikkelen om [SEPPmail 15.0.6 og 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1); følgende gjenopprettingsprosedyre er fortsatt relevant for clustre som fremdeles oppdaterer til 15.0.5.

## Feilbilde

Umiddelbart etter oppdateringen kan webgrensesnittet fortsatt åpnes. Omtrent ti minutter senere mislykkes påloggingen på begge cluster-nodene. At feilen oppstår forsinket og på begge systemene, peker på den replikerte cluster-konfigurasjonen som årsak.

## Gjenoppretting

Følgende trinn endrer cluster-konfigurasjonen. Først må oppdaterte sikkerhetskopier og cluster-identifikatoren være tilgjengelige.

1. Gjenopprett de samtidig opprettede snapshotene av begge cluster-nodene.
2. La én node være avslått etter gjenopprettingen.
3. Last først ned cluster-identifikatoren på noden som kjører, og oppløs deretter clusteret.
4. Obs: Etter oppløsningen starter appliance-enheten umiddelbart på nytt uten ytterligere bekreftelse.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Oppdater den første noden til versjon 15.0.5, og slå den deretter av.
6. Start den andre noden og gjenta de samme trinnene der.
7. Først når begge systemene fungerer hver for seg og har samme versjon, bygger du opp clusteret igjen i henhold til produsentens dokumentasjon.

Denne fremgangsmåten hindrer at en feilaktig konfigurasjon replikeres på nytt mellom nodene under oppdateringen.

## Kilder

1. [SEPPmail-dokumentasjon – «Cluster / høy tilgjengelighet»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): Cluster-typer og replikering av konfigurasjonen over alle noder.
2. [SEPPmail-dokumentasjon – «Administrasjon»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Oppdateringsrekkefølge i clusteret (frontend før backend) og kravet om identiske versjoner.
3. [HIN Mailgateway: Backup og Disaster Recovery i clusteret](/blog/hin-mailgateway-backup-disaster-recovery): En nærmere gjennomgang av cluster-replikering, sikkerhetskopiering og gjenoppretting.
