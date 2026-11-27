---
title: "Facilitator i Microsoft Teams: Hva møteagenten kan og ikke kan"
navTitle: "Teams Facilitator"
description: "Facilitator er en Copilot-agent som skriver felles notater i planlagte Teams-møter, følger agendaen, overvåker tiden og svarer synlig på spørsmål for alle. Artikkelen forklarer funksjoner, lisens- og administratorkrav, nyttige bruksscenarioer og begrensningene for møtetyper, språk og compliance."
date: "2026-11-27"
kategorie: "Microsoft Teams"
timeToRead: "8 min lesetid"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
protokolle:
  - "lizenzierung"
  - "powershell"
hauptthema: "microsoft-teams"
slug: "facilitator-i-microsoft-teams-hva-moteagenten-kan-og-ikke-kan"
translationId: "article-33919f700e0a6587"
translationOf: teams-facilitator
url: https://rafaelpfister.ch/no/blog/facilitator-i-microsoft-teams-hva-moteagenten-kan-og-ikke-kan
translationSourceHash: 796c74d7d8547585c4342621dfa84e867cf70a2cbc477de7f8e82cd88cbc2d0c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-27T09:42:18.363Z
translationReview: automatic
---

Siden Microsoft har rullet ut Facilitator i Teams-møter, har det dukket opp et nytt alternativ i møteinnstillingene i mange leiere: «Allow Copilot and Facilitator». Navnet sier lite om hva som ligger bak. Facilitator er en KI-agent som deltar i møtet og arbeider synlig for alle. Dette gir både bruksområder og begrensninger.

## Hva Facilitator er

Facilitator er en agent fra Microsoft 365 Copilot som fungerer som en møteassistent i et planlagt Teams-møte. Under møtet skriver den notater på en felles Loop-side som alle interne deltakere kan lese og redigere. Den leser agendaen fra invitasjonen, viser en agendatimer på møtescenen og reagerer på @omtaler i møtechatten.

Forskjellen fra de andre KI-funksjonene i Teams ligger i synligheten:

| Funksjon | Hvem ser resultatet | Tidspunkt |
|---|---|---|
| Copilot i Teams-møter | bare personen som spør | under og etter møtet |
| Intelligent Recap | deltakere med Teams Premium- eller Copilot-lisens | etter møtet |
| Facilitator | alle interne deltakere | under møtet, notater også etterpå |

Et spørsmål til Copilot forblir privat. Et spørsmål til Facilitator og svaret vises i møtechatten for alle. Facilitator arbeider dermed for alle deltakerne i fellesskap.

## Oversikt for brukere

For alle funksjoner gjelder følgende: Møtet er et planlagt Teams-møte i din egen organisasjon uten eksterne deltakere, «Copilot og Facilitator tillatt» er slått på i møteinnstillingene, og talespråket ble valgt ved oppstart. Bare den som starter Facilitator eller gir den oppgaver, trenger en Copilot-lisens. Alle andre interne deltakere ser resultatene uten egen Copilot-lisens.

| Funksjon | Hva du kan gjøre med den | Hvem kan bruke den | Andre forutsetninger |
|---|---|---|---|
| Starte Facilitator | Slå på agenten før eller under møtet | Personer med Microsoft 365 Copilot-lisens | Loop i Teams er slått på, appen «Facilitator» er ikke blokkert |
| Direktenotater | Beslutninger, åpne spørsmål og oppgaver registreres fortløpende i fanen «Notes»; du kan korrigere og supplere dem direkte | Lese og redigere: alle interne deltakere | Transkripsjon kjører (starter automatisk); bare tale på det valgte språket registreres, ikke chatmeldinger |
| Agenda og avkryssing | Saklisten fra invitasjonen vises i chatten, og behandlede punkter krysses av | Se: alle interne deltakere | Agendaen står i invitasjonen eller sendes via @Facilitator i chatten; invitasjonen er ikke kryptert; minst én annen person enn arrangøren er i møtet |
| Timer og påminnelser | Nedtelling for et punkt, varsel ved halv tid, før møteslutt og ved overskridelse | Be om timer: Copilot-lisens; se: alle interne deltakere | ingen |
| Spørsmål om møtet | Spør via @Facilitator i chatten, for eksempel «Hva bestemte vi om punkt 2?»; spørsmål og svar er synlige for alle | Stille spørsmål: Copilot-lisens; lese svar: alle interne deltakere | Spørsmål etter møteslutt bare med alternativet «Under og etter møtet» |
| Spørsmål med nettsøk | Kort research under møtet, for eksempel om en produktversjon | Copilot-lisens | Nettsøk for Copilot er ikke deaktivert i leieren |
| Opprette Word-dokument | Opprette et dokument fra notatene, for eksempel et utkast til referat | Copilot-lisens | Lagring i OneDrive til personen som spør |
| Opprette PowerPoint-presentasjon | Opprette en presentasjon fra møtet | Copilot-lisens | Public Preview: deltakelse i Teams Public Preview |
| Overføre oppgaver til Planner | Overføre registrerte oppgaver til Planner med «Accept to sync» | Copilot-lisens | Public Preview: deltakelse i Teams Public Preview |
| Notater i fysisk møte (smarttelefon) | Protokollføre en samtale på stedet med Teams-appen på smarttelefonen | Copilot-lisens | Teams-app på smarttelefonen |
| Notater i Teams Room | Vise chat, notater og timer på romskjermen i møterommet | Copilot-lisens bare for arrangøren | Teams Rooms on Windows; for uplanlagte fysiske møter Teams Rooms Pro (Public Preview) |

## Funksjoner og status

Microsoft oppgir funksjonene separat i administratordokumentasjonen etter generell tilgjengelighet og Public Preview (per november 2026):

| Funksjon | Status |
|---|---|
| Direktenotater i fanen «Notes» (Loop-side, redigerbar i fellesskap) | generelt tilgjengelig |
| Besvare spørsmål fra møteinnholdet og via nettsøk | generelt tilgjengelig |
| Agendatimer med varsler ved halv tid og før møteslutt | generelt tilgjengelig |
| Opprette Word-dokument fra møtet | generelt tilgjengelig |
| @omtale av Facilitator i møtechatten | generelt tilgjengelig |
| Notater på smarttelefon for fysiske møter | generelt tilgjengelig |
| Teams Rooms: chat, notater og timer i rommet | generelt tilgjengelig |
| Registrere oppgaver og synkronisere med Planner | Public Preview |
| Opprette PowerPoint-presentasjon fra møtet | Public Preview |
| Notater for uplanlagte fysiske møter i Teams Rooms | Public Preview |

Om agendaen: Hvis invitasjonen inneholder en saksliste, overtar Facilitator den, publiserer den i chatten og krysser av punkter når de er behandlet. Avkryssingen skjer med intervaller, ikke umiddelbart. Hvis møtet går over den planlagte sluttiden, varsler den om dette. På forespørsel setter den også en nedtelling for et enkelt punkt.

Om notatene: Facilitator oppsummerer beslutninger, åpne spørsmål og oppgaver. Grunnlaget er utelukkende det som blir sagt i transkripsjonen. Det som bare står i møtechatten, tas ikke med i notatene. Notatene kan når som helst korrigeres og suppleres manuelt; Facilitator fortsetter å skrive på samme side.

## Forutsetninger

Flere betingelser må være oppfylt samtidig for å bruke funksjonen:

- **Lisens:** Den som legger Facilitator til i et møte eller slår den på under møtet, trenger en Microsoft 365 Copilot-lisens i tillegg til en egnet Microsoft 365- og Teams-lisens. Deltakere uten Copilot-lisens kan ikke stille spørsmål til Facilitator, men ser notatene, andres spørsmål og svarene dens.
- **Loop:** Loop-opplevelsene i Teams må være aktivert i leieren. Uten Loop finnes det ingen notater.
- **Transkripsjon:** Når Facilitator slås på, starter transkripsjonen automatisk. Hvis møteinnstillingen «Allow Copilot and Facilitator» er satt til «Only during the meeting», kjører Facilitator uten lagret transkripsjon; spørsmål etter møteslutt er da ikke mulig.
- **Appgodkjenning:** Appen «Facilitator» er tillatt som standard, forutsatt at ikke alle apper er blokkert i Teams Admin Center.

I invitasjonen aktiverer du Facilitator under **Options** > **Copilot and other AI** med innstillingen «Allow Copilot and Facilitator». Alternativt kan den slås på under møtet via Copilot-menyen.

## Facilitator er nedtonet

I møtet viser Teams tre brytere i menyen «Møte-KI»: Copilot, Facilitator og Intelligent oppsummering. Hvis alle tre er nedtonet og meldingen «Du har ikke tilgang til KI i dette møtet» vises, skyldes det lisensen eller selve møtet. Hvis bare Facilitator er nedtonet, skyldes det appen eller Loop.

| Symptom | Mulig årsak | Kontroll |
|---|---|---|
| Alle tre bryterne er grå | Kontoen har verken Microsoft 365 Copilot- eller Teams Premium-lisens | Microsoft 365 Admin Center > Brukere > Lisenser. En nylig tildelt lisens blir ofte først aktiv i Teams etter noen timer eller etter ny pålogging |
| Alle tre bryterne er grå | Øyeblikkelig møte («Meet now»), møte startet fra chatten, kanalmøte eller samtale | Opprett møtet som et planlagt Teams-møte via kalenderen |
| Alle tre bryterne er grå | Møte i en annen organisasjon, deltakelse som ekstern person eller gjest | Facilitator er ikke tilgjengelig i eksterne møter |
| Alle tre bryterne er grå | Møteinnstillingen «Tillat Copilot og Facilitator» er satt til «Av», angitt av arrangøren, en møtemal eller en sensitivitetsetikett | Møteinnstillinger > Copilot og annen KI |
| Alle tre bryterne er grå | Teams-møtepolicyen blokkerer Copilot eller transkripsjon | Teams Admin Center > Møter > Møtepolicyer |
| Bare Facilitator er grå | Appen «Facilitator» er blokkert, for eksempel fordi alle apper er sperret | Teams Admin Center > Teams-apper > Administrer apper |
| Bare Facilitator er grå eller ingen notater | Loop for møtenotater er slått av | `Get-SPOTenant` viser `IsCollabMeetingNotesFluidEnabled` |

Lisensen er den vanligste årsaken. Uten Microsoft 365 Copilot-lisens forblir Facilitator-bryteren grå i hvert møte, uavhengig av møteinnstillinger og policyer.

## Hva Facilitator egner seg til

Facilitator gir størst nytte i møter der noen ellers ville ført referat og dermed ikke kunne delta i diskusjonen. Typiske eksempler:

- **Prosjekt- og statusmøter med fast saksliste:** Agendatimeren og varsler ved halv tid holder tidsplanen, og notatene inneholder beslutninger og åpne punkter uten eget referat.
- **Workshoper med beslutninger:** Den felles Loop-siden er synlig under møtet. Deltakerne korrigerer formuleringer direkte i stedet for å samordne et referat i etterkant via e-post.
- **Spørsmål under møtet:** «Hva bestemte vi om punkt 2?» eller et kort nettsøk om en produktversjon besvarer Facilitator i chatten, synlig for alle.
- **Senere oppsummering:** På forespørsel oppretter Facilitator et Word-dokument fra notatene, som lagres i OneDrive til personen som spør.
- **Fysiske møter:** Med Teams-appen på smarttelefonen eller i et Teams Room protokollføres også møter på stedet.

## Hva Facilitator ikke kan

Begrensningene er tydelig beskrevet i Microsoft-dokumentasjonen og gjelder først og fremst møtetyper, språk og compliance.

**Møtetyper som ikke støttes.** Facilitator kjører bare i planlagte Teams-møter. Øyeblikkelige møter («Meet now»), møter startet fra en chat, Teams-samtaler, kanalmøter samt 1:1- og gruppechatter støttes ikke. Dokumentasjonen oppgir ingen støtte for webinarer og Town Halls.

**Ingen eksterne deltakere.** I møter og chatter med eksterne personer er KI-genererte notater ikke tilgjengelige. Eksterne deltakere kan som hovedregel ikke se notatene. Facilitator kan derfor ikke brukes som referatverktøy i møter med kunder eller leverandører.

**Ett språk per møte.** Talespråket velges ved oppstart. Notater opprettes bare for deler på dette språket. I et møte som veksler mellom tysk og fransk, mangler de franske delene. Hvis det valgte språket avviker fra det som snakkes, opprettes det ingen notater. Sveitsertysk er ikke oppført som et eget språk i listen over støttede Copilot-språk.

**Ingen opptak.** Facilitator slår på transkripsjonen, men ikke opptak. De som trenger bilde og lyd, må starte opptaket separat.

**Ingen agenda fra beskyttede invitasjoner.** Hvis invitasjonen er kryptert med en sensitivitetsetikett, kan Facilitator ikke lese agendaen. Det samme gjelder så lenge bare arrangøren er i møtet, fordi den ikke leser arrangørens kopi av invitasjonen.

**Ingen erstatning for et formelt referat.** Notatene er en KI-oppsummering av transkripsjonen. For styremøter, personalsamtaler eller andre møter med juridisk betydning kreves det fortsatt et gjennomgått og godkjent referat.

## Datalagring og compliance

Notatene lagres som en `.loop`-fil i mappen «Meetings» i OneDrive til personen som startet Facilitator. Microsoft behandler dem på samme måte som transkripsjonsdata. For compliance-ansvarlige er tre begrensninger relevante:

- **eDiscovery:** KI-genererte møtenotater registreres ikke automatisk som skyvedlegg i Microsoft Purview eDiscovery.
- **Oppbevaring:** Oppbevaringsetiketter støttes ikke for skyvedlegg i KI-genererte notater.
- **Sensitivitetsetiketter:** Notatene overtar ikke etiketten fra møtet. En etikett kan settes på Loop-komponenten i etterkant. Innstillinger som «Prevent copy and paste» og vannmerker gjelder ikke for svar og notater fra Facilitator.

For organisasjoner med oppbevaringsplikter betyr dette at notater som skal regnes som forretningsdokumentasjon, må overføres til et lagringssted med fungerende oppbevaring. Microsoft beskriver Purview-aspektene på en egen side (se Kilder).

## Styre og deaktivere Facilitator

I Teams Admin Center finner du appen under **Teams apps** > **Manage apps** ved å søke etter «Facilitator». Der kan den tillates eller blokkeres for hele leieren. For enkelte brukergrupper styrer du appen via app-tillatelsespolicyer eller app-sentrert administrasjon.

Loop-notatene i møter, som Facilitator bygger på, avhenger av en SharePoint-leierinnstilling. Den gjelder for alle brukere og kan ikke angis per person:

```powershell
Connect-SPOService -Url "https://contoso-admin.sharepoint.com"
Set-SPOTenant -IsCollabMeetingNotesFluidEnabled $false
```

<details class="options-details">
<summary>Forklarte alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-Url` | Adressen til leierens SharePoint Admin Center (`<tenant>-admin.sharepoint.com`) |
| `-IsCollabMeetingNotesFluidEnabled` | `$false` slår av felles møtenotater og dermed KI-notatene fra Facilitator for alle brukere, `$true` slår dem på igjen |

</details>

Denne innstillingen fjerner også notatfunksjonen for møter uten Facilitator. Hvis bare agenten skal blokkeres, er det mer målrettet å blokkere appen i Teams Admin Center. For møter uten lagret transkripsjon setter du alternativet for Copilot og Facilitator til «Only during the meeting» i møtepolicyen.

## Kilder

1.  [Set up Facilitator in Microsoft Teams for collaborative AI-generated notes](https://learn.microsoft.com/en-us/microsoftteams/facilitator-teams): Administratordokumentasjon med lisenskrav, funksjonsstatus, datalagring, compliance-begrensninger og appstyring (per 18.11.2026).

2.  [Facilitator in Microsoft Teams meetings](https://support.microsoft.com/en-us/teams/copilot/facilitator-in-microsoft-teams-meetings): Sluttbrukerveiledning for møteinnstillinger, agenda, timer, støttede møtetyper og språkvalg.

3.  [Facilitator in Microsoft Teams Rooms](https://learn.microsoft.com/en-us/microsoftteams/rooms/facilitator-teams-rooms): Fysiske møter i Teams Rooms on Windows, lisens for arrangøren, tilordning av talere i rommet.

4.  [Keep meetings focused and productive with Facilitator in Teams](https://techcommunity.microsoft.com/blog/microsoft365insiderblog/keep-meetings-focused-and-productive-with-facilitator-in-teams/4456029): Microsoft 365 Insider-blogg fra oktober 2025 om transkripsjon, grunnlaget for notater og bruk uten lagret transkripsjon.

5.  [Microsoft Purview: Teams Facilitator](https://learn.microsoft.com/en-us/purview/ai-teams-facilitator): Purview-perspektiv på Facilitator-data for oppbevaring og eDiscovery.

6.  [Manage Loop components in OneDrive and SharePoint](https://learn.microsoft.com/microsoft-365/loop/loop-components-configuration): Leierinnstillingene `IsLoopEnabled` og `IsCollabMeetingNotesFluidEnabled` for Loop-komponenter og møtenotater.
