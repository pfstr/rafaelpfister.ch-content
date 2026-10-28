---
title: "E-postsending via et relay: Kontroller TLS og autentisering"
navTitle: "Relay: Kontroller TLS"
description: "En one-pager for applikasjonsansvarlige der applikasjonen sender e-post via et relay: Hvilke tre innstillinger i applikasjonen som teller (port, TLS-modus, pålogging), hva alternativene heter i vanlige miljøer, og hvordan én enkelt test-e-post med Received-header dokumenterer at forbindelsen faktisk er kryptert og autentisert."
date: "2026-08-28"
kategorie: "SMTP og e-postflyt"
timeToRead: "5 min lesetid"
themen:
  - smtp-mailflow
protokolle:
  - "smtp"
  - "tls"
  - "troubleshooting"
slug: "e-postsending-via-et-rele-kontroller-tls-og-autentisering"
translationId: "article-734e79c4a87105e3"
translationOf: mail-relay-tls-authentisierung-pruefen
translationSourceHash: a1f56d0ba84377020bb56c927ee5d8789198061a0f941b463e3fd0e929262082
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T10:05:33.686Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/e-postsending-via-et-rele-kontroller-tls-og-autentisering
---

# E-postsending via et relay: Kontroller TLS og autentisering

Mange applikasjoner sender ikke e-post direkte til Internett, men leverer den til et internt relay: ERP-systemet sender ordrebekreftelser, overvåkingen sender alarmer, og saksbehandlingssystemet sender varsler. E-postteamet drifter relayet; på applikasjonssiden er applikasjonsansvarlig ansvarlig. Ved en revisjon eller en analyse av beskyttelsesbehov havner derfor spørsmålet hos vedkommende: Kobler applikasjonen seg kryptert til relayet, og logger den seg på på riktig måte?

Svaret finnes på to steder som ikke krever noe e-postverktøy eller tilgang til relayet: i SMTP-konfigurasjonen til applikasjonen og i headeren til én enkelt test-e-post. E-postteamet har ansvar for hva relayet selv tilbyr og hvordan det krypterer e-postene videre til mottakeren; på applikasjonssiden holder det å dokumentere sin egen delstrekning.

## Hvor innstillingene finnes

SMTP-konfigurasjonen finnes, avhengig av applikasjonen, på ett av tre steder: i administrasjonsgrensesnittet (som regel under «E-post», «Varsler», «SMTP» eller «Utgående server»), i en konfigurasjonsfil eller i miljøvariablene for utrullingen (typisk `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER` og varianter). Det er alltid de samme opplysningene som søkes: servernavn, port, et krypteringsalternativ og påloggingsopplysninger.

## De tre innstillingene som teller

**For det første: port og TLS-modus.** Begge må passe sammen, for bak valgene ligger det to ulike metoder: Med STARTTLS begynner forbindelsen i klartekst og bytter deretter til TLS, mens den med implisitt TLS (ofte kalt «SSL/TLS» eller «SSL» i skjemaene) er kryptert fra starten av.

| Port | TLS-innstilling i applikasjonen | Vurdering |
|---|---|---|
| 587 | STARTTLS | Ønsket tilstand for innlevering fra applikasjoner |
| 465 | SSL/TLS (implisitt) | også i orden |
| 25 | ingen eller STARTTLS | vanlig for relayer med IP-godkjenning; aktiver likevel TLS-innstillingen dersom relayet tilbyr STARTTLS |
| valgfri | «Ingen» / «None» | Funn: sendingen foregår i klartekst |
| valgfri | «TLS hvis tilgjengelig» / opportunistisk | Funn: faller ved problemer stilltiende tilbake til klartekst; bytt til tvungen TLS |

En feil kombinasjon (for eksempel «SSL/TLS» på port 587) fører til at forbindelsen brytes, ikke til umerket klartekst. De risikable innstillingene er de to siste radene i tabellen, fordi applikasjonen da sender ukryptert uten feilmelding.

**For det andre: sertifikatkontroll.** Mange applikasjoner tilbyr et alternativ som «Ikke kontroller sertifikat», «Allow insecure» eller `verify=false`, som gjerne settes i innføringsprosjekter fordi relayet bruker et internt sertifikat. Forbindelsen forblir riktignok kryptert, men applikasjonen godtar enhver motpart. Hvis alternativet er aktivert, skal det føres opp som et funn i rapporten; den riktige løsningen er tillit til den interne CA-en i stedet for å deaktivere kontrollen.

**For det tredje: pålogging.** Relayer kjenner to modeller: SMTP AUTH med brukernavn og passord eller IP-godkjenning uten konto. Hvilken variant som gjelder, står i e-postteamets relay-godkjenning. Ved SMTP AUTH skal tre punkter stå på sjekklisten: Påloggingen skjer med en dedikert tjenestekonto for applikasjonen (ikke med en personlig konto som deaktiveres ved neste fratreden), passordet er lagret som en hemmelighet i stedet for i klartekst i en konfigurasjonsfil, og TLS-alternativet er aktivt, fordi de vanlige metodene PLAIN og LOGIN ellers overfører påloggingsopplysningene i klartekst.

## Hva innstillingene heter i vanlige miljøer

| Miljø | Kryptering | Pålogging |
|---|---|---|
| Administrasjonsgrensesnitt (ERP, overvåking, appliances) | Nedtrekksliste «Kryptering»: None / STARTTLS / SSL-TLS | Felter for brukernavn/passord; tomme = ingen pålogging |
| Java (Jakarta Mail, Spring) | `mail.smtp.starttls.enable=true` pluss `mail.smtp.starttls.required=true`; for port 465 `mail.smtp.ssl.enable=true` | `mail.smtp.auth=true` |
| .NET | `SmtpClient.EnableSsl=true` (aktiverer STARTTLS); MailKit: `SecureSocketOptions.StartTls` | `Credentials` eller `Authenticate()` |
| PHP (PHPMailer) | `SMTPSecure='tls'` for 587, `'ssl'` for 465 | `SMTPAuth=true` |
| Python (smtplib) | `starttls()` etter at forbindelsen er opprettet, eller `SMTP_SSL` for 465 | `login()` |
| Node.js (Nodemailer) | Port 465: `secure:true`; port 587: `secure:false` pluss `requireTLS:true` | `auth: {user, pass}` |

To punkter i denne tabellen er erfaringsmessig de vanligste funnene: I Java aktiverer `starttls.enable` alene bare opportunistisk TLS; først `starttls.required` hindrer tilbakefall til klartekst. I Nodemailer betyr `secure:false` ikke «ukryptert», men «ingen implisitt TLS»; uten `requireTLS:true` forblir STARTTLS imidlertid også opportunistisk.

## Kontroll: én test-e-post og dens Received-header

Konfigurasjonen angir ønsket tilstand, men beviser ikke hva som skjer på nettet. Beviset står i Received-headeren som relayet legger inn ved mottak av hver e-post. Det holder å sende en test-e-post fra applikasjonen til egen postkasse; vis meldingsheaderen der (Outlook: Fil, Egenskaper, Internett-hoder; Gmail: Vis original), og les den nederste Received-linjen, siden headere vokser nedenfra og opp:

```text
Received: from app01.example.com (app01.example.com [10.1.2.3])
        by relay.example.com (Postfix) with ESMTPSA id 4XyZk12Fzq
        (version=TLSv1.3 cipher=TLS_AES_256_GCM_SHA384);
        Thu, 28 Aug 2026 09:15:04 +0200
```

Nøkkelordet etter `with` er kortversjonen av kontrollresultatet. Kodene er standardiserte (IANA-registeret «Mail Transmission Types»):

| Kode | Betydning | Vurdering |
|---|---|---|
| `SMTP` / `ESMTP` | ukryptert, uten pålogging | Tiltak nødvendig dersom TLS kreves |
| `ESMTPS` | TLS, uten pålogging | i orden ved IP-godkjenning |
| `ESMTPA` | pålogget, men uten TLS | kritisk: påloggingsopplysningene gikk i klartekst |
| `ESMTPSA` | TLS og pålogget | ønsket tilstand ved SMTP AUTH |

Postfix og Exchange legger til TLS-versjon og krypteringsalgoritme i parentes, slik at også foreldede protokollversjoner kan oppdages. For evaluering av lengre headere med flere stasjoner sparer [Mail-Header-Analyzer](https://rafaelpfister.ch/tools/header-analyzer) på dette nettstedet deg for manuelt arbeid; den kjører helt lokalt i nettleseren, og headeren forlater ikke maskinen din.

Hvis headeren forblir uklar eller en foranliggende lastbalanserer endrer forbindelsen, er det på tide å henvende seg til e-postteamet: Relayloggen registrerer for hver innlevering om TLS ble forhandlet frem og hvilken konto applikasjonen logget seg på med.

## Kort sjekkliste for kontrollrapporten

1. SMTP-konfigurasjonen til applikasjonen er funnet (grensesnitt, konfigurasjonsfil eller miljøvariabler) og dokumentert.
2. Port og TLS-modus passer sammen (587/STARTTLS eller 465/SSL-TLS); ingen innstilling «Ingen» eller «TLS hvis tilgjengelig».
3. Sertifikatkontroll er aktiv; en aktivert «Ikke kontroller sertifikat» er registrert som et funn.
4. Påloggingsmodellen er avklart: SMTP AUTH med tjenestekonto og lagring av hemmelighet, eller IP-godkjenning i henhold til relay-godkjenningen.
5. Received-headeren til test-e-posten viser `ESMTPSA` (med konto) eller `ESMTPS` (med IP-godkjenning); `ESMTPA` og `ESMTP` er funn.
6. Dersom kryptering helt frem til mottakeren kreves: formuler dette som et krav til e-postteamet, fordi strekningen etter relayet ligger utenfor applikasjonen.

## Endringslogg

- **25. oktober 2026:** Kildelenker oppdatert til IANAs nye adresser; Jakarta Mail-kilden viser nå til dokumentasjonen for Eclipse Angus Mail.

## Kilder

1.  [RFC 3207: SMTP Service Extension for Secure SMTP over Transport Layer Security](https://www.rfc-editor.org/rfc/rfc3207): definerer STARTTLS og overgangen fra klartekstforbindelsen til TLS.

2.  [RFC 4954: SMTP Service Extension for Authentication](https://www.rfc-editor.org/rfc/rfc4954): definerer SMTP AUTH og metoder som PLAIN og LOGIN.

3.  [RFC 8314: Cleartext Considered Obsolete](https://www.rfc-editor.org/rfc/rfc8314): anbefaler implisitt TLS på port 465 for innlevering fra klienter.

4.  [IANA: Mail Transmission Types](https://www.iana.org/assignments/smtp/smtp.xhtml#smtp-mail-transmission-types): register over `with`-kodene i Received-headeren (ESMTPS, ESMTPA, ESMTPSA).

5.  [Eclipse Angus Mail: Package org.eclipse.angus.mail.smtp](https://eclipse-ee4j.github.io/angus-mail/docs/api/org.eclipse.angus.mail/org/eclipse/angus/mail/smtp/package-summary.html): SMTP-leverandøren i Jakarta Mail-implementasjonen, dokumenterer egenskapene mail.smtp.starttls.enable, starttls.required og ssl.enable.
