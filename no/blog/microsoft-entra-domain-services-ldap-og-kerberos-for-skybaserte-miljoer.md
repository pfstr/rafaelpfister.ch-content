---
title: "Microsoft Entra Domain Services: LDAP og Kerberos for skybaserte miljøer"
navTitle: "Entra Domain Services"
description: "Entra ID snakker ikke LDAP eller Kerberos. Microsoft Entra Domain Services tilbyr et administrert Active Directory-domene som synkroniserer brukere fra Entra ID og tilbyr klassiske protokoller. Virkemåte, begrensninger, kostnader og et praktisk eksempel med en e-postgateway."
date: "2026-07-30"
kategorie: "Active Directory / Entra"
timeToRead: "8 min lesetid"
themen:
  - active-directory-entra
slug: "microsoft-entra-domain-services-ldap-og-kerberos-for-skybaserte-miljoer"
translationId: "article-9c271900a94406b8"
draft: false
translationOf: microsoft-entra-domain-services-ldap-kerberos
translationSourceHash: 6360f60ed2e92d286f0e279f487b62a86fa9a987c2f574b0a53af0d31f0d736b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:22:08.235Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/microsoft-entra-domain-services-ldap-og-kerberos-for-skybaserte-miljoer
---

# Microsoft Entra Domain Services: LDAP og Kerberos for skybaserte miljøer

De som har flyttet brukerne sine fullstendig til Microsoft Entra ID (tidligere Azure Active Directory), merker det senest ved den første appliance-en eller eldre applikasjonen: Entra ID besvarer forespørsler via Microsoft Graph og moderne autentiseringsprotokoller som OAuth og SAML, men ikke via LDAP, Kerberos eller NTLM. En LDAP-bind mot Entra ID er ganske enkelt ikke mulig. For alt som forventer et klassisk Active Directory, tilbyr Microsoft derfor en egen tjeneste: Microsoft Entra Domain Services, tidligere Azure AD Domain Services.

## Hva tjenesten tilbyr

Entra Domain Services er et administrert Active Directory-domene. Microsoft drifter to Windows-domenekontrollere i et Azure-VNet, håndterer patching, replikering og sikkerhetskopiering, og synkroniserer brukere og grupper automatisk fra Entra ID til domenet. Synkroniseringen går bare én vei, fra Entra ID til det administrerte domenet; endringer direkte i domenet flyter ikke tilbake.

Utad oppfører domenet seg som et vanlig Active Directory: Det besvarer LDAP- og LDAPS-forespørsler, støtter Kerberos- og NTLM-autentisering, tillater at VM-er meldes inn i domenet og tilbyr begrensede gruppepolicyer. Applikasjoner og enheter trenger ikke tilpasses for dette; de ser en domenekontroller.

## Hva det brukes til

Tjenesten er rettet mot miljøer som egentlig bare er skybaserte, men som bruker enkelte komponenter med klassiske katalogkrav:

- **Appliances og fagapplikasjoner med LDAP-tilkobling:** Enheter som søker etter brukere via LDAP, evaluerer gruppemedlemskap eller kontrollerer innlogginger via LDAP-bind.
- **Lift-and-shift-migreringer:** Serverarbeidslaster som må forbli domenetilknyttet (Kerberos, NTLM, domeneinnmelding), uten at egne domenekontrollere skal driftes i Azure.
- **Miljøer uten lokalt AD:** Der det aldri har vært et Active Directory, eller det er avviklet, erstatter det administrerte domenet oppsettet av egne DC-er med tilhørende driftsarbeid.

Viktig avgrensning: De som fortsatt drifter et lokalt Active Directory med Entra Connect-synkronisering, trenger som regel ikke tjenesten; appliance-en spør da det eksisterende AD-et. Entra Domain Services fyller gapet når Entra ID er den eneste brukerkilden.

## Arkitektur og oppsett

Det administrerte domenet distribueres i et eget VNet eller subnett og får to faste adresser til domenekontrollere. Arbeidslaster i andre VNet-er når det via VNet-peering; DNS-serverne i de berørte VNet-ene må peke på domenekontrollerne slik at domenenavn og objekter kan slås opp. Tilgangen begrenses via Network Security Groups til de faktisk nødvendige kildene og portene.

Noen særtrekk ved det administrerte domenet som er relevante ved tilkobling av applikasjoner:

- Synkroniserte brukere ligger i OU-en **AADDC Users**, og domenet bruker **onmicrosoft.com**-suffikset uten egen konfigurasjon. Search Base og Bind-DN-er må gjenspeile denne strukturen, for eksempel CN=svc-ldap,OU=AADDC Users,DC=example,DC=onmicrosoft,DC=com.
- Det finnes ingen Domain Administrator. Administrasjonen skjer via den delegerte gruppen AAD DC Administrators; skjemautvidelser er ikke mulig.
- For LDAP-bind-kontoer er en dedikert, upriviligert konto tilstrekkelig; for rene katalogoppslag i Entra ID kreves rollen Directory Readers.

## Problemet med passordhasher

Ett punkt tar jevnlig tid i tester: Kerberos- og NTLM-innlogginger samt LDAP-bind krever passordhasher i det administrerte domenet. For kontoer som kun finnes i skyen, oppretter Entra ID disse hashene først ved neste passordendring etter at tjenesten er aktivert. En nylig synkronisert bruker er altså synlig i katalogen, men kan først logge på etter å ha endret passordet sitt én gang. For hybride kontoer må hashene også synkroniseres fra det lokale AD-et via Entra Connect.

## Secure LDAP steg for steg

Innenfor domenet kjører LDAP som standard ukryptert over port 389. For innlogginger og all tilgang utenfor strengt isolerte nettverk bør forbindelsen bruke Secure LDAP (LDAPS, port 636); tjenesten tilbyr uansett bare kryptert tilgang utenfor VNet-et. Oppsettet består av fire trinn.

**1. Skaff sertifikat.** Secure LDAP trenger et eget sertifikat som lastes opp som PFX inkludert privat nøkkel. Subject eller SAN må dekke det administrerte domenet med wildcard (for eksempel *.example.onmicrosoft.com), siden forespørsler kan havne på begge domenekontrollerne. En offentlig CA, egen PKI eller et særskilt opprettet selvsignert sertifikat kan brukes som kilde. Ved selvsignert sertifikat må kjeden være registrert som klarert på hvert system som spør; ikke alle appliances tillater dette. De som kan velge, er bedre stilt med egen PKI eller en offentlig CA.

**2. Aktiver Secure LDAP.** I portalen under Settings > Secure LDAP slås funksjonen på, og PFX-filen med passord lastes opp. Der kan tilgang via Internett eventuelt frigis; det administrerte domenet får da en offentlig IP-adresse.

**3. Nettverk og DNS.** Den eksterne IP-adressen finnes under Properties. Den tilhørende NSG-regelen åpner TCP/636 og bør begrenses til de faktisk nødvendige kilde-IP-adressene, ikke til Any. For navneoppløsning peker en DNS-oppføring (for eksempel ldaps.example.com) på denne IP-adressen; den må passe til sertifikatet. Intern tilgang går fortsatt direkte mot domenekontrolleradressene.

**4. Test forbindelsen.** Før applikasjonen legges om, lønner det seg å teste med en LDAP-nettleser, ldp.exe eller ldapsearch mot port 636: Bind med tjenestekontoen, deretter et søk under OU-en AADDC Users. Først når bind og søk fungerer korrekt der, er det applikasjonens tur.

For å sette opp selve tjenesten trenger portalkontoen rollene Application Administrator, Domain Services Contributor og Groups Administrator; utrullingen av det administrerte domenet tar godt over en time. I sikkerhetsinnstillingene kan TLS 1.2 også håndheves som minimum.

## Kostnader

Entra Domain Services er en løpende driftskostnad: Tjenesten faktureres per time etter SKU, i tillegg kommer VNet, peering og eventuelle test-VM-er. For ett lite LDAP-bruksområde er det en betydelig pris; alternativet, å drifte egne domenekontrollere som VM-er, kjøper imidlertid besparelsen med ansvar for patching, sikkerhetskopiering og tilgjengelighet.

## Praktisk eksempel: E-postgateway med LDAP-tilkobling

Et konkret eksempel fra appliance-kategorien er SEPPmail Secure E-Mail Gateway. Den bruker LDAP til brukeropprettelse og rettighetsoppslag, og siden fastvare 15.0.6 også til [innlogging på Admin-GUI-en](/blog/seppmail-admin-gui-ldap-authentifizierung). En appliance i Azure-VNet-et når det administrerte domenet via VNet-peering med en dedikert bind-konto (Directory Readers), sikret med NSG-er. Senest for innlogging på Admin-GUI-en, der TLS-alternativet er aktivert som standard, bør forbindelsen bruke Secure LDAP.

## Konklusjon

Entra Domain Services er ikke en erstatning for Entra ID, men en bro: Tjenesten oversetter en skybasert brukerbase til et klassisk AD-domene for alt som krever LDAP, Kerberos eller domeneinnmelding. De som bare må koble til én enkelt applikasjon, bør veie løpende kostnader opp mot modernisering av applikasjonen. Når tjenesten først er på plass, oppfører appliances og eldre applikasjoner seg som i et velkjent AD-miljø, inkludert de beskrevne særtrekkene for OU-struktur, rettigheter og passordhasher.

## Kilder

1.  [Microsoft Learn – «Hva er Microsoft Entra Domain Services?»](https://learn.microsoft.com/de-de/entra/identity/domain-services/overview): Funksjonsomfanget til det administrerte domenet, støttede protokoller og avgrensning mot Entra ID og selvdriftede domenekontrollere.

2.  [Microsoft Learn – «Synkronisering i Entra Domain Services»](https://learn.microsoft.com/de-de/entra/identity/domain-services/synchronization): Enveis synkronisering, OU-struktur og hvordan passordhasher fungerer for kontoer som kun finnes i skyen og hybride kontoer.

3.  [Microsoft Learn – «Konfigurer Secure LDAP»](https://learn.microsoft.com/de-de/entra/identity/domain-services/tutorial-configure-ldaps): LDAPS med eget sertifikat for kryptert LDAP-tilgang.

4.  [Koble SEPPmail Admin-GUI til Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung): Oppsett av LDAP-innlogging på Admin-GUI-en fra fastvare 15.0.6.
