---
title: "Anslut SEPPmail Admin-GUI till Active Directory: konfigurera LDAP-autentisering från 15.0.6"
navTitle: "Admin-LDAP-inloggning"
description: "Sedan firmware 15.0.6 kan administratörer av SEPPmail-appliance autentiseras mot en extern LDAP-server som Active Directory, inklusive gruppmappning till den lokala admin-gruppen. Konfiguration under User > Advanced Settings, steg för steg."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 min lästid"
themen:
  - seppmail
slug: "anslut-seppmail-admin-gui-till-active-directory-konfigurera-ldap-autentisering-fran-15-0-6"
translationId: "article-21092a3dad6b84cb"
draft: false
translationOf: seppmail-admin-gui-ldap-authentifizierung
translationSourceHash: b9a1bdfe75f57458eb57c2e1dcb32fdfd25e1279eb720d4cd4754d49b5aac624
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:24:18.866Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/anslut-seppmail-admin-gui-till-active-directory-konfigurera-ldap-autentisering-fran-15-0-6
---

# Anslut SEPPmail Admin-GUI till Active Directory: konfigurera LDAP-autentisering från 15.0.6

Fram till firmware 15.0.5 stödde administrationsgränssnittet för SEPPmail Secure E-Mail Gateway endast lokala konton. Den som ville arbeta strukturerat skapade en egen lokal användare för varje administratör och lade till den i gruppen admin. Det fungerar, men har de vanliga nackdelarna med lokala konton: egna lösenord per appliance, ingen central offboarding och ingen tillämpning av lösenordsprinciperna från katalogtjänsten. Med patchversionen 15.0.6 ändras detta. Admin-GUI autentiserar vid behov administratörer mot en extern LDAP-server som Active Directory och mappar AD-grupper till lokala grupper på appliance-enheten.

De övriga ändringarna i versionen sammanfattas i artikeln om [SEPPmail 15.0.6 och 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1). Här handlar det endast om den nya externa autentiseringen.

## Vad funktionen gör

Enligt Extended Release Notes lägger 15.0.6 till ett nytt avsnitt, **External Authentication**, under **User > Advanced Settings**. Därmed autentiserar Admin-GUI mot en extern LDAP-server och externa grupper (till exempel AD-säkerhetsgrupper) mappas till lokala grupper på appliance-enheten.

Externt autentiserade användare visas lokalt på appliance-enheten och fungerar som lokala användare, med en skillnad: deras lösenord kan inte ändras på appliance-enheten eftersom det finns på den externa LDAP-servern. Lösenordshanteringen flyttas alltså helt till katalogtjänsten.

Viktigt att skilja på: Appliance-enheten hade redan tidigare extern autentisering, men endast för GINA-webbgränssnittet, konfigurerad per Managed Domain (avsnittet External authentication i domänkonfigurationen). Nytt i 15.0.6 är att även åtkomsten till själva administrationsgränssnittet sker via LDAP.

Jag testar fortfarande om HIN Mailgateway också har fått LDAP-inloggning och kompletterar artikeln därefter. Eftersom HIN-appliance-enheterna bygger på samma SEPPmail-firmware utgår jag från det.

## Förutsättningar

Före konfigurationen bör tre saker finnas på plats:

- **Firmware 15.0.7 eller senare:** Funktionen kom med 15.0.6. Sedan den 2 september 2026 finns patchversion 15.0.7 tillgänglig, som bland annat åtgärdar ett kringgående av MFA-kravet (SEC-100) och Command Injection i konfigurationshanteringen (CVE-2026-84830). Appliance-enheten bör minst ha denna version; funktionsgrenen 15.1 (för närvarande hotfix 15.1.0.1 från den 29 september 2026) innehåller också funktionen.
- **En LDAP-kompatibel katalogtjänst:** Active Directory, OpenLDAP eller liknande. Om användarna endast finns i Entra ID, som inte själv talar LDAP, bygger [Microsoft Entra Domain Services](/blog/microsoft-entra-domain-services-ldap-kerberos) bryggan.
- **Ett bind-konto i katalogtjänsten:** Ett dedikerat, icke-privilegierat tjänstekonto med läsåtkomst som appliance-enheten använder för LDAP-sökningen. Ingen domänadministratör.
- **En AD-grupp för gateway-administratörer:** Till exempel en säkerhetsgrupp med namnet SEPPmail-Admins, som senare mappas till den lokala gruppen admin. Medlemskap i denna grupp avgör då den fullständiga administrativa åtkomsten.

TLS är aktiverat som standard i anslutningsinställningarna och bör förbli så; administratörernas inloggningsuppgifter ska inte skickas okrypterade över nätet. Appliance-enheten måste kunna nå LDAP-servern på den konfigurerade porten (vanligtvis 636 för LDAPS).

## Konfiguration under User > Advanced Settings

Konfigurationen finns i Admin-GUI under **User > Advanced Settings**, i avsnittet **External Authentication**, och består av fyra block.

**1. Connection Settings:** Kryssrutan *Authenticate users to external LDAP server (e.g. Active Directory)* aktiverar funktionen. Därefter anges serveradress, port, alternativet *TLS required* samt tjänstekontots Bind DN och Bind Password.

**2. User Attributes:** Här definieras hur appliance-enheten hittar användarobjekt: LDAP Object Class (i Active Directory vanligtvis person), Search Base (den OU eller container där administratörskontona finns) och e-postattributet (standard: mail).

**3. Group Attributes:** På samma sätt anges gruppobjekten, så att appliance-enheten kan lösa upp gruppmedlemskapen.

**4. Mapping Settings:** Den avgörande delen. Under *Remote Group* väljs gruppen från LDAP-servern och under *Local Group* en eller flera lokala grupper som den mappas till. För fullständig administrativ åtkomst är det gruppen admin; dess medlemmar har samma behörighet som standardanvändaren admin. Den som vill differentiera mappar i stället till begränsade grupper som readonly admin eller till funktionsrelaterade grupper på appliance-enheten.

Innan du sparar är det värt att använda den inbyggda **Login Test**: Med användarnamn och lösenord för ett testkonto kan du kontrollera om anslutning, sökning och autentisering fungerar innan konfigurationen aktiveras.

## Exempelkonfigurationer

Följande värden ska anpassas till den egna miljön (exempeldomän example.com). Fältnamnen motsvarar avsnittet External Authentication på appliance-enheten.

### Active Directory

| Fält | Värde |
|---|---|
| Server | dc01.example.com |
| Port | 636 |
| TLS required | aktiverat |
| Bind DN | CN=svc-seppmail,OU=ServiceAccounts,DC=example,DC=com |
| Bind Password | Tjänstekontots lösenord |
| User: LDAP Object Class | person |
| User: Search Base | OU=IT,DC=example,DC=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | group |
| Group: Search Base | OU=Groups,DC=example,DC=com |
| Mapping: Remote Group | SEPPmail-Admins |
| Mapping: Local Group | admin |

Anmärkningar om Active Directory: Varje nåbar Domain Controller lämpar sig som server; i miljöer med flera platser rekommenderas en DC på samma plats eller ett alias som pekar på flera DC:er. Port 636 är LDAPS; DC:ns certifikat måste därför kunna valideras av appliance-enheten. Search Base bör avgränsas så att den innehåller administratörskontona, men inte hela katalogen. Attributet mail måste vara ifyllt för AD-kontona.

### OpenLDAP

| Fält | Värde |
|---|---|
| Server | ldap01.example.com |
| Port | 636 |
| TLS required | aktiverat |
| Bind DN | cn=seppmail,ou=services,dc=example,dc=com |
| Bind Password | Tjänstekontots lösenord |
| User: LDAP Object Class | inetOrgPerson |
| User: Search Base | ou=people,dc=example,dc=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | groupOfNames |
| Group: Search Base | ou=groups,dc=example,dc=com |
| Mapping: Remote Group | seppmail-admins |
| Mapping: Local Group | admin |

Anmärkningar om OpenLDAP: I typiska konfigurationer finns användare som inetOrgPerson under ou=people. För grupper är groupOfNames det tillförlitliga valet, eftersom medlemskap där representeras via attributet member med fullständigt DN. posixGroup-grupper anger däremot endast sina medlemmar som memberUid (användarnamn i stället för DN); om appliance-enheten kan lösa detta är inte dokumenterat och bör kontrolleras med Login Test före övergången. Om servern endast kör STARTTLS på port 389 ska motsvarande port anges i Server-fältet; anslutningen får under inga omständigheter köras okrypterat.

## Driftanvisningar

Tre punkter förtjänar uppmärksamhet innan LDAP-inloggningen blir den enda vägen in till appliance-enheten:

- **Behåll lokal nödåtkomst.** Externa användares lösenord finns på LDAP-servern. Om katalogtjänsten inte är tillgänglig (nätverksproblem, AD-underhåll eller om gatewayen just ska åtgärda ett problem i just detta nätverk) behövs fortfarande ett lokalt administratörskonto med ett säkert lagrat lösenord. Standardanvändaren admin bör alltså inte avskaffas utan underhållas som en dokumenterad nödåtkomst.
- **MFA är fortfarande relevant.** 15.0.6 har också reviderat MFA-inloggningen: den andra faktorn läggs inte längre till lösenordet utan efterfrågas i ett eget fält. Extern autentisering ersätter inte den andra faktorn. En sårbarhet som under vissa förutsättningar gjorde åtkomst möjlig innan alla MFA-steg var slutförda åtgärdas först i 15.0.7.
- **Offboarding via katalogtjänsten.** Den egentliga vinsten med anslutningen: När en administratör lämnar företaget räcker det att inaktivera AD-kontot eller ta bort personen från den mappade gruppen. Den tidigare nödvändiga uppdateringen av lokala konton på varje appliance-enhet försvinner. De lokalt synliga, externt autentiserade användarobjekten bör ändå regelbundet jämföras med katalogtjänsten.

## Slutsats

LDAP-autentisering för Admin-GUI täpper till en lucka som länge funnits i appliance-enheten: Administratörsåtkomst kan nu styras centralt i katalogtjänsten i stället för per enhet. Tillsammans med det separata MFA-fältet förbättrar 15.0.6 därmed inloggningen till administrationsgränssnittet avsevärt i en enda version. Den som inför funktionen bör hålla gruppmappningen medvetet restriktiv och behålla den lokala nödåtkomsten.

## Ändringshistorik

- **25 oktober 2026:** Rekommenderad lägstanivå höjd från 15.0.6.1 till 15.0.7, som åtgärdar ett kringgående av MFA-kravet (SEC-100) och CVE-2026-84830. Tillagt: Funktionsgrenen 15.1 (för närvarande 15.1.0.1) innehåller också funktionen.

## Källor

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): Post om Admin-GUI-autentisering med funktionsbeskrivning, konfigurationsplats och beteendet för externt autentiserade användare; säkerhetskorrigeringar i 15.0.7 (SEC-100, CVE-2026-84830).

2.  [SEPPmail-dokumentation – «User > Advanced Settings»](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): Referens för fälten i avsnittet External Authentication (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [SEPPmail-dokumentation – «Groups»](https://docs.seppmail.com/ch/07_mi_16_groups.html): Fördefinierade grupper på appliance-enheten; medlemmar i gruppen admin har obegränsad administrativ åtkomst.

4.  [SEPPmail-dokumentation – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Officiella release notes för 15.0.6 med posten om Admin-GUI-autentisering mot externa LDAP-servrar samt efterföljande versioner 15.0.7, 15.1.0 och 15.1.0.1.

5.  [SEPPmail 15.0.6 och 15.0.6.1: Säkerhetskorrigeringar och nya adminfunktioner](/blog/seppmail-releases-15-0-6-und-15-0-6-1): Översikt över alla ändringar i de båda versionerna.
