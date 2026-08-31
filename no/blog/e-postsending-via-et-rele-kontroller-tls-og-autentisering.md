---
title: "E-postsending via et relé: Kontroller TLS og autentisering"
navTitle: "Relé: Kontroller TLS"
description: "En énsides veiledning for Application Managers hvis applikasjon sender e-post via et relé: Hvilke tre innstillinger i applikasjonen som teller (port, TLS-modus, innlogging), hva alternativene heter i vanlige miljøer, og hvordan én enkelt test-e-post med Received-header dokumenterer at forbindelsen faktisk er kryptert og autentisert."
date: "2026-08-28"
kategorie: "SMTP og e-postflyt"
timeToRead: "5 min lesetid"
themen:
  - smtp-mailflow
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "tls"
  - "troubleshooting"
slug: "e-postsending-via-et-rele-kontroller-tls-og-autentisering"
translationId: "article-734e79c4a87105e3"
translationOf: mail-relay-tls-authentisierung-pruefen
url: https://rafaelpfister.ch/no/blog/e-postsending-via-et-rele-kontroller-tls-og-autentisering
translationSourceHash: 51d48e038c5eb870c77828f954ce1ad1d27bc4758889cb492c872eeaede04d9e
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:31:09.208Z
translationReview: automatic
---

# E-postsending via et relé: Kontroller TLS og autentisering

Mange applikasjoner sender ikke e-post direkte til internett, men leverer den til et internt relé: ERP-systemet sine ordrebekreftelser, overvåkingen sine alarmer, ticket-systemet sine varsler. E-postteamet drifter reléet; på applikasjonssiden er Application Manager ansvarlig. Ved en revisjon eller en analyse av beskyttelsesbehov havner derfor spørsmålet hos vedkommende: Kobler applikasjonen seg kryptert til reléet, og logger den seg inn på riktig måte?

Svaret står to steder, uten behov for e-postverktøy eller tilgang til reléet: i SMTP-konfigurasjonen til applikasjonen og i headeren til én enkelt test-e-post. E-postteamet er ansvarlig for hva reléet selv tilbyr og hvordan det krypterer e-postene videre til mottakeren; på applikasjonssiden er det nok å dokumentere sin egen del av overføringsveien.

## Hvor innstillingene finnes

SMTP-konfigurasjonen finnes, avhengig av applikasjonen, på ett av tre steder: i administrasjonsgrensesnittet (vanligvis under «E-post», «Varsler», «SMTP» eller «Utgående server»), i en konfigurasjonsfil eller i miljøvariablene for utrullingen (typisk `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER` og varianter). Det er alltid de samme opplysningene som skal finnes: servernavn, port, et krypteringsalternativ og påloggingsopplysninger.

## De tre innstillingene som teller

**For det første port og TLS-modus.** Begge må passe sammen, fordi valgene dekker to ulike metoder: Med STARTTLS begynner forbindelsen i klartekst og bytter deretter til TLS, mens den ved implisitt TLS (oftest kalt «SSL/TLS» eller «SSL» i skjemaene) er kryptert fra starten av.

| Port | TLS-innstilling i applikasjonen | Vurdering |
|---|---|---|
| 587 | STARTTLS | Ønsket tilstand for innlevering fra applikasjoner |
| 465 | SSL/TLS (implisitt) | også i orden |
| 25 | ingen eller STARTTLS | vanlig for reléer med IP-godkjenning; aktiver likevel TLS-innstillingen dersom reléet tilbyr STARTTLS |
| valgfri | «Ingen» / «None» | Funn: Sending skjer i klartekst |
| valgfri | «TLS hvis tilgjengelig» / opportunistisk | Funn: Faller stille tilbake til klartekst ved et problem; bytt til tvungen TLS |

En feil kombinasjon (for eksempel «SSL/TLS» på port 587) fører til brudd i forbindelsen, ikke til umerket klartekst. De risikable innstillingene er de to siste radene i tabellen, fordi applikasjonen der sender ukryptert uten feilmelding.

**For det andre sertifikatkontrollen.** Mange applikasjoner tilbyr et alternativ som «Ikke kontroller sertifikat», «Allow insecure» eller `verify=false`, som gjerne settes i innføringsprosjekter fordi reléet bruker et internt sertifikat. Forbindelsen forblir riktignok kryptert, men applikasjonen aksepterer enhver motpart. Hvis alternativet er satt, skal det føres som et funn i rapporten; den riktige løsningen er å stole på den interne CA-en i stedet for å slå av kontrollen.

**For det tredje innloggingen.** Reléer kjenner to modeller: SMTP AUTH med brukernavn og passord eller IP-godkjenning uten konto. Hvilken variant som gjelder, står i e-postteamets relégodkjenning. For SMTP AUTH hører tre punkter hjemme på sjekklisten: Innloggingen skjer via en dedikert tjenestekonto for applikasjonen (ikke via en personlig konto som deaktiveres når personen slutter), passordet er lagret som en hemmelighet i stedet for i klartekst i en konfigurasjonsfil, og TLS-alternativet er aktivt, fordi de vanlige metodene PLAIN og LOGIN ellers overfører påloggingsopplysningene i klartekst.

## Hva innstillingene heter i vanlige miljøer

| Miljø | Kryptering | Innlogging |
|---|---|---|
| Administrasjonsgrensesnitt (ERP, overvåking, appliances) | Nedtrekksliste «Kryptering»: None / STARTTLS / SSL-TLS | Felter for brukernavn/passord; tomme = ingen innlogging |
| Java (Jakarta Mail, Spring) | `mail.smtp.starttls.enable=true` pluss `mail.smtp.starttls.required=true`; for port 465 `mail.smtp.ssl.enable=true` | `mail.smtp.auth=true` |
| .NET | `SmtpClient.EnableSsl=true` (aktiverer STARTTLS); MailKit: `SecureSocketOptions.StartTls` | `Credentials` eller `Authenticate()` |
| PHP (PHPMailer) | `SMTPSecure='tls'` for 587, `'ssl'` for 465 | `SMTPAuth=true` |
| Python (smtplib) | `starttls()` etter at forbindelsen er opprettet, eller `SMTP_SSL` for 465 | `login()` |
| Node.js (Nodemailer) | Port 465: `secure:true`; port 587: `secure:false` pluss `requireTLS:true` | `auth: {user, pass}` |

To punkter i denne tabellen er erfaringsmessig de vanligste funnene: I Java aktiverer `starttls.enable` alene bare opportunistisk TLS; først `starttls.required` forhindrer tilbakefall til klartekst. I Nodemailer betyr `secure:false` ikke «ukryptert», men «ingen implisitt TLS»; uten `requireTLS:true` forblir STARTTLS imidlertid også opportunistisk.

## Kontroll: én test-e-post og dens Received-header

Konfigurasjonen angir ønsket tilstand, men beviser ikke hva som faktisk skjer på forbindelsen. Beviset står i Received-headeren som reléet legger til ved mottak av hver e-post. Det holder å sende en test-e-post fra applikasjonen til sin egen postkasse; vis meldingsheaderen der (Outlook: Fil, Egenskaper, Internett-hoder; Gmail: Vis original) og les den nederste Received-linjen, fordi headere vokser nedenfra og opp:

```text
Received: from app01.example.com (app01.example.com [10.1.2.3])
        by relay.example.com (Postfix) with ESMTPSA id 4XyZk12Fzq
        (version=TLSv1.3 cipher=TLS_AES_256_GCM_SHA384);
        Thu, 28 Aug 2026 09:15:04 +0200
```

Nøkkelordet etter `with` er den korte oppsummeringen av kontrollresultatet. Kodene er standardiserte (IANA-registeret «Mail Transmission Types»):

| Kode | Betydning | Vurdering |
|---|---|---|
| `SMTP` / `ESMTP` | ukryptert, uten innlogging | Krever tiltak hvis TLS er påkrevd |
| `ESMTPS` | TLS, uten innlogging | i orden ved IP-godkjenning |
| `ESMTPA` | innlogget, men uten TLS | kritisk: Påloggingsopplysningene gikk i klartekst |
| `ESMTPSA` | TLS og innlogget | Ønsket tilstand ved SMTP AUTH |

Postfix og Exchange legger til TLS-versjon og krypteringsalgoritme i parentes, slik at også foreldede protokollversjoner kan oppdages. For analyse av lengre headere med flere stasjoner kan [Mail-Header-Analyzer](https://rafaelpfister.ch/tools/header-analyzer) på dette nettstedet spare deg for manuelt arbeid; den kjører helt lokalt i nettleseren, og headeren forlater ikke maskinen din.

Hvis headeren forblir uklar, eller en foranstilt lastbalanserer endrer forbindelsesmerkingen, er det på tide å kontakte e-postteamet: Reléloggen dokumenterer for hver innlevering om TLS ble forhandlet frem og hvilken konto applikasjonen logget inn med.

## Kort sjekkliste for kontrollrapporten

1. SMTP-konfigurasjonen til applikasjonen er funnet (grensesnitt, konfigurasjonsfil eller miljøvariabler) og dokumentert.
2. Port og TLS-modus passer sammen (587/STARTTLS eller 465/SSL-TLS); ingen innstilling «Ingen» eller «TLS hvis tilgjengelig».
3. Sertifikatkontrollen er aktiv; et satt «Ikke kontroller sertifikat» er registrert som et funn.
4. Innloggingsmodellen er avklart: SMTP AUTH med tjenestekonto og lagring av hemmelighet, eller IP-godkjenning i henhold til relégodkjenningen.
5. Received-headeren i test-e-posten viser `ESMTPSA` (med konto) eller `ESMTPS` (med IP-godkjenning); `ESMTPA` og `ESMTP` er funn.
6. Hvis kryptering helt frem til mottakeren kreves: Formulert som et krav til e-postteamet, fordi strekningen etter reléet ligger utenfor applikasjonen.

## Kilder

1.  [RFC 3207: SMTP Service Extension for Secure SMTP over Transport Layer Security](https://www.rfc-editor.org/rfc/rfc3207): definerer STARTTLS og overgangen fra klartekstforbindelse til TLS.

2.  [RFC 4954: SMTP Service Extension for Authentication](https://www.rfc-editor.org/rfc/rfc4954): definerer SMTP AUTH og metodene PLAIN og LOGIN.

3.  [RFC 8314: Cleartext Considered Obsolete](https://www.rfc-editor.org/rfc/rfc8314): anbefaler implisitt TLS på port 465 for innlevering fra klienter.

4.  [IANA: Mail Transmission Types](https://www.iana.org/assignments/mail-parameters/mail-parameters.xhtml#mail-parameters-7): register over `with`-kodene i Received-headeren (ESMTPS, ESMTPA, ESMTPSA).

5.  [Jakarta Mail: Package com.sun.mail.smtp](https://jakarta.ee/specifications/mail/2.1/apidocs/jakarta.mail/com/sun/mail/smtp/package-summary): dokumenterer egenskapene mail.smtp.starttls.enable, starttls.required og ssl.enable.
