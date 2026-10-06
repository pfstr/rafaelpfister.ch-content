---
title: "HIN Mail-innlogging: Webmail, Outlook-token, glemt passord og HIN Mail Global"
navTitle: "HIN Mail-innlogging"
description: "Hvor og hvordan du logger inn på HIN Mail: Webmail på webmail.hin.ch, e-posttoken for Outlook og smarttelefon, innlogging uten HIN Client med SMS-kode eller Authenticator App, tilbakestilling av passord og åpning av HIN Mail Global som mottaker uten HIN-konto."
date: "2026-10-05"
kategorie: "HIN Gateway"
timeToRead: "7 min lesetid"
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
slug: "hin-mail-innlogging-webmail-outlook-token-glemt-passord-og-hin-mail-global"
translationId: "article-6f466c817c7c7c66"
translationOf: hin-mail-login
url: https://rafaelpfister.ch/no/blog/hin-mail-innlogging-webmail-outlook-token-glemt-passord-og-hin-mail-global
translationSourceHash: 8f863faa139fd22182bf59e849447e241e339e4fcc4e6a073e6c1a89344f8a06
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T10:41:01.821Z
translationReview: automatic
---

# HIN Mail-innlogging: Webmail, Outlook-token, glemt passord og HIN Mail Global

Bak søket etter «HIN Mail-innlogging» ligger det som regel tre ulike situasjoner: Du er HIN-medlem og vil lese e-postene dine i nettleseren, du vil konfigurere HIN Mail i Outlook eller på smarttelefonen, eller du har som pasient eller ekstern instans mottatt en kryptert HIN-e-post og har ingen HIN-konto. Innloggingen foregår forskjellig i alle de tre tilfellene.

| Situasjon | Adresse | Brukernavn | Passord |
|---|---|---|---|
| Webmail i nettleseren | `webmail.hin.ch` | HIN ID | HIN-passord, sikret med HIN Client, SMS-kode eller Authenticator App |
| Outlook, Thunderbird, Apple Mail, smarttelefon | `imap.mail.hin.ch`, `smtp.mail.hin.ch` | HIN ID | E-posttoken fra `apps.hin.ch` |
| Administrasjon, token, initialiseringspassord | `apps.hin.ch` | HIN ID | HIN-passord pluss en andre faktor |
| Mottaker uten HIN-konto (HIN Mail Global) | Vedlegg `secure-email.html` eller lenke i e-posten | Mobilnummer | SMS-kode |

## HIN Webmail: innlogging i nettleseren

Du får tilgang til Webmail på `https://webmail.hin.ch`. Du logger inn med HIN ID-en din (HIN-brukernavnet, f.eks. `hmuster`) og HIN-passordet ditt. Den andre faktoren leveres av en av følgende metoder:

- **HIN Client:** Hvis klienten kjører på arbeidsstasjonen og er innlogget, åpnes Webmail uten ytterligere forespørsel. Med HIN Client 4.0 foregår innloggingen direkte i nettleseren: Du angir den HIN-beskyttede adressen og blir videresendt til innloggingen.
- **SMS-kode (mTAN):** Etter passordet mottar du en kode på det registrerte mobilnummeret. Denne metoden passer for enheter uten HIN Client, for eksempel en privat bærbar PC.
- **HIN Authenticator App:** Appen (App Store og Google Play, «HIN Authenticator») bekrefter innloggingen på smarttelefonen.

SMS-kode og Authenticator App må aktiveres på forhånd. Hvis begge mangler og HIN Client ikke er tilgjengelig, er det ikke mulig å logge inn på Webmail. Konfigurer derfor minst ett alternativ så lenge klienten fortsatt fungerer.

I Webmail kan du skrive og administrere e-poster, vedlikeholde kontakter og adressebøker, søke i HIN-deltakerkatalogen og opprette flere signaturer. Under brukernavnet viser en stolpe ledig lagringsplass i postkassen; er den rød, må du slette e-poster eller utvide lagringsplassen.

## HIN Mail i Outlook og på smarttelefonen: innlogging med e-posttoken

E-postprogrammer logger ikke inn med HIN-passordet, men med et **e-posttoken**. Dette er den vanligste årsaken til gjentatte passordforespørsler i Outlook: HIN-passordet avvises ved IMAP-innlogging.

Slik oppretter du tokenet:

1. Åpne `https://apps.hin.ch` og logg inn med HIN ID-en (HIN Client, SMS-kode eller Authenticator App).
2. Velg området «HIN Mail».
3. Klikk på «Legg til enhet» og velg «Mail Clients» som enhetstype.
4. Kopier tokenet som vises, og angi det som passord i e-postprogrammet.

Serverinnstillingene:

| Innstilling | Inngående e-post (IMAP) | Utgående e-post (SMTP) |
|---|---|---|
| Server | `imap.mail.hin.ch` | `smtp.mail.hin.ch` |
| Port | `993` | `587` |
| Kryptering | SSL/TLS | STARTTLS |
| Brukernavn | HIN ID | HIN ID |
| Passord | E-posttoken | E-posttoken |
| Autentisering | normal | Aktiver «Utgående e-postserver krever autentisering» |

HIN ID-en, ikke e-postadressen, skal brukes som brukernavn. Outlook foreslår e-postadressen ved automatisk konfigurering. Konfigurasjonen må derfor gjøres manuelt via «Avanserte alternativer» eller «Manuell konfigurering».

Tokenet er knyttet til HIN-identiteten din og kan når som helst tilbakekalles og opprettes på nytt i `apps.hin.ch`. Det anbefales å bruke et eget token for hver enhet: Hvis en smarttelefon blir borte, sperrer du bare denne enheten, mens Outlook på praksis-PC-en fortsetter å fungere. Når tokenet er konfigurert, trenger ikke Outlook HIN Client for å hente e-post.

## Første innlogging uten HIN Client: aktiver identiteten

Nye HIN-identiteter kan aktiveres uten HIN Client. Du trenger brukernavnet og initialiseringspassordet fra HIN-dokumentasjonen.

1. Åpne aktiveringslenken i Servicecenter (`servicecenter.hin.ch`).
2. Angi brukernavn og initialiseringspassord.
3. Opprett et eget passord: minst 10 tegn med tall, store og små bokstaver samt spesialtegn.
4. Konfigurer umiddelbart en andre faktor (SMS-kode eller Authenticator App).
5. Logg deretter inn på `apps.hin.ch`.

Du har 10 minutter til aktiveringen. Etter dette blir initialiseringspassordet tilbakestilt av sikkerhetsgrunner, og du må be om et nytt.

## Glemt HIN-passord

Tilbakestillingen avhenger av om en alternativ innloggingsmetode er konfigurert.

**Med SMS-kode eller Authenticator App:**

1. Logg inn på `apps.hin.ch` med den alternative metoden.
2. Velg fanen «HIN Client» og opprett et nytt initialiseringspassord.
3. Velg «Registrering» i HIN Client, og angi innloggingsnavnet og det nye initialiseringspassordet.
4. Opprett et nytt passord og bekreft «Registrer ny HIN-identitet».

**Uten alternativ metode og uten en innlogget enhet** gjenstår kun HIN Support på telefon (0848 830 740, mandag til fredag kl. 08:00–18:00). Noter det nye initialiseringspassordet før du lukker vinduet.

## HIN Mail Global: åpne kryptert e-post uten HIN-konto

Legekontorer, sykehus og laboratorier sender meldinger til pasienter eller instanser uten HIN-medlemskap via HIN Mail Global. E-posten kommer til deg som en vanlig e-post, men innholdet ligger kryptert i vedlegget `secure-email.html` eller bak lenken «Les kryptert melding». Det finnes ingen HIN-innlogging i egentlig forstand; innloggingen skjer via mobilnummeret ditt.

**Første gang:**

1. Last ned og åpne vedlegget `secure-email.html` (eller klikk på lenken i e-posten). For webmail-tjenester som Gmail, Bluewin eller GMX må du først lagre og deretter åpne filen.
2. Velg språk og angi ditt eget mobilnummer.
3. Angi koden du mottar på SMS. Meldingen vises.

**Ved senere meldinger** er det nok å åpne vedlegget; SMS-koden sendes automatisk til det registrerte nummeret.

Vær oppmerksom på:

- Meldingen er tilgjengelig via lenken i **30 dager**, deretter utløper tilgangen. Lagre eller skriv ut viktig innhold innen denne fristen.
- Videresendte meldinger kan bare åpnes av den opprinnelige mottakeren.
- Du kan svare kryptert.
- En enhet kan merkes som pålitelig; da bortfaller SMS-forespørselen i ett år.
- Hvis det ikke kommer noen kode innen 15 minutter, kan du be om en ny.
- De to nyeste versjonene av Chrome, Safari, Edge og Firefox støttes.

Ved problemer med HIN Mail Global kan HIN Mail Support hjelpe på +41 58 670 48 69. Spørsmål om innholdet i meldingen kan bare besvares av avsenderen.

## Vanlige feil ved HIN Mail-innlogging

| Symptom | Årsak | Løsning |
|---|---|---|
| Outlook spør stadig etter passordet | HIN-passord angitt i stedet for e-posttoken | Opprett token i `apps.hin.ch` og angi det som passord |
| Innlogging i Outlook mislykkes til tross for token | E-postadresse brukt som brukernavn | Bruk HIN ID som brukernavn |
| Sending mislykkes, mottak fungerer | SMTP-autentisering ikke aktivert, feil port eller feil kryptering | Aktiver port `587` med STARTTLS og autentisering |
| Webmail ber om en kode som aldri kommer | Mobilnummeret er utdatert eller SMS-kode er ikke aktivert | Bruk Authenticator App eller logg inn via HIN Client og oppdater nummeret |
| Initialiseringspassord avvises | 10-minuttersfristen er overskredet | Be om et nytt initialiseringspassord |
| Tokenet fungerte, men plutselig ikke lenger | Tokenet er tilbakekalt i `apps.hin.ch` eller enheten er fjernet | Opprett et nytt token for denne enheten |
| HIN Mail Global: lenken åpner ikke lenger noe | 30-dagersfristen er utløpt | Be avsenderen sende på nytt |

## HIN Client 4.0: hva som endres ved innlogging

HIN distribuerer HIN Client 4.0 automatisk frem til 26. oktober 2026; de som ikke får automatisk oppdatering, kan laste den ned fra `download.hin.ch`. Det eksisterende passordet forblir gyldig og bekreftes én gang ved første innlogging med den nye versjonen. Innlogging på HIN-beskyttede tjenester som Webmail foregår deretter direkte i nettleseren.

Versjon 4.0 krever Windows 11 med TPM eller macOS 14 med Security Chip. Enheter som ikke oppfyller disse kravene, kan ikke lenger bruke klienten. Tilgang til Webmail er fortsatt mulig via SMS-kode eller Authenticator App, og henting av e-post i Outlook via e-posttokenet. Detaljer om fristene for plattformfornyelsen finner du i artikkelen [HIN plattformfornyelse 2026](/blog/hin-plattformerneuerung-2026).

## Legekontorer og sykehus med egen e-postserver

Organisasjoner med egen HIN Mailgateway logger ikke inn på HIN Mail. Postkassene ligger på egen Exchange eller i Exchange Online, og gatewayen (en SEPPmail-appliance) krypterer e-postene på vei til og fra HIN-fellesskapet. Innloggingsproblemer gjelder da organisasjonens eget e-postsystem, ikke HIN-plattformen. Denne gatewayen erstattes av den nye HIN Gateway («Stargate»).

<aside class="offer-box">
  <span class="offer-box__tag">Gratis sjekk</span>
  <p><strong>Driver du din egen HIN Mailgateway?</strong> Jeg undersøker miljøet ditt og forteller deg hva som må gjøres før migreringen til «Stargate».</p>
  <a class="offer-box__cta" href="/stargate">Registrer deg nå</a>
</aside>

## Kilder

1.  [HIN Support: HIN Mail og Mobile](https://support.hin.ch/de/thema/hin-mail-mobile/): Oversikt med Webmail-adresse og veiledninger for Outlook, Apple Mail, Thunderbird, iOS og Android.

2.  [HIN Support: Hvordan fungerer Webmail?](https://support.hin.ch/de/service/hin-mail-und-mobile/wie-funktioniert-das-webmail.cfm): Funksjoner i Webmail, lagringsindikator.

3.  [HIN Support: Veiledning for Mail Token Service for Mail Clients](https://support.hin.ch/de/hin-mail-mobile/mail-token-service-mail-clients/): Opprettelse av token i apps.hin.ch, IMAP- og SMTP-servere, porter og kryptering.

4.  [HIN Support: Hva er et token?](https://support.hin.ch/de/service/hin-mail-und-mobile/was-ist-ein-token.cfm): E-posttoken og tilgangstoken, knyttet til HIN-identiteten.

5.  [HIN Support: Aktiver HIN-identitet (uten HIN Client)](https://support.hin.ch/de/hin-2fa/hin-identitaet-aktivieren/): Aktivering i Servicecenter, 10-minuttersfrist, passordregler, andre faktor.

6.  [HIN Support: Glemt passord](https://support.hin.ch/de/thema/hin-client/passwort-vergessen.cfm): Tilbakestilling via initialiseringspassord, telefonstøtte uten alternativ metode.

7.  [HIN Support: HIN Mail til ikke-medlemmer](https://support.hin.ch/en/service/hin-mail-to-non-members.cfm): Tilgjengelighet i 30 dager, SMS-registrering, pålitelige enheter, støttede nettlesere, supporttelefon.

8.  [HIN: HIN Mail Global](https://www.hin.ch/services/hin-mail/hin-mail-global/): Sending av krypterte e-poster til mottakere uten HIN-medlemskap.

9.  [HIN Blogg: Den nye HIN Client er her](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): Nettleserinnlogging, systemkrav med TPM eller Security Chip, utrulling frem til 26. oktober 2026.

10.  [HIN Authenticator i App Store](https://apps.apple.com/ch/app/hin-authenticator/id1535944002): App for den andre faktoren.
