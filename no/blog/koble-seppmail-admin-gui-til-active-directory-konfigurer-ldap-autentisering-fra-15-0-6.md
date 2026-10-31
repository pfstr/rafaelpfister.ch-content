---
title: "Koble SEPPmail Admin-GUI til Active Directory: Konfigurer LDAP-autentisering fra 15.0.6"
navTitle: "Admin-LDAP-pålogging"
description: "Siden fastvare 15.0.6 kan administratorer av SEPPmail-apparatet autentiseres mot en ekstern LDAP-server som Active Directory, inkludert gruppetilordning til den lokale admin-gruppen. Konfigurasjon under User > Advanced Settings, trinn for trinn."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 min lesetid"
themen:
  - seppmail
slug: "koble-seppmail-admin-gui-til-active-directory-konfigurer-ldap-autentisering-fra-15-0-6"
translationId: "article-21092a3dad6b84cb"
draft: false
translationOf: seppmail-admin-gui-ldap-authentifizierung
translationSourceHash: b9a1bdfe75f57458eb57c2e1dcb32fdfd25e1279eb720d4cd4754d49b5aac624
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:24:38.460Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/koble-seppmail-admin-gui-til-active-directory-konfigurer-ldap-autentisering-fra-15-0-6
---

# Koble SEPPmail Admin-GUI til Active Directory: Konfigurer LDAP-autentisering fra 15.0.6

Frem til fastvare 15.0.5 kjente administrasjonsgrensesnittet til SEPPmail Secure E-Mail Gateway bare lokale kontoer. De som ville arbeide ryddig, opprettet en egen lokal bruker for hver administrator og la vedkommende til i gruppen admin. Dette fungerer, men har de vanlige ulempene med lokale kontoer: egne passord per appliance, ingen sentral offboarding og ingen håndheving av passordretningslinjer fra katalogtjenesten. Med patch-utgivelsen 15.0.6 endres dette. Admin-GUI-en autentiserer administratorer mot en ekstern LDAP-server som Active Directory ved behov, og tilordner AD-grupper til lokale grupper på appliance-en.

De øvrige endringene i utgivelsen er oppsummert i artikkelen om [SEPPmail 15.0.6 og 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1). Her handler det kun om den nye eksterne autentiseringen.

## Hva funksjonen gjør

Ifølge Extended Release Notes legger 15.0.6 til en ny seksjon, **External Authentication**, under **User > Advanced Settings**. Dermed autentiserer Admin-GUI-en mot en ekstern LDAP-server, og eksterne grupper (for eksempel AD-sikkerhetsgrupper) tilordnes lokale grupper på appliance-en.

Eksternt autentiserte brukere vises lokalt på appliance-en og oppfører seg som lokale brukere, med én forskjell: Passordet deres kan ikke endres på appliance-en, fordi det ligger på den eksterne LDAP-serveren. Passordansvaret flyttes altså helt til katalogen.

Viktig avgrensning: Appliance-en hadde allerede ekstern autentisering, men bare for GINA-nettgrensesnittet, konfigurert per Managed Domain (seksjonen External authentication i domenekonfigurasjonen). Nytt i 15.0.6 er at også tilgangen til selve administrasjonsgrensesnittet går via LDAP.

Jeg tester fortsatt om HIN Mailgateway også har fått LDAP-pålogging, og oppdaterer artikkelen deretter. Siden HIN-appliance-ene er basert på samme SEPPmail-fastvare, går jeg ut fra det.

## Forutsetninger

Før konfigurasjonen bør tre ting være på plass:

- **Fastvare 15.0.7 eller nyere:** Funksjonen kom med 15.0.6. Siden 2. september 2026 har patch-utgivelsen 15.0.7 vært tilgjengelig. Den retter blant annet en omgåelse av MFA-kravet (SEC-100) og en Command Injection i konfigurasjonsbehandlingen (CVE-2026-84830). Appliance-en bør minst ha denne versjonen; funksjonsgrenen 15.1 (for tiden hotfix 15.1.0.1 fra 29. september 2026) inneholder også funksjonen.
- **En LDAP-kompatibel katalog:** Et Active Directory, OpenLDAP eller tilsvarende. Hvis brukerne bare ligger i Entra ID, som selv ikke snakker LDAP, bygger [Microsoft Entra Domain Services](/blog/microsoft-entra-domain-services-ldap-kerberos) broen.
- **En bind-konto i katalogen:** En dedikert, upriviligert tjenestekonto med lesetilgang som appliance-en bruker til LDAP-søket. Ingen domeneadministrator.
- **En AD-gruppe for gateway-administratorene:** For eksempel en sikkerhetsgruppe kalt SEPPmail-Admins, som senere tilordnes den lokale gruppen admin. Medlemskap i denne gruppen avgjør da full administrativ tilgang.

TLS er aktivert som standard i tilkoblingsinnstillingene og bør forbli aktivert; administratorenes innloggingsdata skal ikke sendes ukryptert over nettverket. Appliance-en må kunne nå LDAP-serveren på den konfigurerte porten (LDAPS vanligvis 636).

## Konfigurasjon under User > Advanced Settings

Konfigurasjonen finnes i Admin-GUI-en under **User > Advanced Settings**, i seksjonen **External Authentication**, og består av fire blokker.

**1. Connection Settings:** Avkrysningsboksen *Authenticate users to external LDAP server (e.g. Active Directory)* aktiverer funksjonen. Deretter følger serveradresse, port, valget *TLS required* samt Bind DN og Bind Password for tjenestekontoen.

**2. User Attributes:** Her defineres hvordan appliance-en finner brukerobjekter: LDAP Object Class (i Active Directory typisk person), Search Base (OU-en eller containeren der administratorkontoene ligger) og e-postattributtet (standard: mail).

**3. Group Attributes:** Tilsvarende informasjon for gruppeobjekter, slik at appliance-en kan løse opp gruppemedlemskapene.

**4. Mapping Settings:** Den avgjørende delen. Under *Remote Group* velges gruppen fra LDAP-serveren, og under *Local Group* én eller flere lokale grupper som den tilordnes. For full administrativ tilgang er dette gruppen admin; medlemmene har samme rettigheter som standardbrukeren admin. De som ønsker differensiering, tilordner i stedet begrensede grupper som readonly admin eller funksjonsrelaterte grupper på appliance-en.

Før lagring er det verdt å bruke den innebygde **Login Test**: Med brukernavn og passord for en testkonto kan du kontrollere at tilkobling, søk og autentisering fungerer før konfigurasjonen blir aktiv.

## Eksempelkonfigurasjoner

Følgende verdier må tilpasses ditt eget miljø (eksempeldomene example.com). Feltnavnene tilsvarer External Authentication-seksjonen på appliance-en.

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

Merknader om Active Directory: Enhver tilgjengelig Domain Controller egner seg som server; i miljøer med flere lokasjoner anbefales en DC på samme lokasjon eller et alias som peker til flere DC-er. Port 636 er LDAPS; sertifikatet til DC-en må derfor kunne valideres av appliance-en. Search Base bør avgrenses slik at den inneholder administratorkontoene, men ikke hele katalogen. Attributtet mail må være vedlikeholdt på AD-kontoene.

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

Merknader om OpenLDAP: I typiske oppsett ligger brukere som inetOrgPerson under ou=people. For grupper er groupOfNames det pålitelige valget, siden medlemskapet der avbildes via member-attributtet med full DN. posixGroup-grupper fører derimot bare medlemmene som memberUid (brukernavn i stedet for DN); om appliance-en kan løse dette opp, er ikke dokumentert og bør kontrolleres med Login Test før omleggingen. Hvis serveren kun kjører med STARTTLS på port 389, skal den aktuelle porten angis i serverfeltet; tilkoblingen skal under ingen omstendigheter kjøre ukryptert.

## Driftsmerknader

Tre punkter bør vies oppmerksomhet før LDAP-pålogging blir den eneste veien inn til appliance-en:

- **Behold lokal nødtilgang.** Passordene til eksterne brukere ligger på LDAP-serveren. Hvis katalogen ikke er tilgjengelig (nettverksproblem, AD-vedlikehold eller gateway-en nettopp skal løse et problem med det samme nettverket), trengs fortsatt en lokal administratorkonto med et sikkert lagret passord. Standardbrukeren admin bør derfor ikke avskaffes, men vedlikeholdes som en dokumentert nødtilgang.
- **MFA er fortsatt relevant.** 15.0.6 har også endret MFA-påloggingen: Den andre faktoren legges ikke lenger til passordet, men etterspørres i et eget felt. Ekstern autentisering erstatter ikke den andre faktoren. En sårbarhet som under bestemte forhold kunne gi tilgang før alle MFA-trinn var fullført, ble først rettet i 15.0.7.
- **Offboarding via katalogen.** Den egentlige gevinsten med integrasjonen: Når en administrator forlater virksomheten, er det nok å deaktivere AD-kontoen eller fjerne personen fra den tilordnede gruppen. Det tidligere nødvendige vedlikeholdet av lokale kontoer på hver appliance faller bort. De lokalt synlige, eksternt autentiserte brukerobjektene bør likevel sammenlignes med katalogen med jevne mellomrom.

## Konklusjon

LDAP-autentisering for Admin-GUI-en lukker et hull som lenge har eksistert på appliance-en: Administratoradganger kan nå styres sentralt i katalogen i stedet for per enhet. Sammen med det separate MFA-feltet forbedrer 15.0.6 dermed påloggingen til administrasjonsgrensesnittet betydelig i én og samme utgivelse. De som innfører funksjonen, bør holde gruppetilordningen bevisst restriktiv og beholde den lokale nødtilgangen.

## Endringslogg

- **25. oktober 2026:** Anbefalt minimumsversjon hevet fra 15.0.6.1 til 15.0.7, som retter en omgåelse av MFA-kravet (SEC-100) og CVE-2026-84830. Lagt til: Funksjonsgrenen 15.1 (for tiden 15.1.0.1) inneholder også funksjonen.

## Kilder

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): Oppføring om Admin-GUI-autentisering med funksjonsbeskrivelse, konfigurasjonssted og oppførselen til eksternt autentiserte brukere; sikkerhetsrettinger i 15.0.7 (SEC-100, CVE-2026-84830).

2.  [SEPPmail-dokumentasjon – «User > Advanced Settings»](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): Referanse for feltene i External Authentication-seksjonen (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [SEPPmail-dokumentasjon – «Groups»](https://docs.seppmail.com/ch/07_mi_16_groups.html): Forhåndsdefinerte grupper på appliance-en; medlemmer av gruppen admin har ubegrenset administrativ tilgang.

4.  [SEPPmail-dokumentasjon – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Offisielle Release Notes for 15.0.6 med oppføringen om Admin-GUI-autentisering mot eksterne LDAP-servere samt oppfølgingsutgivelsene 15.0.7, 15.1.0 og 15.1.0.1.

5.  [SEPPmail 15.0.6 og 15.0.6.1: Sikkerhetsrettinger og nye admin-funksjoner](/blog/seppmail-releases-15-0-6-und-15-0-6-1): Oversikt over alle endringene i de to utgivelsene.
