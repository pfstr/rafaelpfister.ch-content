---
title: "HIN Mail-innlogging: webmail, Outlook-token, glemt passord og HIN Mail Global"
navTitle: "HIN Mail-innlogging"
description: "Hvor og hvordan du logger inn på HIN Mail: webmail på webmail.hin.ch, e-posttoken for Outlook og smarttelefon, innlogging uten HIN Client via SMS-kode eller Authenticator App, tilbakestilling av passord og åpning av HIN Mail Global som mottaker uten HIN-konto."
date: "2026-10-05"
kategorie: "HIN-gateway"
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
translationSourceHash: 00c1c2a195ebd80ccf8c6f7e7318d3b924eff80efe6315bd1d6fc95c05ad3c9f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:31:44.045Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/hin-mail-innlogging-webmail-outlook-token-glemt-passord-og-hin-mail-global
---

# HIN Mail-innlogging: webmail, Outlook-token, glemt passord og HIN Mail Global

Bak søket etter «HIN Mail-innlogging» ligger som regel tre ulike situasjoner: Du er HIN-medlem og vil lese e-post i nettleseren, du vil sette opp HIN Mail i Outlook eller på smarttelefonen, eller du har som pasient eller ekstern instans mottatt en kryptert HIN-e-post og har ingen HIN-konto. Innloggingen foregår på ulike måter i alle tre tilfellene.

| Situasjon | Adresse | Brukernavn | Passord |
|---|---|---|---|
| Webmail i nettleseren | `webmail.hin.ch` | HIN ID | HIN-passord, sikret med HIN Client, SMS-kode eller Authenticator App |
| Outlook, Thunderbird, Apple Mail, smarttelefon | `imap.mail.hin.ch`, `smtp.mail.hin.ch` | HIN ID | E-posttoken fra `apps.hin.ch` |
| Administrasjon, token, initialiseringspassord | `apps.hin.ch` | HIN ID | HIN-passord pluss andre faktor |
| Mottaker uten HIN-konto (HIN Mail Global) | Vedlegg `secure-email.html` eller lenke i e-posten | Mobilnummer | SMS-kode |

## HIN Webmail: innlogging i nettleseren

Du finner webmail på `https://webmail.hin.ch`. Du logger inn med HIN ID-en din (HIN-innloggingsnavnet, f.eks. `hmuster`) og HIN-passordet ditt. Den andre faktoren leveres av en av følgende metoder:

- **HIN Client:** Hvis klienten kjører på arbeidsplassen og du er innlogget, åpnes webmail uten ytterligere spørsmål. Med HIN Client 4.0 foregår innloggingen fra begynnelsen av november 2026 direkte i nettleseren: Du skriver inn den HIN-beskyttede adressen og blir videresendt til innloggingen.
- **SMS-kode (mTAN):** Etter passordet mottar du en kode på det registrerte mobilnummeret. Denne metoden passer for enheter uten HIN Client, for eksempel en privat bærbar PC.
- **HIN Authenticator App:** Appen (App Store og Google Play, «HIN Authenticator») bekrefter innloggingen på smarttelefonen.

SMS-kode og Authenticator App må aktiveres på forhånd. Hvis begge mangler og HIN Client ikke er tilgjengelig, er det ikke mulig å logge inn på webmail. Sett derfor opp minst ett alternativ mens klienten fortsatt fungerer.

I webmail kan du skrive og administrere e-post, vedlikeholde kontakter og adressebøker, søke i HIN-deltakerkatalogen og opprette flere signaturer. Under brukernavnet viser en stolpe ledig lagringsplass i postkassen; er den rød, må du slette e-post eller utvide lagringsplassen.

## HIN Mail i Outlook og på smarttelefonen: innlogging med e-posttoken

E-postprogrammer logger ikke inn med HIN-passordet, men med et **e-posttoken**. Dette er den vanligste årsaken til at Outlook gjentatte ganger spør etter passord: HIN-passordet avvises ved IMAP-innlogging.

Slik oppretter du tokenet:

1. Åpne `https://apps.hin.ch` og logg inn med HIN ID (HIN Client, SMS-kode eller Authenticator App).
2. Velg området «HIN Mail».
3. Klikk på «Legg til enhet» og velg «Mail Clients» som enhetstype.
4. Kopier tokenet som vises, og skriv det inn som passord i e-postprogrammet.

Serverinnstillingene:

| Innstilling | Innkommende e-post (IMAP) | Utgående e-post (SMTP) |
|---|---|---|
| Server | `imap.mail.hin.ch` | `smtp.mail.hin.ch` |
| Port | `993` | `587` |
| Kryptering | SSL/TLS | STARTTLS |
| Brukernavn | HIN ID | HIN ID |
| Passord | E-posttoken | E-posttoken |
| Autentisering | normal | Aktiver «Utgående server krever autentisering» |

HIN ID-en er brukernavnet, ikke e-postadressen. Outlook foreslår e-postadressen ved automatisk oppsett; oppsettet må derfor gjøres manuelt via «Avanserte alternativer» eller «Manuelt oppsett».

Tokenet er knyttet til HIN-identiteten din og kan når som helst tilbakekalles og opprettes på nytt i `apps.hin.ch`. Det anbefales å bruke et eget token for hver enhet: Hvis en smarttelefon går tapt, sperrer du kun denne enheten, mens Outlook på praksis-PC-en fortsatt fungerer. Når tokenet er konfigurert, trenger ikke Outlook HIN Client for å hente e-post.

## Første innlogging uten HIN Client: aktiver identiteten

Nye HIN-identiteter kan aktiveres uten HIN Client. Du trenger brukernavnet og initialiseringspassordet fra HIN-dokumentene.

1. Åpne aktiveringslenken i Servicecenter (`servicecenter.hin.ch`).
2. Skriv inn brukernavn og initialiseringspassord.
3. Angi et eget passord: minst 10 tegn med tall, store og små bokstaver samt spesialtegn.
4. Konfigurer straks en andre faktor (SMS-kode eller Authenticator App).
5. Logg deretter inn på `apps.hin.ch`.

Du har 10 minutter til aktiveringen. Deretter blir initialiseringspassordet tilbakestilt av sikkerhetsgrunner, og du må be om et nytt.

## Glemt HIN-passord

Tilbakestillingen avhenger av om en alternativ innloggingsmetode er konfigurert.

**Med SMS-kode eller Authenticator App:**

1. Logg inn på `apps.hin.ch` med den alternative metoden.
2. Velg fanen «HIN Client» og opprett et nytt initialiseringspassord.
3. Velg «Registrering» i HIN Client, og skriv inn innloggingsnavnet og det nye initialiseringspassordet.
4. Angi et nytt passord og bekreft «Registrer ny HIN-identitet».

**Uten alternativ metode og uten en innlogget enhet** gjenstår bare HIN Support på telefon (0848 830 740, mandag til fredag kl. 08:00–18:00). Noter det nye initialiseringspassordet før du lukker vinduet.

## HIN Mail Global: åpne kryptert e-post uten HIN-konto

Legekontorer, sykehus og laboratorier sender meldinger til pasienter eller instanser uten HIN-medlemskap via HIN Mail Global. E-posten kommer til deg som en vanlig e-post, men innholdet ligger kryptert i vedlegget `secure-email.html` eller bak lenken «Les kryptert melding». Det finnes ingen HIN-innlogging i egentlig forstand her; innloggingen skjer via mobilnummeret ditt.

**Første gang:**

1. Last ned og åpne vedlegget `secure-email.html` (eller klikk på lenken i e-posten). Ved webmail-tjenester som Gmail, Bluewin eller GMX må du først lagre og deretter åpne det.
2. Velg språk og skriv inn ditt eget mobilnummer.
3. Skriv inn koden du mottar på SMS. Meldingen vises.

**Ved senere meldinger** er det nok å åpne vedlegget; SMS-koden sendes automatisk til det registrerte nummeret.

Merk:

- Meldingen er tilgjengelig via lenken i **30 dager**, deretter utløper tilgangen. Lagre eller skriv ut viktig innhold innen denne fristen.
- Videresendte meldinger kan bare åpnes av den opprinnelige mottakeren.
- Du kan svare kryptert.
- En enhet kan merkes som pålitelig; da bortfaller SMS-forespørselen i ett år.
- Hvis det ikke kommer noen kode innen 15 minutter, kan du be om en ny.
- De to nyeste versjonene av Chrome, Safari, Edge og Firefox støttes.

Ved problemer med HIN Mail Global hjelper HIN Mail Support på +41 58 670 48 69. Spørsmål om innholdet i meldingen kan kun besvares av avsenderen.

## Vanlige feil ved HIN Mail-innlogging

| Symptom | Årsak | Løsning |
|---|---|---|
| Outlook spør stadig etter passordet | HIN-passord angitt i stedet for e-posttoken | Opprett token i `apps.hin.ch` og angi det som passord |
| Innlogging i Outlook mislykkes til tross for token | E-postadresse brukt som brukernavn | Bruk HIN ID som brukernavn |
| Sending mislykkes, mottak fungerer | SMTP-autentisering ikke aktivert, feil port eller feil kryptering | Bruk port `587` med STARTTLS, og aktiver autentisering |
| Webmail krever en kode som aldri kommer | Mobilnummeret er utdatert eller SMS-kode er ikke aktivert | Bruk Authenticator App eller logg inn via HIN Client og oppdater nummeret |
| Initialiseringspassordet avvises | 10-minuttersfristen er overskredet | Be om et nytt initialiseringspassord |
| Tokenet fungerte, men gjør plutselig ikke det lenger | Tokenet er tilbakekalt i `apps.hin.ch` eller enheten er fjernet | Opprett et nytt token for denne enheten |
| HIN Mail Global: lenken åpner ikke lenger noe | 30-dagersfristen er utløpt | Be avsenderen om å sende på nytt |

## HIN Client 4.0: hva som endrer seg ved innlogging

HIN distribuerer HIN Client 4.0 automatisk frem til 26. oktober 2026; de som ikke får automatisk oppdatering, laster den ned på `download.hin.ch`. Det eksisterende passordet forblir gyldig; ved første innlogging med den nye versjonen blir du én gang bedt om å endre det. Innlogging på HIN-beskyttede tjenester som webmail foregår fra begynnelsen av november 2026 direkte i nettleseren.

Versjon 4.0 krever Windows 11 med TPM eller macOS 14 med Security Chip. Enheter som ikke oppfyller disse kravene, kan ikke lenger bruke klienten. Tilgang til webmail er fortsatt mulig der via SMS-kode eller Authenticator App, og e-posthenting i Outlook via e-posttokenet. Detaljer om fristene for plattformfornyelsen finnes i artikkelen [HIN plattformfornyelse 2026](/blog/hin-plattformerneuerung-2026).

## Legekontorer og sykehus med egen e-postserver

Organisasjoner med egen HIN Mailgateway logger ikke inn på HIN Mail. Postkassene ligger i egen Exchange eller i Exchange Online, og gatewayen (en SEPPmail-appliance) krypterer e-postene på vei til og fra HIN-fellesskapet. Innloggingsproblemer gjelder da organisasjonens eget e-postsystem, ikke HIN-plattformen. Denne gatewayen erstattes av den nye HIN Gateway («Stargate»).

<aside class="offer-box">
  <span class="offer-box__tag">Kostnadsfri kontroll</span>
  <p><strong>Driver du en egen HIN Mailgateway?</strong> Jeg vurderer miljøet ditt og forteller deg hva som må gjøres før migreringen til «Stargate».</p>
  <a class="offer-box__cta" href="/stargate">Registrer deg nå</a>
</aside>

## Endringslogg

- **25. oktober 2026:** Rettet: Ved første innlogging med HIN Client 4.0 må passordet endres én gang, og innlogging i nettleseren gjelder først fra begynnelsen av november 2026.

## Kilder

1.  [HIN Support: HIN Mail og Mobile](https://support.hin.ch/de/thema/hin-mail-mobile/): Oversikt med webmail-adresse og veiledninger for Outlook, Apple Mail, Thunderbird, iOS og Android.

2.  [HIN Support: Hvordan fungerer webmail?](https://support.hin.ch/de/service/hin-mail-und-mobile/wie-funktioniert-das-webmail.cfm): Funksjoner i webmail, lagringsindikator.

3.  [HIN Support: Veiledning for Mail Token Service for Mail Clients](https://support.hin.ch/de/hin-mail-mobile/mail-token-service-mail-clients/): Opprettelse av token i apps.hin.ch, IMAP- og SMTP-servere, porter og kryptering.

4.  [HIN Support: Hva er et token?](https://support.hin.ch/de/service/hin-mail-und-mobile/was-ist-ein-token.cfm): E-posttoken og tilgangstoken, tilknytning til HIN-identiteten.

5.  [HIN Support: Aktiver HIN-identitet (uten HIN Client)](https://support.hin.ch/de/hin-2fa/hin-identitaet-aktivieren/): Aktivering i Servicecenter, 10-minuttersfrist, passordregler, andre faktor.

6.  [HIN Support: Glemt passord](https://support.hin.ch/de/thema/hin-client/passwort-vergessen.cfm): Tilbakestilling via initialiseringspassord, telefonisk brukerstøtte uten alternativ metode.

7.  [HIN Support: HIN Mail til ikke-medlemmer](https://support.hin.ch/en/service/hin-mail-to-non-members.cfm): 30 dagers tilgjengelighet, SMS-registrering, pålitelige enheter, støttede nettlesere, telefonnummer til brukerstøtte.

8.  [HIN: HIN Mail Global](https://www.hin.ch/services/hin-mail/hin-mail-global/): Sending av kryptert e-post til mottakere uten HIN-medlemskap.

9.  [HIN Blogg: Den nye HIN Client er her](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): Nettleserinnlogging fra begynnelsen av november 2026, passordendring ved første innlogging, systemkrav med TPM eller Security Chip, utrulling frem til 26. oktober 2026.

10.  [HIN Authenticator i App Store](https://apps.apple.com/ch/app/hin-authenticator/id1535944002): App for den andre faktoren.
