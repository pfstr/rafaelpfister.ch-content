---
title: "Koble SEPPmail Admin-GUI til Active Directory: Konfigurer LDAP-autentisering fra 15.0.6"
navTitle: "Admin-LDAP-innlogging"
description: "Siden fastvare 15.0.6 kan administratorer av SEPPmail-appliancen autentisere seg mot en ekstern LDAP-server som Active Directory, inkludert gruppemapping til den lokale admin-gruppen. Konfigurasjon under User > Advanced Settings, trinn for trinn."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 min lesetid"
themen:
  - seppmail
slug: "koble-seppmail-admin-gui-til-active-directory-konfigurer-ldap-autentisering-fra-15-0-6"
translationId: "article-21092a3dad6b84cb"
draft: false
translationOf: seppmail-admin-gui-ldap-authentifizierung
url: https://rafaelpfister.ch/no/blog/koble-seppmail-admin-gui-til-active-directory-konfigurer-ldap-autentisering-fra-15-0-6
translationSourceHash: bb8386d1f880934d4811eb317bcd51d47900fdd493dad90b1d7752bfc25ba55c
translationModel: gpt-5.6-terra
translatedAt: 2026-07-30T12:48:47.369Z
translationReview: automatic
---

# Koble SEPPmail Admin-GUI til Active Directory: Konfigurer LDAP-autentisering fra 15.0.6

Frem til fastvare 15.0.5 kjente administrasjonsgrensesnittet til SEPPmail Secure E-Mail Gateway kun lokale kontoer. De som ønsket å arbeide ryddig, opprettet en egen lokal bruker for hver administrator og la vedkommende til i gruppen admin. Dette fungerer, men har de vanlige ulempene med lokale kontoer: egne passord per appliance, ingen sentral offboarding og ingen håndheving av passordreglene fra katalogtjenesten. Dette endres med patch-utgivelsen 15.0.6. Admin-GUI-en autentiserer administratorer mot en ekstern LDAP-server som Active Directory ved behov, og tilordner AD-grupper til lokale grupper på appliancen.

De øvrige endringene i utgivelsen er oppsummert i artikkelen om [SEPPmail 15.0.6 og 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1). Her handler det kun om den nye eksterne autentiseringen.

## Hva funksjonen gjør

Ifølge Extended Release Notes legger 15.0.6 til en ny seksjon, **External Authentication**, under **User > Advanced Settings**. Dermed autentiserer Admin-GUI-en seg mot en ekstern LDAP-server, og eksterne grupper (for eksempel AD-sikkerhetsgrupper) mappes til lokale grupper på appliancen.

Eksternt autentiserte brukere vises lokalt på appliancen og oppfører seg som lokale brukere, med én forskjell: Passordet deres kan ikke endres på appliancen, siden det ligger på den eksterne LDAP-serveren. Passordansvaret flyttes dermed fullt og helt til katalogen.

Viktig avgrensning: Appliancen hadde allerede ekstern autentisering, men kun for GINA-nettgrensesnittet, konfigurert per Managed Domain (seksjonen External authentication i domeneoppsettet). Nytt i 15.0.6 er at også tilgang til selve administrasjonsgrensesnittet går via LDAP.

Jeg tester fortsatt om HIN Mailgateway også har fått LDAP-innlogging, og oppdaterer artikkelen deretter. Siden HIN-appliancene er basert på samme SEPPmail-fastvare, går jeg ut fra det.

## Forutsetninger

Før oppsettet bør tre ting være på plass:

- **Fastvare 15.0.6.1:** Funksjonen kommer med 15.0.6; på grunn av de to RuleEngine-feilene i utgivelsen er hotfixen 15.0.6.1 det riktige valget direkte.
- **En LDAP-kompatibel katalog:** Active Directory, OpenLDAP eller tilsvarende. Dersom brukerne kun finnes i Entra ID, som selv ikke snakker LDAP, bygger [Microsoft Entra Domain Services](/blog/microsoft-entra-domain-services-ldap-kerberos) broen.
- **En bind-konto i katalogen:** En dedikert, uprivilegert tjenestekonto med lesetilgang som appliancen bruker til LDAP-søket. Ingen domeneadministrator.
- **En AD-gruppe for gateway-administratorene:** For eksempel en sikkerhetsgruppe kalt SEPPmail-Admins, som senere mappes til den lokale gruppen admin. Medlemskap i denne gruppen avgjør da om brukeren får full administrativ tilgang.

TLS er aktivert som standard i tilkoblingsinnstillingene og bør forbli aktivert; administratorenes innloggingsdata hører ikke hjemme ukryptert på nettverket. Appliancen må kunne nå LDAP-serveren på den konfigurerte porten (vanligvis 636 for LDAPS).

## Oppsett under User > Advanced Settings

Konfigurasjonen finnes i Admin-GUI-en under **User > Advanced Settings** i seksjonen **External Authentication**, og består av fire blokker.

**1. Connection Settings:** Avkrysningsboksen *Authenticate users to external LDAP server (e.g. Active Directory)* aktiverer funksjonen. Deretter følger serveradresse, port, alternativet *TLS required* samt Bind DN og Bind Password for tjenestekontoen.

**2. User Attributes:** Her defineres hvordan appliancen finner brukerobjekter: LDAP Object Class (i Active Directory typisk person), Search Base (OU-en eller containeren der administratorkontoene ligger) og e-postattributtet (standard: mail).

**3. Group Attributes:** Tilsvarende opplysninger for gruppeobjekter, slik at appliancen kan løse opp gruppemedlemskapene.

**4. Mapping Settings:** Den avgjørende delen. Under *Remote Group* velges gruppen fra LDAP-serveren, og under *Local Group* velges én eller flere lokale grupper som den tilordnes til. For full administrativ tilgang er dette gruppen admin; medlemmene er likestilt med standardbrukeren admin. De som ønsker differensiering, mapper i stedet til begrensede grupper som readonly admin eller til funksjonsrelaterte grupper på appliancen.

Før lagring er det verdt å bruke den innebygde **Login Test**: Med brukernavn og passord fra en testkonto kan du kontrollere at tilkobling, søk og autentisering fungerer før konfigurasjonen aktiveres.

## Eksempelkonfigurasjoner

Følgende verdier må tilpasses eget miljø (eksempeldomene example.com). Feltnavnene tilsvarer seksjonen External Authentication på appliancen.

### Active Directory

| Felt | Verdi |
|---|---|
| Server | dc01.example.com |
| Port | 636 |
| TLS required | aktivert |
| Bind DN | CN=svc-seppmail,OU=ServiceAccounts,DC=example,DC=com |
| Bind Password | Passord for tjenestekontoen |
| User: LDAP Object Class | person |
| User: Search Base | OU=IT,DC=example,DC=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | group |
| Group: Search Base | OU=Groups,DC=example,DC=com |
| Mapping: Remote Group | SEPPmail-Admins |
| Mapping: Local Group | admin |

Merknader om Active Directory: Enhver tilgjengelig Domain Controller kan brukes som server; i miljøer med flere lokasjoner anbefales en DC på samme sted eller et alias som peker til flere DC-er. Port 636 er LDAPS; for dette må sertifikatet til DC-en kunne valideres av appliancen. Search Base bør avgrenses så stramt at den inneholder administratorkontoene, men ikke hele katalogen. Attributtet mail må være vedlikeholdt på AD-kontoene.

### OpenLDAP

| Felt | Verdi |
|---|---|
| Server | ldap01.example.com |
| Port | 636 |
| TLS required | aktivert |
| Bind DN | cn=seppmail,ou=services,dc=example,dc=com |
| Bind Password | Passord for tjenestekontoen |
| User: LDAP Object Class | inetOrgPerson |
| User: Search Base | ou=people,dc=example,dc=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | groupOfNames |
| Group: Search Base | ou=groups,dc=example,dc=com |
| Mapping: Remote Group | seppmail-admins |
| Mapping: Local Group | admin |

Merknader om OpenLDAP: I typiske oppsett ligger brukere som inetOrgPerson under ou=people. For gruppene er groupOfNames det pålitelige valget, siden medlemskap der representeres via member-attributtet med full DN. posixGroup-grupper fører derimot kun medlemmene som memberUid (brukernavn i stedet for DN); om appliancen kan løse dette opp, er ikke dokumentert og bør kontrolleres med Login Test før overgang. Dersom serveren kun kjører med STARTTLS på port 389, skal den tilsvarende porten angis i serverfeltet; tilkoblingen må under ingen omstendigheter gå ukryptert.

## Driftsmerknader

Tre punkter fortjener oppmerksomhet før LDAP-innlogging blir den eneste veien inn på appliancen:

- **Behold lokal nødtilgang.** Passordene til eksterne brukere ligger på LDAP-serveren. Dersom katalogen ikke er tilgjengelig (nettverksproblem, AD-vedlikehold eller fordi gatewayen nettopp skal løse et problem med det samme nettverket), er det fortsatt nødvendig med en lokal administratorkonto med et sikkert lagret passord. Standardbrukeren admin bør derfor ikke avskaffes, men vedlikeholdes som dokumentert nødtilgang.
- **MFA er fortsatt relevant.** 15.0.6 har også endret MFA-innloggingen: Den andre faktoren legges ikke lenger til passordet, men spørres etter i et eget felt. Ekstern autentisering erstatter ikke den andre faktoren.
- **Offboarding via katalogen.** Den egentlige gevinsten med integrasjonen: Når en administrator slutter i virksomheten, er det tilstrekkelig å deaktivere AD-kontoen eller fjerne brukeren fra den mappede gruppen. Det tidligere nødvendige vedlikeholdet av lokale kontoer på hver appliance faller bort. De lokalt synlige, eksternt autentiserte brukerobjektene bør likevel sammenlignes med katalogen med jevne mellomrom.

## Konklusjon

LDAP-autentiseringen for Admin-GUI-en lukker et hull som lenge har eksistert på appliancen: Administratoradganger kan nå styres sentralt i katalogen i stedet for per enhet. Sammen med det separate MFA-feltet gjør 15.0.6 dermed innloggingen til administrasjonsgrensesnittet betydelig mer moden i én og samme utgivelse. De som innfører funksjonen, bør holde gruppemappingen bevisst restriktiv og ikke ofre den lokale nødtilgangen.

## Kilder

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): Oppføring om Admin-GUI-autentisering med funksjonsbeskrivelse, konfigurasjonssted og oppførselen til eksternt autentiserte brukere.

2.  [SEPPmail-dokumentasjon – «User > Advanced Settings»](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): Referanse for feltene i seksjonen External Authentication (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [SEPPmail-dokumentasjon – «Groups»](https://docs.seppmail.com/ch/07_mi_16_groups.html): Forhåndsdefinerte grupper på appliancen; medlemmer av gruppen admin har ubegrenset administrativ tilgang.

4.  [SEPPmail-dokumentasjon – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Offisielle release notes for 15.0.6 med oppføringen om Admin-GUI-autentisering mot eksterne LDAP-servere.

5.  [SEPPmail 15.0.6 og 15.0.6.1: Sikkerhetsrettinger og nye administrasjonsfunksjoner](/blog/seppmail-releases-15-0-6-und-15-0-6-1): Oversikt over alle endringer i de to utgivelsene.
