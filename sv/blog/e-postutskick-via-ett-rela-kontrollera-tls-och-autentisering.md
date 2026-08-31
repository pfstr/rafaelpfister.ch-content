---
title: "E-postutskick via ett relä: kontrollera TLS och autentisering"
navTitle: "Relä: kontrollera TLS"
description: "En one-pager för Application Managers vars applikation skickar e-post via ett relä: vilka tre inställningar i applikationen som är viktiga (port, TLS-läge, inloggning), vad alternativen heter i vanliga miljöer och hur ett enda testmeddelande via Received-headern visar att anslutningen verkligen är krypterad och autentiserad."
date: "2026-08-28"
kategorie: "SMTP och e-postflöde"
timeToRead: "5 min läsning"
themen:
  - smtp-mailflow
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "tls"
  - "troubleshooting"
slug: "e-postutskick-via-ett-rela-kontrollera-tls-och-autentisering"
translationId: "article-734e79c4a87105e3"
translationOf: mail-relay-tls-authentisierung-pruefen
url: https://rafaelpfister.ch/sv/blog/e-postutskick-via-ett-rela-kontrollera-tls-och-autentisering
translationSourceHash: 51d48e038c5eb870c77828f954ce1ad1d27bc4758889cb492c872eeaede04d9e
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:30:42.789Z
translationReview: automatic
---

# E-postutskick via ett relä: kontrollera TLS och autentisering

Många applikationer skickar inte e-post direkt till internet, utan levererar den till ett internt relä: ERP-systemet sina orderbekräftelser, övervakningen sina larm, ärendehanteringssystemet sina aviseringar. Reläet drivs av e-postteamet; på applikationssidan ansvarar Application Manager. Vid en revision eller en skyddsbehovsanalys hamnar därför frågan hos denne: Ansluter applikationen krypterat till reläet, och autentiserar den sig korrekt?

Svaret finns på två ställen som varken kräver något e-postverktyg eller åtkomst till reläet: i den egna applikationens SMTP-konfiguration och i headern för ett enda testmeddelande. Vad reläet självt erbjuder och hur det krypterar e-post vidare till mottagaren ansvarar e-postteamet för; på applikationssidan räcker det att belägga den egna delsträckan.

## Var inställningarna finns

SMTP-konfigurationen finns beroende på applikation på ett av tre ställen: i administrationsgränssnittet (oftast under ”E-post”, ”Aviseringar”, ”SMTP” eller ”Utgående server”), i en konfigurationsfil eller i distributionsmiljöns miljövariabler (vanligtvis `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER` och varianter). Det är alltid samma uppgifter som efterfrågas: servernamn, port, ett krypteringsalternativ och inloggningsuppgifter.

## De tre inställningar som räknas

**För det första port och TLS-läge.** Båda måste passa ihop, eftersom valen avser två olika metoder: Med STARTTLS börjar anslutningen i klartext och växlar sedan till TLS, medan implicit TLS (oftast kallat ”SSL/TLS” eller ”SSL” i gränssnitt) är krypterat från början.

| Port | TLS-inställning i applikationen | Bedömning |
|---|---|---|
| 587 | STARTTLS | Önskat läge för inlämning från applikationer |
| 465 | SSL/TLS (implicit) | också i ordning |
| 25 | ingen eller STARTTLS | vanligt för reläer med IP-godkännande; aktivera ändå TLS-inställningen om reläet erbjuder STARTTLS |
| valfri | ”Ingen” / ”None” | Resultat: e-post skickas i klartext |
| valfri | ”TLS när tillgängligt” / opportunistiskt | Resultat: återgår tyst till klartext vid problem; byt till tvingande TLS |

En felaktig kombination (exempelvis ”SSL/TLS” på port 587) leder till avbrutna anslutningar, inte till omärkt klartext. De riskabla inställningarna är de två sista raderna i tabellen, eftersom applikationen då skickar okrypterat utan felmeddelande.

**För det andra certifikatkontrollen.** Många applikationer erbjuder ett alternativ som ”Kontrollera inte certifikat”, ”Allow insecure” eller `verify=false`, vilket gärna aktiveras i införandeprojekt eftersom reläet använder ett internt certifikat. Anslutningen förblir då visserligen krypterad, men applikationen accepterar vilken motpart som helst. Om alternativet är aktiverat ska det tas upp som en anmärkning i rapporten; den korrekta lösningen är att lita på den interna CA:n i stället för att stänga av kontrollen.

**För det tredje inloggningen.** Reläer använder två modeller: SMTP AUTH med användarnamn och lösenord eller IP-godkännande utan konto. Vilken variant som gäller framgår av e-postteamets godkännande för reläet. För SMTP AUTH ska tre punkter finnas på checklistan: Inloggningen sker med applikationens dedikerade tjänstekonto (inte med ett personligt konto som inaktiveras vid nästa avgång), lösenordet lagras som en hemlighet i stället för i klartext i en konfigurationsfil, och TLS-alternativet är aktivt, eftersom de vanliga metoderna PLAIN och LOGIN annars överför inloggningsuppgifterna i klartext.

## Så heter inställningarna i vanliga miljöer

| Miljö | Kryptering | Inloggning |
|---|---|---|
| Admin-gränssnitt (ERP, övervakning, appliances) | Rullista ”Kryptering”: None / STARTTLS / SSL-TLS | Fält för användarnamn/lösenord; tomma = ingen inloggning |
| Java (Jakarta Mail, Spring) | `mail.smtp.starttls.enable=true` plus `mail.smtp.starttls.required=true`; för port 465 `mail.smtp.ssl.enable=true` | `mail.smtp.auth=true` |
| .NET | `SmtpClient.EnableSsl=true` (aktiverar STARTTLS); MailKit: `SecureSocketOptions.StartTls` | `Credentials` respektive `Authenticate()` |
| PHP (PHPMailer) | `SMTPSecure='tls'` för 587, `'ssl'` för 465 | `SMTPAuth=true` |
| Python (smtplib) | `starttls()` efter att anslutningen har upprättats eller `SMTP_SSL` för 465 | `login()` |
| Node.js (Nodemailer) | Port 465: `secure:true`; port 587: `secure:false` plus `requireTLS:true` | `auth: {user, pass}` |

Två punkter i denna tabell är erfarenhetsmässigt de vanligaste resultaten: I Java aktiverar `starttls.enable` ensamt endast opportunistisk TLS; först `starttls.required` förhindrar återfall till klartext. I Nodemailer betyder `secure:false` inte ”okrypterat”, utan ”ingen implicit TLS”; utan `requireTLS:true` förblir dock även STARTTLS opportunistiskt.

## Kontroll: ett testmeddelande och dess Received-header

Konfigurationen anger önskat läge, men bevisar inte vad som händer på nätet. Beviset finns i Received-headern som reläet lägger till när det tar emot varje e-postmeddelande. Det räcker att skicka ett testmeddelande från applikationen till den egna postlådan; visa meddelandeheadern där (Outlook: Arkiv, Egenskaper, Internetheaders; Gmail: Visa original) och läs den nedersta Received-raden, eftersom headers växer nedifrån och upp:

```text
Received: from app01.example.com (app01.example.com [10.1.2.3])
        by relay.example.com (Postfix) with ESMTPSA id 4XyZk12Fzq
        (version=TLSv1.3 cipher=TLS_AES_256_GCM_SHA384);
        Thu, 28 Aug 2026 09:15:04 +0200
```

Nyckelordet efter `with` är den korta sammanfattningen av kontrollresultatet. Beteckningarna är standardiserade (IANA-registret ”Mail Transmission Types”):

| Beteckning | Betydelse | Bedömning |
|---|---|---|
| `SMTP` / `ESMTP` | okrypterat, utan inloggning | Åtgärd krävs om TLS krävs |
| `ESMTPS` | TLS, utan inloggning | i ordning vid IP-godkännande |
| `ESMTPA` | inloggad, men utan TLS | kritiskt: inloggningsuppgifter överfördes i klartext |
| `ESMTPSA` | TLS och inloggad | Önskat läge vid SMTP AUTH |

Postfix och Exchange kompletterar med TLS-version och chiffer inom parentes, vilket även gör det möjligt att identifiera föråldrade protokollversioner. För analys av längre headers med flera stationer kan [Mail-Header-Analyzer](https://rafaelpfister.ch/tools/header-analyzer) på denna webbplats göra arbetet åt dig; den körs helt lokalt i webbläsaren och headern lämnar inte din dator.

Om headern förblir oklar eller en framförliggande lastbalanserare ändrar anslutningens stämpel är det dags att kontakta e-postteamet: Reläloggen registrerar för varje inlämning om TLS förhandlades fram och vilket konto applikationen autentiserade sig med.

## Kort checklista för granskningsrapporten

1. Applikationens SMTP-konfiguration har hittats (gränssnitt, konfigurationsfil eller miljövariabler) och dokumenterats.
2. Port och TLS-läge passar ihop (587/STARTTLS eller 465/SSL-TLS); ingen inställning ”Ingen” eller ”TLS när tillgängligt”.
3. Certifikatkontroll aktiv; ett aktiverat ”Kontrollera inte certifikat” har registrerats som en anmärkning.
4. Inloggningsmodell klarlagd: SMTP AUTH med tjänstekonto och lagring av hemlighet, eller IP-godkännande enligt relägodkännandet.
5. Testmeddelandets Received-header visar `ESMTPSA` (med konto) respektive `ESMTPS` (med IP-godkännande); `ESMTPA` och `ESMTP` är anmärkningar.
6. Om kryptering hela vägen till mottagaren krävs: hantera det som ett krav till e-postteamet, eftersom sträckan från reläet ligger utanför applikationen.

## Källor

1.  [RFC 3207: SMTP Service Extension for Secure SMTP over Transport Layer Security](https://www.rfc-editor.org/rfc/rfc3207): definierar STARTTLS och växlingen från klartextanslutning till TLS.

2.  [RFC 4954: SMTP Service Extension for Authentication](https://www.rfc-editor.org/rfc/rfc4954): definierar SMTP AUTH och metoder som PLAIN och LOGIN.

3.  [RFC 8314: Cleartext Considered Obsolete](https://www.rfc-editor.org/rfc/rfc8314): rekommenderar implicit TLS på port 465 för inlämning från klienter.

4.  [IANA: Mail Transmission Types](https://www.iana.org/assignments/mail-parameters/mail-parameters.xhtml#mail-parameters-7): register över `with`-beteckningarna i Received-headern (ESMTPS, ESMTPA, ESMTPSA).

5.  [Jakarta Mail: Package com.sun.mail.smtp](https://jakarta.ee/specifications/mail/2.1/apidocs/jakarta.mail/com/sun/mail/smtp/package-summary): dokumenterar egenskaperna mail.smtp.starttls.enable, starttls.required och ssl.enable.
