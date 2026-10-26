---
title: "HIN Mail-inloggning: webbmail, Outlook-token, glömt lösenord och HIN Mail Global"
navTitle: "HIN Mail-inloggning"
description: "Var och hur du loggar in på HIN Mail: webbmail på webmail.hin.ch, e-posttoken för Outlook och smartphone, inloggning utan HIN Client via SMS-kod eller Authenticator App, lösenordsåterställning och öppning av HIN Mail Global som mottagare utan HIN-konto."
date: "2026-10-05"
kategorie: "HIN-gateway"
timeToRead: "7 min. lästid"
themen:
  - hin-gateway
produkte:
  - "hin"
  - "outlook"
protokolle:
  - "troubleshooting"
  - "verschluesselung"
related:
  - hin-plattformerneuerung-2026
  - hin-mailgateway-backup-disaster-recovery
slug: "hin-mail-inloggning-webbmail-outlook-token-glomt-losenord-och-hin-mail-global"
translationId: "article-6f466c817c7c7c66"
translationOf: hin-mail-login
translationSourceHash: 00c1c2a195ebd80ccf8c6f7e7318d3b924eff80efe6315bd1d6fc95c05ad3c9f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:31:18.138Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/hin-mail-inloggning-webbmail-outlook-token-glomt-losenord-och-hin-mail-global
---

# HIN Mail-inloggning: webbmail, Outlook-token, glömt lösenord och HIN Mail Global

Bakom sökningen efter «HIN Mail Login» finns oftast tre olika situationer: Du är HIN-medlem och vill läsa din e-post i webbläsaren, du vill konfigurera HIN Mail i Outlook eller på smarttelefonen, eller så har du som patient, extern kontakt eller organisation fått ett krypterat HIN-mejl utan att ha något HIN-konto. Inloggningen fungerar olika i alla tre fallen.

| Situation | Adress | Användarnamn | Lösenord |
|---|---|---|---|
| Webbmail i webbläsaren | `webmail.hin.ch` | HIN ID | HIN-lösenord, skyddat med HIN Client, SMS-kod eller Authenticator App |
| Outlook, Thunderbird, Apple Mail, smarttelefon | `imap.mail.hin.ch`, `smtp.mail.hin.ch` | HIN ID | E-posttoken från `apps.hin.ch` |
| Administration, token, initialiseringslösenord | `apps.hin.ch` | HIN ID | HIN-lösenord plus andra faktor |
| Mottagare utan HIN-konto (HIN Mail Global) | Bilaga `secure-email.html` respektive länk i mejlet | Mobilnummer | SMS-kod |

## HIN Webmail: inloggning i webbläsaren

Du når webbmailen på `https://webmail.hin.ch`. Du loggar in med ditt HIN ID (HIN-inloggningsnamnet, t.ex. `hmuster`) och ditt HIN-lösenord. Den andra faktorn tillhandahålls av en av följande metoder:

- **HIN Client:** Om klienten körs på arbetsplatsen och är inloggad öppnas webbmailen utan ytterligare fråga. Med HIN Client 4.0 sker inloggningen från början av november 2026 direkt i webbläsaren: Du anger den HIN-skyddade adressen och skickas vidare till inloggningen.
- **SMS-kod (mTAN):** Efter lösenordet får du en kod till det registrerade mobilnumret. Metoden passar för enheter utan HIN Client, till exempel en privat laptop.
- **HIN Authenticator App:** Appen (App Store och Google Play, «HIN Authenticator») bekräftar inloggningen på smarttelefonen.

SMS-kod och Authenticator App måste aktiveras i förväg. Om båda saknas och HIN Client inte är tillgänglig går det inte att logga in på webbmailen. Konfigurera därför minst ett alternativ medan klienten fortfarande fungerar.

I webbmailen kan du skriva och hantera mejl, underhålla kontakter och adressböcker, söka i HIN-deltagarförteckningen och skapa flera signaturer. Under användarnamnet visar en stapel postlådans lediga lagringsutrymme; om den är röd måste du radera mejl eller utöka lagringsutrymmet.

## HIN Mail i Outlook och på smarttelefonen: inloggning med e-posttoken

E-postprogram loggar inte in med HIN-lösenordet utan med en **e-posttoken**. Detta är den vanligaste orsaken till att Outlook upprepade gånger frågar efter lösenordet: HIN-lösenordet avvisas vid IMAP-inloggning.

Så skapar du token:

1. Öppna `https://apps.hin.ch` och logga in med HIN ID (HIN Client, SMS-kod eller Authenticator App).
2. Välj området «HIN Mail».
3. Klicka på «Lägg till enhet» och välj «Mail Clients» som enhetstyp.
4. Kopiera den visade token och ange den som lösenord i e-postprogrammet.

Serverinställningarna:

| Inställning | Inkommande e-post (IMAP) | Utgående e-post (SMTP) |
|---|---|---|
| Server | `imap.mail.hin.ch` | `smtp.mail.hin.ch` |
| Port | `993` | `587` |
| Kryptering | SSL/TLS | STARTTLS |
| Användarnamn | HIN ID | HIN ID |
| Lösenord | E-posttoken | E-posttoken |
| Autentisering | normal | Aktivera «Servern för utgående e-post kräver autentisering» |

Som användarnamn gäller HIN ID, inte e-postadressen. Outlook föreslår e-postadressen vid automatisk konfiguration; därför måste konfigurationen göras manuellt via «Avancerade alternativ» respektive «Manuell konfiguration».

Token är knuten till din HIN-identitet och kan när som helst återkallas och skapas på nytt i `apps.hin.ch`. En egen token rekommenderas för varje enhet: Om en smarttelefon försvinner spärrar du endast den enheten, medan Outlook på mottagningens dator fortsätter att fungera. När token har konfigurerats behöver Outlook inte längre HIN Client för att hämta e-post.

## Första inloggningen utan HIN Client: aktivera identiteten

Nya HIN-identiteter kan aktiveras utan HIN Client. Du behöver användarnamnet och initialiseringslösenordet från HIN-handlingarna.

1. Öppna aktiveringslänken i Servicecenter (`servicecenter.hin.ch`).
2. Ange användarnamn och initialiseringslösenord.
3. Ange ett eget lösenord: minst 10 tecken med siffror, stora och små bokstäver samt specialtecken.
4. Konfigurera omedelbart en andra faktor (SMS-kod eller Authenticator App).
5. Logga därefter in på `apps.hin.ch`.

Du har 10 minuter på dig för aktiveringen. Därefter återställs initialiseringslösenordet av säkerhetsskäl och du måste begära ett nytt.

## Glömt HIN-lösenord

Återställningen beror på om en alternativ inloggningsmetod har konfigurerats.

**Med SMS-kod eller Authenticator App:**

1. Logga in på `apps.hin.ch` med den alternativa metoden.
2. Välj fliken «HIN Client» och skapa ett nytt initialiseringslösenord.
3. Välj «Registrering» i HIN Client och ange inloggningsnamnet samt det nya initialiseringslösenordet.
4. Ange ett nytt lösenord och bekräfta «Registrera ny HIN-identitet».

**Utan alternativ metod och utan inloggad enhet** återstår endast HIN Support per telefon (0848 830 740, måndag till fredag kl. 08:00–18:00). Anteckna det nya initialiseringslösenordet innan du stänger fönstret.

## HIN Mail Global: öppna krypterat mejl utan HIN-konto

Läkarmottagningar, sjukhus och laboratorier skickar meddelanden till patienter eller organisationer utan HIN-medlemskap via HIN Mail Global. Mejlet kommer till dig som ett vanligt e-postmeddelande, men innehållet är krypterat i bilagan `secure-email.html` respektive bakom länken «Läs krypterat meddelande». Någon HIN-inloggning i egentlig mening finns inte här; inloggningen sker via ditt mobilnummer.

**Första gången:**

1. Ladda ner och öppna bilagan `secure-email.html` (eller klicka på länken i mejlet). Vid webbmailtjänster som Gmail, Bluewin eller GMX ska du först spara och sedan öppna den.
2. Välj språk och ange ditt eget mobilnummer.
3. Ange koden du får via SMS. Meddelandet visas.

**Vid ytterligare meddelanden** räcker det att öppna bilagan; SMS-koden skickas automatiskt till det registrerade numret.

Att tänka på:

- Meddelandet kan hämtas via länken i **30 dagar**, därefter upphör åtkomsten. Spara eller skriv ut viktigt innehåll inom denna period.
- Vidarebefordrade meddelanden kan endast öppnas av den ursprungliga mottagaren.
- Du kan svara krypterat.
- En enhet kan markeras som betrodd; då utelämnas SMS-förfrågan i ett år.
- Om ingen kod kommer inom 15 minuter kan du begära en ny.
- De två aktuella versionerna av Chrome, Safari, Edge och Firefox stöds.

Vid problem med HIN Mail Global hjälper HIN Mail-supporten på +41 58 670 48 69. Frågor om meddelandets innehåll kan endast besvaras av avsändaren.

## Vanliga fel vid HIN Mail-inloggning

| Symptom | Orsak | Lösning |
|---|---|---|
| Outlook frågar ständigt efter lösenordet | HIN-lösenord angivet i stället för e-posttoken | Skapa token i `apps.hin.ch` och ange den som lösenord |
| Inloggning i Outlook misslyckas trots token | E-postadress som användarnamn | Använd HIN ID som användarnamn |
| Skickandet misslyckas, mottagning fungerar | SMTP-autentisering inte aktiverad, fel port eller fel kryptering | Använd port `587` med STARTTLS och aktivera autentisering |
| Webbmail kräver en kod som aldrig kommer | Mobilnumret är inaktuellt eller SMS-kod inte aktiverad | Använd Authenticator App eller logga in via HIN Client och uppdatera numret |
| Initialiseringslösenord avvisas | Fristen på 10 minuter har överskridits | Begär ett nytt initialiseringslösenord |
| Token fungerade, men plötsligt inte längre | Token återkallad i `apps.hin.ch` eller enheten borttagen | Skapa en ny token för denna enhet |
| HIN Mail Global: länken öppnar inte längre något | Fristen på 30 dagar har löpt ut | Be avsändaren att skicka på nytt |

## HIN Client 4.0: vad som ändras vid inloggningen

HIN distribuerar HIN Client 4.0 automatiskt fram till den 26 oktober 2026; den som inte får en automatisk uppdatering laddar ner den från `download.hin.ch`. Det befintliga lösenordet fortsätter att gälla; vid den första inloggningen med den nya versionen uppmanas du en gång att ändra det. Inloggning till HIN-skyddade tjänster som webbmail sker från början av november 2026 direkt i webbläsaren.

Version 4.0 kräver Windows 11 med TPM eller macOS 14 med Security Chip. Enheter som inte uppfyller dessa krav kan inte längre använda klienten. Åtkomst till webbmail är där fortfarande möjlig via SMS-kod eller Authenticator App, och e-posthämtning i Outlook via e-posttoken. Information om tidsfristerna för plattformsförnyelsen finns i artikeln [HIN-plattformsförnyelse 2026](/blog/hin-plattformerneuerung-2026).

## Mottagningar och sjukhus med egen e-postserver

Organisationer med egen HIN Mailgateway loggar inte in på HIN Mail. Postlådorna finns på den egna Exchange-servern eller i Exchange Online, och gatewayen (en SEPPmail-appliance) krypterar mejlen på väg till och från HIN-communityn. Inloggningsproblem gäller då det egna e-postsystemet, inte HIN-plattformen. Denna gateway ersätts av den nya HIN Gateway («Stargate»).

<aside class="offer-box">
  <span class="offer-box__tag">Kostnadsfri kontroll</span>
  <p><strong>Driver du en egen HIN Mailgateway?</strong> Jag granskar din miljö och berättar vad som behöver göras före migreringen till «Stargate».</p>
  <a class="offer-box__cta" href="/stargate">Registrera dig nu</a>
</aside>

## Ändringshistorik

- **25 oktober 2026:** Korrigerat: Vid den första inloggningen med HIN Client 4.0 måste lösenordet ändras en gång, och inloggning i webbläsaren gäller först från början av november 2026.

## Källor

1.  [HIN Support: HIN Mail och Mobile](https://support.hin.ch/de/thema/hin-mail-mobile/): Översikt med webbmailadress och instruktioner för Outlook, Apple Mail, Thunderbird, iOS och Android.

2.  [HIN Support: Hur fungerar webbmailen?](https://support.hin.ch/de/service/hin-mail-und-mobile/wie-funktioniert-das-webmail.cfm): Webbmailens funktioner, lagringsindikator.

3.  [HIN Support: Instruktion för Mail Token Service för Mail Clients](https://support.hin.ch/de/hin-mail-mobile/mail-token-service-mail-clients/): Skapa token i apps.hin.ch, IMAP- och SMTP-servrar, portar och kryptering.

4.  [HIN Support: Vad är en token?](https://support.hin.ch/de/service/hin-mail-und-mobile/was-ist-ein-token.cfm): E-posttoken och åtkomsttoken, koppling till HIN-identiteten.

5.  [HIN Support: Aktivera HIN-identitet (utan HIN Client)](https://support.hin.ch/de/hin-2fa/hin-identitaet-aktivieren/): Aktivering i Servicecenter, 10-minutersfrist, lösenordsregler, andra faktor.

6.  [HIN Support: Glömt lösenord](https://support.hin.ch/de/thema/hin-client/passwort-vergessen.cfm): Återställning via initialiseringslösenord, telefonsupport utan alternativ metod.

7.  [HIN Support: HIN Mail till icke-medlemmar](https://support.hin.ch/en/service/hin-mail-to-non-members.cfm): 30 dagars tillgänglighet, SMS-registrering, betrodda enheter, webbläsare som stöds, telefonsupport.

8.  [HIN: HIN Mail Global](https://www.hin.ch/services/hin-mail/hin-mail-global/): Skickande av krypterade mejl till mottagare utan HIN-medlemskap.

9.  [HIN Blogg: Den nya HIN Client är här](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): Webbläsarinloggning från början av november 2026, lösenordsändring vid första inloggningen, systemkrav med TPM respektive Security Chip, utrullning fram till den 26 oktober 2026.

10.  [HIN Authenticator i App Store](https://apps.apple.com/ch/app/hin-authenticator/id1535944002): App för den andra faktorn.
