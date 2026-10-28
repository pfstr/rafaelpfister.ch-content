---
title: "Skicka e-post via ett relä: Kontrollera TLS och autentisering"
navTitle: "Relä: Kontrollera TLS"
description: "En ensidig guide för applikationsansvariga vars applikation skickar e-post via ett relä: vilka tre inställningar i applikationen som räknas (port, TLS-läge, inloggning), vad alternativen kallas i vanliga miljöer och hur ett enda testmeddelande via Received-headern visar att anslutningen faktiskt är krypterad och autentiserad."
date: "2026-08-28"
kategorie: "SMTP och e-postflöde"
timeToRead: "5 min lästid"
themen:
  - smtp-mailflow
protokolle:
  - "smtp"
  - "tls"
  - "troubleshooting"
slug: "e-postutskick-via-ett-rela-kontrollera-tls-och-autentisering"
translationId: "article-734e79c4a87105e3"
translationOf: mail-relay-tls-authentisierung-pruefen
translationSourceHash: a1f56d0ba84377020bb56c927ee5d8789198061a0f941b463e3fd0e929262082
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T10:05:08.868Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/e-postutskick-via-ett-rela-kontrollera-tls-och-autentisering
---

# Skicka e-post via ett relä: Kontrollera TLS och autentisering

Många applikationer skickar inte själva e-post direkt till internet, utan lämnar den till ett internt relä: ERP-systemet sina orderbekräftelser, övervakningssystemet sina larm och ärendehanteringssystemet sina aviseringar. E-postteamet ansvarar för reläet; på applikationssidan ansvarar applikationsansvarig. Vid en revision eller en skyddsbehovsanalys hamnar därför frågan hos denna person: Ansluter applikationen krypterat till reläet, och autentiserar den sig korrekt?

Svaret finns på två ställen som varken kräver något e-postverktyg eller åtkomst till reläet: i den egna applikationens SMTP-konfiguration och i headern för ett enda testmeddelande. E-postteamet ansvarar för vad reläet självt erbjuder och hur det krypterar e-post vidare till mottagaren; på applikationssidan räcker det att styrka den egna delsträckan.

## Var inställningarna finns

SMTP-konfigurationen finns, beroende på applikation, på ett av tre ställen: i administrationsgränssnittet (oftast under ”E-post”, ”Aviseringar”, ”SMTP” eller ”Utgående server”), i en konfigurationsfil eller i distributionsmiljöns miljövariabler (vanligtvis `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER` och varianter). Det är alltid samma uppgifter som efterfrågas: servernamn, port, ett krypteringsalternativ och inloggningsuppgifter.

## De tre inställningarna som räknas

**Först port och TLS-läge.** Båda måste stämma överens, eftersom valen motsvarar två olika metoder: Med STARTTLS börjar anslutningen i klartext och växlar sedan till TLS, medan den med implicit TLS (oftast kallat ”SSL/TLS” eller ”SSL” i gränssnitt) är krypterad från början.

| Port | TLS-inställning i applikationen | Bedömning |
|---|---|---|
| 587 | STARTTLS | Önskat läge för inlämning från applikationer |
| 465 | SSL/TLS (implicit) | också korrekt |
| 25 | ingen eller STARTTLS | vanligt för reläer med IP-godkännande; aktivera ändå TLS-inställningen om reläet erbjuder STARTTLS |
| valfri | ”Ingen” / ”None” | Resultat: överföringen sker i klartext |
| valfri | ”TLS om tillgängligt” / opportunistiskt | Resultat: återgår tyst till klartext vid problem; byt till tvingande TLS |

En felaktig kombination (till exempel ”SSL/TLS” på port 587) leder till avbrutna anslutningar, inte till omärkt klartext. De riskfyllda inställningarna är de två sista raderna i tabellen, eftersom applikationen då skickar okrypterat utan felmeddelande.

**För det andra certifikatkontrollen.** Många applikationer erbjuder ett alternativ som ”Kontrollera inte certifikat”, ”Allow insecure” eller `verify=false`, vilket gärna aktiveras i införandeprojekt eftersom reläet har ett internt certifikat. Anslutningen förblir visserligen krypterad, men applikationen accepterar då vilken motpart som helst. Om alternativet är aktiverat ska det tas med som ett resultat i rapporten; den korrekta lösningen är att lita på den interna CA:n i stället för att stänga av kontrollen.

**För det tredje inloggningen.** Reläer har två modeller: SMTP AUTH med användarnamn och lösenord eller IP-godkännande utan konto. Vilken variant som gäller framgår av e-postteamets relägodkännande. Vid SMTP AUTH ska tre punkter finnas på checklistan: Inloggningen sker via applikationens dedikerade tjänstkonto (inte via ett personligt konto som inaktiveras när personen slutar), lösenordet lagras som en hemlighet i stället för i klartext i en konfigurationsfil och TLS-alternativet är aktivt, eftersom de vanliga metoderna PLAIN och LOGIN annars överför inloggningsuppgifterna i klartext.

## Vad inställningarna heter i vanliga miljöer

| Miljö | Kryptering | Inloggning |
|---|---|---|
| Administrationsgränssnitt (ERP, övervakning, apparater) | Rullista ”Kryptering”: None / STARTTLS / SSL-TLS | Fält för användarnamn/lösenord; tomma = ingen inloggning |
| Java (Jakarta Mail, Spring) | `mail.smtp.starttls.enable=true` plus `mail.smtp.starttls.required=true`; för port 465 `mail.smtp.ssl.enable=true` | `mail.smtp.auth=true` |
| .NET | `SmtpClient.EnableSsl=true` (aktiverar STARTTLS); MailKit: `SecureSocketOptions.StartTls` | `Credentials` respektive `Authenticate()` |
| PHP (PHPMailer) | `SMTPSecure='tls'` för 587, `'ssl'` för 465 | `SMTPAuth=true` |
| Python (smtplib) | `starttls()` efter att anslutningen har upprättats eller `SMTP_SSL` för 465 | `login()` |
| Node.js (Nodemailer) | Port 465: `secure:true`; port 587: `secure:false` plus `requireTLS:true` | `auth: {user, pass}` |

Två punkter i denna tabell är erfarenhetsmässigt de vanligaste resultaten: I Java aktiverar `starttls.enable` ensamt endast opportunistisk TLS; först `starttls.required` förhindrar återfall till klartext. I Nodemailer betyder `secure:false` inte ”okrypterat”, utan ”ingen implicit TLS”; utan `requireTLS:true` förblir STARTTLS dock också opportunistiskt.

## Kontroll: ett testmeddelande och dess Received-header

Konfigurationen anger det önskade läget, men bevisar inte vad som sker på ledningen. Beviset finns i Received-headern, som reläet lägger till när det tar emot varje e-postmeddelande. Det räcker med ett testmeddelande från applikationen till den egna inkorgen; visa meddelandeheadern där (Outlook: Arkiv, Egenskaper, Internetrubriker; Gmail: Visa original) och läs den nedersta Received-raden, eftersom headern växer nedifrån och upp:

```text
Received: from app01.example.com (app01.example.com [10.1.2.3])
        by relay.example.com (Postfix) with ESMTPSA id 4XyZk12Fzq
        (version=TLSv1.3 cipher=TLS_AES_256_GCM_SHA384);
        Thu, 28 Aug 2026 09:15:04 +0200
```

Nyckelordet efter `with` är den korta sammanfattningen av kontrollresultatet. Koderna är standardiserade (IANA-registret ”Mail Transmission Types”):

| Kod | Betydelse | Bedömning |
|---|---|---|
| `SMTP` / `ESMTP` | okrypterat, utan inloggning | Åtgärd krävs om TLS krävs |
| `ESMTPS` | TLS, utan inloggning | korrekt vid IP-godkännande |
| `ESMTPA` | inloggad, men utan TLS | kritiskt: inloggningsuppgifterna överfördes i klartext |
| `ESMTPSA` | TLS och inloggad | Önskat läge vid SMTP AUTH |

Postfix och Exchange lägger till TLS-version och chiffer inom parentes, vilket även gör det möjligt att identifiera föråldrade protokollversioner. För analys av längre headers med flera stationer kan [Mail-Header-Analyzer](https://rafaelpfister.ch/tools/header-analyzer) på denna webbplats göra arbetet åt dig; den körs helt lokalt i webbläsaren och headern lämnar inte din dator.

Om headern är oklar eller en framförliggande lastbalanserare ändrar anslutningsstämpeln är det dags att kontakta e-postteamet: Reläloggen registrerar för varje inlämning om TLS förhandlades fram och vilket konto applikationen autentiserade sig med.

## Kort checklista för granskningsrapporten

1. Applikationens SMTP-konfiguration hittad (gränssnitt, konfigurationsfil eller miljövariabler) och dokumenterad.
2. Port och TLS-läge stämmer överens (587/STARTTLS eller 465/SSL-TLS); ingen inställning ”Ingen” eller ”TLS om tillgängligt”.
3. Certifikatkontroll aktiv; ett aktiverat ”Kontrollera inte certifikat” har registrerats som resultat.
4. Inloggningsmodell klarlagd: SMTP AUTH med tjänstkonto och hemlighetslagring, eller IP-godkännande enligt relägodkännandet.
5. Received-headern för testmeddelandet visar `ESMTPSA` (med konto) respektive `ESMTPS` (med IP-godkännande); `ESMTPA` och `ESMTP` är resultat.
6. Om kryptering hela vägen till mottagaren krävs: adresserat som ett krav till e-postteamet, eftersom sträckan från reläet ligger utanför applikationen.

## Ändringshistorik

- **25 oktober 2026:** Källänkar uppdaterade till IANA:s nya adresser; Jakarta Mail-källan hänvisar nu till dokumentationen för Eclipse Angus Mail.

## Källor

1.  [RFC 3207: SMTP Service Extension for Secure SMTP over Transport Layer Security](https://www.rfc-editor.org/rfc/rfc3207): definierar STARTTLS och växlingen från klartextanslutning till TLS.

2.  [RFC 4954: SMTP Service Extension for Authentication](https://www.rfc-editor.org/rfc/rfc4954): definierar SMTP AUTH och metoder som PLAIN och LOGIN.

3.  [RFC 8314: Cleartext Considered Obsolete](https://www.rfc-editor.org/rfc/rfc8314): rekommenderar implicit TLS på port 465 för inlämning från klienter.

4.  [IANA: Mail Transmission Types](https://www.iana.org/assignments/smtp/smtp.xhtml#smtp-mail-transmission-types): register över `with`-koderna i Received-headern (ESMTPS, ESMTPA, ESMTPSA).

5.  [Eclipse Angus Mail: Package org.eclipse.angus.mail.smtp](https://eclipse-ee4j.github.io/angus-mail/docs/api/org.eclipse.angus.mail/org/eclipse/angus/mail/smtp/package-summary.html): SMTP-leverantör för Jakarta Mail-implementeringen, dokumenterar egenskaperna mail.smtp.starttls.enable, starttls.required och ssl.enable.
