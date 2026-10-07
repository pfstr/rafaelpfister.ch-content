---
title: "Kerberos: billetter, nøkler og pålitelige tjenester"
blatt: "kerberos"
description: "Kerberos for administratorer: KDC, AS-/TGS-/AP-flyt, principaler og SPN-er, billetter, øktnøkler, keytabs og KVNO, GSS-API, Active Directory og PAC, klareringsforhold, delegering, kryptering, DNS, tid, drift og diagnose."
fakten:
  - label: Oppgave
    wert: Nettverksautentisering via billetter
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-1.1
  - label: Protokollversjon
    wert: Kerberos V5 · RFC 4120
    href: https://datatracker.ietf.org/doc/html/rfc4120
  - label: Tillitsinstans
    wert: KDC med Authentication Service og TGS
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-1.2
  - label: Utveksling
    wert: AS · TGS · klient/server-AP
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-3
  - label: KDC-port
    wert: 88 over UDP og TCP
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=kerberos
  - label: Passordtjeneste
    wert: kpasswd · Port 464
    href: https://datatracker.ietf.org/doc/html/rfc3244
  - label: Tjenestenavn
    wert: service/host@REALM
    href: https://web.mit.edu/kerberos/krb5-latest/doc/admin/princ_dns.html
  - label: Tjenestenøkkel
    wert: Principal · KVNO · Enctype · Key
    href: https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html
  - label: Tidsvindu
    wert: Clock Skew er Realm- og klientpolicy
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-3.2.3
  - label: Applikasjons-API
    wert: GSS-API / SSPI · ofte SPNEGO
    href: https://datatracker.ietf.org/doc/html/rfc4121
  - label: Active Directory
    wert: Domenekontrollere integrerer KDC-en
    href: https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview
  - label: AD-autorisering
    wert: PAC som Ticket-Authorization-Data
    href: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/166d8064-c863-41e1-9c23-edaaa5f36962
werbung:
  - newsletter
ctaThemen:
  - active-directory-entra
translationSourceHash: b5eccfbd21dfaa9b065424961e28940ca63617bba9dbb9f5ab68c1db128df31f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T11:16:13.464Z
translationReview: required
---

# Kerberos: billetter, nøkler og pålitelige tjenester

Kerberos gjør det mulig for en bruker eller prosess å autentisere seg overfor en nettverkstjeneste uten å sende passordet til tjenesten. Klient og server stoler da på et Key Distribution Center (KDC), som utsteder tidsbegrensede billetter og øktnøkler. Ved passordbasert pålogging avledes det likevel en langsiktig nøkkel fra passordet; passordkvalitet, pre-autentisering og beskyttelsen av klienten er derfor fortsatt sikkerhetsrelevante ([RFC 4120, avsnitt 1.1 og 1.2](https://datatracker.ietf.org/doc/html/rfc4120#section-1.1), [RFC 3961, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc3961#section-3)).

For meldings- og katalogadministratorer ligger Kerberos ofte ved siden av [LDAP](/kb/ldap), ikke i stedet for det. LDAP transporterer katalogoperasjoner; Kerberos kan levere identiteten for en LDAP-økt via SASL. Nettgrensesnitt bruker HTTP Negotiate, Windows-tjenester SSPI og Unix-applikasjoner som regel GSS-API. En feil kan derfor oppstå i DNS, tid, KDC-tilgjengelighet, realm-tilordning, billettbuffer, SPN, tjenestekonto, keytab, krypteringstype, PAC eller delegeringspolicy, selv om applikasjonen bare melder «Integrated Authentication failed».

Forklaringen følger en principal fra første kontakt med KDC-en via TGT og tjenestebillett til måltjenesten. Først deretter utdypes Active Directory-utvidelser, delegering, tidsavhengighet, diagnose og gjenoppretting.

## Arkitekturtilnærming: pålitelig tredjepart

Kerberos distribuerer symmetriske nøkler via en pålitelig instans. KDC-en har tilgang til de langsiktige nøklene til principalene i sitt realm og samler logisk to tjenester: Authentication Service, AS, utsteder en Ticket-Granting Ticket, TGT; Ticket-Granting Service, TGS, bytter denne TGT-en mot en billett for en konkret applikasjonstjeneste. Klient/server-utvekslingen, AP, finner deretter sted mellom klient og tjeneste. En tjeneste kan normalt kontrollere en tjenestebillett med sin egen nøkkel uten å kontakte KDC-en på nytt ved hver pålogging ([RFC 4120, avsnitt 1.2 og 3](https://datatracker.ietf.org/doc/html/rfc4120#section-1.2)).

Denne modellen reduserer passordvidereformidling og sentrale nettkontroller, men skaper tydelige feilområder. Uten en tilgjengelig KDC kan nye billetter ikke utstedes; allerede eksisterende billetter kan fortsette å fungere til gyldigheten utløper. En kompromittert KDC-database eller realm-nøkkel truer derimot tillitskjeden for hele realmet. Kerberos er derfor ikke en tilstandsløs plattform for token-signaturer, men et distribuert system av KDC-tilstand, klientbuffere, tjenestenøkler, klokker og navnetjenester.

| Nivå | Teknisk komponent | Ansvarlig tilstand | Administrativt bevis |
|---|---|---|---|
| Applikasjon | HTTP, SMB, LDAP, database, SMTP/IMAP med SASL eller proprietær tjeneste | Applikasjonens økt og autorisering | faktisk valgt autentiseringsmekanisme og målnavn |
| Integrasjons-API | GSS-API, Windows SSPI og ofte SPNEGO | Security Context, delegering og Channel Binding | forhandlet mekanisme, initiator, acceptor og flagg |
| Kerberos AP | `KRB_AP_REQ`, valgfritt `KRB_AP_REP`, eventuelt GSS Wrap/MIC | Tjenestebillett, autentikator og øktnøkkel | målprincipal, billettider, enctype og gjensidig autentisering |
| Kerberos KDC | AS- og TGS-utveksling | Principal-database, realm-nøkler, policyer og billettflagg | KDC-resultat, valgt nøkkel, KVNO og revisjonshendelse |
| Oppdagelse og transport | DNS SRV, UDP/TCP 88, passordtjeneste 464 | Resolverbuffer, realm-tilordning, ruting og pakkestørrelse | faktisk valgt KDC, transport, svartid og reservevalg |
| Persistens | AD DS eller Kerberos-database, keytabs, credential- og replay-buffere | langsiktige nøkler, billetter, PAC, replikering og replay-status | replikeringsstatus, bufferinnhold, filrettigheter og gjenopprettingsprosedyre |

Kerberos autentiserer principaler og kan gi integritet eller konfidensialitet for en GSS-sikkerhetskontekst. Det krypterer ikke automatisk hele nyttedatastrømmen til en applikasjon og gir ikke Perfect Forward Secrecy for øktnøklene som distribueres i Kerberos-kjernen. Applikasjoner kan bruke Kerberos til å autentisere en separat beskyttet kanal; [TLS](/kb/tls) forblir derfor et eget lag med egen serveridentitet og sertifikatkontroll ved HTTPS eller LDAPS ([RFC 4120, avsnitt 10](https://datatracker.ietf.org/doc/html/rfc4120#section-10), [RFC 4121, avsnitt 2 og 4](https://datatracker.ietf.org/doc/html/rfc4121#section-2)).

## Principaler, realmer og nøkkelmateriale

En Kerberos-principal er et navn innenfor et realm. Brukere vises ofte som `alice@EXAMPLE.CH`, tjenester som `HTTP/intranet.example.ch@EXAMPLE.CH`. Delen foran `@` kan inneholde flere komponenter; store og små bokstaver er i utgangspunktet relevante i Kerberos-navnemodellen. Et DNS-domenenavn og et Kerberos-realm er forskjellige navnerom, selv om realmer vanligvis ser ut som DNS-domener med store bokstaver. Klienter trenger derfor en kontrollerbar tilordning fra vertsnavn eller DNS-domene til realm ([RFC 4120, avsnitt 6.1 og 7.2.3](https://datatracker.ietf.org/doc/html/rfc4120#section-6.1), [MIT Kerberos: Mapping hostnames onto realms](https://web.mit.edu/kerberos/krb5-latest/doc/admin/realm_config.html#mapping-hostnames-onto-kerberos-realms)).

Langsiktige nøkler tilhører principaler. Øktnøkler opprettes for en begrenset utveksling. En billett inneholder blant annet klient, server, realm, tidsfelt, flagg, øktnøkkel og Authorization Data; dens krypterte del er beskyttet med en nøkkel til måltjenesten. Klienten mottar den samme øktnøkkelen i en responsdel som er beskyttet for klienten. Den kan dermed transportere billettinnholdet, men ikke selv endre den delen som er kryptert for tjenesten ([RFC 4120, avsnitt 5.3 og 5.4](https://datatracker.ietf.org/doc/html/rfc4120#section-5.3)).

Key Version Number, KVNO, skiller generasjoner av en principal-nøkkel. Encryption Type, Enctype, angir algoritme, nøkkellengde, String-to-Key og kontrollsum. En tjeneste kan ha flere keytab-oppføringer for samme principal med forskjellige KVNO-er eller enctypes for å tåle en kontrollert rotasjon. Hvis billettens KVNO og enctype ikke samsvarer med noen tilgjengelig tjenestenøkkel, kan tjenesten ikke dekryptere billetten. «SPN finnes» beviser derfor ennå ikke at det finnes en passende nøkkel på målsystemet ([RFC 4120, avsnitt 5.2.9](https://datatracker.ietf.org/doc/html/rfc4120#section-5.2.9), [MIT Kerberos: Keytabs](https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html)).

| Objekt | Sted | Konfidensialitet | Driftsgrense |
|---|---|---|---|
| langsiktig principal-nøkkel | KDC-database; ved tjenesten i tillegg keytab eller operativsystemets Key Store | svært kritisk | rotasjon, replikering, KVNO og tillatte enctypes |
| TGT | klientens credential-buffer; kryptert billettdel for `krbtgt/REALM` | nær bearer-token pluss øktnøkkel | levetid, Forwardable-/Renewable-flagg, bufferisolasjon |
| tjenestebillett | credential-buffer og deretter måltjenesten | kan bare dekrypteres for den navngitte tjenesten | SPN, tjenestekonto, målvert, PAC og billett-enctype |
| autentikator | ny for hver AP Request, beskyttet med øktnøkkel | replay-beskyttelse | klienttid, subkey, sekvens og serversidig replay-buffer |
| keytab | fil eller annen keytab-lagring hos tjenesten | beskyttes som et ikke-interaktivt passord | filrettigheter, distribusjon, inventar, rotasjon og sikker sletting |
| replay-buffer | lokal tilstand hos acceptoren | integritetstilstand | kontroller per tjenesteinstans, vert og klyngearkitektur |

Active Directory lagrer SPN-er i det flerverdige attributtet `servicePrincipalName` på en bruker- eller datamaskinkonto. SPN-en kobler tjenestenavnet som klienten setter sammen, til nøyaktig den kontoen hvis nøkkel KDC-en bruker for tjenestebilletten. Et alias, et navn på en lastbalanserer eller en endret tjenestekonto krever derfor en bevisst SPN-tilordning. Dupliserte SPN-er er tvetydige innenfor søkeområdet; en SPN på feil konto fører typisk til at tjenesten ikke kan dekryptere den utstedte billetten ([Microsoft: Service principal names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names), [Microsoft: setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn)).

En keytab er ikke en eksportfil med en identitet som kan gjenopprettes vilkårlig, men en samling reelle langsiktige nøkler. Hver oppføring inneholder principal, KVNO, enctype og nøkkel. Den som kan lese filen, kan utgi seg for å være denne principalen. MIT anbefaler lokal, restriktiv lagring og ingen ubeskyttet overføring. Ved interoperabilitet med Active Directory kan [`ktpass`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ktpass) koble principal, konto og keytab; de valgte parameterne kan da påvirke passord, salt, KVNO eller enctype og må inngå i en testet rotasjonsprosess, ikke på en engangs installasjonslapp ([MIT Kerberos: Application servers](https://web.mit.edu/kerberos/krb5-latest/doc/admin/appl_servers.html)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-kerberos.svg?v=20260813" title="Interaktive Infografik: Kerberos-Discovery, AS-, TGS- und AP-Fluss, Tickets, Schlüssel, Active-Directory-PAC und Delegationsgrenzen" loading="lazy">
  <a href="/images/kb-interaktiv-kerberos.svg?v=20260813">Åpne infografikk om Kerberos-protokollflyten og tillitsgrensene</a>
</iframe>

Når principal, realm og nøkler er avklart, kan billettflyten leses som tre påfølgende samtaler: AS, TGS og til slutt applikasjonstjenesten.

## Protokollflyt: AS, TGS og AP

AS-utvekslingen begynner med `KRB_AS_REQ`. En KDC kan svare med `KDC_ERR_PREAUTH_REQUIRED` og angi de støttede pre-autentiseringsprosedyrene. Med det utbredte krypterte tidsstempelet beviser klienten kjennskap til sin langsiktige nøkkel før KDC-en utsteder en TGT. `KRB_AS_REP` inneholder TGT-en, som er kryptert for TGS-en, og en responsdel for klienten. PKINIT erstatter dette første beviset med offentlig nøkkel-kryptografi og sertifikater; Kerberos FAST kan herde pre-autentisering i en beskyttet tunnel ([RFC 4120, avsnitt 3.1 og 5.2.7](https://datatracker.ietf.org/doc/html/rfc4120#section-3.1), [RFC 4556](https://datatracker.ietf.org/doc/html/rfc4556), [RFC 6113, avsnitt 5](https://datatracker.ietf.org/doc/html/rfc6113#section-5)).

I TGS-utvekslingen sender klienten `KRB_TGS_REQ` med TGT-en, en autentikator og ønsket tjenesteprincipal. KDC-en kontrollerer realm-policy, billettflagg, målprincipal og støttede nøkler og leverer i `KRB_TGS_REP` en tjenestebillett pluss en ny klient/tjeneste-øktnøkkel. Brukerpassordet behøves ikke igjen. En allerede eksisterende TGT kan derfor brukes for mange tjenester, til levetid, policy eller buffertilstand krever en ny initialautentisering ([RFC 4120, avsnitt 3.3](https://datatracker.ietf.org/doc/html/rfc4120#section-3.3)).

Ved AP-utvekslingen presenterer klienten `KRB_AP_REQ` for tjenesten: tjenestebilletten og en fersk autentikator beskyttet med øktnøkkelen. Tjenesten dekrypterer billetten med sin langsiktige nøkkel, kontrollerer målprincipal, tider, flagg og replay-tilstand og mottar derfra øktnøkkelen. Hvis klienten ber om gjensidig autentisering, svarer tjenesten med `KRB_AP_REP`. Først dette svaret beviser kryptografisk for klienten at motparten har tjenestenøkkelen ([RFC 4120, avsnitt 3.2 og 5.5](https://datatracker.ietf.org/doc/html/rfc4120#section-3.2)).

| Utveksling | Request | Suksessrespons | Kritiske inndata | Typisk feilgrense |
|---|---|---|---|---|
| AS | `KRB_AS_REQ` | `KRB_AS_REP` med TGT | klientprincipal, realm, pre-auth, tillatte enctypes | ukjent konto, pre-auth, tid, manglende klientnøkkel |
| TGS | `KRB_TGS_REQ` | `KRB_TGS_REP` med tjenestebillett | TGT, autentikator, SPN, flagg, målnøkler | SPN, realm-referral, policy, delegering, enctype |
| AP | `KRB_AP_REQ` | valgfritt `KRB_AP_REP` | tjenestebillett, autentikator, tjenestenøkkel, replay-buffer | feil tjenestekonto, keytab/KVNO, tid, replay |
| Feil | en av requestene | `KRB_ERROR` | feilkode pluss valgfri e-data og servertid | behold numerisk kode og berørt protokollfase |

Pre-autentisering beskytter ikke mot alle offline-angrep, men endrer forutsetningen. Kontoer uten påkrevd pre-autentisering kan levere et AS-svar der den passordbeskyttede delen kan kontrolleres offline. Tjenestebilletter kan også analyseres offline; svake passord for tjenestekontoer og RC4 forsterker denne risikoen. Den effektive beskyttelsen består av pre-autentisering, sterke tilfeldige tjenestenøkler eller gMSA, moderne enctypes, begrensede rettigheter og revisjonsdata, ikke bare av at Kerberos finnes ([RFC 6113, avsnitt 1](https://datatracker.ietf.org/doc/html/rfc6113#section-1), [RFC 8429, avsnitt 5](https://datatracker.ietf.org/doc/html/rfc8429#section-5), [Microsoft: Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)).

## Billetter, tider, flagg og transport

En billett har `authtime`, valgfritt `starttime`, `endtime` og, for fornybare billetter, `renew-till`. TGT-en er ikke et universelt tilgangstoken: Den er rettet mot TGS-en og brukes til å hente flere billetter. En tjenestebillett er rettet mot nøyaktig serverprincipalen. Levetid og fornyelse styres av KDC-policy og kan begrenses per realm eller konto. Faste verdier som «ti timer» er derfor produktstandarder, ikke en egenskap ved Kerberos-standarden ([RFC 4120, avsnitt 2.3 og 5.3](https://datatracker.ietf.org/doc/html/rfc4120#section-2.3)).

Flagg som `forwardable`, `forwarded`, `proxiable`, `proxy`, `renewable`, `initial`, `pre-authent` og `ok-as-delegate` endrer hvordan en billett kan brukes. De er ikke dekorative diagnosefelt. Et double hop kan mislykkes selv om den første tjenestebilletten er gyldig, fordi et nødvendig flagg eller en KDC-policy mangler. Omvendt utvider en videresendbar TGT virkningen av en kompromittert tjeneste. Billettflagg hører derfor til ethvert delegerings- og hendelsesbevis ([RFC 4120, avsnitt 2](https://datatracker.ietf.org/doc/html/rfc4120#section-2)).

Autentikatorer og enkelte pre-autentiseringsmetoder bruker tid til å kontrollere ferskhet. RFC 4120 lar tillatt avvik være åpen som lokal policy. Active Directory bruker fem minutter som standard; denne toleransen erstatter ikke presis tidssynkronisering. Klient, KDC og måltjeneste er avgjørende: En klient kan få en billett og likevel mislykkes hos tjenesten med `KRB_AP_ERR_SKEW` dersom klokken der avviker ([RFC 4120, avsnitt 3.2.3 og 7.5.1](https://datatracker.ietf.org/doc/html/rfc4120#section-3.2.3), [Microsoft: Kerberos troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance), [Microsoft: Windows Time Service](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service)).

Kerberos bruker port 88 over UDP og TCP. RFC 4120 krever TCP-støtte og beskriver UDP som valgfritt; for store UDP-svar kan med `KRB_ERR_RESPONSE_TOO_BIG` utløse et nytt forsøk over TCP. PAC, gruppemedlemskap og andre Authorization Data øker billettstørrelsen. En brannmur som bare tillater små UDP-tester eller blokkerer TCP 88, kan derfor skape feil som avhenger av bruker eller gruppe ([RFC 4120, avsnitt 7.2.1](https://datatracker.ietf.org/doc/html/rfc4120#section-7.2.1), [Microsoft: Kerberos KDC configuration keys](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-protocol-registry-kdc-configuration-keys)).

Tjenestebilletten er bare det kryptografiske beviset. For at HTTP, LDAP eller SMB skal kunne bruke den, bygger GSS-API, SSPI og SPNEGO Kerberos inn i det aktuelle applikasjonsprotokollet.

## Innbygging i applikasjonsprotokoller

GSS-API gir applikasjoner en mekanismeuavhengig Security Context; RFC 4121 definerer Kerberos-V5-mekanismen for dette. Windows SSPI fyller en tilsvarende rolle. SPNEGO forhandler mellom tilbudte GSS-mekanismer og vises ofte som `Negotiate` i HTTP. Det synlige headernavnet beviser ikke automatisk at Kerberos ble valgt i stedet for NTLM. Diagnosen må fange den faktisk forhandlede mekanismen og det forespurte Target Name ([RFC 4121](https://datatracker.ietf.org/doc/html/rfc4121), [RFC 4178](https://datatracker.ietf.org/doc/html/rfc4178), [Microsoft: Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview)).

SASL GSS-API binder den samme Kerberos-mekanismen inn i applikasjonsprotokoller. Dette er for eksempel mulig med [LDAP](/kb/ldap), IMAP eller SMTP når klient og server tilbyr mekanismen. Etter en vellykket GSS-kontekst kan SASL Security Layers levere integritet eller konfidensialitet. Om de faktisk brukes og hvordan de spiller sammen med TLS, er konfigurasjon i den konkrete applikasjonen; `GSSAPI` i Capability-listen alene beviser verken SPN eller Channel Protection ([RFC 4752](https://datatracker.ietf.org/doc/html/rfc4752)).

Klienten danner tjenestenavnet av Service Class og målvert. For HTTP er vanligvis `HTTP/fqdn`, for LDAP `ldap/fqdn` relevant. Alias, CNAME, omvendt oppslag, proxy, klyngenavn og vertsnavnet i URL-en kan endre den dannede identiteten. Klienten må be om billetten for den samme principalen som den mottakende tjenesten har nøkkelen til. En lastbalanserer løser ikke denne bindingen; alle backendinstanser trenger en konsistent tjenesteidentitet og nøkkelstrategi ([MIT Kerberos: Application servers – DNS](https://web.mit.edu/kerberos/krb5-latest/doc/admin/appl_servers.html#getting-dns-information-correct), [Microsoft: Service principal names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names)).

## Active Directory som Kerberos-realm

I Active Directory Domain Services er KDC-en integrert i domenekontrolleren. Bruker-, datamaskin- og tjenestekontoer er Kerberos-principaler; katalogen leverer nøkler, SPN-er, grupper, kontoflagg og policyer. `krbtgt`-kontoen representerer domenets TGS. KDC-tilgjengelighet og datakonsistens i KDC-en følger dermed DC Locator, DNS, AD-replikering, nettsteder og gjenopprettingsmodellen for AD DS, ikke en separat Kerberos-klyngeprotokoll ([Microsoft: Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview), [MS-KILE: Kerberos V5 Synopsis](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/b4af186e-b2ff-43f9-b18e-eedb366abf13), [DC Locator](/kb/ldap#active-directory-als-ldap-serverprofil)).

AD legger vanligvis et Privilege Attribute Certificate, PAC, til billetter som Authorization Data. Det kan blant annet inneholde SIDs, gruppemedlemskap, profil- og policyinformasjon samt signaturer. KDC-en oppretter og signerer disse dataene; tjenesten bruker dem for Windows-autorisering eller lar dem eventuelt validere. Kerberos-kjerneautentisering og AD-autorisering er derfor separate utsagn: En kryptografisk gyldig principal har ikke automatisk ønsket tilgang ([MS-PAC](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/166d8064-c863-41e1-9c23-edaaa5f36962), [MS-KILE: PAC Generation](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/c25d48df-67f0-4c5f-9e46-27a7d5710909)).

Nøkkelmateriale og SPN-attributter replikeres med Active Directory. Etter endringer i konto, passord eller SPN kan ulike DC-er derfor kortvarig bruke ulike tilstander. Hvis klienten for TGS og administratoren for kontrollen treffer ulike DC-er, virker en feil intermittent. Et robust bevis angir utstedende KDC, mål-DC for katalogoppslaget, KVNO, billett-enctype og replikeringsstatus. Dette gjelder særlig ved manuelle keytab-rotasjoner og tjenester på tvers av steder.

Valg av enctype er snittet av klienttilbud, KDC-policy, målkonto-nøkler og tjenestestøtte. AES-kompatibel programvare er ikke tilstrekkelig hvis kontoen ikke har tilsvarende nøkler eller `msDS-SupportedEncryptionTypes` og domenepolicy utelukker dem. RFC 8429 klassifiserer RC4 og 3DES for Kerberos som foreldet; Microsoft dokumenterer inventarisering via sikkerhetshendelsene 4768 og 4769. En utfasing begynner med måling og nøkkelgenerering, ikke med global deaktivering av én bit ([RFC 8429](https://datatracker.ietf.org/doc/html/rfc8429), [RFC 8009](https://datatracker.ietf.org/doc/html/rfc8009), [Microsoft: Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)).

For Windows-tjenester reduserer Group Managed Service Accounts, gMSA, behovet for manuell administrasjon av passord og SPN. De erstatter imidlertid ikke kontrollen av hvilken identitet prosessen faktisk kjører under, hvilke verter som får lese det administrerte passordet, og hvilke SPN-er som er registrert på kontoen. For apparater eller Unix-tjenester er en keytab ofte fortsatt nødvendig; rotasjonen må synkroniseres med AD-kontoen ([Microsoft: Service Accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-service-accounts)).

Innenfor et AD-domene er denne veien direkte. Ved tilgang på tvers av realm- eller domenegrenser kommer Cross-Realm-billetter og eventuelt flere mellomledd i tillegg.

## Klareringsforhold og Cross-Realm-baner

Et realm-klareringsforhold slår ikke sammen principal-databaser. Cross-Realm-autentisering bruker TGT-er for `krbtgt/TARGET@SOURCE` og eventuelt flere mellomrealmer. Klienten følger en billettbane til den når realmet til måltjenesten. RFC 6806 supplerer med referrals og navnekanonisering, slik de særlig brukes i AD-miljøer. Banen som er synlig i bufferen, er derfor mer utsagnskraftig enn den generelle påstanden «klareringsforholdet er grønt» ([RFC 4120, avsnitt 1.1 og 3.3.3](https://datatracker.ietf.org/doc/html/rfc4120#section-3.3.3), [RFC 6806](https://datatracker.ietf.org/doc/html/rfc6806)).

Retning og transitivitet for klareringsforhold, Selective Authentication, SID-filtrering, Name Suffix Routing og tilgjengelige enctypes begrenser hva en Cross-Realm-TGT praktisk muliggjør. SPN-er må være søkbare og entydige i riktig forest. Et lokalt `setspn -Q`-treff beviser ingen entydighet på tvers av forest; [`setspn`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn) har forest- og domenealternativer for dette. Diagnosen dokumenterer hver referral-TGT, ikke bare den siste feilen med tjenestebilletten ([Microsoft: Windows Authentication Concepts](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/windows-authentication-concepts), [Microsoft: setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn)).

En billett til frontenden gir ikke automatisk denne rett til å kontakte en backend på vegne av brukeren. Det er her double-hop-problemet begynner.

## Delegering og Double Hop

Ved normal AP-utveksling mottar en frontend ingen fritt brukbar brukernøkkel. Skal den få tilgang til en backend på vegne av brukeren, trenger den en delegeringsmodell. Forwarded-TGT- eller unconstrained delegation gir tjenesten en gjenbrukbar TGT og utvider virkningen langt utover én enkelt backend. En kompromittert frontend kan bruke den til billetter til andre tjenester; denne formen er derfor en stor utvidelse av tillit ([RFC 4120, avsnitt 2.5 og 2.6](https://datatracker.ietf.org/doc/html/rfc4120#section-2.5), [MS-SFU: Protocol Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/1fb9caca-449f-4183-8f7a-1a5fc7e7290a)).

Microsofts Service-for-User-utvidelser deler opp problemet. Med S4U2self kan en tjeneste hente en billett til seg selv på vegne av en bruker, for eksempel etter en annen frontend-autentisering. Med S4U2proxy ber den, under KDC-policy, om en billett til en annen tjeneste på vegne av denne brukeren. Klassisk constrained delegation lagrer tillatte mål på frontendkontoen; resource-based constrained delegation angir tillatte innringere på ressurskontoen. I begge tilfeller inngår mål-SPN, billettflagg, kontoinnstillinger og trustbane i avgjørelsen ([MS-SFU: Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/36d103d2-61a6-42d5-a725-74de3205cdaf), [MS-SFU: Introduction](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/8ee85a47-7526-4184-a7c5-25a5e4155d7d)).

Delegering er autorisering for videreformidling av identitet, ikke bare et kompatibilitetsvalg. Administratoren inventariserer frontendprincipal, backend-SPN-er, tillatte brukerklasser, Protocol Transition, trustgrenser og det teknisk minste målomfanget. En vellykket første hop beviser ikke den andre; en direkte backendtest i brukerens kontekst omgår omvendt delegeringsgrensen og kan gi et falskt positivt resultat.

## Driftsmodell, overvåking og gjenoppretting

Etter billettflyt og delegering kommer driftsspørsmålet: Hvilken Kerberos-komponent må være tilgjengelig, hvilket bevis viser tilstanden og hva kan gjenopprettes i en nødsituasjon? Tabellen knytter disse spørsmålene til de involverte rollene.

| Rolle | Tilstand som skal overvåkes | Ledemetrikk eller bevis | Vanlig blindflekk |
|---|---|---|---|
| Klient | realm-tilordning, KDC-valg, klokke og credential-buffer | AS-/TGS-latens, bufferlevetid, KDC og feilkode | test bruker et annet DNS-navn eller en annen brukerpåloggingsøkt |
| KDC | principal-nøkler, policy, replikering, trust og revisjon | 4768/4769/4771, feilrate per kode, enctype og utstedende DC | samlede feil uten mål-SPN og klienttilbud |
| Tjeneste | tjenestekonto, SPN, keytab/Key Store, replay-buffer og klokke | AP-suksess, KVNO, billett-enctype, målprincipal | port åpen, men prosessen kjører under en annen identitet |
| Frontend med delegering | S4U-/videresendingspolicy og backendmål | første og andre hop separat, delegeringsbane og billettflagg | direkte backendtest omgår double hop |
| Realm-/forest-drift | KDC-database, realm-nøkler, AD-replikering og gjenoppretting | replikert nøkkeltilstand, sikret konfigurasjon, testet restore | tjeneste-keytab og KDC-generasjon glir fra hverandre |

Billettbuffere er driftstilstand. En brukerprosess, tjenestekonto, container eller Windows Logon Session kan se en annen buffer enn det interaktive administratorskallet. Å slette og hente en billett på nytt er en målrettet test, men ingen reparasjon av den underliggende SPN-, nøkkel- eller replikeringsårsaken. Før purge registreres principal, mål-SPN, KDC, KVNO, enctype, flagg og tidsfelt; ellers forsvinner det beste feilbeviset.

Windows logger TGT-forespørsler under 4768, tjenestebilletter under 4769, pre-autentiseringsfeil under 4771 og ytterligere AS-feil under 4772, dersom riktige Advanced Audit Policies er aktive. Volumet er høyt på KDC-er. Overvåking trenger derfor aggregering etter Result Code, klient, måltjeneste, DC og enctype samt baseliner, i stedet for å behandle hver vellykkede TGS Request som en alarm ([Microsoft: Advanced Audit Policy Configuration](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration)).

Gjenoppretting avhenger av KDC-implementasjonen. I AD DS inngår KDC-data og `krbtgt` i modellen for System State- og forest-gjenoppretting; en vilkårlig databasefil eller en LDIF-eksport er ikke en gyldig Kerberos-sikkerhetskopi. Operatører av selvstendige MIT-realmer må sikre KDC-database, stash-/master key-materiale, konfigurasjon, ACL-er og replikering samlet. Tjeneste-keytabs må inventariseres i tillegg og etter en restore kontrolleres for konsistens mellom KVNO og nøkkel ([Microsoft: Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state), [MIT Kerberos: Backups of secure hosts](https://web.mit.edu/kerberos/krb5-latest/doc/admin/admin_commands/kdb5_util.html)).

Ved feilsøking kontrolleres veien baklengs: målnavn og SPN, eksisterende tjenestebillett, TGS-svar, TGT, realm-finning, DNS og tid.

## Diagnoseverktøy

Diagnosen begynner fra samme nettverkssone, med samme målnavn, realm, bruker- eller tjenestekontekst og samme Logon Session som applikasjonen. Testdata bruker reserverte navn. Utdata fra billetter og keytabs kan avsløre principaler og infrastruktur; nøkkelmateriale må aldri havne i billetter, chat eller prosessargumenter.

### Finn KDC og passordtjeneste via DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-Dienstsuche">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName -Type SRV "_kerberos._tcp.example.ch"
Resolve-DnsName -Type SRV "_kerberos._udp.example.ch"
Resolve-DnsName -Type SRV "_kpasswd._tcp.example.ch"</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +short SRV _kerberos._tcp.example.ch
dig +short SRV _kerberos._udp.example.ch
dig +short SRV _kpasswd._tcp.example.ch</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) viser prioritet, vekt, port og Target. Deretter må A-/AAAA-oppløsning og tilgjengelighet for hvert leverte Target kontrolleres. RFC 4120 definerer DNS-SRV-oppdagelse, men tillater lokal realm-konfigurasjon; en tom SRV-test beviser derfor bare en feil dersom den konkrete klienten bruker DNS-oppdagelse ([RFC 4120, avsnitt 7.2.3](https://datatracker.ietf.org/doc/html/rfc4120#section-7.2.3), [RFC 2782](https://datatracker.ietf.org/doc/html/rfc2782)).

### Sammenlign tid og TCP-tilgjengelighet

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-Zeit- und Portprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">w32tm /query /status
w32tm /stripchart /computer:dc1.example.ch /samples:5 /dataonly
Test-NetConnection dc1.example.ch -Port 88</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">timedatectl status
chronyc tracking
nc -vz dc1.example.ch 88</code></pre>
  </div>
</div>

[`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings#w32tm-command-line-tool) og [`timedatectl`](https://man7.org/linux/man-pages/man1/timedatectl.1.html) viser kilde og synkroniseringsstatus; [`chronyc`](https://chrony-project.org/doc/4.7/chronyc.html) supplerer med offset- og sporingsdata for Chrony. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) eller [`nc`](https://man.openbsd.org/nc) beviser bare TCP 88. UDP, KDC-protokoll, realm og pre-autentisering krever en faktisk AS-/TGS-test.

### Hent og vis billetter på nytt målrettet

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-Ticketprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">klist purge
klist get HTTP/intranet.example.ch
klist</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">kdestroy
KRB5_TRACE=/dev/stderr kinit alice@EXAMPLE.CH
kvno HTTP/intranet.example.ch@EXAMPLE.CH
klist -ef</code></pre>
  </div>
</div>

Windows-[`klist`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/klist) arbeider i konteksten til den valgte Logon Session. Under MIT Kerberos sletter [`kdestroy`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kdestroy.html), henter [`kinit`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kinit.html) og [`kvno`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html) samt viser [`klist`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/klist.html) credentials. [`KRB5_TRACE`](https://web.mit.edu/kerberos/krb5-current/doc/user/user_config/kerberos.html) gjør KDC-valg og protokollbane synlig. Før `purge` eller `kdestroy` skal en eksisterende feilbillett dokumenteres.

### Kontroller SPN og tjenestenøkkel mot hverandre

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-SPN- und Keytabprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">setspn -Q HTTP/intranet.example.ch
setspn -X -F

Get-ADUser svc-web -Properties @(
  "servicePrincipalName"
  "msDS-SupportedEncryptionTypes"
)</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">klist -kte /etc/krb5.keytab
kvno -k /etc/krb5.keytab \
  HTTP/intranet.example.ch@EXAMPLE.CH</code></pre>
  </div>
</div>

[`setspn`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn) viser AD-målkontoen og søker med `-X -F` etter duplikater i hele forestet. [`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) leser SPN-er og deklarerte enctypes; en manglende verdi har produktspesifikk fallback-semantikk og må ikke generelt tolkes som «ingen AES». MIT-[`klist`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/klist.html) viser principal, KVNO og enctype i keytaben; [`kvno -k`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html) ber om en billett og validerer den mot den angitte keytaben.

### Knytt feil til en protokollfase

| Kode eller symptom | Fase og vanlig grense | Neste bevis |
|---|---|---|
| `KDC_ERR_C_PRINCIPAL_UNKNOWN` | AS: klientprincipal ukjent i valgt realm | realm-tilordning, UPN/principal, utstedende KDC og replikering |
| `KDC_ERR_PREAUTH_FAILED` | AS: nøkkel, passord, sertifikat eller pre-auth-metode avvist | klienttid, pre-auth-type, kontonøkkel og KDC-revisjon 4771 |
| `KDC_ERR_S_PRINCIPAL_UNKNOWN` | TGS: mål-SPN ikke funnet eller ikke entydig oppløsbar | nøyaktig forespurt SPN, søk i hele forestet og målrealm |
| `KDC_ERR_ETYPE_NOSUPP` | AS/TGS: ingen felles enctype-/nøkkelsnittmengde | klienttilbud, KDC-policy, kontonøkler, keytab og 4768/4769 |
| `KRB_AP_ERR_MODIFIED` | AP: billetten passer ikke nøkkelen til tjenesten som svarer | SPN-konto, prosessidentitet, keytab, KVNO, enctype og backendnode |
| `KRB_AP_ERR_SKEW` | AS/AP: tid utenfor toleransen | klient-, KDC- og tjenestetid samt respektive tidskilde |
| `KRB_AP_ERR_TKT_EXPIRED` | AP: billett utenfor gyldighetsvinduet | buffer, `endtime`, fornyelse, klienttid og ny initialisering |
| `KDC_ERR_BADOPTION` | TGS/S4U: flagg, delegering eller policy ikke tillatt | Forwardable-flagg, frontendkonto, backend-SPN og delegeringskonfigurasjon |
| Kerberos-billett finnes, applikasjonen bruker NTLM | Mechanism Negotiation eller Target Name | SPNEGO-resultat, URL/FQDN, sone-/klientpolicy og faktisk dannet SPN |

Feilkodene er standardisert i RFC 4120; Windows legger til status- og revisjonskontekst. Automatisering bør beholde den numeriske koden, fasen, KDC-en, klientprincipalen og målprincipalen. Fritekst alene er verken stabil eller entydig. For pakkeanalyse kan [Wireshark](https://www.wireshark.org/docs/dfref/k/kerberos.html) filtrere på `kerberos`; krypterte billettdeler forblir med hensikt uleselige uten passende nøkler ([RFC 4120, avsnitt 7.5.9](https://datatracker.ietf.org/doc/html/rfc4120#section-7.5.9), [Microsoft: Kerberos troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance)).

## Teknisk historie

Kerberos oppsto tidlig på 1980-tallet i MIT Project Athena. Protokollen bygger konseptuelt på arbeider om pålitelig tredjepart av Needham og Schroeder samt Denning og Sacco. Versjonene 1 til 4 ble utviklet i Athena-miljøet; versjon 4 var den første bredt anvendte utgaven. Navnet viser til Kerberos, den flerhodede vokteren i gresk mytologi, og står for KDC-ens sentrale tillitsrolle ([RFC 4120, avsnitt 1](https://datatracker.ietf.org/doc/html/rfc4120#section-1), [MIT Kerberos Consortium: Documentation](https://kerberos.org/docs/index.html)).

Kerberos V5 fjernet begrensninger i versjon 4 knyttet til navngivning, billettlevetider, kryptografi, Cross-Realm og utvidbarhet. RFC 1510 standardiserte V5 i 1993. RFC 4120 erstattet denne spesifikasjonen i 2005 med presiseringer og en fullstendig ASN.1-beskrivelse. Protokollfamilien ble deretter utvidet modulært, blant annet med PKINIT, Pre-Authentication Framework og FAST, GSS-API, referrals samt nye AES-profiler ([RFC 1510](https://datatracker.ietf.org/doc/html/rfc1510), [RFC 4120](https://datatracker.ietf.org/doc/html/rfc4120), [RFC 4556](https://datatracker.ietf.org/doc/html/rfc4556), [RFC 6113](https://datatracker.ietf.org/doc/html/rfc6113)).

Microsoft gjorde Kerberos V5 til den sentrale domenautentiseringsprotokollen med Windows 2000 og koblet den til AD-principaler, SPN-er, PAC, SSPI, trust-referrals og delegeringsutvidelser. Samtidig forble MIT Kerberos, Heimdal og andre implementasjoner interoperable via IETF-protokoller og GSS-API. Kryptografien utviklet seg fra DES og senere RC4 til AES-profiler; RFC 8429 innledet utfasing av 3DES og RC4 i 2018. Den tekniske historien forklarer hvorfor eldre enheter, gamle tjenestekontoer og trust-nøkler fortsatt synliggjør enctype-grenser, uten at artikkelen fastslår en flyktig produktversjonsstatus ([MS-KILE](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/), [RFC 3962](https://datatracker.ietf.org/doc/html/rfc3962), [RFC 8009](https://datatracker.ietf.org/doc/html/rfc8009), [RFC 8429](https://datatracker.ietf.org/doc/html/rfc8429)).

## Kilder

- [IETF RFC 3244 – Microsoft Windows 2000 Kerberos Change Password and Set Password Protocols](https://datatracker.ietf.org/doc/html/rfc3244)
- [MIT Kerberos – Mapping Hostnames onto Kerberos Realms](https://web.mit.edu/kerberos/krb5-latest/doc/admin/princ_dns.html)
- [IANA – Kerberos service names and ports](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=kerberos)
- [RFC 4120 – The Kerberos Network Authentication Service (V5)](https://datatracker.ietf.org/doc/html/rfc4120) – arkitektur, meldinger, billetter, flagg, feilkoder, transport og sikkerhetsmodell.
- [RFC 3961, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc3961)
- [RFC 4121 – Kerberos V5 GSS-API Mechanism](https://datatracker.ietf.org/doc/html/rfc4121) – GSS-kontekst, tokens, integritet og konfidensialitet.
- [MIT Kerberos: Mapping hostnames onto realms](https://web.mit.edu/kerberos/krb5-latest/doc/admin/realm_config.html)
- [MIT Kerberos – Keytabs](https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html) – keytab-innhold, KVNO, enctype og nøkkel.
- [Microsoft – Service principal names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names) – SPN-entydighet og binding til tjenestekontoer.
- [Microsoft – setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn) – SPN-forespørsel, registrering og søk etter duplikater.
- [Microsoft – ktpass](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ktpass) – AD-/keytab-interoperabilitet.
- [MIT Kerberos – Application servers](https://web.mit.edu/kerberos/krb5-latest/doc/admin/appl_servers.html) – keytab-beskyttelse, rotasjon, klokke og DNS.
- [RFC 4556 – PKINIT](https://datatracker.ietf.org/doc/html/rfc4556) – Public-Key-Pre-Authentication.
- [RFC 6113 – Generalized Framework for Kerberos Pre-Authentication](https://datatracker.ietf.org/doc/html/rfc6113) – pre-auth-rammeverk og FAST.
- [RFC 8429 – Deprecate 3DES and RC4 in Kerberos](https://datatracker.ietf.org/doc/html/rfc8429) – IETF Best Current Practice for foreldede enctypes.
- [Microsoft – Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos) – enctype-inventar, kontonøkler og hendelser.
- [Microsoft – Kerberos authentication troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance) – Windows-feilkoder, tid, SPN og Double Hop.
- [Microsoft – Windows Time Service](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service) – tidstjeneste og Kerberos-avhengighet.
- [Microsoft – Kerberos KDC registry and protocol settings](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-protocol-registry-kdc-configuration-keys) – UDP-størrelse, TCP-fallback og KDC-diagnose.
- [RFC 4178 – SPNEGO](https://datatracker.ietf.org/doc/html/rfc4178) – forhandling av GSS-mekanismer.
- [Microsoft – Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview) – Windows-arkitektur, KDC, SSPI og gjensidig autentisering.
- [RFC 4752 – SASL GSS-API Mechanism](https://datatracker.ietf.org/doc/html/rfc4752) – innbygging i LDAP, IMAP, SMTP og andre SASL-protokoller.
- [MS-KILE – Kerberos V5 Synopsis](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/b4af186e-b2ff-43f9-b18e-eedb366abf13) – AS-, TGS- og AP-utveksling i Windows.
- [MS-PAC – Privilege Attribute Certificate](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/166d8064-c863-41e1-9c23-edaaa5f36962) – grupper, SIDs, profiler, policyer og signaturer.
- [MS-KILE – PAC Generation](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/c25d48df-67f0-4c5f-9e46-27a7d5710909) – opprettelse av PAC-Authorization-Data.
- [RFC 8009 – AES with HMAC-SHA2 for Kerberos 5](https://datatracker.ietf.org/doc/html/rfc8009) – AES-SHA2-enctypes.
- [Microsoft – Service Accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-service-accounts) – gMSA, passord- og SPN-administrasjon.
- [RFC 6806 – Kerberos Principal Name Canonicalization and Cross-Realm Referrals](https://datatracker.ietf.org/doc/html/rfc6806) – referrals og navnekanonisering.
- [Microsoft – Windows Authentication Concepts](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/windows-authentication-concepts) – klareringsforhold, Protocol Transition og constrained delegation.
- [MS-SFU – Protocol Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/1fb9caca-449f-4183-8f7a-1a5fc7e7290a) – delegeringsflyter og risikoen ved videresendte TGT-er.
- [MS-SFU – Service for User Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/36d103d2-61a6-42d5-a725-74de3205cdaf) – S4U2self og S4U2proxy.
- [MS-SFU – Introduction](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/8ee85a47-7526-4184-a7c5-25a5e4155d7d) – Protocol Transition og constrained delegation.
- [Microsoft – Advanced Audit Policy Configuration](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration) – hendelser 4768, 4769, 4771 og 4772.
- [Microsoft – Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state) – AD DS-/KDC-sikkerhetskopi.
- [MIT Kerberos – kdb5_util](https://web.mit.edu/kerberos/krb5-latest/doc/admin/admin_commands/kdb5_util.html) – KDC-database og sikkerhetskopi i MIT Kerberos.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – DNS-SRV-forespørsler i Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – DNS-SRV-forespørsler på Unix-systemer.
- [RFC 2782 – A DNS RR for specifying the location of services](https://datatracker.ietf.org/doc/html/rfc2782) – SRV-prioritet, vekt, port og Target.
- [Microsoft – w32tm](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings) – Windows-tidskilde og offsetdiagnose.
- [Linux man-pages – timedatectl(1)](https://man7.org/linux/man-pages/man1/timedatectl.1.html) – tidssynkroniseringsstatus på Linux.
- [Chrony – chronyc](https://chrony-project.org/doc/4.7/chronyc.html) – offset-, kilde- og sporingsdiagnose.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – TCP-tilkoblingsdiagnose i Windows.
- [OpenBSD – nc(1)](https://man.openbsd.org/nc) – TCP-porttest på Unix-systemer.
- [Microsoft – klist](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/klist) – Windows-billettbuffere og test av tjenestebillett.
- [MIT Kerberos – kdestroy](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kdestroy.html) – slett credential-buffer.
- [MIT Kerberos – kinit](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kinit.html) – hent TGT og pre-auth-alternativer.
- [MIT Kerberos – kvno](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html) – tjenestebillett og keytab-validering.
- [MIT Kerberos – klist](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/klist.html) – visning av credential-buffer og keytab.
- [MIT Kerberos – kerberos environment](https://web.mit.edu/kerberos/krb5-current/doc/user/user_config/kerberos.html) – `KRB5_TRACE` og buffer-/keytabvariabler.
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) – SPN- og enctype-attributter for en tjenestekonto.
- [Wireshark – Kerberos display filter reference](https://www.wireshark.org/docs/dfref/k/kerberos.html) – protokollfelt og displayfiltre.
- [MIT Kerberos Consortium – Documentation](https://kerberos.org/docs/index.html) – opprinnelse i Project Athena og versjonshistorie.
- [RFC 1510 – The Kerberos Network Authentication Service (V5)](https://datatracker.ietf.org/doc/html/rfc1510) – historisk V5-kjernespesifikasjon fra 1993.
- [MS-KILE – Kerberos Protocol Extensions](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/) – Active Directory-Kerberos og Microsoft-utvidelser.
- [RFC 3962 – AES Encryption for Kerberos 5](https://datatracker.ietf.org/doc/html/rfc3962) – AES-SHA1-profiler.
