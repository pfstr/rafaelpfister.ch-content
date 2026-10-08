---
title: "LDAP: protokoll, datamodell och katalogdrift"
blatt: "ldap"
description: "LDAP för administratörer: protokollstack och BER-wireformat, DIT, Distinguished Names, schema, bindning och SASL, sökning och kontroller, TLS, Active Directory, Global Catalog, replikeringsgränser, skalning och diagnostik."
fakten:
  - label: Namn
    wert: Lightweight Directory Access Protocol
    href: https://datatracker.ietf.org/doc/html/rfc4510
  - label: Protokollversion
    wert: LDAPv3 · RFC 4510 till 4519
    href: https://datatracker.ietf.org/doc/html/rfc4510#section-1
  - label: Wireformat
    wert: ASN.1-strukturer, BER-kodade
    href: https://datatracker.ietf.org/doc/html/rfc4511#section-5.1
  - label: Transport
    wert: TCP; valfritt TLS och SASL ovanpå
    href: https://datatracker.ietf.org/doc/html/rfc4511#section-5
  - label: Portar
    wert: 389 LDAP · 636 LDAP över TLS
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap
  - label: Datamodell
    wert: DIT med namngivna poster och attribut
    href: https://datatracker.ietf.org/doc/html/rfc4512#section-2
  - label: Namn
    wert: DN med ordnade RDN:er
    href: https://datatracker.ietf.org/doc/html/rfc4514
  - label: Filter
    wert: Prefixsyntax enligt RFC 4515
    href: https://datatracker.ietf.org/doc/html/rfc4515
  - label: Bindning
    wert: anonymous, simple eller SASL
    href: https://datatracker.ietf.org/doc/html/rfc4513#section-5
  - label: StartTLS
    wert: Extended Operation i befintlig session
    href: https://datatracker.ietf.org/doc/html/rfc4511#section-4.14
  - label: Paginering
    wert: Simple Paged Results Control
    href: https://datatracker.ietf.org/doc/html/rfc2696
  - label: AD Global Catalog
    wert: 3268 LDAP · 3269 LDAP över TLS
    href: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1010e59a-cf64-410c-a05c-a3a6c715261a
werbung:
  - newsletter
ctaThemen:
  - active-directory-entra
  - seppmail
translationSourceHash: 7b219213aa84d6ce78de262f2cd3a5ba23a220cc1669e134bceb3a325b06de33
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T11:25:28.863Z
translationReview: required
---

# LDAP: protokoll, datamodell och katalogdrift

LDAP är det gemensamma språk som program använder för att komma åt katalogtjänster. Med det kan en klient söka efter namngivna poster, läsa eller ändra attribut och autentisera sig mot katalogen. Protokollet definierar meddelanden, operationer och felkoder. Hur en server lagrar sina data, replikerar dem eller skyddar mot avbrott är däremot upp till respektive implementering. Active Directory Domain Services, OpenLDAP och 389 Directory Server talar därför LDAP utan att internt vara samma plattform ([RFC 4510, avsnitt 1 och 2](https://datatracker.ietf.org/doc/html/rfc4510#section-1), [RFC 4511, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc4511#section-3)).

Denna åtskillnad är avgörande i meddelandedriften. En gateway kan före SMTP-mottagning kontrollera om en mottagare finns, slå upp grupper för en policy eller logga in en administratör. Om denna fråga misslyckas eller returnerar inaktuella data är det inte bara «LDAP-störning»: beroende på integration kan meddelanden avvisas, regler tillämpas fel eller inloggningar blockeras. Administratören måste därför veta var på vägen från DNS-namnet till det lästa attributet som avbrottet finns.

Förklaringen följer denna väg. Först hittar klienten en server och upprättar en skyddad session. Därefter autentiserar den sig, skickar en sökning och tolkar svaren. Först när detta normala flöde är tydligt går det att placera in schema, Active Directory-särdrag, replikering, skalning och återställning på ett meningsfullt sätt.

## Protokollstack och sessionsmodell

Innan ett program kan söka behöver det ett konkret tjänstmål. I Active Directory-miljöer tillhandahåller DNS-SRV-poster möjliga Domain Controllers eller Global Catalogs; andra produkter använder statiska FQDN:er, egen tjänsteupptäckt eller en lastbalanserare. Detta val avgör inte bara IP-adressen utan även plats, serverroll och det namn mot vilket certifikatet kontrolleras. Ett porttest mot en godtycklig nåbar server besvarar därför ännu inte om programmet når sitt avsedda mål.

På det valda målet bygger LDAP en [TCP](/kb/tcp)-anslutning. Port 389 börjar som LDAP och kan växla till en skyddad session med StartTLS Extended Operation. Port 636 är registrerad hos IANA som `ldaps` och används bland annat av Active Directory för TLS som startar omedelbart. I båda fallen måste klienten kontrollera certifikatkedjan och servernamnet; «krypterad» och «ansluten till rätt server» är två olika bevis ([RFC 4511, avsnitt 4.14 och 5](https://datatracker.ietf.org/doc/html/rfc4511#section-4.14), [IANA Service Name Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap), [MS-ADTS, Using SSL/TLS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/8e73932f-70cf-46d6-88b1-8d9f86235e81)).

Inom denna anslutning överför LDAP inga läsbara kommandorader som [SMTP](/kb/smtp). Meddelandena beskrivs som ASN.1-strukturer och kodas med Basic Encoding Rules, BER. Varje `LDAPMessage` innehåller ett `messageID`, exakt en operation och valfria kontroller. Med Message-ID kan en långlivad anslutning hålla isär flera pågående operationer; deras svar behöver inte komma i frågeordning. En lyckad TCP-handshake säger alltså inget om BER-avkodning, bindning eller en helt avslutad sökning ([RFC 4511, avsnitt 3.1, 4.1.1 och 5.1](https://datatracker.ietf.org/doc/html/rfc4511#section-4.1.1)).

| Lager | Standardiserat innehåll | Observation relevant för administratören |
|---|---|---|
| Program | Bind, Search, Compare, Modify, Add, Delete, ModifyDN, Extended Operations och Controls | Result Code, `diagnosticMessage`, Entries, References och Controls |
| Kodning | ASN.1-datatyper i BER | Avkodningsfel, maximal requeststorlek, Message-ID och OID:er |
| Säkerhet | TLS samt SASL-mekanismer och deras Security Layer | Certifikatnamn, Trust Chain, bindningsmetod, Signing, Channel Binding |
| Transport | långlivad TCP-anslutning | DNS-mål, port, anslutningslatens, resets, Idle Timeout och poolstatus |
| Internt i servern | DIT, schema, ACL, index, lagring och replikering | inte standardiserat av LDAP; produkt- och topologispecifikt |

För ändringar gäller en viktig gräns: en enskild LDAP-operation är atomär inom sitt omfång, men flera poster utgör ingen gemensam transaktion i kärnprotokollet. RFC 5805 beskriver en experimentell transaktionsutökning, vars stöd klienten måste identifiera vid Root DSE. Även där måste det kontrolleras hur repliker ser ändringen. Provisioneringsprocesser behöver därför egna regler för upprepning, delfel och avstämning i stället för en tyst förutsatt databastransaktion ([RFC 4511, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc4511#section-3), [RFC 5805, avsnitt 1 och 3](https://datatracker.ietf.org/doc/html/rfc5805#section-1)).

## Datamodell: DIT, Entry, attribut och schema

Efter sessionsuppbyggnaden måste klienten kunna ange var och vad den söker. LDAP organiserar därför katalogdata som ett Directory Information Tree, förkortat DIT. Varje Entry har ett unikt Distinguished Name och attribut. Schemat beskriver vilka attribut som finns, hur deras värden jämförs och vilka Object Classes de kräver eller tillåter. Utan denna modell är sökbas, filter och resultat bara strängar utan tillförlitlig betydelse ([RFC 4512, avsnitt 2 och 3](https://datatracker.ietf.org/doc/html/rfc4512#section-2), [RFC 4511, avsnitt 4.1.7](https://datatracker.ietf.org/doc/html/rfc4511#section-4.1.7)).

Distinguished Name utgör sökvägen till en post i trädet. I `cn=Mail Gateway,ou=Services,dc=example,dc=ch` betecknar `cn=Mail Gateway` det lokala Relative Distinguished Name; följande RDN:er leder via containern till namnroten. Eftersom RDN:er kan vara flervärda och tecken som komma, plus eller omvänt snedstreck måste escape-kodas får programvara inte bearbeta ett DN genom enkel uppdelning vid kommatecken. Den behöver en RFC-4514-kompatibel parser ([RFC 4512, avsnitt 2.3](https://datatracker.ietf.org/doc/html/rfc4512#section-2.3), [RFC 4514, avsnitt 2 och 3](https://datatracker.ietf.org/doc/html/rfc4514#section-2)).

```text
dn: cn=Mail Gateway,ou=Services,dc=example,dc=ch
objectClass: top
objectClass: person
objectClass: organizationalPerson
cn: Mail Gateway
sn: Gateway
mail: mail-gateway@example.ch
```

För export och import finns LDIF, en standardiserad textrepresentation. LDIF avbildar Entries eller ändringsposter, men är inte wireformatet för den pågående LDAP-sessionen. Radbrytning, Base64-värden och Change Records följer egna regler. Framför allt innehåller en export bara det som servern och behörigheterna gör synligt; operativa attribut, ACL:er eller backendstatus kan saknas. En LDIF-dump är därför ett datautdrag, men inte automatiskt en återställningsbar serverbackup ([RFC 2849, avsnitt 2 och 4](https://datatracker.ietf.org/doc/html/rfc2849#section-2)).

Betydelsen hos ett attributvärde kommer först från schemat. En Matching Rule som `caseIgnoreMatch`, `integerMatch` eller en DN-jämförelse avgör om två värden är lika och vilka filter som fungerar för dem. På wire-nivå framträder värden först som Octet Strings; syntax och attributtyp ger dem deras tolkning. Egna schemaelement behöver därför varaktigt unika OID:er, definierade syntaxer och Matching Rules samt en utrullning som tar hänsyn till server och alla beroende klienter samtidigt ([RFC 4512, avsnitt 4](https://datatracker.ietf.org/doc/html/rfc4512#section-4), [RFC 4517](https://datatracker.ietf.org/doc/html/rfc4517), [RFC 4520](https://datatracker.ietf.org/doc/html/rfc4520)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-ldap.svg?v=20260813" title="Interaktive Infografik: LDAP-Protokollstack, Nachrichtenschicht, DIT, Serverarchitektur und Admin-Diagnosepunkte" loading="lazy">
  <a href="/images/kb-interaktiv-ldap.svg?v=20260813">Öppna infografik om LDAP-protokoll och katalogarkitektur</a>
</iframe>

## Operationer och tillståndsändringar

Med transport, namn och schema finns grundstrukturen; nu börjar den egentliga protokolldialogen. Den första avgörande tillståndsändringen är vanligen `Bind`. Den fastställer under vilken identitet och med vilka därav härledda rättigheter följande operationer körs. En ny bindning ersätter detta tillstånd. Medan servern bearbetar en bindning får klienten inte starta andra operationer på samma anslutning ([RFC 4511, avsnitt 3.1 och 4.2](https://datatracker.ietf.org/doc/html/rfc4511#section-4.2)).

Efter lyckad bindning kan klienten läsa eller skriva. En sökning ger inte ett enda stort svar utan noll eller flera `SearchResultEntry`-meddelanden, eventuellt References och till sist exakt ett `SearchResultDone`. Först detta slutresultat visar om sekvensen var fullständig. `Modify`, `Add`, `Delete` och `ModifyDN` ändrar poster; `Compare` kontrollerar ett attributvärde enligt dess Matching Rule utan att returnera en vanlig sökträff ([RFC 4511, avsnitt 4.5 till 4.9](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5)).

Även sessionsslutet har tydlig semantik. `Unbind` är en ensidig begäran om stängning och har inget svar. `Abandon` ber servern att avbryta en viss operation, men garanterar inte avbrottet. Om TCP i stället bryts försvinner alla pågående operationer. Vid en skrivoperation kan klienten då inte säkert veta om ändringen hann träda i kraft före eller efter anslutningsförlusten; en retry kräver därför först en tillståndsavstämning i stället för blind upprepning ([RFC 4511, avsnitt 4.3 och 4.11](https://datatracker.ietf.org/doc/html/rfc4511#section-4.3)).

| Operation | Typisk användning | Gräns som en klient måste hantera |
|---|---|---|
| Bind | Tjänstkonto, användarkontroll eller SASL-autentisering | lyckad TCP-/TLS-anslutning är ännu ingen lyckad bindning |
| Search | Läsa mottagare, grupper, adresser, policyer och Root DSE | flera Entries, References, gränser, Controls och slutresultat |
| Compare | Kontrollera ett känt attributvärde på serversidan | resultatet är `compareTrue` eller `compareFalse`, inte ett Search-resultat |
| Modify/Add/Delete/ModifyDN | Provisionering och livscykel | atomärt per operation, men utan kärnprotokollstransaktion över flera Entries |
| Extended Operation | StartTLS, Password Modify eller leverantörsspecifika funktioner | kontrollera OID och stöd på målservern |
| Controls | Paging, sortering, Assertion, Sync eller leverantörsfunktion | okänd kritisk Control måste leda till fel |

Controls och Extended Operations kompletterar detta flöde utan att införa en ny LDAP-version. Varje Control har en OID, en Criticality och valfritt ett BER-kodat värde. Om en klient markerar en okänd eller icke körbar Control som kritisk måste operationen misslyckas med `unavailableCriticalExtension`; annars får servern ignorera den. Före Paging, Sync eller en leverantörsfunktion läser en korrekt klient därför bland annat `supportedControl`, `supportedExtension`, `supportedFeatures`, `supportedLDAPVersion` och `supportedSASLMechanisms` från Root DSE ([RFC 4511, avsnitt 4.1.11](https://datatracker.ietf.org/doc/html/rfc4511#section-4.1.11), [RFC 4512, avsnitt 5.1](https://datatracker.ietf.org/doc/html/rfc4512#section-5.1)).

## Bind, SASL och TLS-förtroendegränser

Bindningen avgör vem servern tillskriver nästa sökning. Vid Simple Bind måste tre fall skiljas åt: tomt DN och tomt lösenord ger anonym åtkomst. Ett icke tomt DN med tomt lösenord är en *unauthenticated Bind* och bekräftar uttryckligen inte den angivna identiteten, även om servern kan returnera `success`. Först ett icke tomt DN med icke tomt lösenord utgör normal namn-/lösenordsautentisering. Klienter bör därför avvisa tomma lösenord redan före requestet; servrar bör inte av misstag tillåta unauthenticated Binds ([RFC 4513, avsnitt 5.1.1 till 5.1.3 och 6.3.1](https://datatracker.ietf.org/doc/html/rfc4513#section-5.1)).

Vid denna lösenordsautentisering känner servern till den presenterade hemligheten. Transporten måste därför inte bara vara krypterad utan även autentiserad. Detta omfattar en giltig certifikatkedja och kontroll av att det konfigurerade DNS-namnet finns i certifikatet. Den som accepterar alla certifikat eller använder en IP-adress kan upprätta en krypterad kanal till fel motpart. Samma kontroll gäller StartTLS och LDAP med omedelbar TLS ([RFC 4513, avsnitt 3.1 och 5.1.3](https://datatracker.ietf.org/doc/html/rfc4513#section-3.1), [RFC 9525, avsnitt 2 och 4](https://datatracker.ietf.org/doc/html/rfc9525#section-4), [TLS](/kb/tls)).

SASL tillåter olika autentiseringsmekanismer i stället för enbart lösenordsbindning och kan dessutom förhandla fram skydd för efterföljande LDAP-meddelanden. I Active Directory förekommer särskilt Negotiate, Kerberos och NTLM. LDAP Signing skyddar där integriteten hos vissa SASL-sessioner; Channel Binding knyter autentiseringen till den underliggande TLS-anslutningen. TLS, Signing och Channel Binding löser således relaterade men inte identiska problem. Ett test måste avbilda produktens faktiska bindningstyp ([RFC 4513, avsnitt 5.2](https://datatracker.ietf.org/doc/html/rfc4513#section-5.2), [Microsoft: LDAP signing](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing), [MS-ADTS, Channel Binding](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/2c3ea153-eff9-46a7-8614-19b677efa4e0)).

Vid införande av striktare AD-principer räcker det därför inte att titta på Windows-versionen. Nya AD-DS-distributioner på Windows Server 2025 kräver LDAP Signing som standard, medan uppgraderingar övertar befintliga inställningar. Microsoft anger Directory Service-händelserna 2886 till 2889 för Signing samt 3039 till 3041 för Channel Binding. Dessa revisionsdata visar vilka klienter, portar och bindningsmetoder som faktiskt skulle påverkas; först därefter kan genomdrivandet planeras på ett hållbart dataunderlag ([Microsoft: LDAP signing, Default Security Behavior och Event Monitoring](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing), [Microsoft: LDAP session security after ADV190023](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/ldap-session-security-settings-requirements-adv190023)).

## Search: bas, scope, filter och attributprojektion

Efter säker bindning följer operationen som de flesta integrationer är beroende av: Search. Requestet anger ett Base DN, scope, aliashantering, egna Size- och Time-Limits, ett filter och önskade attribut. `baseObject` läser endast basposten, `singleLevel` dess direkta barn och `wholeSubtree` hela underträdet inklusive basen. Servern får sätta striktare gränser. Noll träffar med `success` är ett giltigt svar; `noSuchObject` betyder däremot att sökbasen saknas eller inte är synlig för denna identitet ([RFC 4511, avsnitt 4.5.1 och 4.5.2](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5.1)).

Filtret beskriver inte fri SQL-logik utan ett träd i prefixnotation. `(&(objectClass=person)(mail=*@example.ch))` sammanfogar exempelvis ett Equality-uttryck med ett Substring-uttryck. `|` står för OR, `!` för NOT, `=*` för Presence och `:=` för ett Extensible Match. Huruvida en jämförelse använder versal-/gemen-känslighet, talordning eller DN-semantik avgörs av attributets Matching Rule ([RFC 4511, avsnitt 4.5.1](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5.1), [RFC 4515](https://datatracker.ietf.org/doc/html/rfc4515), [RFC 4517](https://datatracker.ietf.org/doc/html/rfc4517)).

Därmed blir skapandet av filter en säkerhetsuppgift. Värden från användarinmatningar måste kodas enligt RFC 4515; särskilt `*`, parenteser, omvänt snedstreck, NUL och ogiltiga UTF-8-oktetar får inte hamna råa i uttrycket. Strängkonkatenering kan annars ändra filterstrukturen och möjliggöra LDAP-injektion. DN-escaping enligt RFC 4514 följer andra regler och ersätter inte filterkodning ([RFC 4515, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc4515#section-3), [RFC 4514, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc4514#section-3)).

Utöver filtret bestämmer attributlistan hur mycket servern returnerar. En tom lista begär alla vanliga användarattribut, `1.1` inga attribut, `*` alla användarattribut och `+` enligt RFC 3673 alla operativa attribut. ACL:er kan fortfarande dölja värden. Produktionsklienter bör bara begära attribut som behövs: stora flervärden belastar nätverk, avkodare och minne och kan utlösa egna servergränser ([RFC 4511, avsnitt 4.5.1.8](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5.1.8), [RFC 3673](https://datatracker.ietf.org/doc/html/rfc3673)).

### Paging, sortering och föränderliga resultat

Större resultatmängder överförs vanligen med Simple Paged Results Control. Servern skickar med en ogenomskinlig cookie för varje sida som klienten skickar tillbaka tillsammans med samma request. Denna cookie är varken en offset eller en permanent cursor. Om kataloginnehållet förändras under sekvensen kan poster saknas eller förekomma dubbelt. Paging begränsar alltså datamängden per svar men skapar ingen konsekvent snapshot ([RFC 2696, avsnitt 2 och 3](https://datatracker.ietf.org/doc/html/rfc2696#section-2)).

Active Directory gör denna skillnad synlig i vardagen: LDAP Policy `MaxPageSize` begränsar opaginerade resultat till 1000 objekt som standard. En import som får exakt 1000 poster har därför inte bevisat sin fullständighet. Klienten måste bearbeta sidor och cookies korrekt och upptäcka avbrott. Ytterligare policyer begränsar frågetid, Receive Buffer och samtidiga Result Sets. I drift bör därför Page Size, antal sidor, senaste cookieförlopp, timeout och återstart loggas ([MS-ADTS, LDAP Policies](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/3f0137a1-63df-400c-bf97-e1040f055a99), [Microsoft: Paging Search Results](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ldap/paging-search-results)).

## Active Directory som LDAP-serverprofil

De tidigare reglerna gäller LDAP generellt. Active Directory Domain Services är en konkret serverimplementering med ytterligare roller och konventioner. Dess data är fördelade över Naming Contexts; en Domain Controller har minst Schema, Configuration och sin egen Domain-Naming-Context. Root DSE har det tomma DN:t och anger bland annat `defaultNamingContext`, `configurationNamingContext`, `schemaNamingContext`, alla `namingContexts`, servernamnet och stödda mekanismer. Efter TCP och TLS är denna post det första test som faktiskt säger något om den uppnådda katalogtjänsten ([RFC 4512, avsnitt 5.1](https://datatracker.ietf.org/doc/html/rfc4512#section-5.1), [Microsoft RootDSE](https://learn.microsoft.com/en-us/windows/win32/adschema/rootdse), [MS-ADTS, rootDSE Attributes](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/96f7b086-1ca3-4764-9a08-33f8f7a543db)).

Valet mellan Domain Controller och Global Catalog ändrar sökresultatet. En DC tillhandahåller LDAP på 389 respektive 636 och känner till hela Domain-Naming-Context för sin domän. Global Catalog använder dessutom 3268 eller 3269 och har en partiell replik från alla domäner i skogen. Den kan hitta objekt i hela skogen men returnerar för externa domäner bara attribut ur Partial Attribute Set. En lyckad träff bevisar därför ännu inte att attributet som programmet behöver finns tillgängligt ([MS-ADTS, Ports](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1010e59a-cf64-410c-a05c-a3a6c715261a), [Microsoft: Searching the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/searching-global-catalog-contents), [Microsoft: Attributes included in the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/attributes-included-in-the-global-catalog)).

Även filter kan bli AD-specifika. Matching Rule `1.2.840.113556.1.4.1941`, `LDAP_MATCHING_RULE_TRANSITIVE_EVAL`, följer exempelvis länkade attribut och kan utvärdera nästlade grupper. Dess stöd anges inte bara i `supportedControl`. Dessutom innehåller `memberOf` inte Primary Group. Ett auktoriseringsbeslut baserat på gruppmedlemskap måste därför uttryckligen ta hänsyn till gruppnästling, Primary Group, gruppscope, ACL-synlighet och replikeringsstatus ([MS-ADTS, LDAP Matching Rules](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/4e638665-f466-4597-93c4-12f2ebfabab5), [Microsoft: Primary group membership](https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/user-prov-sync/exclude-user-primary-group)).

Vilken server som besvarar dessa frågor avgörs för Windows-klienter av DC Locator tillsammans med [DNS](/kb/dns)-SRV-poster. Plats- och rollrelaterade poster ger kandidater med prioritet och vikt. En statiskt angiven IP kringgår detta urval och försvårar certifikatkontrollen. En enkel TCP-lastbalanserare fördelar visserligen anslutningar, men känner utan ytterligare logik varken skrivbara DC:er eller Global Catalogs, Naming Contexts eller replikeringsstatus ([Microsoft: DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator), [Microsoft: Verify LDAP SRV records](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created)).

Slutligen får LDAP-nåbarhet inte förväxlas med frisk replikering. AD replikerar katalogändringar via Directory Replication Service Remote Protocol; OpenLDAPs `syncrepl` använder däremot LDAP Content Synchronization med Provider, Consumer och Cookies. En testsökning kan visa att en viss server svarar. Om alla servrar har samma ändringar och återhämtar sig efter ett avbrott måste kontrolleras med verktygen för respektive plattform ([MS-DRSR, Relationship to Other Protocols](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/0597aff6-0177-4d52-99f2-14a5441bc3c1), [RFC 4533](https://datatracker.ietf.org/doc/html/rfc4533), [OpenLDAP Administrator's Guide: Replication](https://www.openldap.org/doc/admin25/replication.html)).

## Integrations- och driftsmodeller

För driften är det nu mindre viktigt att en produkt «stöder LDAP» än hur den använder LDAP. Vid en **uppslagning vid körning** väntar ett meddelande eller en session direkt på Search och serversvar. Vid **Credential Check** söker ett tekniskt konto först efter användarens DN och utför därefter en andra bindning med det angivna lösenordet. En **import eller cache** läser däremot många poster och arbetar med en lokal kopia fram till nästa körning. Dessa mönster får olika följder för latens, lösenordshantering, failover och dataålder; produktdokumentationen måste beskriva det konkreta beteendet ([RFC 4511, avsnitt 4.2 och 4.5](https://datatracker.ietf.org/doc/html/rfc4511#section-4.2), [RFC 2696](https://datatracker.ietf.org/doc/html/rfc2696)).

| Mönster | Kritisk väg | Omedelbart driftsbevis |
|---|---|---|
| Uppslagning vid körning | DNS, Connect, TLS, pool, Bind, Search och serversvar per åtgärd | p50/p95/p99 per operation, poolmättnad, Result Codes, fallbackmål |
| Credential Check | Användarsökning plus andra bindning med användarlösenord | DN-upplösning, spärr av tomma lösenord, TLS-namnkontroll, lockout-beteende |
| Periodisk import | fullständig, paginerad uppräkning och commit till lokal cache | Page-/Cookie-förlopp, objektantal, raderingsmodell, senaste lyckade commit |
| Change Sync | leverantörsspecifik eller LDAP Sync-cursor | cursorbeständighet, replay, resync och raderade objekt |

Oavsett mönster behöver klienten separata timeouter för Connect, Bind, Operation och Idle. En Connection Pool sparar TCP-, TLS- och bindningsuppbyggnad men bär anslutningens autentiseringstillstånd med sig. Döda sessioner måste upptäckas, och en anslutning får inte av misstag växla mellan användare eller tenants. Failover behöver en spårbar målordning, begränsade upprepningar och en väg tillbaka till föredraget mål. Annars multiplicerar parallella retryer belastningen just under ett katalogavbrott ([RFC 4511, avsnitt 3.1, 4.2 och 5.3](https://datatracker.ietf.org/doc/html/rfc4511#section-3.1)).

På serversidan avgör Base DN, scope, filter och attributlista arbetet. Ett selektivt likhetsvillkor på ett indexerat attribut är något annat än en inledande substring eller ett stort OR-uttryck. LDAP publicerar ingen exekveringsplan och föreskriver ingen indexteknik. Administratören måste därför korrelera verkliga produktfilter med resultatmängd, p95-/p99-latens och servermått. En snabb sökning efter ett enskilt testkonto bevisar inte att en mottagarkontroll skalar under topplast ([OpenLDAP Administrator's Guide: Performance Tuning](https://www.openldap.org/doc/admin26/tuning.html), [MS-ADTS: LDAP Policies](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/3f0137a1-63df-400c-bf97-e1040f055a99)).

Övervakning bör dela upp flödet i samma steg som felsökningen: DNS-val, TCP- och TLS-uppbyggnad, Bind, Search-latens, Result Code, antal träffar och Paging-förlopp. Till detta kommer poolbeläggning, retryfrekvens och status för import eller sync. En enda syntetisk bindning kan bekräfta nåbarhet, men identifierar varken saknade attribut, en ofullständig import eller en replikeringspartner som ligger efter.

För backup och recovery räcker inte heller det synliga kataloginnehållet. Schema, ACL:er, backend- och serverkonfiguration, nycklar och certifikat, replikeringsidentiteter samt metoden för att återföra en återställd nod till topologin måste säkerhetskopieras. Active Directory använder för detta System State och egna Forest Recovery-steg; OpenLDAP beror på sin backend. Handboken skiljer till exempel en LMDB-säkerhetskopia från `slapcat` och varnar för semantiskt inkonsekventa LDIF-tillstånd vid ändringar i flera delar. LDAP definierar ingen backupmekanism ([Microsoft: Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state), [OpenLDAP Administrator's Guide: Directory Backups](https://www.openldap.org/doc/admin26/maintenance.html)).

Ett återställningstest är först slutfört när en klient hittar den återställda tjänsten via det avsedda DNS-namnet, TLS och Bind lyckas, Root DSE och schema stämmer, verkliga sökningar ger fullständiga attribut och replikeringen kontrollerat startar igen. Därmed återvänder recovery till artikelns början: hela vägen räknas, inte bara en startad databas.

## Diagnostikverktyg

Diagnostiken följer samma väg som en produktiv fråga. Den börjar i den berörda applikationens nät och använder dess DNS-namn, Truststore, bindningsmetod, Base DN, filter och attributlista. Ett test från administratörens laptop kan annars lyckas medan gatewayen fortsätter att använda en annan DC, ett annat CA eller ett annat scope. Exemplen använder reserverade namn och läser bara metadata; bindningslösenord hör varken hemma i shellhistoriken eller i processargument. `ldapsearch -W` frågar efter dem interaktivt.

### Identifiera tjänstmål via DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-Dienstsuche">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) visar targets, portar, prioriteter och vikter. Därefter ska A-/AAAA-upplösning, platsanknytning och nåbarhet för varje faktiskt valbart target kontrolleras. En enskild nåbar DC åtgärdar inte en felaktig SRV-uppsättning ([Microsoft: DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator)).

### Kontrollera TCP och implicit TLS på port 636

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) visar först bara TCP-connect. Den efterföljande [.NET `SslStream`](https://learn.microsoft.com/dotnet/api/system.net.security.sslstream) respektive [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) kontrollerar TLS med det konfigurerade DNS-namnet. `s_client -showcerts` visar endast certifikaten som servern skickat och utgör i sig inget lyckat bevis för Chain eller Hostname. StartTLS på 389 kan under Unix kontrolleras separat med `openssl s_client -starttls ldap` ([OpenSSL `s_client`](https://docs.openssl.org/master/man1/openssl-s_client/), [RFC 4511, avsnitt 4.14](https://datatracker.ietf.org/doc/html/rfc4511#section-4.14)).

### Läs Root DSE och funktioner

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-Root-DSE-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-ADRootDSE`](https://learn.microsoft.com/powershell/module/activedirectory/get-adrootdse?view=windowsserver2025-ps) använder här ActiveDirectory-modulen och som standard den inloggade Windows-identiteten. [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch&sektion=1&apropos=0&manpath=OpenLDAP+2.6-Release) tvingar med `-ZZ` fram lyckad StartTLS och läser anonymt bara Root-DSE-attribut som servern har frisläppt. En saknad OID bevisar att just detta mål inte publicerar funktionen; det säger inget om andra klusternoder.

### Återskapa en verklig sökning med scope, filter och paging

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-Suchprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser?view=windowsserver2025-ps) accepterar med `-LDAPFilter` den RFC-nära filtersyntaxen och utför paging via `-ResultPageSize`. [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch&sektion=1&apropos=0&manpath=OpenLDAP+2.6-Release) använder `-E pr=500/noprompt` för Paged Results Control och `-W` för interaktiv lösenordsfråga. Testet måste utöver träffen även dokumentera slutresultat, antal sidor, returnerade attribut och körtid.

### Tilldela fel till en gräns

| Observation | Protokollbetydelse | Nästa tillförlitliga bevis |
|---|---|---|
| Timeout före TLS | Måluppslagning, routing, brandvägg, listener eller uttömd pool | SRV/A/AAAA, TCP-handshake, serverlistener och anslutningslatens |
| Certifikatfel | Chain, giltighet, namn eller klientens Trust stämmer inte | skickad Chain, Trust Anchor, SAN mot exakt konfigurerad FQDN |
| `strongAuthRequired` / `confidentialityRequired` | Servern kräver starkare bindnings- eller skyddsmetod | port, lyckad StartTLS, SASL-mekanism, Signing-/CBT-policy |
| `invalidCredentials` | Presenterande bindningsidentitet eller credentials avvisades | exakt bindningstyp och DN; ingen lösenordsloggning |
| `invalidDNSyntax` | DN är syntaktiskt ogiltigt | RFC-4514-kodning och faktiskt DN från Search Result |
| `noSuchObject` med `matchedDN` | Base DN saknas eller är osynligt från en överordnad nod | Root DSE, Naming Context, ACL-synlighet och `matchedDN` |
| `sizeLimitExceeded` | Klient- eller servergräns före fullständigt resultat | Paging-Control, Page-Cookies, LDAP Policy och totalräkning |
| `adminLimitExceeded` / `busy` / `unavailable` | Serverresurs eller administrativ gräns | servermått, Query Policy, filterkostnad, retryfrekvens och målnod |
| noll träffar med `success` | Giltig sökning utan synlig matchning | jämför Base, scope, filter, ACL, målnod och replikeringsstatus |

Numeriska Result Codes hör till LDAP-protokollet; `diagnosticMessage` och ytterligare AD-subkoder är däremot implementationsspecifik kontext. Automatisering bör därför först utvärdera Result Code och logga texten som komplement. För `busy` och `unavailable` behöver varje klient en begränsad retrybudget med backoff. Obegränsade upprepningar gör ett enskilt katalogproblem till en belastningstopp i alla beroende system ([RFC 4511, avsnitt 4.1.9 och bilaga A](https://datatracker.ietf.org/doc/html/rfc4511#appendix-A)).

## Teknisk historia

LDAP uppstod inte som en självständig katalogdatabas. X.500 hade i slutet av 1980-talet definierat en omfattande katalogmodell och Directory Access Protocol. RFC 1487 beskrev 1993 en lättare åtkomst till denna modell; RFC 1777 följde 1995 som LDAP Version 2. «Lightweight» avsåg den förenklade protokollåtkomsten jämfört med DAP, inte små kataloger eller låg driftsmässig betydelse. Den tidiga utvecklingen är nära förknippad med Tim Howes och University of Michigan ([RFC 1487](https://datatracker.ietf.org/doc/html/rfc1487), [RFC 1777](https://datatracker.ietf.org/doc/html/rfc1777)).

LDAPv3 publicerades 1997 med RFC 2251 och följddokument. Utbyggbara operationer, Controls, SASL, internationalisering och den reviderade datamodellen gjorde det till grunden för dagens implementationer. LDAPbis-arbetet ordnade om denna status 2006: RFC 4510 fungerar som roadmap, RFC 4511 beskriver protokollet, RFC 4512 informationsmodellen och RFC 4513 säkerheten; RFC 4514 till 4519 kompletterar representationer, URL:er, syntaxer och schema ([RFC 2251](https://datatracker.ietf.org/doc/html/rfc2251), [RFC 4510, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc4510#section-3)).

Parallellt utvecklades mycket olika servrar. OpenLDAP uppstod 1998 ur University of Michigan-implementeringen och fortsatte `slapd`, bibliotek och verktyg som ett open-source-projekt. Active Directory tog med Windows 2000 LDAPv3-profilen med eget schema, Naming Contexts, Controls, Matching Rules och separat replikeringsprotokoll till bred företagsanvändning ([OpenLDAP Release Road Map](https://www.openldap.org/software/roadmap.html), [OpenLDAP Administrator's Guide – Preface](https://www.openldap.org/doc/admin26/preface.html), [MS-ADTS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/)).

Denna historia förklarar den viktigaste driftsregeln: LDAP förenhetligar åtkomsten, inte den interna arkitekturen. Den som flyttar en klient från OpenLDAP till AD DS eller mellan två appliances måste därför kontrollera mer än host, port och Bind-DN. Schema, Controls, gränser, gruppupplösning, replikering och recovery förblir produktegenskaper.

## Källor

- [RFC 4510, avsnitt 1 och 2](https://datatracker.ietf.org/doc/html/rfc4510)
- [RFC 4511 – LDAP: The Protocol](https://datatracker.ietf.org/doc/html/rfc4511) – meddelandelager, operationer, BER, TCP, StartTLS och Result Codes.
- [IANA – Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap) – `ldap` 389 och `ldaps` 636.
- [MS-ADTS – Using SSL/TLS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/8e73932f-70cf-46d6-88b1-8d9f86235e81) – implicit TLS och StartTLS i Active Directory.
- [RFC 5805, avsnitt 1 och 3](https://datatracker.ietf.org/doc/html/rfc5805)
- [RFC 4512 – LDAP Directory Information Models](https://datatracker.ietf.org/doc/html/rfc4512) – DIT, Entries, attribut, schema, Root DSE och subschema.
- [RFC 4514, avsnitt 2 och 3](https://datatracker.ietf.org/doc/html/rfc4514)
- [RFC 2849, avsnitt 2 och 4](https://datatracker.ietf.org/doc/html/rfc2849)
- [RFC 4517 – LDAP Syntaxes and Matching Rules](https://datatracker.ietf.org/doc/html/rfc4517) – standardsyntaxer och jämförelseregler.
- [RFC 4520 – IANA Considerations for LDAP](https://datatracker.ietf.org/doc/html/rfc4520) – registrering av OID:er och protokollparametrar.
- [RFC 4513 – LDAP Authentication Methods and Security Mechanisms](https://datatracker.ietf.org/doc/html/rfc4513) – bindningsmetoder, SASL, TLS och säkerhetsgränser.
- [RFC 9525, avsnitt 2 och 4](https://datatracker.ietf.org/doc/html/rfc9525)
- [Microsoft – LDAP signing for AD DS](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing) – Signing, Channel Binding, standardvärden och händelser.
- [MS-ADTS – Channel Binding](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/2c3ea153-eff9-46a7-8614-19b677efa4e0) – LDAP Channel Binding i Active Directory.
- [Microsoft – LDAP session security after ADV190023](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/ldap-session-security-settings-requirements-adv190023) – säkerhetskrav per Bind- och TLS-modell.
- [RFC 4515 – String Representation of Search Filters](https://datatracker.ietf.org/doc/html/rfc4515) – filtergrammatik och Value Encoding.
- [RFC 3673 – All Operational Attributes](https://datatracker.ietf.org/doc/html/rfc3673) – `+` som attributväljare för operativa attribut.
- [RFC 2696 – Simple Paged Results Control](https://datatracker.ietf.org/doc/html/rfc2696) – sidor, cookies och konsistensgränser.
- [MS-ADTS – LDAP Policies](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/3f0137a1-63df-400c-bf97-e1040f055a99) – administrativa sök- och resursgränser.
- [Microsoft – Paging Search Results](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ldap/paging-search-results) – Paged Search i Active Directory.
- [Microsoft – RootDSE](https://learn.microsoft.com/en-us/windows/win32/adschema/rootdse) – Naming Contexts och serverfunktioner.
- [MS-ADTS – rootDSE Attributes](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/96f7b086-1ca3-4764-9a08-33f8f7a543db) – AD-specifika Root-DSE-attribut.
- [MS-ADTS – Ports](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1010e59a-cf64-410c-a05c-a3a6c715261a) – LDAP-, LDAPS- och Global-Catalog-portar.
- [Microsoft – Searching the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/searching-global-catalog-contents) – sökning i hela skogen och partiell replik.
- [Microsoft – Attributes included in the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/attributes-included-in-the-global-catalog) – Partial Attribute Set för Global Catalog.
- [MS-ADTS – LDAP Matching Rules](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/4e638665-f466-4597-93c4-12f2ebfabab5) – AD-specifika Extensible-Match-OID:er.
- [Microsoft – Primary group membership](https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/user-prov-sync/exclude-user-primary-group) – avgränsning mellan `memberOf` och Primary Group.
- [Microsoft – DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator) – DNS-SRV-val, LDAP Ping och platsanknytning.
- [Microsoft – Verify LDAP SRV records](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created) – SRV-registrering för Domain Controllers.
- [MS-DRSR – Relationship to Other Protocols](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/0597aff6-0177-4d52-99f2-14a5441bc3c1) – avgränsning av AD-replikering från LDAP.
- [RFC 4533 – LDAP Content Synchronization Operation](https://datatracker.ietf.org/doc/html/rfc4533) – LDAP-Sync-Controls, cookies och tillståndsmodell.
- [OpenLDAP Administrator's Guide – Replication](https://www.openldap.org/doc/admin25/replication.html) – `syncrepl`, cookies och Provider-/Consumer-modell.
- [OpenLDAP Administrator's Guide – Performance Tuning](https://www.openldap.org/doc/admin26/tuning.html) – indexering och interna sökkostnader i servern.
- [Microsoft – Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state) – System State-säkerhetskopiering för AD DS.
- [OpenLDAP Administrator's Guide – Directory Backups](https://www.openldap.org/doc/admin26/maintenance.html) – LMDB-säkerhetskopia, `slapcat` och konsistensgränser.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – DNS- och SRV-frågor i Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – DNS- och SRV-frågor i Unix-system.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – TCP-anslutningsdiagnostik i Windows.
- [Microsoft Learn – SslStream](https://learn.microsoft.com/dotnet/api/system.net.security.sslstream) – TLS-handshake och certifikatkontroll med .NET.
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/) – TLS-handshake, namnkontroll och LDAP-StartTLS.
- [Microsoft Learn – Get-ADRootDSE](https://learn.microsoft.com/powershell/module/activedirectory/get-adrootdse?view=windowsserver2025-ps) – Root-DSE-diagnostik i Windows.
- [OpenLDAP – ldapsearch(1)](https://www.openldap.org/software/man.cgi?query=ldapsearch&sektion=1&apropos=0&manpath=OpenLDAP+2.6-Release) – Search-, StartTLS-, SASL- och Control-alternativ.
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser?view=windowsserver2025-ps) – LDAPFilter, SearchBase, scope och Paging.
- [RFC 1487 – X.500 Lightweight Directory Access Protocol](https://datatracker.ietf.org/doc/html/rfc1487) – första LDAP-specifikationen från 1993.
- [RFC 1777 – Lightweight Directory Access Protocol](https://datatracker.ietf.org/doc/html/rfc1777) – LDAPv2 och historisk protokollmodell.
- [RFC 2251 – Lightweight Directory Access Protocol v3](https://datatracker.ietf.org/doc/html/rfc2251) – första LDAPv3-kärnspecifikationen.
- [OpenLDAP – Release Road Map](https://www.openldap.org/software/roadmap.html) – Release 1.0 i augusti 1998.
- [OpenLDAP Administrator's Guide – Preface](https://www.openldap.org/doc/admin26/preface.html) – University of Michigan-LDAP som grund för projektet.
- [MS-ADTS – Active Directory Technical Specification](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/) – LDAP-serverprofil för AD DS och AD LDS.
