---
slug: "hin-mailgateway-15-0-5-atgarda-inloggningsfel-efter-klusteruppdateringen"
title: "HIN Mailgateway 15.0.5: Åtgärda inloggningsfel efter klusteruppdateringen"
navTitle: "Inloggningsfel 15.0.5"
description: "Efter uppdateringen av ett HIN Mailgateway-kluster till version 15.0.5 slutar inloggningen att fungera på båda noderna efter några minuter. Denna metod återställer appliances på ett kontrollerat sätt."
date: "2026-06-19"
kategorie: "HIN-Gateway"
timeToRead: "3 min. lästid"
themen:
  - hin-gateway
draft: false
translationOf: "hin-update-issue-version-15.0.5"
translationId: article-bd1908eec39f9c26
translationReview: automatic
translationSourceHash: d40e2a9637cb0711ca33ed8ffde7827192362f668bebafb7b1f6862fda736d04
translatedAt: 2026-08-08T14:23:43.587Z
url: https://rafaelpfister.ch/sv/blog/hin-mailgateway-15-0-5-atgarda-inloggningsfel-efter-klusteruppdateringen
translationModel: gpt-5.6-terra
---

# HIN Mailgateway 15.0.5: Åtgärda inloggningsfel efter klusteruppdateringen

Vid uppdatering av en HIN Mailgateway från 14.1.4.2 till 15.0.5 kan ett fel i klusterreplikeringen slå ut inloggningen på båda appliances. Enskilda system påverkas inte. Tillverkaren känner till problemet och planerar en korrigering i en kommande version.

**Uppdatering från den 29 juli 2026:** Den aviserade korrigeringen är här. Patch-versionen 15.0.6 förhindrar lösenordsrehashning när klustermedlemmar kör olika firmwareversioner. Det är exakt den konfiguration som utlöste felet som beskrivs här. Kontexten finns i artikeln om [SEPPmail 15.0.6 och 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1); följande återställningsprocedur är fortfarande relevant för kluster som ännu uppdaterar till 15.0.5.

## Felbild

Omedelbart efter uppdateringen går det fortfarande att öppna webbgränssnittet. Efter cirka tio minuter misslyckas inloggningen på båda klusternoderna. Att felet uppstår med fördröjning och på båda systemen pekar på den replikerade klusterkonfigurationen som orsak.

## Återställning

Följande steg ändrar klusterkonfigurationen. Aktuella säkerhetskopior och klusteridentifieraren måste finnas tillgängliga i förväg.

1. Återställ de samtidigt skapade ögonblicksbilderna för båda klusternoderna.
2. Låt en nod vara avstängd efter återställningen.
3. Ladda först ned klusteridentifieraren på den körande noden och upplös sedan klustret.
4. Observera: När klustret har upplösts startar appliance omedelbart om utan ytterligare bekräftelse.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Uppdatera den första noden till version 15.0.5 och stäng sedan av den.
6. Starta den andra noden och upprepa samma steg där.
7. Återskapa klustret enligt tillverkarens dokumentation först när båda systemen fungerar var för sig och har samma versionsnivå.

Detta förfarande förhindrar att en felaktig konfiguration replikeras mellan noderna igen under uppdateringen.

## Källor

1. [SEPPmail-dokumentation – «Cluster / Hochverfügbarkeit»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): Klustertyper och replikering av konfigurationen över alla noder.
2. [SEPPmail-dokumentation – «Administration»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Uppdateringsordning i klustret (frontend före backend) och kravet på identiska versionsnivåer.
3. [HIN Mailgateway: Backup & Disaster Recovery im Cluster](/blog/hin-mailgateway-backup-disaster-recovery): Fördjupad genomgång av klusterreplikering, säkerhetskopiering och återställning.
