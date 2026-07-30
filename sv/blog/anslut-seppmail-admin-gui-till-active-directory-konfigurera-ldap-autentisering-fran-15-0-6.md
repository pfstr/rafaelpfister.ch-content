---
title: "Anslut SEPPmail Admin-GUI till Active Directory: konfigurera LDAP-autentisering från 15.0.6"
navTitle: "Admin-LDAP-inloggning"
description: "Sedan firmware 15.0.6 kan administratörer av SEPPmail-appliancen autentisera sig mot en extern LDAP-server som Active Directory, inklusive gruppmappning till den lokala admin-gruppen. Konfiguration under User > Advanced Settings, steg för steg."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 min lästid"
themen:
  - seppmail
slug: "anslut-seppmail-admin-gui-till-active-directory-konfigurera-ldap-autentisering-fran-15-0-6"
translationId: "article-21092a3dad6b84cb"
draft: false
translationOf: seppmail-admin-gui-ldap-authentifizierung
url: https://rafaelpfister.ch/sv/blog/anslut-seppmail-admin-gui-till-active-directory-konfigurera-ldap-autentisering-fran-15-0-6
translationSourceHash: bb8386d1f880934d4811eb317bcd51d47900fdd493dad90b1d7752bfc25ba55c
translationModel: gpt-5.6-terra
translatedAt: 2026-07-30T12:48:21.465Z
translationReview: automatic
---

# Anslut SEPPmail Admin-GUI till Active Directory: konfigurera LDAP-autentisering från 15.0.6

Fram till firmware 15.0.5 kände administrationsgränssnittet för SEPPmail Secure E-Mail Gateway endast till lokala konton. Den som ville arbeta korrekt skapade en egen lokal användare för varje administratör och lade till den i gruppen admin. Det fungerar, men har de vanliga nackdelarna med lokala konton: egna lösenord per appliance, ingen central offboarding och ingen tillämpning av lösenordspolicyer från katalogtjänsten. Med patchversion 15.0.6 ändras detta. Admin-GUI autentiserar vid behov administratörer mot en extern LDAP-server som Active Directory och mappar AD-grupper till lokala grupper på appliancen.

De övriga ändringarna i versionen sammanfattas i artikeln om [SEPPmail 15.0.6 och 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1). Här handlar det endast om den nya externa autentiseringen.

## Vad funktionen gör

Enligt Extended Release Notes lägger 15.0.6 till ett nytt avsnitt, **External Authentication**, under **User > Advanced Settings**. Därmed autentiserar Admin-GUI mot en extern LDAP-server, och externa grupper (till exempel AD-säkerhetsgrupper) mappas till lokala grupper på appliancen.

Externt autentiserade användare visas lokalt på appliancen och fungerar som lokala användare, med en skillnad: deras lösenord kan inte ändras på appliancen eftersom det finns på den externa LDAP-servern. Lösenordsansvaret flyttas alltså helt till katalogen.

Viktigt för avgränsningen: Appliancen hade redan tidigare extern autentisering, men endast för GINA-webbgränssnittet, konfigurerad per Managed Domain (avsnittet External authentication i domänkonfigurationen). Nytt i 15.0.6 är att även åtkomsten till själva administrationsgränssnittet sker via LDAP.

Jag testar fortfarande om HIN Mailgateway också har fått LDAP-inloggning och kompletterar artikeln därefter. Eftersom HIN-applianserna bygger på samma SEPPmail-firmware utgår jag från det.

## Förutsättningar

Före konfigurationen bör fyra saker finnas på plats:

- **Firmware 15.0.6.1:** Funktionen kommer med 15.0.6; på grund av versionens två RuleEngine-fel är hotfixen 15.0.6.1 direkt det rätta valet.
- **En LDAP-kompatibel katalog:** Active Directory, OpenLDAP eller motsvarande. Om användarna endast finns i Entra ID, som i sig inte talar LDAP, bygger [Microsoft Entra Domain Services](/blog/microsoft-entra-domain-services-ldap-kerberos) bron.
- **Ett bind-konto i katalogen:** Ett dedikerat, oprivilegierat tjänstkonto med läsåtkomst som appliancen använder för LDAP-sökningen. Ingen domänadministratör.
- **En AD-grupp för gateway-administratörer:** Till exempel en säkerhetsgrupp SEPPmail-Admins, som senare mappas till den lokala gruppen admin. Medlemskap i denna grupp avgör då den fullständiga administrativa åtkomsten.

TLS är som standard aktiverat i anslutningsinställningarna och bör förbli så; administratörernas inloggningsuppgifter ska inte skickas okrypterade över nätverket. Appliancen måste kunna nå LDAP-servern på den konfigurerade porten (LDAPS vanligtvis 636).

## Konfiguration under User > Advanced Settings

Konfigurationen finns i Admin-GUI under **User > Advanced Settings** i avsnittet **External Authentication** och består av fyra block.

**1. Connection Settings:** Kryssrutan *Authenticate users to external LDAP server (e.g. Active Directory)* aktiverar funktionen. Därefter följer serveradress, port, alternativet *TLS required* samt tjänstkontots Bind DN och Bind Password.

**2. User Attributes:** Här definieras hur appliancen hittar användarobjekt: LDAP Object Class (vanligtvis person i Active Directory), Search Base (den OU eller behållare där administratörskontona finns) och e-postattributet (standard: mail).

**3. Group Attributes:** Motsvarande uppgifter för gruppobjekt så att appliancen kan lösa gruppmedlemskapen.

**4. Mapping Settings:** Den avgörande delen. Under *Remote Group* väljs gruppen från LDAP-servern, och under *Local Group* en eller flera lokala grupper som den mappas till. För fullständig administrativ åtkomst är det gruppen admin; dess medlemmar är likställda med standardanvändaren admin. Den som vill differentiera mappar i stället till begränsade grupper som readonly admin eller till funktionsrelaterade grupper på appliancen.

Innan du sparar är det värt att använda det inbyggda **Login Test**: Med användarnamn och lösenord för ett testkonto går det att kontrollera om anslutning, sökning och autentisering fungerar innan konfigurationen aktiveras.

## Exempelkonfigurationer

Följande värden måste anpassas till den egna miljön (exempeldomän example.com). Fältnamnen motsvarar avsnittet External Authentication på appliancen.

### Active Directory

| Fält | Värde |
|---|---|
| Server | dc01.example.com |
| Port | 636 |
| TLS required | aktiverat |
| Bind DN | CN=svc-seppmail,OU=ServiceAccounts,DC=example,DC=com |
| Bind Password | Tjänstkontots lösenord |
| User: LDAP Object Class | person |
| User: Search Base | OU=IT,DC=example,DC=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | group |
| Group: Search Base | OU=Groups,DC=example,DC=com |
| Mapping: Remote Group | SEPPmail-Admins |
| Mapping: Local Group | admin |

Anmärkningar om Active Directory: Alla nåbara domänkontrollanter lämpar sig som server; i miljöer med flera platser rekommenderas en DC på samma plats eller ett alias som pekar på flera DC:er. Port 636 är LDAPS; för detta måste DC:ns certifikat kunna valideras av appliancen. Search Base bör begränsas så att den innehåller administratörskontona, men inte hela katalogen. Attributet mail måste vara ifyllt på AD-kontona.

### OpenLDAP

| Fält | Värde |
|---|---|
| Server | ldap01.example.com |
| Port | 636 |
| TLS required | aktiverat |
| Bind DN | cn=seppmail,ou=services,dc=example,dc=com |
| Bind Password | Tjänstkontots lösenord |
| User: LDAP Object Class | inetOrgPerson |
| User: Search Base | ou=people,dc=example,dc=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | groupOfNames |
| Group: Search Base | ou=groups,dc=example,dc=com |
| Mapping: Remote Group | seppmail-admins |
| Mapping: Local Group | admin |

Anmärkningar om OpenLDAP: I typiska installationer finns användare som inetOrgPerson under ou=people. För grupper är groupOfNames det tillförlitliga valet, eftersom medlemskapet där representeras via member-attributet med fullständigt DN. posixGroup-grupper anger däremot endast sina medlemmar som memberUid (användarnamn i stället för DN); om appliancen kan lösa detta är inte dokumenterat och bör kontrolleras med Login Test före övergången. Om servern endast kör STARTTLS på port 389 ska motsvarande port anges i serverfältet; anslutningen får under inga omständigheter vara okrypterad.

## Driftanvisningar

Tre punkter förtjänar uppmärksamhet innan LDAP-inloggning blir den enda vägen in i appliancen:

- **Behåll lokal nödåtkomst.** Externa användares lösenord finns på LDAP-servern. Om katalogen inte är tillgänglig (nätverksproblem, AD-underhåll eller om gatewayen just ska åtgärda ett problem i just detta nätverk) behövs fortfarande ett lokalt administratörskonto med säkert förvarat lösenord. Standardanvändaren admin bör därför inte avskaffas utan underhållas som dokumenterad nödåtkomst.
- **MFA är fortfarande relevant.** 15.0.6 har också omarbetat MFA-inloggningen: Den andra faktorn läggs inte längre till lösenordet utan efterfrågas i ett eget fält. Extern autentisering ersätter inte den andra faktorn.
- **Offboarding via katalogen.** Den egentliga vinsten med anslutningen: När en administratör lämnar företaget räcker det att inaktivera AD-kontot eller ta bort personen från den mappade gruppen. Det tidigare nödvändiga underhållet av lokala konton på varje appliance försvinner. De lokalt synliga, externt autentiserade användarobjekten bör ändå regelbundet jämföras med katalogen.

## Slutsats

LDAP-autentisering för Admin-GUI täpper till en lucka som länge funnits på appliancen: Administratörsåtkomst kan nu styras centralt i katalogen i stället för per enhet. Tillsammans med det separata MFA-fältet gör 15.0.6 därmed inloggningen till administrationsgränssnittet betydligt mognare i en och samma version. Den som inför funktionen bör hålla gruppmappningen medvetet restriktiv och inte offra den lokala nödåtkomsten.

## Källor

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): Post om Admin-GUI-autentisering med funktionsbeskrivning, konfigurationsplats och beteendet för externt autentiserade användare.

2.  [SEPPmail-dokumentation – «User > Advanced Settings»](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): Referens för fälten i avsnittet External Authentication (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [SEPPmail-dokumentation – «Groups»](https://docs.seppmail.com/ch/07_mi_16_groups.html): Fördefinierade grupper på appliancen; medlemmar i gruppen admin har obegränsad administrativ åtkomst.

4.  [SEPPmail-dokumentation – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Officiella release notes för 15.0.6 med posten om Admin-GUI-autentisering mot externa LDAP-servrar.

5.  [SEPPmail 15.0.6 och 15.0.6.1: säkerhetskorrigeringar och nya adminfunktioner](/blog/seppmail-releases-15-0-6-und-15-0-6-1): Översikt över alla ändringar i de båda versionerna.
