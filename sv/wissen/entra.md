---
title: "Microsoft Entra ID: Klientorganisation, token och hybrididentitet"
blatt: "entra"
description: "Microsoft Entra ID för infrastruktur- och meddelandeadministratörer: klientorganisations- och katalogobjekt, Security Token Service, OAuth/OIDC och SAML, appar och tjänstehuvudnamn, arbetsbelastningsidentiteter, medgivande och auktorisering, villkorad åtkomst, enheter, hybridsynkronisering, Domain Services, Microsoft Graph, loggning, återställning och teknisk historik."
fakten:
  - label: Systemroll
    wert: multitenant molnkatalog- och identitetstjänst med token-, policy- och administrationskontrollplan
    href: https://learn.microsoft.com/en-us/entra/fundamentals/whatis
  - label: Klientorganisation
    wert: fristående katalog-/policygräns med klientorganisations-ID, verifierade domäner och objekt
    href: https://learn.microsoft.com/en-us/entra/identity-platform/single-and-multi-tenant-apps
  - label: Protokoll
    wert: OAuth 2.0 · OpenID Connect · SAML 2.0 · WS-Federation över HTTPS
    href: https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols
  - label: Administrations-API
    wert: Microsoft Graph för katalog-, identitets-, policy-, gransknings- och programobjekt
    href: https://learn.microsoft.com/en-us/graph/overview
  - label: Appmodell
    wert: Application Object som definition · Service Principal som lokal instans i klientorganisationen
    href: https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals
  - label: Användartoken
    wert: ID Token för klientsession · Access Token för Resource API · Refresh Token för förnyelse
    href: https://learn.microsoft.com/en-us/entra/identity-platform/security-tokens
  - label: Arbetsbelastningsidentitet
    wert: Service Principal eller Managed Identity; certifikat, hemlighet eller federerad autentiseringsuppgift
    href: https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-overview
  - label: Auktorisering
    wert: Delegated Scopes · Application Roles · Directory Roles · resurs-/objektpolicy
    href: https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview
  - label: Åtkomstpolicy
    wert: Conditional Access utvärderar signaler och tillämpar kontroller vid tokenbeslut
    href: https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview
  - label: Hybridbrygga
    wert: Connect Sync eller Cloud Sync replikerar utvalda AD-DS-objekt och attribut
    href: https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync
  - label: Äldre domäntjänster
    wert: Entra Domain Services tillhandahåller hanterat Kerberos, NTLM, LDAP och Domain Join
    href: https://learn.microsoft.com/en-us/entra/identity/domain-services/overview
  - label: Drifttillstånd
    wert: tokenutfärdande · policyresultat · synkronisering/provisionering · autentiseringsuppgifters förfall · inloggning/granskning · nödkonto
    href: https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-sign-ins
werbung:
  - newsletter
ctaThemen:
  - active-directory-entra
  - microsoft-entra
  - powershell
translationSourceHash: 6d1a58b89bfeb1ae9f9b24608e56afa198b91d6e6681bb8692d749a36e5e1ef3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:09:27.335Z
translationReview: automatic
---

# Microsoft Entra ID: Klientorganisation, token och hybrididentitet

Microsoft Entra ID är en molnkatalog, identitetsprovider och policytillämpningspunkt för Microsoft- och tredjepartsapplikationer. Den lagrar användare, grupper, enheter, applikationer, tjänstehuvudnamn och roller; dess Security Token Service utfärdar signerade token efter lyckad autentisering och policykontroll. Microsoft Graph utgör administrationskontrollplanet. Dessa roller måste betraktas separat: ett befintligt användarobjekt garanterar inte en lyckad inloggning, en giltig token ingen verksamhetsmässig objektbehörighet och ett synkroniserat attribut ingen omedelbar effekt i varje måltjänst.

För meddelandeadministratörer ligger Entra på flera kritiska vägar: användare och `proxyAddresses` matar Exchange Online, grupper styr distributionslistor och åtkomst, appbehörigheter möjliggör automatiserad åtkomst till postlådor eller Graph, Conditional Access påverkar administratörs- och klientinloggningar, och hybridsynkronisering kopplar Active Directory Domain Services till molnet. Klientorganisationen är därmed inte bara ”AD på internet”, utan en självständig identitets- och auktoriseringsplattform.

Förklaringen följer en identitet från inloggning till åtkomst till en konkret resurs. Först förklaras klientorganisation, katalog och token, därefter roller, applikationer, enheter, hybridanslutning, protokoll och återställning.

Microsoft Entra ID kombinerar en katalog med tokenutfärdande och policykontroll. En användare får inte åtkomst ”till Entra”, utan begär en token för en konkret resurs; Entra kontrollerar då identitet, applikation, enhet och policy.

## Arkitektur: Directory, STS, Policy och Resource

Microsoft beskriver Entra ID som en molnbaserad tjänst för identitets- och åtkomsthantering. Dess huvudkomponenter är:

1. **Directory:** Objekt, attribut, relationer, roller, domäner och lokal konfiguration för klientorganisationen.
2. **Security Token Service (STS):** Protokollslutpunkter, autentisering, tokenutfärdande och nyckelpublicering.
3. **Policy Engines:** Conditional Access, Identity Protection, Authentication Methods, Consent och andra åtkomstbeslut.
4. **Provisioning/Sync:** Replikering eller etablering till och från AD DS, SaaS och HR-källor.
5. **Microsoft Graph:** API för katalog-, identitets-, gransknings- och policyhantering.
6. **Resource Services:** Exchange Online, Graph, egna API:er och SaaS validerar token och verkställer sin auktorisering.

Entra har stöd för flera klientorganisationer. En klientorganisation utgör en administrativ och policymässig gräns, inte automatiskt en fullständig data- eller nätverksisolering för alla konsumerade SaaS-tjänster ([Microsoft Entra fundamentals – What is Microsoft Entra ID?](https://learn.microsoft.com/en-us/entra/fundamentals/whatis)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-entra.svg?v=20260813" title="Interaktive Infografik: Microsoft-Entra-Aufruf von Benutzer oder Workload über Authentisierung, Conditional Access und STS bis Token, Resource API und Microsoft Graph sowie Tenant-, App-, Hybrid-, Geräte- und Recoverygrenzen" loading="lazy">
  <a href="/images/kb-interaktiv-entra.svg?v=20260813">Öppna den interaktiva grafiken direkt</a>.
</iframe>

## Teknikstack ur administratörsperspektiv

För en SaaS-tjänst med flera klientorganisationer är den interna teknikstacken med programmeringsspråk, databaser och orkestrering inte en produktegenskap som kunden kan kontrollera. Att härleda den från värdnamn, klientbibliotek eller platsannonser vore värdelöst för drift och återställning. Den verifierbara **teknikstacken ur administratörsperspektiv** består av de publicerade avtalen: HTTPS som transport, OAuth 2.0, OpenID Connect, SAML och WS-Federation för identitet, JWT/JWS och JWKS för token och nycklar, Microsoft Graph som REST-/OData-kontrollplan samt agentbaserad synkronisering för hybrididentiteter. Microsoft dokumenterar uttryckligen dessa protokoll och Graph som gränssnitt som stöds ([Microsoft identity platform protocols](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols), [Microsoft Graph overview](https://learn.microsoft.com/en-us/graph/overview)).

För administrationen är därför inte ett antaget serverspråk avgörande, utan den konkreta kombinationen av klientorganisation, authority, protokollversion, tokenformat, Graph-slutpunkt, SDK- eller PowerShell-modul, synkroniseringsagent och resurstjänst. Var och en av dessa lager har egna gränser för versionshantering, behörigheter, loggning och fel.

## Klientorganisation, domäner och objektankare

Varje klientorganisation har ett oföränderligt klientorganisations-ID. Verifierade domännamn erbjuder läsbara inloggningsnamn och routningskoppling, men ersätter inte klientorganisations-ID:t. Ett domänbyte eller en konfiguration med flera klientorganisationer får därför inte identifieras enbart utifrån UPN-suffixet.

Objekt har klientorganisationslokala ID:n. Användare kan vara medlemmar eller gäster; en B2B-gäst är ett objekt i resursklientorganisationen med koppling till en extern identitet. Application Objects har ett globalt klient-/applikations-ID, medan Service Principals dessutom har ett eget Object ID i respektive klientorganisation. För join och synkronisering blir ytterligare ankare som `onPremisesImmutableId`, enhets-ID:n och källattribut relevanta.

Administratörer dokumenterar minst:

- klientorganisations-ID, primära och verifierade domäner;
- Object ID i stället för endast visningsnamn eller UPN;
- Home Tenant jämfört med Resource Tenant;
- källsystem och Source of Authority per attribut;
- tillstånd för mjuk borttagning/återställning och livscykel;
- licens-, roll- och grupprelationer som separata objekt.

Microsoft skiljer mellan Single Tenant- och Multi Tenant-applikationer utifrån vilka kataloger som får använda konton och Service Principals ([Microsoft identity platform – Single- and multi-tenant apps](https://learn.microsoft.com/en-us/entra/identity-platform/single-and-multi-tenant-apps)).

## Entra ID är inte Active Directory Domain Services

AD DS använder domäner, skogar, domänkontrollanter, LDAP, Kerberos, NTLM, DNS-integration, grupprinciper och replikering. Entra ID använder klientorganisationsobjekt, HTTPS-slutpunkter, moderna federations-/tokenprotokoll och molnbaserad policy. Det exponerar ingen allmän LDAP- eller Kerberos-slutpunkt för applikationer.

| Egenskap | AD DS | Entra ID |
|---|---|---|
| Topologi | Skog, domän, platser, domänkontrollant | Klientorganisation och global molntjänst |
| Primära protokoll | Kerberos, LDAP, DNS, SMB/RPC | OAuth 2.0, OpenID Connect, SAML, HTTPS/Graph |
| Enhetsanslutning | Domain Join och datorobjekt | Registered, Entra Joined, Hybrid Joined |
| Policy | GPO, ACL:er, Kerberos-/LDAP-konfiguration | Conditional Access, roller, Consent, token-/apppolicy |
| Applikation | Service Account/SPN, LDAP Bind, Kerberos | App Registration, Service Principal, Managed Identity |

Microsofts jämförelse pekar uttryckligen på de olika protokoll- och administrationsmodellerna ([Microsoft Learn – Compare Active Directory to Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/fundamentals/compare)). En appliance med [LDAP](/kb/ldap)-bindning kan inte autentisera direkt mot Entra ID utan gateway eller Domain Services; ett OAuth-API förstår omvänt ingen [Kerberos](/kb/kerberos)-biljett.

## Protokollslutpunkter och metadata

OpenID Connect Discovery-slutpunkten publicerar issuer, Authorization Endpoint, Token Endpoint, JWKS URI och funktioner som stöds. Klienter använder klientorganisationsspecifika authorities, till exempel ett konkret klientorganisations-ID; `common`, `organizations` eller `consumers` tillåter bredare kontotyper och ändrar issuer-kontroll och klientorganisationsbehörighet.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Entra-Protokollmetadaten">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$tenant = $env:ENTRA_TENANT_ID
$metadata = Invoke-RestMethod "https://login.microsoftonline.com/$tenant/v2.0/.well-known/openid-configuration"
$metadata | Select-Object issuer,authorization_endpoint,token_endpoint,jwks_uri
Invoke-RestMethod $metadata.jwks_uri | Select-Object -ExpandProperty keys | Select-Object kid,kty,use</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">metadata="https://login.microsoftonline.com/$ENTRA_TENANT_ID/v2.0/.well-known/openid-configuration"
curl --silent --show-error --fail "$metadata" | jq '{issuer,authorization_endpoint,token_endpoint,jwks_uri}'
jwks_uri="$(curl --silent --show-error --fail "$metadata" | jq -r .jwks_uri)"
curl --silent --show-error --fail "$jwks_uri" | jq '.keys[] | {kid,kty,use}'</code></pre>
  </div>
</div>

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod), [`curl`](https://curl.se/docs/manpage.html) och [`jq`](https://jqlang.org/manual/) läser offentlig metadata. De utför ingen tokenvalidering. OIDC Discovery- och JWKS-format är standardiserade ([OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html), [RFC 7517 – JSON Web Key](https://www.rfc-editor.org/rfc/rfc7517.html)).

När klientorganisation, domän och objektankare har klargjorts följer själva inloggningsvägen. OAuth 2.0, OpenID Connect och SAML fyller olika uppgifter och levererar olika artefakter.

## OAuth 2.0, OpenID Connect och SAML

OAuth 2.0 delegerar auktorisering för resurs-API:er. OpenID Connect lägger till ett autentiseringslager med ID Token och UserInfo. SAML transporterar signerade assertioner mellan Identity Provider och Service Provider. Microsoft publicerar de protokollvarianter som Identity Platform stöder ([Microsoft identity platform protocols](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols), [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html), [OASIS SAML 2.0 Technical Overview](https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html)).

De vanliga OAuth-flödena skiljer mellan identitet och klienttyp:

- **Authorization Code med PKCE:** interaktiv användare, Public eller Confidential Client.
- **Client Credentials:** arbetsbelastning utan användare; Application Permissions/App Roles.
- **On-Behalf-Of:** mellanliggande API byter en användartoken mot en token för ett underordnat API.
- **Device Code:** enhet utan bekväm webbläsare; koden bekräftas på en annan enhet.
- **Refresh Token:** förnyar åtkomst utan fullständig interaktiv inloggning, men är fortfarande underkastad policy- och återkallelsehändelser.

Microsoft dokumenterar varje flöde med egna gränser för begäran, autentiseringsuppgifter och säkerhet ([Authorization Code Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow), [Client Credentials Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow), [On-Behalf-Of Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow), [Device Authorization Grant](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-device-code)).

## Tokentyper och claims

En **ID Token** är avsedd för klienten och styrker en autentisering. En **Access Token** är avsedd för ett resurs-API; endast denna resurs ska validera den. En **Refresh Token** är en autentiseringsuppgift gentemot Authorization Server och skickas aldrig till ett resurs-API ([Microsoft identity platform – Security tokens](https://learn.microsoft.com/en-us/entra/identity-platform/security-tokens)).

För JWT-baserade token är bland annat följande relevanta:

| Claim | Operativ betydelse |
|---|---|
| `iss` | förväntad issuer inklusive semantik för klientorganisation/slutpunkt |
| `aud` | resurs-API som token är avsedd för |
| `tid` | klientorganisationskontext |
| `oid` / `sub` | objekt- respektive subjektrelaterat ankare med olika omfång |
| `scp` | delegerade scopes för en användartoken |
| `roles` | Application Roles respektive rollclaims |
| `exp`, `nbf`, `iat` | tidsgränser; servertid och tolerans är relevanta |
| `amr`, `acr` | autentiseringsinformation; får inte universellt tolkas som policyersättning |
| `groups` | gruppclaim eller overage-signal när mängden inte ryms i token |

Valideringen kontrollerar signatur, algoritm, issuer, audience, tid och applikationsspecifika claims. JWT Best Current Practices varnar för algoritmförväxling, Cross-JWT-Confusion och blint förtroende för mottagna claims ([RFC 8725 – JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html), [Microsoft – Validate tokens](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens#validate-tokens)).

En token kan dekodas lokalt för diagnostik; riktiga token kopieras varken till externa webbplatser eller ärenden. Avkodning är ingen signaturkontroll.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für sichere Tokenanalyse">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$parts = $env:ACCESS_TOKEN -split '\.'
$payload = $parts[1].Replace('-','+').Replace('_','/')
$payload += '=' * ((4 - $payload.Length % 4) % 4)
$claims = [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($payload)) | ConvertFrom-Json
$claims | Select-Object iss,aud,tid,oid,sub,scp,roles,iat,nbf,exp</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">payload="$(printf '%s' "$ACCESS_TOKEN" | cut -d. -f2 | tr '_-' '/+')"
case $((${#payload} % 4)) in 2) payload="${payload}==";; 3) payload="${payload}=";; esac
printf '%s' "$payload" | base64 --decode 2&gt;/dev/null | jq '{iss,aud,tid,oid,sub,scp,roles,iat,nbf,exp}'</code></pre>
  </div>
</div>

[`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json), [`cut`](https://www.gnu.org/software/coreutils/manual/html_node/cut-invocation.html), [`tr`](https://www.gnu.org/software/coreutils/manual/html_node/tr-invocation.html) och [`base64`](https://www.gnu.org/software/coreutils/manual/html_node/base64-invocation.html) bearbetar endast en lokal kopia. Efter analysen kasseras den; loggar och shell-historik får inte lagra token.

## Application Object och Service Principal

En App Registration skapar ett **Application Object** i Home Tenant. Det beskriver bland annat Client ID, Redirect URI:er, App Roles, begärda API-behörigheter och autentiseringsuppgifter. När applikationen används eller ges consent i en klientorganisation finns där en **Service Principal** som en klientorganisationslokal instans. Managed Identities är särskilda Service Principals vars livscykel för autentiseringsuppgifter hanteras av Azure ([Application objects and service principals](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals)).

Dessa ID:n får inte förväxlas:

- `appId`/Client ID identifierar applikationsdefinitionen på protokollnivå;
- Application Object ID identifierar registreringsobjektet i Home Tenant;
- Service Principal Object ID identifierar instansen i Resource Tenant;
- Resource-/API-App ID respektive Audience identifierar tokenmålet.

Ett visningsnamn är inte unikt och inte ett ankare för automatisering. Att ta bort och skapa på nytt kan inte godtyckligt återskapa samma Client ID och skapar nya objektrelationer.

En giltig token besvarar ännu inte vad en applikation får göra. Delegerade och rena applikationsbehörigheter skiljer sig åt genom huruvida en användarkontext ingår och vem som godkänner rättigheterna.

## Delegated Permissions, Application Permissions och Consent

Delegated Permissions verkar i kontexten för en inloggad användare och visas vanligen som `scp`. Appen kan inte automatiskt göra mer än vad användare och klientorganisationspolicy tillåter. Application Permissions verkar utan användare som App Roles i `roles`-claimen och kan möjliggöra omfattande åtkomst ([Permissions and consent overview](https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview)).

Consent skapar klientorganisationslokala grants och App Role-tilldelningar. Admin Consent är inte bara en bekräftelse av en dialogruta, utan en auktoriseringsändring. Inventering och granskning omfattar:

- Client- och Resource-Service Principal;
- delegerade scope-grants och Application App-Role-Assignments;
- vem som gav consent, när och via vilken process;
- faktisk användning, Publisher Verification och ursprung;
- objekt-/postlådebegränsning i resursen, om tillgängligt;
- effekt vid återkallelse och driftstopp.

I Exchange Online kan en Application Permission dessutom begränsas genom en Exchange-specifik åtkomstpolicy eller RBAC for Applications. Graph-/Entra-consentobjektet ensamt återger inte den verksamhetsmässiga postlåderäckvidden fullt ut ([Exchange Online – Role Based Access Control for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)).

## Workload Identities och autentiseringsuppgifter

En Workload Identity är en programvaruidentitet, vanligtvis en Service Principal eller Managed Identity. Autentiseringsuppgifter kan vara Client Secret, certifikat/privat nyckel, Managed Identity-plattformautentiseringsuppgift eller Federated Identity Credential. Hemligheter är bearer credentials; certifikat och federation förbättrar nyckelinnehav eller undviker lagrade långlivade hemligheter, men kräver egen trustdrift ([Microsoft Entra Workload ID overview](https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-overview)).

**Workload Identity Federation** förlitar sig på en extern OIDC-issuer och binder issuer, subject och audience till en Service Principal eller en User-Assigned Managed Identity. CI-systemet, Kubernetes-tjänstekontot eller molnarbetsbelastningen byter sin kortlivade externa token mot en Entra-token utan att lagra en statisk hemlighet ([Workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)).

Drift av autentiseringsuppgifter omfattar utfärdare, lagring, åtkomst till privat nyckel, NotBefore/Expiry, överlappning, rotation, senaste användning och återkallelse. Utgångsdatumet i Application Object bevisar inte att en integration faktiskt använder denna autentiseringsuppgift.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Entra-App- und Credentialinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Connect-MgGraph -Scopes 'Application.Read.All','Directory.Read.All'
Get-MgContext
$app = Get-MgApplication -Filter "appId eq '$env:CLIENT_ID'"
$sp = Get-MgServicePrincipal -Filter "appId eq '$env:CLIENT_ID'"
$app | Select-Object Id,AppId,DisplayName,SignInAudience,PasswordCredentials,KeyCredentials
$sp | Select-Object Id,AppId,DisplayName,ServicePrincipalType,AccountEnabled</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">az account show --output json | jq '{tenantId,user,name}'
az ad app list --filter "appId eq '$CLIENT_ID'" --output json | jq '.[] | {id,appId,displayName,signInAudience,passwordCredentials,keyCredentials}'
az ad sp list --filter "appId eq '$CLIENT_ID'" --output json | jq '.[] | {id,appId,displayName,servicePrincipalType,accountEnabled}'</code></pre>
  </div>
</div>

[`Connect-MgGraph`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/connect-mggraph), [`Get-MgContext`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/get-mgcontext), [`Get-MgApplication`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.applications/get-mgapplication) och [`Get-MgServicePrincipal`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.applications/get-mgserviceprincipal) visar Graph-kontexten och båda objekttyperna. [`az account show`](https://learn.microsoft.com/en-us/cli/azure/account#az-account-show), [`az ad app`](https://learn.microsoft.com/en-us/cli/azure/ad/app) och [`az ad sp`](https://learn.microsoft.com/en-us/cli/azure/ad/sp) visar Azure CLI-vyn. Utdata kan innehålla metadata om autentiseringsuppgifter och klientorganisationsinventering.

## Conditional Access och Continuous Access Evaluation

Conditional Access behandlar signaler som användare/arbetsbelastning, målresurs, enhet, plats, risk, klienttyp och autentiseringsstyrka och tillämpar grant- eller session controls. Policyer samverkar; en lyckad enskild policy är inte ett samlat resultat ([Conditional Access overview](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview)).

Report-only och What If hjälper vid planering, men ersätter inte en pilotgrupp med riktiga klienter. Särskilda fall relevanta för meddelanden är Legacy Authentication, SMTP AUTH, mobilklienter, tjänstekonton, Break Glass-konton, administratörsportaler och non-interactive sign-ins.

Access Tokens är normalt giltiga fram till utgången. Continuous Access Evaluation gör det möjligt för resurser och klienter som stöds att ta hänsyn till kritiska händelser och policyändringar tidigare ([Continuous Access Evaluation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation)). CAE är ingen global omedelbar återkallelse för varje applikation; resursen och klienten måste stödja protokollet.

## MFA, Authentication Methods och Authentication Strengths

MFA är ett policyuttalande, inte en enskild produktmetod. Authentication Methods Policies styr vilka metoder som får registreras och användas; Conditional Access Authentication Strengths kan kräva konkreta metodkombinationer ([Authentication methods overview](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-methods), [Conditional Access authentication strengths](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-strengths)).

Nätfiskeresistenta metoder som FIDO2/Passkeys eller certifikatbaserad autentisering förändrar registrerings-, återställnings- och enhetsprocesser. Temporary Access Pass kan möjliggöra bootstrap och är självt en tidsbegränsad autentiseringsuppgift. Helpdesk-återställning, ny telefonregistrering och förlorade enheter är arbetsflöden med hög risk och hör hemma i granskningsspåret.

## Directory Roles, PIM och Administrative Units

Directory Roles auktoriserar administrationsfunktioner för Entra och Microsoft 365. Roller kan tilldelas direkt, via grupper eller tidsbegränsat genom Privileged Identity Management. PIM skiljer mellan eligible och active, approval, MFA, motivering, varaktighet och Access Reviews ([Microsoft Entra PIM overview](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)).

Administrative Units kan begränsa omfånget för utvalda roller till delmängder av objekt. Inte varje roll eller åtgärd stöder detta omfång; Graph-, Exchange- och säkerhetsportaler har delvis egna RBAC-modeller ([Administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/administrative-units)).

Privilegium dokumenteras som en kedja: tilldelningskälla → aktivering → token/roll → målservice-RBAC → konkret objektåtgärd. ”Global Administrator finns” förklarar inte automatiskt en 403 i Exchange eller Graph.

## Enhetsidentitet och Primary Refresh Token

Entra Registered, Entra Joined och Hybrid Entra Joined är olika enhetsrelationer. Registered kopplar vanligtvis en personlig eller externt hanterad enhet; Joined använder Entra som primär organisationskoppling; Hybrid Joined kopplar AD DS Domain Join med Entra-registrering ([Microsoft Entra device identity](https://learn.microsoft.com/en-us/entra/identity/devices/overview)).

I Windows stöder Primary Refresh Token Single Sign-On och bär enhets- samt autentiseringskontext. Utfärdande och förnyelse av PRT beror på användar-, enhets- och nyckeltillstånd; det är ingen vanlig Refresh Token som administratörer ska kopiera ([Primary Refresh Token](https://learn.microsoft.com/en-us/entra/identity/devices/concept-primary-refresh-token)).

Enhetsstatus, MDM-compliance, Hybrid Join, certifikat och Conditional Access kan vara tidsmässigt osynkroniserade. Diagnostik kontrollerar därför tillsammans lokal join-information, Entra-enhetsobjekt, hanteringsobjekt, sign-in-logg och policyresultat.

I hybridmiljöer börjar identiteten ofta i det lokala Active Directory. Synkronisering överför utvalda objekt och attribut, men gör inte Entra ID till en LDAP- eller Kerberos-ersättning.

## Hybrididentitet: Connect Sync och Cloud Sync

Microsoft Entra Connect Sync kör en synkroniseringsmotor på Windows Server och kan representera omfattande regler och vissa hybridfunktioner. Cloud Sync använder lättviktiga Provisioning Agents och molnstyrd konfiguration; funktionsomfång och topologi skiljer sig åt ([Microsoft Entra Connect overview](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-azure-ad-connect), [Microsoft Entra Cloud Sync overview](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync)).

Ett synkroniseringssystem har tre nivåer:

1. **Connector Spaces** eller käll-/målconnector med importerade objekt;
2. **Metaverse-/mappningslogik** respektive Cloud Provisioning-konfiguration;
3. **Export** till Entra med målobjekt, attributflöde och feltillstånd.

Matchning och Source Anchor förhindrar dubbletter. Filter, joinregler, attributprioritet och writeback definierar dataägarskap. En lyckad scheduler-körning bevisar inte att varje objekt har exporterats; karantän, provisioningfel och bearbetning i molntjänsten kontrolleras separat.

### Autentisering i hybriddrift

- **Password Hash Synchronization (PHS):** en härledd hash lagras i Entra; molnautentisering är möjlig vid on-prem-fel.
- **Pass-through Authentication (PTA):** agenter validerar lösenord mot AD DS; agentvägen blir driftsrelevant.
- **Federation:** Entra dirigerar autentisering till ett federerat STS; certifikat, claims rules och tillgänglighet utökar felområdet.

Microsoft beskriver urval och avvägningar för dessa inloggningsmetoder ([Choose the right authentication method](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/choose-ad-authn)). Seamless SSO och PRT är ytterligare mekanismer och inte synonyma med den primära autentiseringsmetoden.

## Entra Domain Services

Microsoft Entra Domain Services tillhandahåller en hanterad domän med Domain Join, grupprinciper, LDAP, Kerberos och NTLM. Användare, grupper och autentiseringshashar synkroniseras från Entra till den hanterade domänen. Kunder får inga Domain Admin- eller Enterprise Admin-rättigheter och hanterar inte domänkontrollanter själva ([Microsoft Entra Domain Services overview](https://learn.microsoft.com/en-us/entra/identity/domain-services/overview)).

Domain Services är ingen dubbelriktad brygga till AD DS och ingen ersättning för Entra-tokenprotokoll. Äldre applikationer ser LDAP-/Kerberosobjekt i den hanterade domänen; moderna SaaS-applikationer fortsätter att använda Entra. Secure LDAP kräver certifikat, publicerad nätverksgräns, NSG/firewall och credentialpolicy.

## External Identities och Cross-Tenant-åtkomst

B2B Collaboration skapar gästobjekt i Resource Tenant, medan autentisering ofta sker i Home Tenant. Cross-Tenant Access Settings styr inbound och outbound trust, förtroende för MFA-/enhetsclaims och organisatoriska relationer ([Microsoft Entra B2B collaboration overview](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b), [Cross-tenant access overview](https://learn.microsoft.com/en-us/entra/external-id/cross-tenant-access-overview)).

Resource Tenant auktoriserar fortsatt sina data. Ett borttaget eller inaktiverat hemkonto, ett befintligt gästobjekt, Access Packages och gruppmedlemskap kan ha olika livscykler. Periodiska Access Reviews och sponsor-/ägarprocesser sluter denna lucka.

## Microsoft Graph som Control Plane

Microsoft Graph exponerar resurssökvägar som `/users`, `/groups`, `/applications`, `/servicePrincipals`, `/policies` och `/auditLogs`. Behörighet, API-version, paging, throttling och eventual consistency för vissa frågor ingår i avtalet ([Microsoft Graph overview](https://learn.microsoft.com/en-us/graph/overview)).

Delta Query levererar ändringar sedan ett initialt tillstånd via ogenomskinliga token; Change Notifications skickar webhooks, men ersätter inte en avstämningskörning ([Microsoft Graph delta query](https://learn.microsoft.com/en-us/graph/delta-query-overview), [Microsoft Graph change notifications](https://learn.microsoft.com/en-us/graph/change-notifications-overview)). För [API:er](/kb/apis) gäller även här idempotens, pagination, 429/Retry-After och Request ID.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Microsoft-Graph-Objektprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Connect-MgGraph -Scopes 'User.Read.All','AuditLog.Read.All'
Get-MgContext
Get-MgUser -UserId $env:USER_OBJECT_ID -Property Id,UserPrincipalName,AccountEnabled,OnPremisesSyncEnabled,ProxyAddresses
Invoke-MgGraphRequest -Method GET -Uri "v1.0/auditLogs/signIns?`$filter=userId eq '$env:USER_OBJECT_ID'&amp;`$top=20"</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">az account get-access-token --resource-type ms-graph --output json | jq '{tenant,expires_on}'
token="$(az account get-access-token --resource-type ms-graph --query accessToken -o tsv)"
curl --silent --show-error --fail \
  --header "Authorization: Bearer $token" \
  "https://graph.microsoft.com/v1.0/users/$USER_OBJECT_ID?%24select=id,userPrincipalName,accountEnabled,onPremisesSyncEnabled,proxyAddresses" | jq .</code></pre>
  </div>
</div>

[`Get-MgUser`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguser) och [`Invoke-MgGraphRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/invoke-mggraphrequest) använder den visade Graph-kontexten. [`az account get-access-token`](https://learn.microsoft.com/en-us/cli/azure/account#az-account-get-access-token) levererar en kortlivad CLI-token; den loggas inte och lagras inte permanent. Graph-svar innehåller personuppgifter och säkerhetsrelevant data.

## Nätverks- och TLS-beroenden

Entra är en distribuerad HTTPS-tjänst. Proxy, brandvägg, DNS, TLS-inspektion, tid och endpoint allowlist påverkar autentisering, Graph, Device Registration, Sync och återkallelse. En statisk IP-lista är inte alltid rätt modell; Microsoft publicerar Service Tags och URL-/IP-kategorier för Microsoft 365 och Entra-relaterade slutpunkter ([Microsoft 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges), [Azure service tags overview](https://learn.microsoft.com/en-us/azure/virtual-network/service-tags-overview)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Entra-Netzwerkdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName login.microsoftonline.com
Test-NetConnection login.microsoftonline.com -Port 443 -InformationLevel Detailed
Invoke-WebRequest "https://login.microsoftonline.com/$env:ENTRA_TENANT_ID/v2.0/.well-known/openid-configuration" -UseBasicParsing
w32tm /query /status</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig login.microsoftonline.com A
nc -vz login.microsoftonline.com 443
openssl s_client -connect login.microsoftonline.com:443 -servername login.microsoftonline.com -alpn h2,http/1.1 &lt;/dev/null
timedatectl show --property=NTPSynchronized --property=TimeUSec</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) kontrollerar först namnupplösningen. [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection), [`nc`](https://man.openbsd.org/nc) och [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) separerar TCP och [TLS](/kb/tls). [`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) kontrollerar HTTP, [`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings) och [`timedatectl`](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html) tidskällan.

## Sign-in Logs, Audit Logs och Provisioning Logs

Sign-in Logs dokumenterar interaktiva, non-interactive, Service Principal- och Managed Identity-inloggningar med status, Conditional Access-utvärdering, klient-, enhets- och riskinformation. Audit Logs dokumenterar katalog- och policyändringar. Provisioning Logs visar etableringssteg och mappningsfel ([Microsoft Entra sign-in logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-sign-ins), [Microsoft Entra audit logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-audit-logs), [Provisioning logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-provisioning-logs)).

Correlation ID, Request ID, tid i UTC, klientorganisation, Client ID, Resource ID, användar-/Service Principal-ID och felkod utgör minsta bevisunderlag. Portaltexter kompletteras med felkod och token-/protokollfas; samma UI-meddelanden kan ha olika orsaker.

Loggkvarhållning beror på licens och exportkonfiguration och kan ändras. Långsiktiga bevis skickas före incidenten via Diagnostic Settings eller stödda exportvägar till ett kontrollerat loggsystem.

## Break Glass, backup och återställning

Entra är en SaaS-tjänst; kunder säkerhetskopierar inte någon domänkontrollants databas. De måste dock skydda konfiguration, externa autentiseringsuppgifter och administrativ återställbarhet:

- minst två cloud-only Emergency Access Accounts med oberoende starka metoder;
- undantag från Conditional Access endast så långt som behövs för återställning och under övervakning;
- konfiguration för klientorganisation/domän/federation/cross-tenant och rollexport;
- App Registrations, Service Principals, grants, App Roles och metadata om autentiseringsuppgifter;
- policies för Conditional Access, Authentication Methods, PIM och livscykel;
- hybridsynkroniseringsregler, Source Anchors, agent-/serverkonfiguration och stagingväg;
- externa beroenden av CA/KMS/federation/DNS/Break Glass;
- export av Audit-/Sign-in-Logs och ändringsärenden.

Microsoft rekommenderar dedikerade Emergency Access Accounts vars användning larmar och testas regelbundet ([Manage emergency access accounts](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access)).

Mjukt borttagna objekt har produktspecifika återställningsfönster och begränsningar som kan ändras. Återställningsrunbooks länkar därför till respektive Microsoft-dokumentation och testar separat användare, grupp, app, Service Principal, felkonfiguration av Conditional Access och förlorad federations-/domänväg.

Felsökningen följer tokenbeslutet: identifiera användare och applikation, kontrollera inloggningsmetod och Conditional Access, läs tokenclaims och utvärdera slutligen auktoriseringen på resursen.

## Diagnostik enligt beslutsfas

Ett Entra-fel kan avgränsas snabbare om fasen först fastställs: inloggning, tokenutfärdande eller målprogrammets beslut. Tabellen kopplar lämpliga bevis till varje fas.

| Fas | Bevis | Typisk felklass |
|---|---|---|
| Discovery | Authority, klientorganisations-ID, OIDC-metadata, DNS/TLS | fel klientorganisation/moln, proxy, tid, endpoint |
| Klient | Client ID, Redirect URI, flow, credentialtyp | appdefinition, Reply URL, Secret/Cert/Federation |
| Autentisering | Sign-in Log, metod, Device, risk | credential, MFA, användare/arbetsbelastning inaktiverad |
| Conditional Access | policyresultat, Target Resource, Conditions | scope, Grant Control, Device, Location, Client |
| Token | `iss`, `aud`, `tid`, `scp`/`roles`, tid | fel Resource/Authority, Consent, claim/klocka |
| Resource API | Request ID, Resource-RBAC, objektpolicy | Graph-/Exchange-roll, appåtkomst, objektomfång |
| Provisioning/Sync | Connector/Agent, mapping, Export/Provisioning Log | Source Anchor, filter, dubblett, karantän |

Ett Entra-fel löses inte genom godtyckliga nya inloggningar. Först fasen avgör om nätverk, klientkonfiguration, identitet, policy, token eller resursauktorisering måste korrigeras.

## Teknisk historik

Microsoft utvecklade Windows Azure Active Directory som en multitenant molnkatalog- och federationstjänst för Microsoft Online Services och tredjepartsapplikationer. Azure AD använde moderna tokenprotokoll och var arkitektoniskt aldrig en värdbaserad AD DS-domänkontrollant. Microsoft Graph ersatte flera äldre API:er som gemensamt molnkontrollplan.

Hybrid Identity uppstod ur synkroniserings- och federationsverktyg som DirSync, Azure AD Sync, AD FS och senare Azure AD Connect. Password Hash Sync, Pass-through Authentication, Seamless SSO, Cloud Sync och Workload Federation kompletterade olika driftmodeller. Managed Identities flyttade rotation av autentiseringsuppgifter för Azure-arbetsbelastningar till plattformen.

År 2023 bytte Microsoft namn på Azure Active Directory till **Microsoft Entra ID**; protokollslutpunkter, API:er, klientorganisations- och licensgrunder förblev del av samma tjänstekontinuitet ([Microsoft – New name for Azure Active Directory](https://learn.microsoft.com/en-us/entra/fundamentals/new-name)). Historiken förklarar dagens blandning av `login.microsoftonline.com`, `AzureAD`-äldre beteckningar, Microsoft Graph, AD DS-hybridtermer och Entra-produktnamn.

## Adminchecklista i korthet

Avslutningsvis betraktas identiteter, applikationer, roller, policyer och återställning tillsammans. Checklistan fungerar som ett kort driftbevis för dessa sammanhängande kontroller.

| Fråga | Driftbevis |
|---|---|
| Vilken klientorganisation? | klientorganisations-ID, Cloud/Authority, verifierad domän, Home-/Resource Tenant |
| Vilket objekt? | Object ID, typ, Source of Authority, UPN/Mail endast som attribut, borttagningstillstånd |
| Vilken klient? | App-/Client ID, Application Object, Service Principal i Resource Tenant, Redirect/Flow |
| Vilken identitet? | användare/gäst/enhet/Service Principal/Managed Identity, credential- eller federationskälla |
| Vilken token? | typ, issuer, audience, klientorganisation, subject/object, `scp`/`roles`, tid och signaturnyckel-ID |
| Vilken auktorisering? | Consent Grant/App-Role, Directory-/Resource-RBAC, objekt-/postlådescope |
| Vilken policy? | Conditional Access, Auth Strength, risk, Device, Location, Session/CAE |
| Vilken hybridkälla? | Connect/Cloud Sync, Source Anchor, filter, mapping, exportstatus, autentiseringsmetod |
| Vilka bevis? | UTC, Correlation-/Request ID, Sign-in/Audit/Provisioning Log, Graph-status |
| Hur återställs det? | Emergency Accounts, policies/roller/appar/grants, hybridkonfiguration, domäner/federation, externa nycklar och loggar |

## Källor

- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Entra fundamentals – What is Microsoft Entra ID?](https://learn.microsoft.com/en-us/entra/fundamentals/whatis)
- [Microsoft identity platform – Single- and multi-tenant apps](https://learn.microsoft.com/en-us/entra/identity-platform/single-and-multi-tenant-apps)
- [Microsoft Learn – Compare AD DS to Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/fundamentals/compare)
- [Microsoft Learn – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [curl manual](https://curl.se/docs/manpage.html)
- [jq manual](https://jqlang.org/manual/)
- [OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html)
- [RFC 7517 – JSON Web Key](https://www.rfc-editor.org/rfc/rfc7517.html)
- [Microsoft identity platform protocols](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [OASIS SAML 2.0 Technical Overview](https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html)
- [Authorization Code Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow)
- [Client Credentials Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow)
- [On-Behalf-Of Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow)
- [Device Authorization Grant](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-device-code)
- [Microsoft identity platform – Security tokens](https://learn.microsoft.com/en-us/entra/identity-platform/security-tokens)
- [RFC 8725 – JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html)
- [Microsoft identity platform – Access tokens and validation](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens)
- [Microsoft Learn – ConvertFrom-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json)
- [GNU coreutils – cut](https://www.gnu.org/software/coreutils/manual/html_node/cut-invocation.html)
- [GNU coreutils – tr](https://www.gnu.org/software/coreutils/manual/html_node/tr-invocation.html)
- [GNU coreutils – base64](https://www.gnu.org/software/coreutils/manual/html_node/base64-invocation.html)
- [Application objects and service principals](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals)
- [Permissions and consent overview](https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview)
- [Exchange Online – RBAC for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)
- [Microsoft Entra Workload ID overview](https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-overview)
- [Workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)
- [Microsoft Graph PowerShell – Connect-MgGraph](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/connect-mggraph)
- [Microsoft Graph PowerShell – Get-MgContext](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/get-mgcontext)
- [Microsoft Graph PowerShell – Get-MgApplication](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.applications/get-mgapplication)
- [Microsoft Graph PowerShell – Get-MgServicePrincipal](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.applications/get-mgserviceprincipal)
- [Azure CLI – az account show](https://learn.microsoft.com/en-us/cli/azure/account)
- [Azure CLI – az ad app](https://learn.microsoft.com/en-us/cli/azure/ad/app)
- [Azure CLI – az ad sp](https://learn.microsoft.com/en-us/cli/azure/ad/sp)
- [Conditional Access overview](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview)
- [Continuous Access Evaluation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation)
- [Authentication methods overview](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-methods)
- [Conditional Access authentication strengths](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-strengths)
- [Microsoft Entra PIM overview](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)
- [Administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/administrative-units)
- [Microsoft Entra device identity](https://learn.microsoft.com/en-us/entra/identity/devices/overview)
- [Primary Refresh Token](https://learn.microsoft.com/en-us/entra/identity/devices/concept-primary-refresh-token)
- [Microsoft Entra Connect overview](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-azure-ad-connect)
- [Microsoft Entra Cloud Sync overview](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync)
- [Choose the right authentication method](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/choose-ad-authn)
- [Microsoft Entra Domain Services overview](https://learn.microsoft.com/en-us/entra/identity/domain-services/overview)
- [Microsoft Entra B2B collaboration overview](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b)
- [Cross-tenant access overview](https://learn.microsoft.com/en-us/entra/external-id/cross-tenant-access-overview)
- [Microsoft Graph overview](https://learn.microsoft.com/en-us/graph/overview)
- [Microsoft Graph delta query](https://learn.microsoft.com/en-us/graph/delta-query-overview)
- [Microsoft Graph change notifications](https://learn.microsoft.com/en-us/graph/change-notifications-overview)
- [Microsoft Graph PowerShell – Get-MgUser](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguser)
- [Microsoft Graph PowerShell – Invoke-MgGraphRequest](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/invoke-mggraphrequest)
- [Microsoft 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges)
- [Azure service tags overview](https://learn.microsoft.com/en-us/azure/virtual-network/service-tags-overview)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – Windows Time tools](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings)
- [systemd – timedatectl](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html)
- [Microsoft Entra sign-in logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-sign-ins)
- [Microsoft Entra audit logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-audit-logs)
- [Microsoft Entra provisioning logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-provisioning-logs)
- [Manage emergency access accounts](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access)
- [Microsoft – New name for Azure Active Directory](https://learn.microsoft.com/en-us/entra/fundamentals/new-name)
