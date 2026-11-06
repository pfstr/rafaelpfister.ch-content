---
slug: "hin-mailgateway-15-0-5-atgarda-inloggningsfel-efter-klusteruppdateringen"
title: "HIN Mailgateway 15.0.5: åtgärda inloggningsfel efter klusteruppdateringen"
navTitle: "Inloggningsfel 15.0.5"
description: "Efter uppdateringen av ett HIN Mailgateway-kluster till version 15.0.5 slutar inloggningen att fungera på båda noderna efter några minuter. Denna procedur återställer apparaterna kontrollerat."
date: "2026-06-19"
kategorie: "HIN-gateway"
timeToRead: "3 min läsning"
themen:
  - hin-gateway
draft: false
translationOf: "hin-update-issue-version-15.0.5"
translationId: article-bd1908eec39f9c26
translationReview: automatic
translationSourceHash: 126aded044bfb102e020aaf45b7b77f6658f77a6de88a71d1554b1fabeefce2a
translatedAt: 2026-11-06T03:29:04.781Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/hin-mailgateway-15-0-5-atgarda-inloggningsfel-efter-klusteruppdateringen
---

# HIN Mailgateway 15.0.5: åtgärda inloggningsfel efter klusteruppdateringen

Vid uppdatering av en HIN Mailgateway från 14.1.4.2 till 15.0.5 kan ett fel i klusterreplikeringen leda till att inloggningen slutar fungera på båda apparaterna. Enskilda system påverkas inte. Tillverkaren känner till problemet och planerar en korrigering i en kommande version.

**Uppdatering den 29 juli 2026:** Den aviserade korrigeringen är här. Patchversionen 15.0.6 förhindrar lösenordsrehashning när klustermedlemmar kör olika firmwareversioner. Det är exakt den konstellation som utlöste felet som beskrivs här. En genomgång finns i artikeln om [SEPPmail 15.0.6 och 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1); följande återställningsprocedur är fortfarande relevant för kluster som ännu uppdaterar till 15.0.5.

## Felbild

Omedelbart efter uppdateringen går det fortfarande att öppna webbgränssnittet. Efter cirka tio minuter misslyckas inloggningen på båda klusternoderna. Att felet uppstår fördröjt och på båda systemen tyder på att den replikerade klusterkonfigurationen är orsaken.

## Återställning

Följande steg ändrar klusterkonfigurationen. Aktuella säkerhetskopior och klusteridentifieraren måste finnas tillgängliga i förväg.

1. Återställ de samtidigt skapade ögonblicksbilderna för båda klusternoderna.
2. Låt en nod vara avstängd efter återställningen.
3. På den körande noden hämtar du först klusteridentifieraren och löser sedan upp klustret.
4. Observera: Efter upplösningen startar apparaten omedelbart om utan ytterligare bekräftelse.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Uppdatera den första noden till version 15.0.5 och stäng sedan av den.
6. Starta den andra noden och upprepa samma steg där.
7. Bygg upp klustret igen enligt tillverkarens dokumentation först när båda systemen fungerar separat och har samma versionsnivå.

Denna metod förhindrar att en felaktig konfiguration replikeras mellan noderna igen under uppdateringen.

## Ändringslogg

- **25 oktober 2026:** Rättat: Uppdateringsordningen i klustret är enligt SEPPmail-dokumentationen backend före frontend, inte tvärtom.

## Källor

1. [SEPPmail-dokumentation – «Kluster / hög tillgänglighet»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): Klustertyper och replikering av konfigurationen över alla noder.
2. [SEPPmail-dokumentation – «Administration»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Uppdateringsordning i klustret (backend före frontend) och kravet på identiska versionsnivåer.
3. [HIN Mailgateway: Backup & Disaster Recovery i klustret](/blog/hin-mailgateway-backup-disaster-recovery): Fördjupad genomgång av klusterreplikering, säkerhetskopiering och återställning.
