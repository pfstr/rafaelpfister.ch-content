---
title: "Exchange-sikkerhetsoppdateringer fra oktober 2026: seks sårbarheter, siste oppdatering for Exchange 2016 og 2019"
navTitle: "Exchange SU 10/2026"
description: "Oktober-SU-en lukker seks sårbarheter, inkludert kritisk ekstern kjøring av kode via en manipulert e-post, og retter .ics-feilen samt WordBreaker-deadlåsen. For Exchange 2016 og 2019 er dette den siste oppdateringen, også i ESU-programmet."
date: "2026-10-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min lesetid"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "exchange-sikkerhetsoppdateringer-fra-oktober-2026-seks-sarbarheter-siste-oppdatering-for"
translationId: article-173db9431239a95f
translationOf: exchange-security-updates-oktober-2026
translationSourceHash: a834949e09b3f4b6e7f16be2dd6cf0ecedd0ccf1f4fc8dd6f409336407aaaca2
translationModel: gpt-5.6-terra
translatedAt: 2026-10-16T08:27:07.708Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/exchange-sikkerhetsoppdateringer-fra-oktober-2026-seks-sarbarheter-siste-oppdatering-for
---

# Exchange-sikkerhetsoppdateringer fra oktober 2026: seks sårbarheter, siste oppdatering for Exchange 2016 og 2019

Microsoft publiserte 13. oktober 2026 sikkerhetsoppdateringer (SU-er) for Exchange Server SE, Exchange 2019 og Exchange 2016. Ifølge KB-artiklene lukker de seks sårbarheter. Ingen av dem var offentlig kjent på forhånd, og ifølge Security Update Guide blir ingen aktivt utnyttet. Microsoft vurderer imidlertid to som «Exploitation More Likely», deriblant en Remote Code Execution vurdert som *Critical*, som en uautentisert angriper kan utløse med en manipulert e-post. Denne måneden er spesiell av to grunner: SU-en retter de to åpne Known Issues fra septemberoppdateringene (HTTP 500 for publiserte kalendere og ContentEngine-deadlåsen i Exchange SE), og for Exchange 2016 og 2019 er dette ifølge Microsoft den siste sikkerhetsoppdateringen noensinne, også for kunder i Period-2-ESU-programmet.

## Hvilke Exchange-versjoner oppdateringen er tilgjengelig for

SU-ene fra 13. oktober 2026 er tilgjengelige for følgende versjoner:

- **Exchange Server Subscription Edition (SE) RTM**: KB5128895, build 15.2.2562.56; offentlig tilgjengelig.
- **Exchange Server 2019 CU15**: KB5128896, build 15.2.1748.55; kun via **Period-2-ESU-programmet**.
- **Exchange Server 2019 CU14**: KB5128897, build 15.2.1544.50; kun via Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5128898, build 15.1.2507.77; kun via Period 2 ESU.

Du kan sammenligne din egen versjon med oversikten over [Exchange-buildnumre](/tools/exchange-builds).

Exchange 2016 og 2019 er out of support. Ifølge KB-artiklene får organisasjoner i Period-2-ESU-programmet sikkerhetsoppdateringer frem til slutten av oktober 2026; alle andre bør migrere til Exchange SE. Exchange Online er ifølge kunngjøringen allerede beskyttet; i hybridmiljøer trenger likevel hver Exchange-server SU-en, også rene administrasjonsservere og maskiner med Exchange Management Tools. Hva ESU-slutten konkret betyr, står i avsnittet [Siste oppdatering for Exchange 2016 og 2019](#letztes-update-für-exchange-2016-und-2019).

## Oversikt over sårbarhetene

| CVE | Type | CVSS |
| --- | --- | --- |
| CVE-2026-70279 | Spoofing (Cross-Site-Scripting) | 9.3 |
| CVE-2026-77910 | Elevation of Privilege | 8.8 |
| CVE-2026-70571 | Remote Code Execution | 8.1 |
| CVE-2026-87692 | Elevation of Privilege | 7.8 |
| CVE-2026-87695 | Elevation of Privilege | 7.8 |
| CVE-2026-87696 | Elevation of Privilege | 7.5 |

Alle de fire KB-artiklene oppgir de samme seks CVE-ene. I Security Update Guide mangler imidlertid Exchange 2019 CU15 for CVE-2026-87692; der er bare Exchange SE, 2019 CU14 og 2016 CU23 oppført som berørt (per 14. oktober 2026). Frankys Web tolker dette slik at 2019 CU15 ikke er berørt av denne sårbarheten. For installasjonen spiller forskjellen ingen rolle, siden SU-en på CU15 uansett lukker alle de øvrige sårbarhetene.

**[CVE-2026-70571](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70571)** er månedens eneste sårbarhet som Microsoft vurderer som *Critical* (Remote Code Execution, CVSS 8.1). Årsaken er et stakkbasert bufferoverløp. Ifølge FAQ-en i Security Update Guide kan en uautentisert angriper sende en manipulert e-post til en berørt Exchange-server og dermed utløse en intern konverteringsprosess; ingen brukerinteraksjon er nødvendig. Angrepskompleksiteten er høy ifølge CVSS-vektoren (AC:H); Microsoft skriver at vellykket utnyttelse ikke er garantert og krever inngående forståelse av målmiljøet. Likevel vurderer Microsoft utnyttelse som «Exploitation More Likely». Angriperen trenger ikke en konto i organisasjonen, det er nok at serveren mottar e-post.

**[CVE-2026-70279](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70279)** har den høyeste verdien med CVSS 9.3 og er også vurdert som «Exploitation More Likely». Det dreier seg om Cross-Site-Scripting: En uautentisert angriper sender en manipulert lenke; åpner en administrator den, kan angriperen utføre handlinger med administratorens rettigheter og dermed få tilgang til eller endre organisasjonens data. Ifølge FAQ-en forutsetter dette brukerinteraksjon, vanligvis gjennom sosial manipulering. Alvorlighetsgraden er *Important*.

**[CVE-2026-77910](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-77910)** (Elevation of Privilege, CVSS 8.8) krever bare en konto med lave rettigheter. Ifølge FAQ-en kan en angriper manipulere en delt hurtigbuffer i Exchange Control Panel og dermed få installert et ondsinnet e-posttillegg i postboksen til en annen bruker uten at denne trenger å gjøre noe. Microsoft beskriver samme angrepsvei for [CVE-2026-87696](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87696) (CVSS 7.5), der som følge av utilstrekkelig kontroll av kryptografiske signaturer og med høy angrepskompleksitet.

De to øvrige sårbarhetene, [CVE-2026-87692](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87692) (deserialisering av ikke-klarerte data) og [CVE-2026-87695](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87695), er lokale rettighetsutvidelser med CVSS 7.8: En innlogget angriper med lave rettigheter kan ifølge Microsoft få SYSTEM-rettigheter. Begge er vurdert som «Exploitation Less Likely», det samme gjelder CVE-2026-77910 og CVE-2026-87696.

Om **CVE-2026-96940** fra september-v2: Oktoberkunngjøringen nevner ikke v2. Ifølge FAQ-en i kunngjøringen er SU-er kumulative; den som er på en støttet CU, installerer bare den nyeste SU-en. Frankys Web bekrefter at oktober-SU-en inneholder rettelsen fra v2. Servere som fortsatt kjører den første september-SU-en fra 8. september, kan derfor installere oktober-SU-en direkte; detaljer om v2 finnes i [artikkelen om september-v2](/blog/exchange-security-updates-september-2026-v2).

## Siste oppdatering for Exchange 2016 og 2019

Kunngjøringen er entydig på dette punktet: Oktober 2026 er den siste måneden der kunder i Period-2-ESU-programmet får nye sikkerhetsoppdateringer for Exchange 2016 og 2019. Fra neste utgivelse publiseres oppdateringer bare for Exchange SE. Fremtidige kunngjøringer og CVE-meldinger vil ikke lenger nevne 2016 og 2019, og Microsoft skriver uttrykkelig: «We will stop testing Exchange 2016 and 2019 for vulnerabilities.» At en CVE ikke nevner disse versjonene, betyr derfor ikke at de ikke er berørt; Microsoft understreker i kunngjøringen at de gamle versjonene ikke blir sikre av den grunn. Frankys Web påpeker i tillegg at Exchange SE og 2019 CU15 teknisk sett er svært like, slik at fremtidige SE-sårbarheter med høy sannsynlighet også berører 2019.

Ifølge kunngjøringen kan Period 2 ESU fortsatt kjøpes i en begrenset periode. Det bringer imidlertid en organisasjon bare opp til nivået for oktober 2026-oppdateringen; det kommer ingen flere oppdateringer. De som allerede har kjøpt en ESU og har spørsmål om tilgang, kan ifølge KB-artiklene henvende seg til `ExchangeandSfBServerESUInquiry@service.microsoft.com`.

For drift betyr dette:

1. **Exchange 2016 og 2019 med ESU**: Installer oktober-SU-en. Deretter fryses versjonen, og enhver sårbarhet som blir kjent senere, forblir åpen.

2. **Exchange 2016 og 2019 uten ESU**: Disse serverne har ikke fått sikkerhetsoppdateringer siden mai 2026. Microsoft anbefaler migrering til Exchange SE så snart som mulig.

3. **Hybridmiljøer**: I tillegg til manglende oppdateringer begrenser og blokkerer Exchange Online e-postflyten fra utdaterte lokale servere; reglene finnes i [artikkelen om Transport Enforcement](/blog/exchange-online-transport-enforcement-hybrid-server).

## Rettede problemer og hva du må gjøre etterpå

**Publiserte kalendere (.ics) gir HTTP 500 til kalenderapper** (alle versjoner). Problemet, som har eksistert siden august-SU-en, er rettet med oktober-SU-en; [KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672) oppgir nå installasjon av de fire oktoberoppdateringene eller nyere som løsning. Supportartikkelen beskriver ikke lenger den tidligere URL-rewrite-løsningen på nettstedet «Exchange Back End», og Microsoft gir ingen veiledning om fjerning av regelen (per 14. oktober 2026). Hvis du har opprettet regelen, dokumenter hvilke servere den er aktiv på, og test et kalenderabonnement før du fjerner den.

**ContentEngine-deadlås på grunn av manglende koreanske WordBreaker-regelfiler** (bare Exchange SE). Ifølge [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) er problemet rettet med KB5128895 eller en nyere oppdatering, og etter installasjon av oktoberoppdateringen er «no additional action» nødvendig. Dette gjelder også servere der du har brukt den manuelle løsningen fra [septemberartikkelen](/blog/exchange-security-updates-september-2026#workaround-für-den-wordbreaker-deadlock): Microsoft krever ikke at de kopierte regelfilene fjernes. Servere uten løsning trenger ikke lenger å legge til filene manuelt.

**Ledig/opptatt for delegerte postbokser i hybridmiljøer med ren Graph API-konfigurasjon** (bare Exchange SE). Dette problemet rettes ikke av oktober-SU-en: [KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) melder nå «The issue has been fixed in the service». Motsetningen fra [v2-artikkelen](/blog/exchange-security-updates-september-2026-v2) er dermed avklart. Hvis du har angitt løsningen, fjern den igjen i henhold til KB5127092:

```powershell
Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=True"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Alternativene forklart</summary>

| Alternativ | Effekt |
|---|---|
| `Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature` | Viser gjeldende verdi for overstyringen; bare ved `Enabled=False` er neste kommando nødvendig. |
| `-Parameters "Enabled=True"` | Slår videresending av tilgjengelighetsforespørsler via Microsoft Graph på igjen. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Retter diagnosekallet mot topologitjenesten. |
| `-Component VariantConfiguration -Argument Refresh` | Laster Variant Configuration på nytt, slik at endringen trer i kraft umiddelbart. |

</details>

Kontroller deretter ledig/opptatt-forespørselen for en delegert Exchange Online-postboks fra Outlook eller OWA.

## Kjente problemer

Verken kunngjøringen eller de fire KB-artiklene nevner Known Issues for oktober-SU-en (per 14. oktober 2026). Den 15. oktober 2026 supplerte Microsoft kunngjøringen: Windows Server-oppdateringene fra oktober 2026 kan føre til at Exchange-servere feiler etter omtrent én dag med `MapiExceptionMdbOffline` (ec=1142), også uten installert Exchange-SU. Symptomer og fremgangsmåte finnes i [artikkelen om Exchange-feil etter Windows-oppdateringene](/blog/exchange-windows-updates-oktober-2026). De tre åpne problemene fra septemberoppdateringene er dermed løst: to gjennom oktober-SU-en, ett gjennom en korrigering i Exchange Online (se over). Wrapper-problemet i delte postbokser var allerede rettet med september-SU-en.

## Installasjon og oppfølging

Microsoft angir den kjente fremgangsmåten i kunngjøringen: inventariser med [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), finn oppdateringsveien med [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) ved utdatert CU-versjon, installer SU-en, start serveren på nytt og kontroller at alle Exchange-tjenester kjører. Ved feil viser Microsoft til [SetupAssist-skriptet](https://aka.ms/ExSetupAssist). Etter installasjonen viser Health Checker ifølge KB5128895 om SU-en er riktig installert og om ytterligere trinn er nødvendige. Security Update Guide oppgir ingen påkrevd omstart for oktoberbyggene; kunngjøringen anbefaler likevel dette etter oppsettet.

Installasjonsfilen for Exchange SE heter ifølge KB5128895 `ExchangeSubscriptionEdition-KB5128895-x64-en.exe` (Download Center: «SU11»), for Exchange 2019 CU15 `Exchange2019-KB5128896-x64-en.exe`, for CU14 `Exchange2019-KB5128897-x64-en.exe` og for Exchange 2016 `Exchange2016-KB5128898-x64-en.exe`. Pakkene er CU-spesifikke.

For hybridmiljøer gjelder ifølge FAQ-en: Hvis autentiseringssertifikatet byttes etter installasjon av en SU, må Hybrid Configuration Wizard kjøres på nytt. På Windows Server 2025 vises installerte Exchange-SU-er ikke i Kontrollpanel; Microsoft fraråder å avinstallere SU-er.

Fra tidligere måneder gjenstår disse etterarbeidene dersom de ikke allerede er utført: Fjern Wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` (se [septemberartikkelen](/blog/exchange-security-updates-september-2026)) og kontroller om CVE-2026-42897-avbøtingen (M2.1.0) fortsatt er aktiv (se [artikkelen om juli-SU-en](/blog/exchange-security-updates-juli-2026)).

## Anbefalt fremgangsmåte

Installer oktober-SU-en raskt på alle Exchange-servere og maskiner med Management Tools: Den kritiske Remote Code Execution-sårbarheten CVE-2026-70571 kan utløses uten konto via en e-post, og Microsoft vurderer utnyttelse av den som mer sannsynlig, på samme måte som Cross-Site-Scripting-sårbarheten CVE-2026-70279. Servere med den første september-SU-en får dermed også rettelsen for CVE-2026-96940. Fjern deretter ledig/opptatt-overstyringen dersom den er angitt, og kontroller publiserte kalendere. For Exchange 2016 og 2019 er dette siste oppdatering; planlegg migreringen til Exchange SE nå, for fra november får disse serverne ikke flere sikkerhetsoppdateringer.

## Kilder

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-october-2026-exchange-server-security-updates/ba-p/4563014): Kunngjøring fra 13. oktober 2026 med støttede versjoner, rettede problemer, slutten på oppdateringer for Exchange 2016/2019 og FAQ (lest via RSS-feeden til Exchange Team Blogs; ingen kommentarer fra Exchange-teamet, per 14. oktober 2026).

2.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM October 13, 2026 (KB5128895) – Microsoft Support](https://support.microsoft.com/help/5128895): CVE-liste, rettede problemer, installasjonsfil og merknad om Health Checker.

3.  [Description of the security update for Microsoft Exchange Server 2019 CU15 October 13, 2026 (KB5128896) – Microsoft Support](https://support.microsoft.com/help/5128896): KB-artikkel for Exchange 2019 CU15 med ESU-merknad.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU14 October 13, 2026 (KB5128897) – Microsoft Support](https://support.microsoft.com/help/5128897): KB-artikkel for Exchange 2019 CU14 med ESU-kontaktadresse.

5.  [Description of the security update for Microsoft Exchange Server 2016 CU23 October 13, 2026 (KB5128898) – Microsoft Support](https://support.microsoft.com/help/5128898): KB-artikkel for Exchange 2016 CU23.

6.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Type, CVSS, alvorlighetsgrad, vurdering av utnyttelse, FAQ og berørte build-versjoner for de seks CVE-ene fra oktober samt CVE-2026-96940.

7.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Buildnumre for oktober-SU-ene og de foregående oppdateringene.

8.  [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Rettelse med oktoberoppdateringene.

9.  [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Rettelse med KB5128895, ingen ytterligere trinn nødvendig.

10. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Korrigering i tjenesten og kommandoer for å tilbakestille SettingOverride.

11. [Neue Sicherheitsupdates für Exchange Server (Oktober 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-oktober-2026/): Tyskspråklig gjennomgang av de seks CVE-ene, merknad om den inkluderte v2-rettelsen og vurdering av ESU-slutten.

12. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventarisering av CU/SU-versjoner før og etter installasjonen.
