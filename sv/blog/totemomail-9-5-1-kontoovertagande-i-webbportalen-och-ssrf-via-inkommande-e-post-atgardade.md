---
title: "totemomail 9.5.1: Kontoövertagande i webbportalen och SSRF via inkommande e-post åtgärdade"
navTitle: "totemomail 9.5.1"
description: "totemomail 9.5.1 åtgärdar åtta säkerhetsbrister och uppdaterar totemomail till den nivå som Kiteworks-advisories för Email Protection Gateway, publicerade i slutet av september, avser. Två brister kan utnyttjas utan inloggning: ett fel i webbportalens lösenordsåterställning som möjliggör övertagande av administratörskonton samt serverbaserade hämtningar av URL:er från inkommande meddelanden. Dessutom åtgärdas Path Traversal-brister i Admin API och Mail API, och Tomcat- och Batik-biblioteken har uppdaterats."
date: "2026-10-31"
kategorie: "Totemomail"
timeToRead: "5 min. lästid"
themen:
  - totemomail
  - sicherheitsluecken
produkte:
  - "totemomail"
  - "sicherheitsluecken"
protokolle:
  - "releases"
  - "haertung"
slug: "totemomail-9-5-1-kontoovertagande-i-webbportalen-och-ssrf-via-inkommande-e-post-atgardade"
translationId: "article-2bd15f63632ab728"
draft: false
translationOf: totemomail-9-5-1-sicherheitsupdate
url: https://rafaelpfister.ch/sv/blog/totemomail-9-5-1-kontoovertagande-i-webbportalen-och-ssrf-via-inkommande-e-post-atgardade
translationSourceHash: 5d441bdc7ec39d7fcef423e826bf14772f11bca2aa3dbdf4e1e3c83297131635
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T19:47:18.292Z
translationReview: automatic
---

# totemomail 9.5.1: Kontoövertagande i webbportalen och SSRF via inkommande e-post åtgärdade

totemo har släppt totemomail 9.5.1 (Build 101). Patchen innehåller åtta säkerhetskorrigeringar, en förbättring och tre felrättningar. Två av bristerna kan utnyttjas utan inloggning: ett fel i webbportalens lösenordsåterställning, som även kan användas för att ta över administratörskonton, samt serverbaserade hämtningar av URL:er i inkommande meddelanden. Båda berör komponenter som i de flesta installationer är åtkomliga från internet. Planera därför uppdateringen snarast.

## Direkt från 9.2.1 till 9.5.1

Release Notes listar 9.5.1 direkt efter 9.2.1; versionerna 9.3 och 9.4 förekommer inte i totemomails historik. En post i 9.5.1 hänvisar ändå till en «security hardening introduced in 9.3.1». Mellanversionerna kommer från Kiteworks Email Protection Gateway (EPG), som bygger på samma kodbas: Kiteworks säkerhetsmeddelanden anger EPG 9.3.1, 9.4.0, 9.4.1, 9.5.0 och 9.5.1 som åtgärdade versioner. För totemomail-installationer med 9.2.1 är 9.5.1 nästa uppdatering; Release Notes listar ingen ytterligare patch däremellan.

## Brister utan inloggning

**Server-Side Request Forgery via meddelandeinnehållet.** totemomail hämtade URL:er från behandlade meddelanden på serversidan utan att begränsa målet tillräckligt. En avsändare utan inloggning kunde utforma ett meddelande så att gatewayen skickade förfrågningar till interna tjänster eller metadataändpunkter för molninstanser och returnerade svaren. Det räcker med ett e-postmeddelande som gatewayen behandlar; med en gateway i det inkommande e-postflödet kan vilken extern avsändare som helst göra detta. Detta är särskilt känsligt i installationer i Azure, AWS eller andra moln, där metadataändpunkten `169.254.169.254` ger konfigurationsdata och, beroende på plattform, instansens åtkomsttoken. Release Notes anger inget CVE-nummer för denna brist. Kiteworks listar fem EPG-advisories om åtkomst till interna nätverksresurser (CVE-2026-102095, CVE-2026-102102 till CVE-2026-102105, samtliga CVSS 9.1, åtgärdade i EPG 9.5.0); de kortfattade advisory-texterna visar inte vilket av dem som motsvarar totemomail-posten.

**Lösenordsåterställning i webbportalen (CVE-2026-85066).** Ett fel i processen för återställning av lösenord gjorde det möjligt att hoppa över verifieringsstegen och ange ett nytt lösenord för ett befintligt konto. Release Notes omfattar uttryckligen administratörskonton. Kiteworks bedömer bristen i det tillhörande advisoryt GHSA-rwpq-5xfv-54pv till CVSS 9.8 (kritisk) och anger EPG 9.5.0 som åtgärdad version. Webbportalen är avsedd för externa mottagare av krypterade meddelanden och är därför vanligtvis offentligt åtkomlig.

## Brister med administratörsrättigheter

Ytterligare tre korrigeringar kräver inloggning med administratörsrättigheter. I kombination med kontoövertagandet i webbportalen får dessa krav dock mindre betydelse.

- **Admin API, köhantering (CVE-2026-34843):** Genom otillräckligt skyddade filuppladdningar kunde filer skrivas till valfria platser i filsystemet. Enligt Release Notes möjliggjorde detta Remote Code Execution. Advisoryt GHSA-mhc9-4pph-2r9x bedömer bristen till CVSS 7.2; i EPG är den åtgärdad sedan 9.4.0.
- **Rule Designer:** Vid import av en regelkonfiguration kunde bearbetningspipelinen ladda och köra kod utanför de godkända Mail Processing-komponenterna. Rättigheter för att konfigurera e-postregler krävs.
- **Mail API:** Vid uppackning av uppladdade ZIP-filer fanns en Path Traversal-brist. Filer packas nu endast upp i den temporära katalogen.

Den fjärde korrigeringen berör egenskapen `security.login.requestingKeysAvailable`. Tidigare dolde den i webbportalen endast länken för att begära offentliga nycklar; själva funktionen var fortfarande åtkomlig via en direkt förfrågan. Från och med 9.5.1 inaktiverar egenskapen funktionen. Om du har angett egenskapen för att förhindra nyckelförfrågningar fungerar den först med denna uppdatering som förväntat.

## Vad Kiteworks-registret dessutom listar

Kiteworks publicerade samlat 125 säkerhetsmeddelanden i sitt advisory-register på GitHub den 30 september 2026, varav 28 för Email Protection Gateway. totemomails Release Notes för 9.5.1 nämner betydligt färre punkter. Registret skiljer inte mellan EPG och totemomail; det går därför inte att avgöra om varje EPG-brist även berör totemomail. De kritiska EPG-posterna är:

| CVE | CVSS | Påverkan enligt advisory | Åtgärdad från EPG |
|---|---|---|---|
| CVE-2026-54154 | 10.0 | Kodexekvering med root-rättigheter utan inloggning | 9.4.1 |
| CVE-2026-85066 | 9.8 | Övertagande av användarkonton (lösenordsåterställning) | 9.5.0 |
| CVE-2026-85065 | 9.8 | Övertagande av användarkonton | 9.5.0 |
| CVE-2026-102149 | 9.4 | Övertagande av användarkonton | 9.5.1 |
| CVE-2026-102106 | 9.1 | Ändring av administrationskonfigurationen | 9.5.0 |
| CVE-2026-102095, CVE-2026-102102 till 102105 | 9.1 | Åtkomst till interna nätverksresurser | 9.5.0 |

Därtill kommer 13 poster klassade som höga, de flesta av dem kodexekvering med administratörsrättigheter (CVSS 7.2). Den högsta angivna åtgärdade versionen är 9.5.1; Kiteworks beskrev 9.5.1 som den aktuella versionen i slutet av september, där alla kända sårbarheter hade beaktats. Den som uppdaterar totemomail till 9.5.1 har därmed den version som alla EPG-advisories i registret avser.

## Uppdaterade bibliotek

De medföljande biblioteken från Apache Tomcat och Apache XML Graphics Batik har uppdaterats. Release Notes nämner CVE-2026-50229, CVE-2026-53404, CVE-2026-53434, CVE-2026-55276, CVE-2026-55955, CVE-2026-55956 och CVE-2026-59083, utan att tilldela dem separat till de två biblioteken. Dessutom uppdaterades woodstox-core-asl på grund av en sårbarhet; inget CVE-nummer anges för detta.

## Förbättring och felrättningar

- **EPG-migrering:** Vid migrering från en Oracle-databas stöder totemomail nu Oracle Wallet. Om inget databaslösenord är konfigurerat ansluter migreringen via Wallet.
- **RMS:** Meddelanden kunde fastna under bearbetningen eftersom RMS-åtgärder väntade obegränsat på autentisering. Väntetiden är nu begränsad till en minut. Detta berör installationer med Microsoft Purview Information Protection (RMS).
- **TRE:** Funktionen «Open Online» för Registered Envelopes fungerar igen. Den hade slutat fungera sedan den nämnda härdningen i 9.3.1.
- **Licensrapport:** För molnlicenser räknar rapporten åter interna användare med fördefinierade funktionstyper korrekt. Artikeln om [licensgränsen och rensning via LDAP](/blog/totemomail-licensed-user-limit-ldap-cleanup) beskriver hur övergivna konton påverkar licensräkningen.

## Installation

Patchen kan installeras på system från totemomail 7 b985 under Windows, Linux, Solaris och FreeBSD. Logga in på administrationskonsolen, ladda upp den medföljande TPF-filen under `[Instanzname]` > «Update» och starta uppdateringen. Vid flera instanser måste detta göras separat för varje instans. totemomail startar sedan om automatiskt om möjligt; planera in ett underhållsfönster och skapa en säkerhetskopia eller snapshot i förväg.

Tills uppdateringen har installerats minskar två åtgärder risken: gör administrationskonsolen och Admin API åtkomliga endast från administrationsnätverket och begränsa gatewayens utgående anslutningar till de mål som faktiskt behövs. Vid molninstallationer är metadataändpunkten `169.254.169.254` ett av de mål som gatewayen inte behöver kunna nå. Båda åtgärderna ersätter inte patchen; bristen i webbportalens lösenordsåterställning åtgärdas endast genom uppdateringen.

## Källor

1.  totemomail Release Notes 9.5.1 (Build 101), totemo ag: installationsanvisning och Release Notes som PDF, tillgängliga för kunder via tillverkarens support; grund för alla uppgifter i denna artikel.

2.  [Kiteworks Security Advisories på GitHub](https://github.com/kiteworks/security-advisories/security/advisories): Officiellt register med CVSS-värden och åtgärdade versioner för Email Protection Gateway, Core, MFT och Advanced Forms.

3.  [GHSA-rwpq-5xfv-54pv: Kontoövertagande (CVE-2026-85066)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-rwpq-5xfv-54pv): Advisory om felet i lösenordsåterställningen, CVSS 9.8, åtgärdat i EPG 9.5.0.

4.  [GHSA-5xhq-9wq3-rvj6: Godtycklig kodexekvering (CVE-2026-54154)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5xhq-9wq3-rvj6): Den mest kritiska EPG-bristen med CVSS 10.0, åtgärdad i EPG 9.4.1.

5.  [P.K. Sharma: Kiteworks publicerade 125 advisories](https://www.pk-sharma.com/briefing/kiteworks-125-advisories-which-is-the-shutdown-flaw): Analys av publiceringen den 30 september 2026 med antal per produkt och allvarlighetsgrad samt Kiteworks uttalanden om 9.5.1.

6.  [Förstå e-postrouting mellan totemomail och Exchange Online](/blog/totemomail-m365): Hur totemomail behandlar inkommande meddelanden; relevant för frågan om vilka e-postmeddelanden som kan utlösa SSRF-bristen.

7.  [Gränsen för Totemomail-licenser nådd: rensa övergivna användare via LDAP](/blog/totemomail-licensed-user-limit-ldap-cleanup): Bakgrund till licensräkningen som licensrapportkorrigeringen berör.
