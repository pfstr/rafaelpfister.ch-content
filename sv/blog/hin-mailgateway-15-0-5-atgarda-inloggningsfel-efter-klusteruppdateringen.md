---
slug: "hin-mailgateway-15-0-5-atgarda-inloggningsfel-efter-klusteruppdateringen"
title: "HIN Mailgateway 15.0.5: Åtgärda inloggningsfel efter klusteruppdateringen"
navTitle: "Inloggningsfel 15.0.5"
description: "Efter uppdateringen av ett HIN Mailgateway-kluster till version 15.0.5 slutar inloggningen att fungera på båda noderna efter några minuter. Denna metod återställer appliances på ett kontrollerat sätt."
date: "2026-06-19"
kategorie: "HIN-gateway"
timeToRead: "3 min lästid"
themen:
  - hin-gateway
draft: false
translationOf: "hin-update-issue-version-15.0.5"
url: "https://rafaelpfister.ch/sv/blog/hin-mailgateway-15-0-5-atgarda-inloggningsfel-efter-klusteruppdateringen"
---

# HIN Mailgateway 15.0.5: Åtgärda inloggningsfel efter klusteruppdateringen

Vid uppdatering av en HIN Mailgateway från 14.1.4.2 till 15.0.5 kan ett fel i klusterreplikeringen slå ut inloggningen på båda appliances. Enskilda system påverkas inte. Tillverkaren känner till problemet och planerar en korrigering i en kommande version.

## Felbild

Omedelbart efter uppdateringen går det fortfarande att öppna webbgränssnittet. Ungefär tio minuter senare misslyckas inloggningen på båda klusternoderna. Att felet uppstår fördröjt och på båda systemen pekar på den replikerade klusterkonfigurationen som orsak.

## Återställning

Följande steg ändrar klusterkonfigurationen. Aktuella säkerhetskopior och klusteridentifieraren måste finnas tillgängliga i förväg.

1. Återställ de samtidigt skapade ögonblicksbilderna för båda klusternoderna.
2. Låt en nod vara avstängd efter återställningen.
3. Hämta först klusteridentifieraren på den nod som körs och upplös sedan klustret.
4. Obs: Efter upplösningen startar appliance omedelbart om utan ytterligare bekräftelse.

![](../images/hin-update-issue-version-15.0.5/YSaXyzS9jLOD9utH0H2AEDOdnjI.png)

5. Uppdatera den första noden till version 15.0.5 och stäng sedan av den.
6. Starta den andra noden och upprepa samma steg där.
7. Bygg inte upp klustret igen enligt tillverkarens dokumentation förrän båda systemen fungerar var för sig och har samma versionsnivå.

Detta förfarande förhindrar att en felaktig konfiguration replikeras mellan noderna på nytt under uppdateringen.

## Källor

1. [SEPPmail-dokumentation – «Kluster / hög tillgänglighet»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html): Klustertyper och replikering av konfigurationen över alla noder.
2. [SEPPmail-dokumentation – «Administration»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Uppdateringsordning i klustret (frontend före backend) och kravet på identiska versionsnivåer.
3. [HIN Mailgateway: Säkerhetskopiering och katastrofåterställning i kluster](/blog/hin-mailgateway-backup-disaster-recovery): Fördjupad genomgång av klusterreplikering, säkerhetskopiering och återställning.
