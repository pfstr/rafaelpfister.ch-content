---
title: "Exchange-sikkerhetsoppdateringer for november 2026: ni sårbarheter, første oppdatering kun for Exchange SE"
navTitle: "Exchange SU 11/2026"
description: "November-SU-en lukker ni sårbarheter i Exchange Server SE, inkludert en kritisk ekstern kjøring av kode der en konto med lave rettigheter er tilstrekkelig. Exchange 2016 og 2019 får ikke flere oppdateringer. Exchange-feilene etter Windows-oppdateringene i oktober løses av Windows-oppdateringen i november."
date: "2026-11-11"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min lesetid"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
slug: "exchange-sikkerhetsoppdateringer-for-november-2026-ni-sarbarheter-forste-oppdatering-kun-for"
translationId: article-cfb739155e545cc3
translationOf: exchange-security-updates-november-2026
url: https://rafaelpfister.ch/no/blog/exchange-sikkerhetsoppdateringer-for-november-2026-ni-sarbarheter-forste-oppdatering-kun-for
translationSourceHash: 1ce238b3219fc9c3670ec3137e807538c62e566ad22bd60fbee45dcf6dce9193
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:25:45.051Z
translationReview: automatic
---

# Exchange-sikkerhetsoppdateringer for november 2026: ni sårbarheter, første oppdatering kun for Exchange SE

Microsoft publiserte 10. november 2026 en sikkerhetsoppdatering (SU) for Exchange Server SE. Ifølge KB-artikkelen lukker den ni sårbarheter. Ingen av dem var offentlig kjent på forhånd, og ifølge Security Update Guide blir ingen aktivt utnyttet. Microsoft klassifiserer én ekstern kjøring av kode som *Critical* og «Exploitation More Likely»; en konto med lave rettigheter er tilstrekkelig. Denne måneden er spesiell av to grunner: Det er den første SU-en som kun utgis for Exchange SE, og Exchange-feilene etter Windows Server-oppdateringene i oktober er løst med Windows-oppdateringene i november.

## Hvilke Exchange-versjoner oppdateringen er tilgjengelig for

SU-en fra 10. november 2026 er bare tilgjengelig for én versjon:

- **Exchange Server Subscription Edition (SE) RTM**: KB5133453, build 15.2.2562.57; offentlig tilgjengelig.

Du kan sammenligne din egen versjon med oversikten over [Exchange-buildnumre](/tools/exchange-builds).

Det finnes ingen oppdatering for Exchange 2016 og 2019. [Oktober-SU-en](/blog/exchange-security-updates-oktober-2026) var den siste, også for kunder i Period 2 ESU-programmet, og ifølge kunngjøringen er dette programmet avsluttet. Ifølge Microsoft er Exchange Online allerede beskyttet; i hybridmiljøer trenger likevel hver Exchange-server SU-en, også rene administrasjonsservere og maskiner med Exchange Management Tools.

## Oversikt over sårbarhetene

| CVE | Type | CVSS |
| --- | --- | --- |
| CVE-2026-118207 | Ekstern kjøring av kode | 8.8 |
| CVE-2026-118149 | Ekstern kjøring av kode | 8.8 |
| CVE-2026-118150 | Rettighetsutvidelse | 8.8 |
| CVE-2026-118154 | Rettighetsutvidelse | 8.8 |
| CVE-2026-118153 | Tjenestenekt | 7.5 |
| CVE-2026-118151 | Rettighetsutvidelse | 7.2 |
| CVE-2026-118157 | Rettighetsutvidelse | 7.2 |
| CVE-2026-118208 | Forfalskning | 7.1 |
| CVE-2026-118152 | Forfalskning | 6.5 |

Alle de ni sårbarhetene kan nås via nettverket (AV:N) og har lav angrepskompleksitet (AC:L) ifølge CVSS-vektoren. Security Update Guide oppgir kun Exchange Server SE RTM som berørt produkt.

**[CVE-2026-118207](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118207) er månedens eneste sårbarhet som Microsoft klassifiserer som *Critical* (ekstern kjøring av kode, CVSS 8.8), og den eneste med vurderingen «Exploitation More Likely». Ifølge FAQ-en i Security Update Guide kan en autentisert angriper med lave rettigheter sende en manipulert forespørsel til en berørt server og dermed kjøre kode på serveren; ingen brukerinteraksjon er nødvendig. Microsoft oppgir utilstrekkelig inndatavalidering (CWE-706) som årsak. Et vanlig postboksbruker, for eksempel med et passord stjålet gjennom phishing, er dermed tilstrekkelig for angrepet.

**[CVE-2026-118149](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118149) er den andre sårbarheten for ekstern kjøring av kode (CVSS 8.8, *Important*). Også her sender en autentisert angriper ifølge FAQ-en en manipulert forespørsel til en berørt tjeneste og kan deretter kjøre kode på målsystemet; årsaken er kodeinjeksjon (CWE-94). Microsoft vurderer utnyttelse som «Exploitation Unlikely».

**[CVE-2026-118153](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118153) (tjenestenekt, CVSS 7.5) er den eneste sårbarheten som verken krever en konto eller brukerinteraksjon. Ifølge beskrivelsen kan en uautentisert angriper sette tjenesten ut av drift over nettverket; årsaken er ineffektiv algoritmisk kompleksitet (CWE-407). Ifølge vektoren er kun tilgjengeligheten berørt, ikke konfidensialitet eller integritet.

De fire rettighetsutvidelsene skiller seg fra hverandre i kravene: For [CVE-2026-118150](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118150) og [CVE-2026-118154](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118154) (begge CVSS 8.8) er en konto med lave rettigheter tilstrekkelig, mens angriperen allerede må ha høye rettigheter for [CVE-2026-118151](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118151) og [CVE-2026-118157](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118157) (begge CVSS 7.2). Ved CVE-2026-118151 kan angriperen ifølge FAQ-en oppnå SYSTEM-rettigheter.

Av de to forfalskningssårbarhetene krever [CVE-2026-118208](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118208) (CVSS 7.1) en konto med lave rettigheter. [CVE-2026-118152](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118152) (CVSS 6.5) er en Cross-Site Request Forgery: Angriperen trenger ingen konto, men en bruker må medvirke (UI:R). Alle sårbarhetene unntatt CVE-2026-118207 og CVE-2026-118149 er klassifisert som «Exploitation Less Likely».

## Exchange 2016 og 2019: ingen oppdatering, ingen videre testing

Kunngjøringen nevner Exchange 2016 og 2019 kun i en merknad: Disse versjonene testes ikke lenger for sårbarheter, og det kommer ikke flere oppdateringer. Microsoft skriver at de må anses som permanent sårbare («permanently vulnerable») og at organisasjonen umiddelbart bør oppdatere. At de ni CVE-ene fra november bare oppgir Exchange SE, sier derfor ingenting om hvorvidt de eldre versjonene er berørt. Frankys Web påpeker at Exchange SE og 2019 CU15 er teknisk nært beslektet, og at minst en del av sårbarhetene trolig også finnes i Exchange 2019.

For drift betyr dette:

1. **Exchange 2019 CU15 og eldre, Exchange 2016**: Versjonen forblir frosset på oktober-SU-en (med ESU) eller eldre. De ni sårbarhetene fra november forblir åpne på disse serverne dersom de finnes der, og Microsoft tester dette ikke lenger.

2. **Migrering**: Microsoft anbefaler umiddelbar overgang til Exchange SE.

3. **Hybridmiljøer**: Exchange Online begrenser og blokkerer e-postflyten fra utdaterte lokale servere; reglene finnes i [artikkelen om transporthåndheving](/blog/exchange-online-transport-enforcement-hybrid-server).

## Løste problemer

**Exchange-feil etter Windows Server-oppdateringene i oktober** (alle versjoner). Problemet med `MapiExceptionMdbOffline` (ec=1142) løses ikke av Exchange-SU-en: Ifølge kunngjøringen må Windows Server-oppdateringene fra november 2026 installeres. Release Notes for disse oppdateringene lister løsningen under «[Exchange Server]» og oppgir problemet som løst med oppdateringer fra 10. november 2026: KB5131080 for Windows Server 2025, KB5131090 for Windows Server 2022, KB5131085 for Windows Server 2019 og KB5131291 for Windows Server 2016. Ifølge kunngjøringen spiller rekkefølgen du installerer Exchange-SU-en og Windows-oppdateringen i ingen rolle. Hvis du har angitt registerverdien `EpmapSavedContextCap` fra [artikkelen om Exchange-feilene](/blog/exchange-windows-updates-oktober-2026), kan du la den stå; etter installasjon av novemberoppdateringene for Windows Server er den ifølge Microsoft ikke lenger nødvendig.

**Feil ledig/opptatt-visning via Microsoft Graph i hybridmiljøer** (kun Exchange SE). Ifølge [KB5135867](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5135867) fikk brukere med lokal postboks i klassisk Outlook ikke korrekt ledig/opptatt-informasjon for Exchange Online-postbokser når Graph API var aktivert; kun tidsblokker ble vist. Med Exchange Web Services (EWS) i stedet for Graph API var visningen korrekt. Problemet er løst med KB5133453 eller en nyere oppdatering. Støtteartikkelen nevner ikke noe etterarbeid. Dersom du gikk tilbake til EWS på grunn av denne feilen, bør du etter installasjonen kontrollere ledig/opptatt-oppslaget for en Exchange Online-postboks før du aktiverer Graph API igjen.

## Kjente problemer

Verken kunngjøringen eller KB5133453 nevner Known Issues for november-SU-en (per 11. november 2026). Kunngjøringen har ingen kommentarer fra Exchange-teamet, og Microsoft påpeker at dokumentasjonen kan være ufullstendig på publiseringstidspunktet. Det finnes ikke lenger noen åpne Known Issues fra tidligere måneder: Problemene med septemberoppdateringene har vært løst siden oktober-SU-en, og Windows-problemet fra oktober er løst med novemberoppdateringene for Windows Server (se ovenfor).

## Installasjon og etterarbeid

Microsoft oppgir den kjente fremgangsmåten i kunngjøringen: Inventariser med [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) hvilke servere som trenger oppdateringer eller manuelle tiltak, installer SU-en, start serveren på nytt og kontroller at alle Exchange-tjenester kjører. Dersom tjenester er deaktivert etter installasjonen, ble installasjonen avbrutt. Ved feil under eller etter installasjonen viser Microsoft til [SetupAssist-skriptet](https://aka.ms/ExSetupAssist). Etter installasjonen viser Health Checker ifølge KB5133453 om SU-en er riktig installert og om ytterligere tiltak er nødvendig.

Installasjonsfilen heter ifølge KB5133453 `ExchangeSubscriptionEdition-KB5133453-x64-en.exe` (Download Center: «SU12»). SU-en erstatter oktober-SU-en KB5128895. SU-er er kumulative: Hvis du har installert en eldre SU på Exchange SE RTM, installerer du direkte november-SU-en.

For hybridmiljøer gjelder følgende ifølge FAQ-en: Dersom Auth-sertifikatet byttes etter installasjon av en SU, må Hybrid Configuration Wizard kjøres på nytt. På Windows Server 2025 vises ikke installerte Exchange-SU-er i Kontrollpanel; Microsoft fraråder å avinstallere SU-er.

Fra tidligere måneder gjenstår disse etterarbeidene dersom de ennå ikke er utført: Fjern ledig/opptatt-overstyringen `EnableRouteThroughMSGraphFeature` (se [oktoberartikkelen](/blog/exchange-security-updates-oktober-2026)), fjern Wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` (se [septemberartikkelen](/blog/exchange-security-updates-september-2026)) og kontroller om CVE-2026-42897-tiltaket (M2.1.0) fortsatt er aktivt (se [artikkelen om juli-SU-en](/blog/exchange-security-updates-juli-2026)).

## Anbefalt fremgangsmåte

Installer november-SU-en raskt på alle servere med Exchange SE og på maskinene med Management Tools: For den kritiske eksterne kjøringen av kode CVE-2026-118207 er en konto med lave rettigheter tilstrekkelig, og Microsoft anser utnyttelse som mer sannsynlig. Installer novemberoppdateringene for Windows Server i samme vedlikeholdsvindu, slik at årsaken til feilene i oktober blir løst. Det finnes ikke lenger noen oppdatering for Exchange 2016 og 2019; ifølge Microsoft må disse serverne anses som permanent sårbare og bør umiddelbart erstattes med Exchange SE.

## Kilder

1.  [Released: November 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-november-2026-exchange-server-security-updates/4570368): Kunngjøring fra 10. november 2026 med støttet versjon, merknad om Exchange 2016/2019, det løste Windows-problemet og FAQ (ingen kommentarer fra Exchange-teamet, per 11. november 2026).

2.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM November 10, 2026 (KB5133453) – Microsoft Support](https://support.microsoft.com/help/5133453): CVE-liste, løst problem, installasjonsfil, erstattet oppdatering og merknad om Health Checker.

3.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Type, CVSS, alvorlighetsgrad, vurdering av utnyttelse, FAQ og berørt produkt for de ni CVE-ene fra november.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Buildnummer for november-SU-en.

5.  [Microsoft graph displays incorrect free/busy availability information in Microsoft Exchange server SE hybrid setup – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5135867): Symptomer og løsning med KB5133453.

6.  [Exchange servers may become unstable after installation of October 2026 Windows updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/exchange-servers-may-become-unstable-after-installation-of/ba-p/4565382): Innlegg fra Exchange-teamet om Windows-problemet i oktober, som kunngjøringen viser til.

7.  [November 10, 2026: KB5131080 (OS Build 26100.33822) – Microsoft Support](https://support.microsoft.com/help/5131080): Release Notes for Windows Server 2025 med Exchange-løsningen og Known Issue markert som løst.

8.  [November 10, 2026: KB5131090 (OS Build 20348.5870) – Microsoft Support](https://support.microsoft.com/help/5131090): Release Notes for Windows Server 2022 med samme løsning.

9.  [November 10, 2026: KB5131085 (OS Build 17763.9487) – Microsoft Support](https://support.microsoft.com/help/5131085): Release Notes for Windows Server 2019 med samme løsning.

10. [November 10, 2026: KB5131291 (OS Build 14393.9688) – Microsoft Support](https://support.microsoft.com/help/5131291): Release Notes for Windows Server 2016 med samme løsning.

11. [Neue Sicherheitsupdates für Exchange Server (November 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-november-2026/): Tyskspråklig gjennomgang av de ni CVE-ene og vurdering for Exchange 2016 og 2019.

12. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventarisering av CU/SU-versjoner før og etter installasjon.
