---
title: "LDAP: protokoll, datamodell og katalogdrift"
blatt: "ldap"
description: "LDAP for administratorer: protokollstakk og BER-wireformat, DIT, Distinguished Names, skjema, bind og SASL, søk og kontroller, TLS, Active Directory, Global Catalog, replikeringsgrenser, skalering og diagnostikk."
fakten:
  - label: Navn
    wert: Lightweight Directory Access Protocol
    href: https://datatracker.ietf.org/doc/html/rfc4510
  - label: Protokollversjon
    wert: LDAPv3 · RFC 4510 til 4519
    href: https://datatracker.ietf.org/doc/html/rfc4510#section-1
  - label: Wireformat
    wert: ASN.1-strukturer, BER-kodet
    href: https://datatracker.ietf.org/doc/html/rfc4511#section-5.1
  - label: Transport
    wert: TCP; valgfritt TLS og SASL over dette
    href: https://datatracker.ietf.org/doc/html/rfc4511#section-5
  - label: Porter
    wert: 389 LDAP · 636 LDAP over TLS
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap
  - label: Datamodell
    wert: DIT med navngitte oppføringer og attributter
    href: https://datatracker.ietf.org/doc/html/rfc4512#section-2
  - label: Navn
    wert: DN av ordnede RDN-er
    href: https://datatracker.ietf.org/doc/html/rfc4514
  - label: Filtre
    wert: Prefikssyntaks i henhold til RFC 4515
    href: https://datatracker.ietf.org/doc/html/rfc4515
  - label: Bind
    wert: anonymous, simple eller SASL
    href: https://datatracker.ietf.org/doc/html/rfc4513#section-5
  - label: StartTLS
    wert: Extended Operation i en eksisterende økt
    href: https://datatracker.ietf.org/doc/html/rfc4511#section-4.14
  - label: Paginering
    wert: Simple Paged Results Control
    href: https://datatracker.ietf.org/doc/html/rfc2696
  - label: AD Global Catalog
    wert: 3268 LDAP · 3269 LDAP over TLS
    href: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1010e59a-cf64-410c-a05c-a3a6c715261a
werbung:
  - newsletter
ctaThemen:
  - active-directory-entra
  - seppmail
translationSourceHash: 7b219213aa84d6ce78de262f2cd3a5ba23a220cc1669e134bceb3a325b06de33
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T11:27:03.810Z
translationReview: required
---

# LDAP: protokoll, datamodell og katalogdrift

LDAP er det felles språket som applikasjoner bruker for å få tilgang til katalogtjenester. En klient kan bruke det til å søke etter navngitte oppføringer, lese eller endre attributter og autentisere seg mot katalogen. Protokollen definerer meldinger, operasjoner og feilkoder. Hvordan en server lagrer dataene sine, replikerer dem eller beskytter dem mot feil, er derimot opp til den enkelte implementasjonen. Active Directory Domain Services, OpenLDAP og 389 Directory Server snakker derfor LDAP uten å være den samme plattformen internt ([RFC 4510, avsnitt 1 og 2](https://datatracker.ietf.org/doc/html/rfc4510#section-1), [RFC 4511, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc4511#section-3)).

Dette skillet er avgjørende i meldingsdrift. En gateway kan kontrollere om en mottaker finnes før SMTP-aksept, løse opp grupper for en policy eller logge på en administrator. Hvis dette oppslaget svikter eller returnerer utdaterte data, er det ikke bare «LDAP-feil»: Avhengig av integrasjonen blir meldinger avvist, regler brukt feil eller pålogginger blokkert. Administratoren må derfor vite hvor på veien fra DNS-navnet til det leste attributtet forbindelsen brytes.

Forklaringen følger denne veien. Først finner klienten en server og etablerer en beskyttet økt. Deretter autentiserer den seg, utfører et søk og tolker svarene. Først når denne normale flyten er klar, kan skjema, Active Directory-særegenheter, replikering, skalering og gjenoppretting settes i riktig sammenheng.

## Protokollstakk og øktmodell

Før en applikasjon kan søke, trenger den et konkret tjenestemål. I Active Directory-miljøer leverer DNS-SRV-poster mulige Domain Controllers eller Global Catalogs; andre produkter bruker statiske FQDN-er, egen tjenesteoppdagelse eller en lastbalanserer. Dette valget bestemmer ikke bare IP-adressen, men også plassering, serverrolle og navnet sertifikatet kontrolleres mot. En porttest mot en vilkårlig tilgjengelig server besvarer derfor ikke om applikasjonen når det tiltenkte målet.

På det valgte målet oppretter LDAP en [TCP](/kb/tcp)-forbindelse. Port 389 starter som LDAP og kan bytte til en beskyttet økt med StartTLS Extended Operation. Port 636 er registrert hos IANA som `ldaps` og brukes blant annet av Active Directory for TLS som starter umiddelbart. I begge tilfeller må klienten kontrollere sertifikatkjeden og servernavnet; «kryptert» og «koblet til riktig server» er to forskjellige bevis ([RFC 4511, avsnitt 4.14 og 5](https://datatracker.ietf.org/doc/html/rfc4511#section-4.14), [IANA Service Name Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap), [MS-ADTS, Using SSL/TLS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/8e73932f-70cf-46d6-88b1-8d9f86235e81)).

Innenfor denne forbindelsen overfører LDAP ikke lesbare kommandolinjer som [SMTP](/kb/smtp). Meldingene er beskrevet som ASN.1-strukturer og kodet med Basic Encoding Rules, BER. Hver `LDAPMessage` inneholder en `messageID`, nøyaktig én operasjon og eventuelt kontroller. Via meldings-ID-en kan en langvarig forbindelse skille mellom flere pågående operasjoner; svarene deres trenger ikke å ankomme i forespørselsrekkefølge. En vellykket TCP-handshake sier dermed ingenting om BER-dekoding, bind eller et fullført søk ([RFC 4511, avsnitt 3.1, 4.1.1 og 5.1](https://datatracker.ietf.org/doc/html/rfc4511#section-4.1.1)).

| Lag | Standardisert innhold | Administrativt relevant observasjon |
|---|---|---|
| Applikasjon | Bind, Search, Compare, Modify, Add, Delete, ModifyDN, Extended Operations og Controls | Result Code, `diagnosticMessage`, Entries, References og Controls |
| Koding | ASN.1-datatyper i BER | Dekoderfeil, maksimal forespørselsstørrelse, Message-ID og OID-er |
| Sikkerhet | TLS samt SASL-mekanismer og deres Security Layer | Sertifikatnavn, Trust Chain, bind-metode, Signing, Channel Binding |
| Transport | Langvarig TCP-forbindelse | DNS-mål, port, tilkoblingslatens, resets, Idle Timeout og pooltilstand |
| Serverinternt | DIT, skjema, ACL, indeks, lagring og replikering | Ikke standardisert av LDAP; produkt- og topologispesifikt |

For endringer gjelder en viktig grense: En enkelt LDAP-operasjon er atomær innenfor sitt omfang, men flere oppføringer danner ingen felles kjernprotokolltransaksjon. RFC 5805 beskriver en eksperimentell transaksjonsutvidelse, hvis støtte klienten må oppdage på Root DSE. Selv da må det kontrolleres hvordan replikaer ser endringen. Klargjøringsprosesser trenger derfor egne regler for gjentakelse, delfeil og avstemming, i stedet for en stilltiende antatt databasetransaksjon ([RFC 4511, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc4511#section-3), [RFC 5805, avsnitt 1 og 3](https://datatracker.ietf.org/doc/html/rfc5805#section-1)).

## Datamodell: DIT, oppføring, attributt og skjema

Etter at økten er opprettet, må klienten kunne angi hvor og hva den søker etter. LDAP organiserer katalogdataene som et Directory Information Tree, kort DIT. Hver oppføring har et unikt Distinguished Name og attributter. Skjemaet beskriver hvilke attributter som finnes, hvordan verdiene deres sammenlignes, og hvilke Object Classes de krever eller tillater. Uten denne modellen er søkebase, filter og resultater bare strenger uten pålitelig betydning ([RFC 4512, avsnitt 2 og 3](https://datatracker.ietf.org/doc/html/rfc4512#section-2), [RFC 4511, avsnitt 4.1.7](https://datatracker.ietf.org/doc/html/rfc4511#section-4.1.7)).

Distinguished Name danner banen til en oppføring i treet. Ved `cn=Mail Gateway,ou=Services,dc=example,dc=ch` betegner `cn=Mail Gateway` den lokale Relative Distinguished Name; de påfølgende RDN-ene leder via beholderen til navneroten. Siden RDN-er kan være flerverdige og tegn som komma, pluss eller omvendt skråstrek escapes, må programvare ikke behandle en DN ved enkel splitting på komma. Den trenger en RFC-4514-kompatibel parser ([RFC 4512, avsnitt 2.3](https://datatracker.ietf.org/doc/html/rfc4512#section-2.3), [RFC 4514, avsnitt 2 og 3](https://datatracker.ietf.org/doc/html/rfc4514#section-2)).

```text
dn: cn=Mail Gateway,ou=Services,dc=example,dc=ch
objectClass: top
objectClass: person
objectClass: organizationalPerson
cn: Mail Gateway
sn: Gateway
mail: mail-gateway@example.ch
```

For eksport og import finnes LDIF som en standardisert tekstrepresentasjon. LDIF representerer oppføringer eller endringsposter, men er ikke wireformatet for den aktive LDAP-økten. Linjebryting, Base64-verdier og Change Records følger egne regler. Fremfor alt inneholder en eksport bare det serveren og tillatelsene gjør synlig; operative attributter, ACL-er eller backendtilstand kan mangle. En LDIF-dump er derfor et datauttrekk, men ikke automatisk en gjenopprettbar serversikkerhetskopi ([RFC 2849, avsnitt 2 og 4](https://datatracker.ietf.org/doc/html/rfc2849#section-2)).

Betydningen av en attributtverdi kommer først fra skjemaet. En Matching Rule som `caseIgnoreMatch`, `integerMatch` eller en DN-sammenligning avgjør om to verdier er like, og hvilke filtre som fungerer på dem. På wire-nivå vises verdier først som Octet Strings; syntaks og attributttype gir dem tolkning. Egne skjemaelementer trenger derfor varig unike OID-er, definerte syntakser og Matching Rules, samt en utrulling som tar hensyn til servere og alle avhengige klienter samlet ([RFC 4512, avsnitt 4](https://datatracker.ietf.org/doc/html/rfc4512#section-4), [RFC 4517](https://datatracker.ietf.org/doc/html/rfc4517), [RFC 4520](https://datatracker.ietf.org/doc/html/rfc4520)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-ldap.svg?v=20260813" title="Interaktive Infografik: LDAP-Protokollstack, Nachrichtenschicht, DIT, Serverarchitektur und Admin-Diagnosepunkte" loading="lazy">
  <a href="/images/kb-interaktiv-ldap.svg?v=20260813">Åpne infografikk om LDAP-protokoll og katalogarkitektur</a>
</iframe>

## Operasjoner og tilstandsendringer

Med transport, navn og skjema på plass står grunnstrukturen klar; nå begynner den egentlige protokolldialogen. Den første avgjørende tilstandsendringen er vanligvis `Bind`. Den fastsetter hvilken identitet og hvilke avledede rettigheter de påfølgende operasjonene kjører med. En ny bind erstatter denne tilstanden. Så lenge serveren behandler en bind, kan klienten ikke starte andre operasjoner på samme forbindelse ([RFC 4511, avsnitt 3.1 og 4.2](https://datatracker.ietf.org/doc/html/rfc4511#section-4.2)).

Etter vellykket bind kan klienten lese eller skrive. Et søk leverer ikke ett enkelt stort svar, men null eller flere `SearchResultEntry`-meldinger, eventuelt References og til slutt nøyaktig ett `SearchResultDone`. Først dette sluttresultatet viser om sekvensen var komplett. `Modify`, `Add`, `Delete` og `ModifyDN` endrer oppføringer; `Compare` kontrollerer en attributtverdi etter Matching Rule uten å returnere et ordinært søkeresultat ([RFC 4511, avsnitt 4.5 til 4.9](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5)).

Også øktavslutningen har en klar semantikk. `Unbind` er en ensidig forespørsel om å lukke og har ikke noe svar. `Abandon` ber serveren om å avbryte en bestemt operasjon, men garanterer ikke dette avbruddet. Hvis TCP i stedet brytes, forsvinner alle pågående operasjoner. Ved en skriveoperasjon kan klienten da ikke vite sikkert om endringen trådte i kraft før eller etter forbindelsestapet; en retry krever derfor først en tilstandsavstemming i stedet for blind gjentakelse ([RFC 4511, avsnitt 4.3 og 4.11](https://datatracker.ietf.org/doc/html/rfc4511#section-4.3)).

| Operasjon | Typisk bruk | Grense klienten må håndtere |
|---|---|---|
| Bind | Tjenestekonto, brukerkontroll eller SASL-autentisering | Vellykket TCP-/TLS-forbindelse er ennå ikke en vellykket bind |
| Search | Mottakere, grupper, adresser, policyer og Root DSE leses | Flere Entries, References, grenser, Controls og avsluttende resultat |
| Compare | Kontrollere en kjent attributtverdi på serversiden | Resultatet er `compareTrue` eller `compareFalse`, ikke Search-resultat |
| Modify/Add/Delete/ModifyDN | Klargjøring og livssyklus | Atomær per operasjon, men uten kjernprotokolltransaksjon på tvers av flere Entries |
| Extended Operation | StartTLS, Password Modify eller leverandørspesifikke funksjoner | Kontroller OID og støtte på målserveren |
| Controls | Paging, sortering, Assertion, Sync eller leverandørfunksjon | Ukjent kritisk Control må føre til feil |

Controls og Extended Operations supplerer denne flyten uten å innføre en ny LDAP-versjon. Hver Control har en OID, en Criticality og eventuelt en BER-kodet verdi. Hvis en klient markerer en ukjent eller ikke-eksekverbar Control som kritisk, må operasjonen feile med `unavailableCriticalExtension`; ellers kan serveren ignorere den. Før Paging, Sync eller en leverandørfunksjon leser en korrekt klient derfor blant annet `supportedControl`, `supportedExtension`, `supportedFeatures`, `supportedLDAPVersion` og `supportedSASLMechanisms` på Root DSE ([RFC 4511, avsnitt 4.1.11](https://datatracker.ietf.org/doc/html/rfc4511#section-4.1.11), [RFC 4512, avsnitt 5.1](https://datatracker.ietf.org/doc/html/rfc4512#section-5.1)).

## Bind, SASL og TLS-tillitsgrenser

Bind avgjør hvem serveren tilordner det neste søket til. Ved Simple Bind må tre tilfeller skilles: Tom DN og tomt passord gir anonym tilgang. En ikke-tom DN med tomt passord er en *unauthenticated Bind* og bekrefter uttrykkelig ikke den oppgitte identiteten, selv om serveren kan returnere `success`. Først en ikke-tom DN med ikke-tomt passord utgjør vanlig navn/passord-autentisering. Klienter bør derfor avvise tomme passord før forespørselen; servere bør ikke utilsiktet tillate unauthenticated Binds ([RFC 4513, avsnitt 5.1.1 til 5.1.3 og 6.3.1](https://datatracker.ietf.org/doc/html/rfc4513#section-5.1)).

Ved denne passordautentiseringen kjenner serveren den presenterte hemmeligheten. Transporten må derfor ikke bare være kryptert, men også autentisert. Dette omfatter en gyldig sertifikatkjede og kontroll av om det konfigurerte DNS-navnet finnes i sertifikatet. Den som aksepterer alle sertifikater eller bruker en IP-adresse, kan opprette en kryptert kanal til feil motpart. Den samme kontrollen gjelder for StartTLS og LDAP med umiddelbar TLS ([RFC 4513, avsnitt 3.1 og 5.1.3](https://datatracker.ietf.org/doc/html/rfc4513#section-3.1), [RFC 9525, avsnitt 2 og 4](https://datatracker.ietf.org/doc/html/rfc9525#section-4), [TLS](/kb/tls)).

SASL tillater ulike autentiseringsmekanismer i stedet for en ren passord-bind og kan i tillegg forhandle fram beskyttelse for påfølgende LDAP-meldinger. I Active Directory finnes særlig Negotiate, Kerberos og NTLM. LDAP Signing beskytter der integriteten til bestemte SASL-økter; Channel Binding knytter autentiseringen til den underliggende TLS-forbindelsen. TLS, Signing og Channel Binding løser dermed beslektede, men ikke identiske problemer. En test må gjenspeile produktets faktiske bind-type ([RFC 4513, avsnitt 5.2](https://datatracker.ietf.org/doc/html/rfc4513#section-5.2), [Microsoft: LDAP signing](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing), [MS-ADTS, Channel Binding](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/2c3ea153-eff9-46a7-8614-19b677efa4e0)).

For å innføre strengere AD-policyer er det derfor ikke nok å se på Windows-versjonen. Nye AD DS-distribusjoner på Windows Server 2025 krever LDAP Signing som standard, mens oppgraderinger overtar eksisterende innstillinger. Microsoft oppgir Directory Service-hendelsene 2886 til 2889 for Signing og 3039 til 3041 for Channel Binding. Disse revisjonsdataene viser hvilke klienter, porter og bind-metoder som faktisk ville bli berørt; først deretter kan håndheving planlegges på et solid datagrunnlag ([Microsoft: LDAP signing, Default Security Behavior og Event Monitoring](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing), [Microsoft: LDAP session security after ADV190023](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/ldap-session-security-settings-requirements-adv190023)).

## Search: base, scope, filter og attributtprojeksjon

Etter sikker bind følger operasjonen de fleste integrasjoner er avhengige av: Search. Forespørselen angir en Base DN, scope, aliashåndtering, egne size- og time-limits, et filter og ønskede attributter. `baseObject` leser bare baseoppføringen, `singleLevel` dens direkte barn og `wholeSubtree` hele undertreet inkludert basen. Serveren kan sette strengere grenser. Null treff med `success` er et gyldig svar; `noSuchObject` betyr derimot at søkebasen mangler eller ikke er synlig for denne identiteten ([RFC 4511, avsnitt 4.5.1 og 4.5.2](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5.1)).

Filteret beskriver ikke fri SQL-logikk, men et tre i prefiksnotasjon. `(&(objectClass=person)(mail=*@example.ch))` kombinerer for eksempel et Equality-uttrykk med et Substring-uttrykk. `|` står for OR, `!` for NOT, `=*` for Presence og `:=` for et Extensible Match. Om en sammenligning bruker store/små bokstaver, tallrekkefølge eller DN-semantikk, avgjøres av Matching Rule for det aktuelle attributtet ([RFC 4511, avsnitt 4.5.1](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5.1), [RFC 4515](https://datatracker.ietf.org/doc/html/rfc4515), [RFC 4517](https://datatracker.ietf.org/doc/html/rfc4517)).

Dermed blir generering av filtre en sikkerhetsoppgave. Verdier fra brukerinndata må kodes etter RFC 4515; særlig `*`, parenteser, omvendt skråstrek, NUL og ugyldige UTF-8-oktetts må ikke komme rått inn i uttrykket. Strengkonkatenering kan ellers endre filterstrukturen og muliggjøre LDAP-injeksjon. DN-escaping etter RFC 4514 følger andre regler og erstatter ikke filterkoding ([RFC 4515, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc4515#section-3), [RFC 4514, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc4514#section-3)).

Ved siden av filteret bestemmer attributtlisten hvor mye serveren returnerer. En tom liste ber om alle vanlige brukerattributter, `1.1` ingen attributter, `*` alle brukerattributter og `+` etter RFC 3673 alle operative attributter. ACL-er kan fortsatt skjule verdier. Produksjonsklienter bør bare be om nødvendige attributter: Store flerverdier belaster nettverk, dekoder og minne og kan utløse egne servergrenser ([RFC 4511, avsnitt 4.5.1.8](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5.1.8), [RFC 3673](https://datatracker.ietf.org/doc/html/rfc3673)).

### Paging, sortering og resultater som endrer seg

Større treffmengder overføres vanligvis med Simple Paged Results Control. Serveren legger ved en ugjennomsiktig cookie for hver side, som klienten sender tilbake sammen med samme forespørsel. Denne cookien er ikke en offset eller en varig cursor. Hvis kataloginnholdet endrer seg underveis, kan oppføringer mangle eller forekomme dobbelt. Paging begrenser dermed datamengden per svar, men skaper ikke et konsistent snapshot ([RFC 2696, avsnitt 2 og 3](https://datatracker.ietf.org/doc/html/rfc2696#section-2)).

Active Directory synliggjør dette skillet i praksis: LDAP-policyen `MaxPageSize` begrenser upaginerte resultater som standard til 1000 objekter. En import som mottar nøyaktig 1000 oppføringer, har derfor ikke bevist at den er fullstendig. Klienten må behandle sider og cookies korrekt og oppdage avbrudd. Andre policyer begrenser spørrevarighet, Receive Buffer og samtidig holdte Result Sets. I drift må derfor Page Size, antall sider, siste cookie-fremdrift, timeout og gjenoppstart loggføres ([MS-ADTS, LDAP Policies](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/3f0137a1-63df-400c-bf97-e1040f055a99), [Microsoft: Paging Search Results](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ldap/paging-search-results)).

## Active Directory som LDAP-serverprofil

Reglene hittil gjelder LDAP generelt. Active Directory Domain Services er en konkret serverimplementasjon med ekstra roller og konvensjoner. Dataene er fordelt på Naming Contexts; en Domain Controller har minst Schema, Configuration og sin egen Domain Naming Context. Root DSE har tom DN og oppgir blant annet `defaultNamingContext`, `configurationNamingContext`, `schemaNamingContext`, alle `namingContexts`, servernavnet og støttede mekanismer. Etter TCP og TLS er denne oppføringen den første testen som faktisk sier noe om den nådde katalogtjenesten ([RFC 4512, avsnitt 5.1](https://datatracker.ietf.org/doc/html/rfc4512#section-5.1), [Microsoft RootDSE](https://learn.microsoft.com/en-us/windows/win32/adschema/rootdse), [MS-ADTS, rootDSE Attributes](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/96f7b086-1ca3-4764-9a08-33f8f7a543db)).

Valget mellom Domain Controller og Global Catalog endrer søkeresultatet. En DC betjener LDAP på 389 eller 636 og kjenner full Domain Naming Context for sitt domene. Global Catalog bruker i tillegg 3268 eller 3269 og har en partiell replika fra alle domener i skogen. Den kan finne objekter på tvers av skogen, men for eksterne domener leverer den bare attributter fra Partial Attribute Set. Et vellykket treff beviser derfor ikke at attributtet applikasjonen trenger, finnes ([MS-ADTS, Ports](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1010e59a-cf64-410c-a05c-a3a6c715261a), [Microsoft: Searching the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/searching-global-catalog-contents), [Microsoft: Attributes included in the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/attributes-included-in-the-global-catalog)).

Også filtre kan bli AD-spesifikke. Matching Rule `1.2.840.113556.1.4.1941`, `LDAP_MATCHING_RULE_TRANSITIVE_EVAL`, følger for eksempel koblede attributter og kan evaluere nestede grupper. Støtten står ikke bare i `supportedControl`. Dessuten inneholder `memberOf` ikke Primary Group. En autorisasjonsbeslutning basert på gruppemedlemskap må derfor uttrykkelig ta hensyn til gruppenesting, Primary Group, gruppescope, ACL-synlighet og replikeringsstatus ([MS-ADTS, LDAP Matching Rules](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/4e638665-f466-4597-93c4-12f2ebfabab5), [Microsoft: Primary group membership](https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/user-prov-sync/exclude-user-primary-group)).

Hvilken server som besvarer disse forespørslene, avgjøres for Windows-klienter av DC Locator sammen med [DNS](/kb/dns)-SRV-poster. Steds- og rollebaserte poster gir kandidater med prioritet og vekt. En statisk angitt IP omgår dette valget og vanskeliggjør sertifikatkontroll. En enkel TCP-lastbalanserer fordeler riktignok forbindelser, men kjenner uten ekstra logikk verken skrivbare DC-er eller Global Catalogs, Naming Contexts eller replikeringsstatus ([Microsoft: DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator), [Microsoft: Verify LDAP SRV records](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created)).

Til slutt må LDAP-tilgjengelighet ikke forveksles med sunn replikering. AD replikerer katalogendringer over Directory Replication Service Remote Protocol; OpenLDAPs `syncrepl` bruker derimot LDAP Content Synchronization med Provider, Consumer og cookies. Et testsøk kan vise at en bestemt server svarer. Om alle servere har de samme endringene og tar igjen etter en feil, må kontrolleres med verktøyene for den aktuelle plattformen ([MS-DRSR, Relationship to Other Protocols](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/0597aff6-0177-4d52-99f2-14a5441bc3c1), [RFC 4533](https://datatracker.ietf.org/doc/html/rfc4533), [OpenLDAP Administrator's Guide: Replication](https://www.openldap.org/doc/admin25/replication.html)).

## Integrasjons- og driftsmodeller

For drift er det nå mindre viktig at et produkt «støtter LDAP», enn hvordan det bruker LDAP. Ved et **oppslag i kjøretid** venter en melding eller økt direkte på Search og serversvar. Ved **Credential Check** søker en teknisk konto først etter brukerens DN og utfører deretter en andre bind med det oppgitte passordet. En **import eller cache** leser derimot mange oppføringer og arbeider med en lokal kopi til neste kjøring. Disse mønstrene har ulike konsekvenser for latens, passordbehandling, failover og dataalder; produktdokumentasjonen må angi den konkrete atferden ([RFC 4511, avsnitt 4.2 og 4.5](https://datatracker.ietf.org/doc/html/rfc4511#section-4.2), [RFC 2696](https://datatracker.ietf.org/doc/html/rfc2696)).

| Mønster | Kritisk bane | Umiddelbart driftsbevis |
|---|---|---|
| Oppslag i kjøretid | DNS, Connect, TLS, pool, Bind, Search og serversvar per hendelse | p50/p95/p99 per operasjon, poolmetning, Result Codes, fallbackmål |
| Credential Check | Brukersøk pluss andre bind med brukerpassord | DN-oppløsning, Empty-Password-sperre, TLS-navnekontroll, lockout-atferd |
| Periodisk import | Fullstendig, paginert enumerering og commit i lokal cache | Page-/cookie-fremdrift, objektantall, slettemodell, siste vellykkede commit |
| Change Sync | Leverandørspesifikk eller LDAP Sync-cursor | Cursorpersistens, replay, resync og slettede objekter |

Uavhengig av mønsteret trenger klienten separate tidsavbrudd for Connect, Bind, Operation og Idle. En Connection Pool sparer oppsett av TCP, TLS og Bind, men bærer med seg forbindelsens autentiseringstilstand. Døde økter må oppdages, og en forbindelse må ikke utilsiktet skifte mellom brukere eller leietakere. Failover trenger en sporbar målrekkefølge, begrensede gjentakelser og en vei tilbake til foretrukket mål. Ellers mangedobler parallelle retries lasten nettopp under et katalogutfall ([RFC 4511, avsnitt 3.1, 4.2 og 5.3](https://datatracker.ietf.org/doc/html/rfc4511#section-3.1)).

På serversiden bestemmer Base DN, Scope, Filter og attributtliste arbeidet. En selektiv likhetsbetingelse på et indeksert attributt er noe annet enn en innledende substring eller et stort OR-uttrykk. LDAP publiserer ingen eksekveringsplan og foreskriver ingen indeksteknikk. Administratoren må derfor korrelere de faktiske produktfiltrene med resultatmengde, p95-/p99-latens og servermetrikker. Et raskt søk etter én testkonto beviser ikke at en mottakerkontroll skalerer under toppbelastning ([OpenLDAP Administrator's Guide: Performance Tuning](https://www.openldap.org/doc/admin26/tuning.html), [MS-ADTS: LDAP Policies](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/3f0137a1-63df-400c-bf97-e1040f055a99)).

Overvåking bør dele prosessen inn i de samme trinnene som feilsøkingen: DNS-valg, TCP- og TLS-oppsett, Bind, Search-latens, Result Code, treffantall og paging-fremdrift. I tillegg kommer poolbelegg, retryrate og status for import eller synkronisering. En enkelt syntetisk bind kan bekrefte tilgjengelighet, men oppdager verken manglende attributter, en ufullstendig import eller en replikeringspartner som ligger etter.

For backup og recovery er heller ikke det synlige kataloginnholdet tilstrekkelig. Skjema, ACL-er, backend- og serverkonfigurasjon, nøkler og sertifikater, replikeringsidentiteter samt prosedyren for å ta en gjenopprettet node inn i topologien igjen må sikres. Active Directory bruker System State og egne Forest Recovery-trinn til dette; OpenLDAP avhenger av sin backend. Håndboken skiller for eksempel mellom en LMDB-sikkerhetskopi og `slapcat` og påpeker semantisk inkonsistente LDIF-tilstander ved endringer i flere deler. LDAP definerer ingen backupmekanisme ([Microsoft: Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state), [OpenLDAP Administrator's Guide: Directory Backups](https://www.openldap.org/doc/admin26/maintenance.html)).

En gjenopprettingstest er først fullført når en klient finner den gjenopprettede tjenesten via det tiltenkte DNS-navnet, TLS og Bind lykkes, Root DSE og skjemaet stemmer, reelle søk leverer fullstendige attributter og replikeringen starter kontrollert igjen. Dermed fører recovery tilbake til artikkelens begynnelse: Hele banen teller, ikke bare en startet database.

## Diagnoseverktøy

Diagnosen følger samme bane som en produksjonsforespørsel. Den starter i nettverket til den berørte applikasjonen og bruker dens DNS-navn, Truststore, bind-metode, Base DN, filter og attributtliste. En test fra administratorens bærbare PC kan ellers lykkes mens gatewayen fortsatt bruker en annen DC, en annen CA eller et annet scope. Eksemplene bruker reserverte navn og leser bare metadata; bind-passord hører verken hjemme i shellhistorikken eller i prosessargumenter. `ldapsearch -W` ber om dem interaktivt.

### Finn tjenestemål via DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-Dienstsuche">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName _ldap._tcp.dc._msdcs.example.ch -Type SRV -DnsOnly
Resolve-DnsName _ldap._tcp.gc._msdcs.example.ch -Type SRV -DnsOnly</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +noall +answer SRV _ldap._tcp.dc._msdcs.example.ch
dig +noall +answer SRV _ldap._tcp.gc._msdcs.example.ch</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) viser mål, porter, prioriteter og vekter. Deretter må A-/AAAA-oppløsning, stedstilknytning og tilgjengelighet for hvert faktisk valgbare mål kontrolleres. Én tilgjengelig DC retter ikke et feilaktig SRV-sett ([Microsoft: DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator)).

### Kontroller TCP og implisitt TLS på port 636

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection dc1.example.ch -Port 636 -InformationLevel Detailed

$tcp = [Net.Sockets.TcpClient]::new("dc1.example.ch", 636)
$tls = [Net.Security.SslStream]::new($tcp.GetStream(), $false)
$tls.AuthenticateAsClient("dc1.example.ch")
$tls.SslProtocol
$tls.RemoteCertificate.Subject
$tls.RemoteCertificate.GetExpirationDateString()
$tls.Dispose(); $tcp.Dispose()</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl s_client \
  -connect dc1.example.ch:636 \
  -servername dc1.example.ch \
  -verify_hostname dc1.example.ch \
  -verify_return_error -brief</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) beviser først bare TCP-tilkoblingen. Den påfølgende [.NET `SslStream`](https://learn.microsoft.com/dotnet/api/system.net.security.sslstream) eller [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) kontrollerer TLS med det konfigurerte DNS-navnet. `s_client -showcerts` viser bare sertifikatene serveren sender og er i seg selv ikke bevis på en vellykket Chain- eller Hostname-kontroll. StartTLS på 389 kan kontrolleres separat under Unix med `openssl s_client -starttls ldap` ([OpenSSL `s_client`](https://docs.openssl.org/master/man1/openssl-s_client/), [RFC 4511, avsnitt 4.14](https://datatracker.ietf.org/doc/html/rfc4511#section-4.14)).

### Les Root DSE og funksjoner

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-Root-DSE-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ADRootDSE -Server dc1.example.ch -Properties @(
  "defaultNamingContext"
  "namingContexts"
  "supportedLDAPVersion"
  "supportedControl"
  "supportedSASLMechanisms"
)</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ldapsearch -LLL -x -ZZ -H ldap://dc1.example.ch \
  -s base -b "" \
  defaultNamingContext namingContexts supportedLDAPVersion \
  supportedControl supportedSASLMechanisms</code></pre>
  </div>
</div>

[`Get-ADRootDSE`](https://learn.microsoft.com/powershell/module/activedirectory/get-adrootdse?view=windowsserver2025-ps) bruker her ActiveDirectory-modulen og som standard den påloggede Windows-identiteten. [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch&sektion=1&apropos=0&manpath=OpenLDAP+2.6-Release) tvinger med `-ZZ` frem vellykket StartTLS og leser anonymt bare Root DSE-attributtene serveren har frigitt. En manglende OID beviser at akkurat dette målet ikke publiserer funksjonen; den sier ingenting om andre clusternoder.

### Gjenskap reelt søk med scope, filter og paging

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-Suchprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$params = @{
  Server         = "dc1.example.ch"
  SearchBase     = "OU=People,DC=example,DC=ch"
  SearchScope    = "Subtree"
  LDAPFilter     = "(&(objectClass=user)(mail=admin@example.ch))"
  Properties     = @("mail", "proxyAddresses", "memberOf")
  ResultPageSize = 500
}

Get-ADUser @params</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ldapsearch -LLL -x -ZZ -H ldap://dc1.example.ch \
  -D "CN=svc-lookup,OU=Services,DC=example,DC=ch" -W \
  -b "OU=People,DC=example,DC=ch" -s sub \
  -E pr=500/noprompt \
  "(&(objectClass=user)(mail=admin@example.ch))" \
  mail proxyAddresses memberOf</code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser?view=windowsserver2025-ps) godtar med `-LDAPFilter` den RFC-nære filtersyntaksen og utfører paging via `-ResultPageSize` . [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch&sektion=1&apropos=0&manpath=OpenLDAP+2.6-Release) bruker `-E pr=500/noprompt` for Paged Results Control og `-W` for interaktiv passordspørring. Testen må, i tillegg til treffet, dokumentere avsluttende resultat, antall sider, returnerte attributter og kjøretid.

### Tilordne feil til en grense

| Observasjon | Protokollbetydning | Neste solide bevis |
|---|---|---|
| Timeout før TLS | Måloppløsning, ruting, brannmur, listener eller uttømt pool | SRV/A/AAAA, TCP-handshake, serverlistener og tilkoblingslatens |
| Sertifikatfeil | Chain, gyldighet, navn eller klienttillit stemmer ikke | Sendt Chain, Trust Anchor, SAN mot nøyaktig konfigurert FQDN |
| `strongAuthRequired` / `confidentialityRequired` | Serveren krever sterkere bind- eller beskyttelsesmetode | Port, StartTLS-suksess, SASL-mekanisme, Signing-/CBT-policy |
| `invalidCredentials` | Presentert bind-identitet eller credentials avvist | Nøyaktig bind-type og DN; ingen passordlogging |
| `invalidDNSyntax` | DN syntaktisk ugyldig | RFC-4514-koding og faktisk DN fra Search Result |
| `noSuchObject` med `matchedDN` | Base DN mangler eller er usynlig fra en overordnet node | Root DSE, Naming Context, ACL-synlighet og `matchedDN` |
| `sizeLimitExceeded` | Klient- eller servergrense før fullstendig resultat | Paging-Control, Page-Cookies, LDAP Policy og totalt antall |
| `adminLimitExceeded` / `busy` / `unavailable` | Serverressurs eller administrativ grense | Servermetrikker, Query Policy, filterkostnad, retryrate og målnode |
| Null treff ved `success` | Gyldig søk uten synlig match | Sammenlign base, scope, filter, ACL, målnode og replikeringsstatus |

Numeriske Result Codes hører til LDAP-protokollen; `diagnosticMessage` og ekstra AD-subkoder er derimot implementasjonsspesifikk kontekst. Automatisering bør derfor først evaluere Result Code og loggføre teksten som tillegg. For `busy` og `unavailable` trenger hver klient et begrenset retrybudsjett med backoff. Ubegrensede gjentakelser gjør ett enkelt katalogproblem til en lasttopp i alle avhengige systemer ([RFC 4511, avsnitt 4.1.9 og vedlegg A](https://datatracker.ietf.org/doc/html/rfc4511#appendix-A)).

## Teknisk historie

LDAP oppsto ikke som en uavhengig katalogdatabase. X.500 hadde på slutten av 1980-årene definert en omfattende katalogmodell og Directory Access Protocol. RFC 1487 beskrev i 1993 en lettere tilgang til denne modellen; RFC 1777 fulgte i 1995 som LDAP Version 2. «Lightweight» viste til den forenklede protokolltilgangen sammenlignet med DAP, ikke til små kataloger eller liten driftsmessig betydning. Den tidlige utviklingen er tett knyttet til Tim Howes og University of Michigan ([RFC 1487](https://datatracker.ietf.org/doc/html/rfc1487), [RFC 1777](https://datatracker.ietf.org/doc/html/rfc1777)).

LDAPv3 ble publisert i 1997 med RFC 2251 og ledsagende dokumenter. Utvidbare operasjoner, Controls, SASL, internasjonalisering og den reviderte datamodellen gjorde det til grunnlaget for dagens implementasjoner. LDAPbis-arbeidet omorganiserte denne statusen i 2006: RFC 4510 fungerer som roadmap, RFC 4511 beskriver protokollen, RFC 4512 informasjonsmodellen og RFC 4513 sikkerheten; RFC 4514 til 4519 utfyller representasjoner, URL-er, syntakser og skjema ([RFC 2251](https://datatracker.ietf.org/doc/html/rfc2251), [RFC 4510, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc4510#section-3)).

Parallelt utviklet det seg svært ulike servere. OpenLDAP oppsto i 1998 fra University of Michigan-implementasjonen og videreførte `slapd`, biblioteker og verktøy som et åpen kildekode-prosjekt. Active Directory brakte med Windows 2000 en LDAPv3-profil med eget skjema, Naming Contexts, Controls, Matching Rules og separat replikeringsprotokoll inn i bred virksomhetsbruk ([OpenLDAP Release Road Map](https://www.openldap.org/software/roadmap.html), [OpenLDAP Administrator's Guide – Preface](https://www.openldap.org/doc/admin26/preface.html), [MS-ADTS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/)).

Denne historien forklarer den viktigste driftsregelen: LDAP standardiserer tilgangen, ikke den interne arkitekturen. Den som flytter en klient fra OpenLDAP til AD DS eller mellom to appliances, må derfor kontrollere mer enn host, port og Bind-DN. Skjema, Controls, grenser, gruppeoppløsning, replikering og recovery forblir produktegenskaper.

## Kilder

- [RFC 4510, avsnitt 1 og 2](https://datatracker.ietf.org/doc/html/rfc4510)
- [RFC 4511 – LDAP: The Protocol](https://datatracker.ietf.org/doc/html/rfc4511) – meldingslag, operasjoner, BER, TCP, StartTLS og Result Codes.
- [IANA – Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap) – `ldap` 389 og `ldaps` 636.
- [MS-ADTS – Using SSL/TLS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/8e73932f-70cf-46d6-88b1-8d9f86235e81) – implisitt TLS og StartTLS i Active Directory.
- [RFC 5805, avsnitt 1 og 3](https://datatracker.ietf.org/doc/html/rfc5805)
- [RFC 4512 – LDAP Directory Information Models](https://datatracker.ietf.org/doc/html/rfc4512) – DIT, Entries, attributter, skjema, Root DSE og Subschema.
- [RFC 4514, avsnitt 2 og 3](https://datatracker.ietf.org/doc/html/rfc4514)
- [RFC 2849, avsnitt 2 og 4](https://datatracker.ietf.org/doc/html/rfc2849)
- [RFC 4517 – LDAP Syntaxes and Matching Rules](https://datatracker.ietf.org/doc/html/rfc4517) – standardsyntakser og sammenligningsregler.
- [RFC 4520 – IANA Considerations for LDAP](https://datatracker.ietf.org/doc/html/rfc4520) – registrering av OID-er og protokollparametere.
- [RFC 4513 – LDAP Authentication Methods and Security Mechanisms](https://datatracker.ietf.org/doc/html/rfc4513) – bind-metoder, SASL, TLS og sikkerhetsgrenser.
- [RFC 9525, avsnitt 2 og 4](https://datatracker.ietf.org/doc/html/rfc9525)
- [Microsoft – LDAP signing for AD DS](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing) – Signing, Channel Binding, standarder og hendelser.
- [MS-ADTS – Channel Binding](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/2c3ea153-eff9-46a7-8614-19b677efa4e0) – LDAP Channel Binding i Active Directory.
- [Microsoft – LDAP session security after ADV190023](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/ldap-session-security-settings-requirements-adv190023) – sikkerhetskrav per bind- og TLS-modell.
- [RFC 4515 – String Representation of Search Filters](https://datatracker.ietf.org/doc/html/rfc4515) – filtergrammatikk og verdikoding.
- [RFC 3673 – All Operational Attributes](https://datatracker.ietf.org/doc/html/rfc3673) – `+` som attributtvelger for operative attributter.
- [RFC 2696 – Simple Paged Results Control](https://datatracker.ietf.org/doc/html/rfc2696) – sider, cookies og konsistensgrenser.
- [MS-ADTS – LDAP Policies](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/3f0137a1-63df-400c-bf97-e1040f055a99) – administrative søke- og ressursgrenser.
- [Microsoft – Paging Search Results](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ldap/paging-search-results) – Paged Search i Active Directory.
- [Microsoft – RootDSE](https://learn.microsoft.com/en-us/windows/win32/adschema/rootdse) – Naming Contexts og serverfunksjoner.
- [MS-ADTS – rootDSE Attributes](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/96f7b086-1ca3-4764-9a08-33f8f7a543db) – AD-spesifikke Root DSE-attributter.
- [MS-ADTS – Ports](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1010e59a-cf64-410c-a05c-a3a6c715261a) – LDAP-, LDAPS- og Global Catalog-porter.
- [Microsoft – Searching the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/searching-global-catalog-contents) – søk på tvers av skogen og partiell replika.
- [Microsoft – Attributes included in the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/attributes-included-in-the-global-catalog) – Partial Attribute Set i Global Catalog.
- [MS-ADTS – LDAP Matching Rules](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/4e638665-f466-4597-93c4-12f2ebfabab5) – AD-spesifikke Extensible-Match-OID-er.
- [Microsoft – Primary group membership](https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/user-prov-sync/exclude-user-primary-group) – avgrensning mellom `memberOf` og Primary Group.
- [Microsoft – DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator) – DNS-SRV-valg, LDAP Ping og stedstilknytning.
- [Microsoft – Verify LDAP SRV records](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created) – SRV-registrering for Domain Controllers.
- [MS-DRSR – Relationship to Other Protocols](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/0597aff6-0177-4d52-99f2-14a5441bc3c1) – avgrensning av AD-replikering fra LDAP.
- [RFC 4533 – LDAP Content Synchronization Operation](https://datatracker.ietf.org/doc/html/rfc4533) – LDAP Sync-Controls, cookies og tilstandsmodell.
- [OpenLDAP Administrator's Guide – Replication](https://www.openldap.org/doc/admin25/replication.html) – `syncrepl`, cookies og Provider-/Consumer-modell.
- [OpenLDAP Administrator's Guide – Performance Tuning](https://www.openldap.org/doc/admin26/tuning.html) – indeksering og serverinterne søkekostnader.
- [Microsoft – Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state) – System State-sikkerhetskopiering for AD DS.
- [OpenLDAP Administrator's Guide – Directory Backups](https://www.openldap.org/doc/admin26/maintenance.html) – LMDB-sikkerhetskopi, `slapcat` og konsistensgrenser.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – DNS- og SRV-forespørsler i Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – DNS- og SRV-forespørsler i Unix-systemer.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – TCP-tilkoblingsdiagnostikk i Windows.
- [Microsoft Learn – SslStream](https://learn.microsoft.com/dotnet/api/system.net.security.sslstream) – TLS-handshake og sertifikatkontroll med .NET.
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/) – TLS-handshake, navnekontroll og LDAP-StartTLS.
- [Microsoft Learn – Get-ADRootDSE](https://learn.microsoft.com/powershell/module/activedirectory/get-adrootdse?view=windowsserver2025-ps) – Root DSE-diagnostikk i Windows.
- [OpenLDAP – ldapsearch(1)](https://www.openldap.org/software/man.cgi?query=ldapsearch&sektion=1&apropos=0&manpath=OpenLDAP+2.6-Release) – Search-, StartTLS-, SASL- og Control-alternativer.
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser?view=windowsserver2025-ps) – LDAPFilter, SearchBase, Scope og Paging.
- [RFC 1487 – X.500 Lightweight Directory Access Protocol](https://datatracker.ietf.org/doc/html/rfc1487) – første LDAP-spesifikasjon fra 1993.
- [RFC 1777 – Lightweight Directory Access Protocol](https://datatracker.ietf.org/doc/html/rfc1777) – LDAPv2 og historisk protokollmodell.
- [RFC 2251 – Lightweight Directory Access Protocol v3](https://datatracker.ietf.org/doc/html/rfc2251) – første LDAPv3-kjernespesifikasjon.
- [OpenLDAP – Release Road Map](https://www.openldap.org/software/roadmap.html) – Release 1.0 i august 1998.
- [OpenLDAP Administrator's Guide – Preface](https://www.openldap.org/doc/admin26/preface.html) – University of Michigan LDAP som grunnlag for prosjektet.
- [MS-ADTS – Active Directory Technical Specification](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/) – LDAP-serverprofil for AD DS og AD LDS.
