---
slug: "hin-mailgateway-15-0-5-los-innloggingsfeil-etter-klyngeoppdateringen"
title: "HIN Mailgateway 15.0.5: Løs innloggingsfeil etter klyngeoppdateringen"
navTitle: "Innloggingsfeil 15.0.5"
description: "Etter oppdatering av en HIN Mailgateway-klynge til versjon 15.0.5 slutter innloggingen å fungere på begge nodene etter noen minutter. Denne fremgangsmåten setter apparatene kontrollert i drift igjen."
date: "2026-06-19"
kategorie: "HIN-gateway"
timeToRead: "3 min. lesetid"
themen:
  - hin-gateway
draft: false
translationOf: "hin-update-issue-version-15.0.5"
url: "https://rafaelpfister.ch/no/blog/hin-mailgateway-15-0-5-los-innloggingsfeil-etter-klyngeoppdateringen"
translationId: article-bd1908eec39f9c26
translationReview: automatic
translationSourceHash: 3bf0ad28c6b9b80f5644d7281912c3966fd7d0632665afcc13e055bda963e5c2
translatedAt: 2026-07-29T12:29:38.968Z
---

# HIN Mailgateway 15.0.5: Løs innloggingsfeil etter klyngeoppdateringen

Ved oppdatering av en HIN Mailgateway fra 14.1.4.2 til 15.0.5 kan en feil i klyngereplikeringen sette innloggingen ut av spill på begge apparatene. Enkeltstående systemer er ikke berørt. Produsenten kjenner til problemet og planlegger en rettelse i en kommende versjon.

## Feilbilde

Rett etter oppdateringen kan webgrensesnittet fortsatt åpnes. Omtrent ti minutter senere mislykkes innloggingen på begge klyngenodene. At feilen oppstår forsinket og på begge systemene, peker på den replikerte klyngekonfigurasjonen som årsak.

## Gjenoppretting

Følgende trinn endrer klyngekonfigurasjonen. Først må oppdaterte sikkerhetskopier og klyngeidentifikatoren være tilgjengelige.

1. Gjenopprett de samtidig opprettede snapshotene av begge klyngenodene.
2. Etter gjenopprettingen lar du én node være avslått.
3. På noden som kjører, laster du først ned klyngeidentifikatoren og oppløser deretter klyngen.
4. Merk: Etter oppløsningen starter apparatet umiddelbart på nytt uten ytterligere bekreftelse.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Oppdater den første noden til versjon 15.0.5, og slå den deretter av.
6. Start den andre noden og gjenta de samme trinnene der.
7. Først når begge systemene fungerer enkeltvis og har samme versjon, bygger du opp klyngen igjen i henhold til produsentens dokumentasjon.

Denne fremgangsmåten forhindrer at en feilaktig konfigurasjon replikeres på nytt mellom nodene under oppdateringen.

## Kilder

1. [SEPPmail-dokumentasjon – «Klynge / høy tilgjengelighet»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): Klyngetyper og replikering av konfigurasjonen over alle noder.
2. [SEPPmail-dokumentasjon – «Administrasjon»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Oppdateringsrekkefølge i klyngen (frontend før backend) og kravet om identiske versjoner.
3. [HIN Mailgateway: Sikkerhetskopiering og katastrofegjenoppretting i klyngen](/blog/hin-mailgateway-backup-disaster-recovery): inngående gjennomgang av klyngereplikering, sikkerhetskopiering og gjenoppretting.
