---
title: "Microsoft Entra ID: Tenant, tokens og hybrididentitet"
blatt: "entra"
description: "Microsoft Entra ID for infrastruktur- og meldingsadministratorer: tenant- og katalogobjekter, Security Token Service, OAuth/OIDC og SAML, apper og tjenestehovedenheter, arbeidsbelastningsidentiteter, samtykke og autorisering, Conditional Access, enheter, hybridsynkronisering, Domain Services, Microsoft Graph, logging, gjenoppretting og teknisk historikk."
fakten:
  - label: Systemrolle
    wert: skybasert katalog- og identitetstjeneste for flere leietakere med token-, policy- og administrasjons-control plane
    href: https://learn.microsoft.com/en-us/entra/fundamentals/whatis
  - label: Leietaker
    wert: selvstendig Directory-/policygrense med Tenant-ID, verifiserte domener og objekter
    href: https://learn.microsoft.com/en-us/entra/identity-platform/single-and-multi-tenant-apps
  - label: Protokoller
    wert: OAuth 2.0 · OpenID Connect · SAML 2.0 · WS-Federation over HTTPS
    href: https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols
  - label: Administrasjons-API
    wert: Microsoft Graph for Directory-, Identity-, Policy-, Audit- og applikasjonsobjekter
    href: https://learn.microsoft.com/en-us/graph/overview
  - label: Appmodell
    wert: Application Object som definisjon · Service Principal som lokal instans i tenant
    href: https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals
  - label: Brukertokener
    wert: ID Token for klientøkt · Access Token for Resource API · Refresh Token for fornyelse
    href: https://learn.microsoft.com/en-us/entra/identity-platform/security-tokens
  - label: Arbeidsbelastningsidentitet
    wert: Service Principal eller Managed Identity; sertifikat, hemmelighet eller føderert legitimasjon
    href: https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-overview
  - label: Autorisasjon
    wert: Delegated Scopes · Application Roles · Directory Roles · Resource-/objektpolicy
    href: https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview
  - label: Tilgangspolicy
    wert: Conditional Access evaluerer signaler og håndhever kontroller ved tokenavgjørelser
    href: https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview
  - label: Hybridbro
    wert: Connect Sync eller Cloud Sync replikerer utvalgte AD-DS-objekter og attributter
    href: https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync
  - label: Eldre domenetjenester
    wert: Entra Domain Services tilbyr administrert Kerberos, NTLM, LDAP og Domain Join
    href: https://learn.microsoft.com/en-us/entra/identity/domain-services/overview
  - label: Driftstilstand
    wert: Token-Issuance · policyresultat · Sync/Provisioning · credentialutløp · Sign-in/Audit · Break Glass
    href: https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-sign-ins
werbung:
  - newsletter
ctaThemen:
  - active-directory-entra
  - microsoft-entra
  - powershell
translationSourceHash: 6d1a58b89bfeb1ae9f9b24608e56afa198b91d6e6681bb8692d749a36e5e1ef3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T11:15:35.833Z
translationReview: automatic
---

# Microsoft Entra ID: Tenant, tokens og hybrididentitet

Microsoft Entra ID er en skykatalog, Identity Provider og Policy Enforcement Point for Microsoft- og tredjepartsapplikasjoner. Den lagrer brukere, grupper, enheter, applikasjoner, Service Principals og roller; Security Token Service utsteder signerte tokens etter vellykket autentisering og policykontroll. Microsoft Graph utgjør administrasjons-control plane. Disse rollene må vurderes separat: Et eksisterende brukerobjekt garanterer ikke vellykket pålogging, et gyldig token garanterer ikke faglig objektautorisasjon, og et synkronisert attributt får ikke nødvendigvis umiddelbar effekt i hver måltjeneste.

For meldingsadministratorer ligger Entra i flere kritiske stier: Brukere og `proxyAddresses` mater Exchange Online, grupper styrer distribusjon og tilgang, apprettigheter tillater automatisert postboks- eller Graphtilgang, Conditional Access påvirker administrator- og klientpålogginger, og hybridsynkronisering kobler Active Directory Domain Services til skyen. Tenant er derfor ikke bare «AD på Internett», men en selvstendig identitets- og autorisasjonsplattform.

Forklaringen følger en identitet fra pålogging til tilgang til en konkret ressurs. Først forklares tenant, katalog og token, deretter roller, applikasjoner, enheter, hybridtilkobling, protokoller og gjenoppretting.

Microsoft Entra ID kobler en katalog med tokenutstedelse og policykontroll. En bruker får ikke tilgang «til Entra», men ber om et token for en konkret ressurs; Entra kontrollerer da identitet, applikasjon, enhet og policy.

## Arkitektur: Directory, STS, Policy og Resource

Microsoft beskriver Entra ID som en skybasert identitets- og tilgangsadministrasjonstjeneste. Hovedflatene er:

1. **Directory:** Objekter, attributter, relasjoner, roller, domener og tenantlokal konfigurasjon.
2. **Security Token Service (STS):** Protokollendepunkter, autentisering, tokenutstedelse og nøkkelpublisering.
3. **Policy Engines:** Conditional Access, Identity Protection, Authentication Methods, Consent og andre tilgangsavgjørelser.
4. **Provisioning/Sync:** Replikering eller klargjøring til og fra AD DS, SaaS og HR-kilder.
5. **Microsoft Graph:** API for katalog-, identitets-, revisjons- og policyadministrasjon.
6. **Resource Services:** Exchange Online, Graph, egne API-er og SaaS validerer tokens og håndhever autorisasjonen deres.

Entra støtter flere leietakere. En tenant utgjør en administrativ og policymessig grense, men ikke automatisk fullstendig data- eller nettverksisolasjon for alle SaaS-tjenester som brukes ([Microsoft Entra fundamentals – What is Microsoft Entra ID?](https://learn.microsoft.com/en-us/entra/fundamentals/whatis)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-entra.svg?v=20260813" title="Interaktive Infografik: Microsoft-Entra-Aufruf von Benutzer oder Workload über Authentisierung, Conditional Access und STS bis Token, Resource API und Microsoft Graph sowie Tenant-, App-, Hybrid-, Geräte- und Recoverygrenzen" loading="lazy">
  <a href="/images/kb-interaktiv-entra.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

## Teknologistack fra et administratorperspektiv

For en SaaS-tjeneste med flere leietakere er den interne stakken av programmeringsspråk, databaser og orkestrering ikke en produktegenskap kunden kontrollerer. Å utlede den fra vertsnavn, klientbiblioteker eller stillingsannonser ville være verdiløst for drift og gjenoppretting. Den dokumenterbare **teknologistacken fra et administratorperspektiv** består av de publiserte avtalene: HTTPS som transport, OAuth 2.0, OpenID Connect, SAML og WS-Federation for identitet, JWT/JWS og JWKS for tokens og nøkler, Microsoft Graph som REST-/OData-control plane samt agentbasert synkronisering for hybrididentiteter. Microsoft dokumenterer uttrykkelig disse protokollene og Graph som støttede grensesnitt ([Microsoft identity platform protocols](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols), [Microsoft Graph overview](https://learn.microsoft.com/en-us/graph/overview)).

For administrasjon er derfor ikke et antatt serverspråk avgjørende, men den konkrete kombinasjonen av tenant, authority, protokollversjon, tokenformat, Graph-endepunkt, SDK- eller PowerShell-modul, synkroniseringsagent og Resource Service. Hvert av disse lagene har egne grenser for versjonering, rettigheter, logging og feil.

## Tenant, domener og objektankere

Hver tenant har en uforanderlig Tenant-ID. Verifiserte domenenavn gir lesbare påloggingsnavn og rutingtilknytning, men erstatter ikke Tenant-ID-en. Et domeneendring eller oppsett med flere leietakere må derfor ikke identifiseres utelukkende ut fra UPN-suffikset.

Objekter har tenantlokale ID-er. Brukere kan være Member eller Guest; en B2B-gjest er et objekt i Resource Tenant med tilknytning til en ekstern identitet. Application Objects har en global Client-/Application-ID, og Service Principals har i tillegg sin egen Object-ID i den aktuelle tenant. For join og synkronisering blir ytterligere ankere som `onPremisesImmutableId`, enhets-ID-er og kildeattributter relevante.

Administratorer dokumenterer minst:

- Tenant-ID, primære og verifiserte domener;
- Object-ID i stedet for bare visningsnavn eller UPN;
- Home Tenant kontra Resource Tenant;
- kildesystem og Source of Authority per attributt;
- Soft-Delete-/Restore-tilstand og livssyklus;
- lisens-, rolle- og grupperelasjoner som separate objekter.

Microsoft skiller mellom Single-Tenant- og Multi-Tenant-applikasjoner ut fra hvilke kataloger som kan bruke kontoer og Service Principals ([Microsoft identity platform – Single- and multi-tenant apps](https://learn.microsoft.com/en-us/entra/identity-platform/single-and-multi-tenant-apps)).

## Entra ID er ikke Active Directory Domain Services

AD DS bruker domener, forests, Domain Controllers, LDAP, Kerberos, NTLM, DNS-integrasjon, gruppepolicyer og replikering. Entra ID bruker tenantobjekter, HTTPS-endepunkter, moderne federation-/tokenprotokoller og skybasert policy. Den eksponerer ikke et generelt LDAP- eller Kerberos-endepunkt for applikasjoner.

| Kjennetegn | AD DS | Entra ID |
|---|---|---|
| Topologi | Forest, Domain, Sites, Domain Controller | Tenant og global skytjeneste |
| Primærprotokoller | Kerberos, LDAP, DNS, SMB/RPC | OAuth 2.0, OpenID Connect, SAML, HTTPS/Graph |
| Enhetstilknytning | Domain Join og datamaskinobjekt | Registered, Entra Joined, Hybrid Joined |
| Policy | GPO, ACL-er, Kerberos-/LDAP-konfigurasjon | Conditional Access, roller, Consent, token-/apppolicy |
| Applikasjon | Service Account/SPN, LDAP Bind, Kerberos | App Registration, Service Principal, Managed Identity |

Microsofts sammenligning viser uttrykkelig til de ulike protokoll- og administrasjonsmodellene ([Microsoft Learn – Compare Active Directory to Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/fundamentals/compare)). En appliance med [LDAP](/kb/ldap)-binding kan ikke autentisere direkte mot Entra ID uten gateway eller Domain Services; et OAuth-API forstår omvendt ikke en [Kerberos](/kb/kerberos)-ticket.

## Protokollendepunkter og metadata

OpenID Connect Discovery-endepunktet publiserer Issuer, Authorization Endpoint, Token Endpoint, JWKS URI og støttede funksjoner. Klienter bruker tenantspesifikke authorities som en konkret Tenant-ID; `common`, `organizations` eller `consumers` tillater bredere kontotyper og endrer issuer-kontroll og tenantgodkjenning.

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

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod), [`curl`](https://curl.se/docs/manpage.html) og [`jq`](https://jqlang.org/manual/) leser offentlige metadata. De utfører ingen tokenvalidering. OIDC Discovery- og JWKS-formater er standardiserte ([OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html), [RFC 7517 – JSON Web Key](https://www.rfc-editor.org/rfc/rfc7517.html)).

Når tenant, domene og objektankere er avklart, følger selve påloggingsveien. OAuth 2.0, OpenID Connect og SAML har forskjellige oppgaver og leverer forskjellige artefakter.

## OAuth 2.0, OpenID Connect og SAML

OAuth 2.0 delegerer autorisasjon for Resource API-er. OpenID Connect legger til et autentiseringslag med ID Token og UserInfo. SAML transporterer signerte assertions mellom Identity Provider og Service Provider. Microsoft publiserer de støttede protokollvariantene i Identity Platform ([Microsoft identity platform protocols](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols), [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html), [OASIS SAML 2.0 Technical Overview](https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html)).

De vanlige OAuth-flytene skiller mellom identitet og klienttype:

- **Authorization Code med PKCE:** interaktiv bruker, Public eller Confidential Client.
- **Client Credentials:** arbeidsbelastning uten bruker; Application Permissions/App Roles.
- **On-Behalf-Of:** mellomliggende API bytter et brukertoken mot et token for et etterfølgende API.
- **Device Code:** enhet uten praktisk nettleser; koden bekreftes på en annen enhet.
- **Refresh Token:** fornyer tilgang uten full interaktiv pålogging, men er fortsatt underlagt policy- og revocation-hendelser.

Microsoft dokumenterer hver flyt med egne request-, credential- og sikkerhetsgrenser ([Authorization Code Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow), [Client Credentials Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow), [On-Behalf-Of Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow), [Device Authorization Grant](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-device-code)).

## Tokentyper og claims

Et **ID Token** er beregnet for klienten og dokumenterer en autentisering. Et **Access Token** er beregnet for et Resource API; bare denne ressursen skal validere det. Et **Refresh Token** er en credential mot Authorization Server og sendes aldri til et Resource API ([Microsoft identity platform – Security tokens](https://learn.microsoft.com/en-us/entra/identity-platform/security-tokens)).

For JWT-baserte tokens er blant annet følgende relevant:

| Claim | Driftsbetydning |
|---|---|
| `iss` | forventet issuer, inkludert tenant-/endpoint-semantikk |
| `aud` | Resource API som tokenet er beregnet for |
| `tid` | tenantkontekst |
| `oid` / `sub` | objekt- eller subjektrelatert anker med forskjellig scope |
| `scp` | delegerte scopes for et brukertoken |
| `roles` | Application Roles eller rolleclaims |
| `exp`, `nbf`, `iat` | tidsgrenser; servertid og toleranse er relevant |
| `amr`, `acr` | autentiseringsinformasjon; skal ikke universelt tolkes som policyerstatning |
| `groups` | gruppeclaim eller overagesignal når mengden ikke får plass i tokenet |

Validering kontrollerer signatur, algoritme, issuer, audience, tid og applikasjonsspesifikke claims. JWT Best Current Practices advarer mot algoritmeforveksling, Cross-JWT-Confusion og blind tillit til mottatte claims ([RFC 8725 – JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html), [Microsoft – Validate tokens](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens#validate-tokens)).

Et token kan dekodes lokalt for diagnose; reelle tokens kopieres aldri til fremmede nettsteder eller saker. Dekoding er ikke en signaturkontroll.

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

[`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json), [`cut`](https://www.gnu.org/software/coreutils/manual/html_node/cut-invocation.html), [`tr`](https://www.gnu.org/software/coreutils/manual/html_node/tr-invocation.html) og [`base64`](https://www.gnu.org/software/coreutils/manual/html_node/base64-invocation.html) behandler bare en lokal kopi. Etter analysen forkastes den; logger og shellhistorikk må ikke lagre tokenet.

## Application Object og Service Principal

En App Registration oppretter et **Application Object** i Home Tenant. Det beskriver blant annet Client-ID, Redirect URIs, App Roles, forespurte API-rettigheter og credentials. Når applikasjonen brukes eller consentieres i en tenant, finnes det der en **Service Principal** som tenantlokal instans. Managed Identities er spesielle Service Principals der Azure administrerer credential-livssyklusen ([Application objects and service principals](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals)).

Disse ID-ene må ikke forveksles:

- `appId`/Client-ID identifiserer applikasjonsdefinisjonen på protokollnivå;
- Application Object-ID identifiserer registreringsobjektet i Home Tenant;
- Service-Principal-Object-ID identifiserer instansen i Resource Tenant;
- Resource-/API-App-ID eller Audience identifiserer tokenmålet.

Et visningsnavn er ikke entydig og er ikke et automatiseringsanker. Sletting og nyoppretting kan ikke fritt gjenskape samme Client-ID og oppretter nye objektrelasjoner.

Et gyldig token besvarer fortsatt ikke hva en applikasjon kan gjøre. Delegerte og rene applikasjonsrettigheter skiller seg ved om en brukerkontekst er involvert og hvem som godkjenner rettighetene.

## Delegated Permissions, Application Permissions og Consent

Delegated Permissions virker i konteksten til en pålogget bruker og vises typisk som `scp`. Appen kan ikke automatisk mer enn brukeren og tenantpolicyen tillater. Application Permissions virker uten bruker som App Roles i `roles`-claimet og kan muliggjøre omfattende tilgang ([Permissions and consent overview](https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview)).

Consent oppretter tenantlokale grants og app-rolletildelinger. Admin Consent er ikke bare en bekreftelse av en dialog, men en autorisasjonsendring. Inventar og gjennomgang omfatter:

- Client- og Resource-Service-Principal;
- delegerte scope-grants og Application App-Role-Assignments;
- hvem som consentierte, når og gjennom hvilken prosess;
- faktisk bruk, Publisher Verification og opprinnelse;
- objekt-/postboksbegrensning i ressursen, der det er tilgjengelig;
- effekt av tilbakekalling og utfall.

I Exchange Online kan en Application Permission i tillegg begrenses av Exchange-spesifikk tilgangspolicy eller RBAC for Applications. Graph-/Entra-consentobjektet alene avbilder ikke fullt ut den faglige postboksrekkevidden ([Exchange Online – Role Based Access Control for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)).

## Workload Identities og Credentials

En Workload Identity er en programvareidentitet, vanligvis Service Principal eller Managed Identity. Credentials kan være Client Secret, sertifikat/privat nøkkel, Managed-Identity-plattformcredential eller Federated Identity Credential. Secrets er bearer credentials; sertifikater og federation forbedrer nøkkelbesittelse eller unngår lagrede langvarige secrets, men krever egen tillitsdrift ([Microsoft Entra Workload ID overview](https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-overview)).

**Workload Identity Federation** stoler på en ekstern OIDC-issuer og knytter issuer, subject og audience til en Service Principal eller en User-Assigned Managed Identity. CI-system, Kubernetes-servicekonto eller cloud workload bytter sitt kortvarige eksterne token mot et Entra-token uten å lagre en statisk secret ([Workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)).

Credential-drift omfatter utsteder, lagring, private-key-tilgang, NotBefore/Expiry, overlapp, rotasjon, siste bruk og tilbakekalling. Utløpsdatoen i Application Object beviser ikke at en integrasjon faktisk bruker denne credentialen.

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

[`Connect-MgGraph`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/connect-mggraph), [`Get-MgContext`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/get-mgcontext), [`Get-MgApplication`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.applications/get-mgapplication) og [`Get-MgServicePrincipal`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.applications/get-mgserviceprincipal) viser Graph-kontekst og begge objekttypene. [`az account show`](https://learn.microsoft.com/en-us/cli/azure/account#az-account-show), [`az ad app`](https://learn.microsoft.com/en-us/cli/azure/ad/app) og [`az ad sp`](https://learn.microsoft.com/en-us/cli/azure/ad/sp) viser Azure CLI-visningen. Utdata kan inneholde credentialmetadata og tenantinventar.

## Conditional Access og Continuous Access Evaluation

Conditional Access behandler signaler som bruker/workload, målressurs, enhet, plassering, risiko, klienttype og autentiseringsstyrke, og anvender grant- eller session controls. Policyer virker sammen; en enkelt vellykket policy er ikke et samlet resultat ([Conditional Access overview](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview)).

Report-only og What If hjelper ved planlegging, men erstatter ikke en pilotgruppe med reelle klienter. Meldingsrelevante spesialtilfeller er Legacy Authentication, SMTP AUTH, Mobile Clients, Service Accounts, Break-Glass-kontoer, administratorportaler og non-interactive sign-ins.

Access Tokens er normalt gyldige til de utløper. Continuous Access Evaluation lar støttede resources og klienter ta hensyn til kritiske hendelser og policyendringer tidligere ([Continuous Access Evaluation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation)). CAE er ikke en global umiddelbar tilbakekalling for hver applikasjon; ressurs og klient må støtte protokollen.

## MFA, Authentication Methods og Authentication Strengths

MFA er en policyuttalelse, ikke én enkelt produktmetode. Authentication Methods Policies styrer hvilke metoder som kan registreres og brukes; Conditional Access Authentication Strengths kan kreve konkrete metodekombinasjoner ([Authentication methods overview](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-methods), [Conditional Access authentication strengths](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-strengths)).

Phishingresistente metoder som FIDO2/Passkeys eller sertifikatbasert autentisering endrer enrollment-, recovery- og enhetsprosesser. Temporary Access Pass kan muliggjøre bootstrap og er selv en tidsbegrenset credential. Helpdesk-tilbakestilling, ny telefonregistrering og mistede enheter er arbeidsflyter med høy risiko og hører hjemme i revisjonssporet.

## Directory Roles, PIM og Administrative Units

Directory Roles autoriserer Entra- og Microsoft 365-administrasjonsfunksjoner. Roller kan tildeles direkte, via grupper eller tidsbegrenset gjennom Privileged Identity Management. PIM skiller mellom eligible og active, Approval, MFA, begrunnelse, varighet og Access Reviews ([Microsoft Entra PIM overview](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)).

Administrative Units kan begrense scope for utvalgte roller til delmengder av objekter. Ikke alle roller eller operasjoner støtter dette scopet; Graph-, Exchange- og sikkerhetsportaler har delvis egne RBAC-modeller ([Administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/administrative-units)).

Privilegier dokumenteres som en kjede: tildelingskilde → aktivering → token/rolle → målservice-RBAC → konkret objekthandling. «Global Administrator finnes» forklarer ikke automatisk en 403 i Exchange eller Graph.

## Enhetsidentitet og Primary Refresh Token

Entra Registered, Entra Joined og Hybrid Entra Joined er ulike enhetsrelasjoner. Registered knytter vanligvis en personlig eller eksternt administrert enhet; Joined bruker Entra som primær organisasjonstilknytning; Hybrid Joined kombinerer AD-DS-Domain Join med Entra-registrering ([Microsoft Entra device identity](https://learn.microsoft.com/en-us/entra/identity/devices/overview)).

På Windows støtter Primary Refresh Token Single Sign-On og bærer enhets- og autentiseringskontekst. PRT-utstedelse og -fornyelse avhenger av bruker-, enhets- og nøkkeltilstand; det er ikke et vanlig Refresh Token som administratorer bør kopiere ([Primary Refresh Token](https://learn.microsoft.com/en-us/entra/identity/devices/concept-primary-refresh-token)).

Enhetsstatus, MDM-compliance, Hybrid Join, sertifikater og Conditional Access kan være tidsmessig usynkroniserte. Diagnosen kontrollerer derfor lokal joininformasjon, Entra-enhetsobjekt, administrasjonsobjekt, sign-in-logg og policyresultat samlet.

I hybride miljøer starter identiteten ofte i lokal Active Directory. Synkronisering overfører utvalgte objekter og attributter, men gjør ikke Entra ID til en LDAP- eller Kerberos-erstatning.

## Hybrididentitet: Connect Sync og Cloud Sync

Microsoft Entra Connect Sync kjører en synkroniseringsmotor på Windows Server og kan håndtere omfattende regler og bestemte hybridfunksjoner. Cloud Sync bruker lette Provisioning Agents og skyadministrert konfigurasjon; funksjonsomfang og topologi er forskjellige ([Microsoft Entra Connect overview](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-azure-ad-connect), [Microsoft Entra Cloud Sync overview](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync)).

Et synkroniseringssystem har tre nivåer:

1. **Connector Spaces** eller kilde-/målkobling med importerte objekter;
2. **Metaverse-/mappinglogikk** eller Cloud Provisioning-konfigurasjon;
3. **Export** til Entra med målobjekt, attributtflyt og feiltilstand.

Matching og Source Anchor hindrer duplikater. Filtre, joinregler, attributtprioritet og writeback definerer dataeierskap. En vellykket schedulerkjøring beviser ikke at hvert objekt ble eksportert; karantene, provisioningfeil og skytjenestebehandling kontrolleres separat.

### Autentisering i hybriddrift

- **Password Hash Synchronization (PHS):** en avledet hash lagres i Entra; skyauthentisering er mulig ved feil lokalt.
- **Pass-through Authentication (PTA):** agents validerer passord mot AD DS; agentstien blir driftsrelevant.
- **Federation:** Entra videresender autentisering til en føderert STS; sertifikater, Claims Rules og tilgjengelighet utvider feilområdet.

Microsoft beskriver valg og avveininger for disse påloggingsmetodene ([Choose the right authentication method](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/choose-ad-authn)). Seamless SSO og PRT er tilleggsmekanismer og ikke synonymer for den primære autentiseringsmetoden.

## Entra Domain Services

Microsoft Entra Domain Services tilbyr et administrert domene med Domain Join, gruppepolicyer, LDAP, Kerberos og NTLM. Brukere, grupper og credential-hasher synkroniseres fra Entra til det administrerte domenet. Kunder får ingen Domain- eller Enterprise-Admin-rettigheter og administrerer ikke Domain Controllers selv ([Microsoft Entra Domain Services overview](https://learn.microsoft.com/en-us/entra/identity/domain-services/overview)).

Domain Services er ikke en toveis bro til AD DS og ikke en erstatning for Entra-tokenprotokoller. Eldre applikasjoner ser LDAP-/Kerberosobjekter i Managed Domain; moderne SaaS-applikasjoner bruker fortsatt Entra. Secure LDAP krever sertifikat, publisert nettverksgrense, NSG/firewall og credentialpolicy.

## External Identities og Cross-Tenant-tilgang

B2B Collaboration oppretter gjesteobjekter i Resource Tenant, mens autentisering ofte skjer i Home Tenant. Cross-Tenant Access Settings styrer inbound- og outbound-trust, tillit til MFA-/enhetsclaims og organisatoriske relasjoner ([Microsoft Entra B2B collaboration overview](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b), [Cross-tenant access overview](https://learn.microsoft.com/en-us/entra/external-id/cross-tenant-access-overview)).

Resource Tenant autoriserer fortsatt sine data. En slettet eller deaktivert Home-konto, et eksisterende gjesteobjekt, Access Packages og gruppemedlemskap kan ha ulike livssykluser. Regelmessige Access Reviews og sponsor-/ownerprosesser lukker dette gapet.

## Microsoft Graph som Control Plane

Microsoft Graph eksponerer ressurstier som `/users`, `/groups`, `/applications`, `/servicePrincipals`, `/policies` og `/auditLogs`. Rettighet, API-versjon, paging, throttling og eventual consistency for visse spørringer er en del av avtalen ([Microsoft Graph overview](https://learn.microsoft.com/en-us/graph/overview)).

Delta Query leverer endringer siden en initial tilstand gjennom opake tokens; Change Notifications sender webhooks, men erstatter ikke en reconciliation-kjøring ([Microsoft Graph delta query](https://learn.microsoft.com/en-us/graph/delta-query-overview), [Microsoft Graph change notifications](https://learn.microsoft.com/en-us/graph/change-notifications-overview)). For [API-er](/kb/apis) gjelder også idempotens, paginering, 429/Retry-After og Request-ID.

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

[`Get-MgUser`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguser) og [`Invoke-MgGraphRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/invoke-mggraphrequest) bruker den viste Graph-konteksten. [`az account get-access-token`](https://learn.microsoft.com/en-us/cli/azure/account#az-account-get-access-token) leverer et kortvarig CLI-token; det logges ikke eller lagres permanent. Graph-svar inneholder personopplysninger og sikkerhetsrelevante data.

## Nettverks- og TLS-avhengigheter

Entra er en distribuert HTTPS-tjeneste. Proxy, firewall, DNS, TLS-inspeksjon, tid og endpoint-allowlist påvirker autentisering, Graph, Device Registration, Sync og Revocation. En statisk IP-liste er ikke alltid riktig modell; Microsoft publiserer Service Tags og URL-/IP-kategorier for Microsoft 365- og Entra-relaterte endepunkter ([Microsoft 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges), [Azure service tags overview](https://learn.microsoft.com/en-us/azure/virtual-network/service-tags-overview)).

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) kontrollerer først navneoppløsningen. [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection), [`nc`](https://man.openbsd.org/nc) og [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) skiller TCP og [TLS](/kb/tls). [`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) kontrollerer HTTP, [`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings) og [`timedatectl`](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html) tidskilden.

## Sign-in Logs, Audit Logs og Provisioning Logs

Sign-in Logs dokumenterer interaktive, non-interactive, Service-Principal- og Managed-Identity-pålogginger med status, Conditional-Access-evaluering, klient-, enhets- og risikoinformasjon. Audit Logs dokumenterer Directory- og policyendringer. Provisioning Logs viser klargjøringstrinn og mappingfeil ([Microsoft Entra sign-in logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-sign-ins), [Microsoft Entra audit logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-audit-logs), [Provisioning logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-provisioning-logs)).

Correlation ID, Request ID, tid i UTC, tenant, Client-ID, Resource-ID, bruker-/Service-Principal-ID og feilkode utgjør minimumsdokumentasjonen. Portaltekster kompletteres med feilkode og token-/protokollfase; like UI-meldinger kan ha ulike årsaker.

Loggoppbevaring avhenger av lisens og eksportkonfigurasjon og kan endres. Langsiktig dokumentasjon sendes før hendelsen gjennom Diagnostic Settings eller støttede eksportbaner til et kontrollert loggsystem.

## Break Glass, sikkerhetskopiering og gjenoppretting

Entra er en SaaS-tjeneste; kunder sikkerhetskopierer ikke en Domain Controller-database. De må likevel beskytte konfigurasjon, eksterne credentials og administrativ gjenopprettbarhet:

- minst to skybaserte Emergency Access Accounts med uavhengige sterke metoder;
- unntak fra Conditional Access bare så langt som nødvendig for recovery og med overvåking;
- Tenant-/Domain-/Federation-/Cross-Tenant-konfigurasjon og rolleeksport;
- App Registrations, Service Principals, Grants, App Roles og credentialmetadata;
- Conditional-Access-, Authentication-Methods-, PIM- og Lifecycle-policyer;
- Hybrid Sync-regler, Source Anchors, agent-/serverkonfigurasjon og stagingsti;
- eksterne CA-/KMS-/Federation-/DNS-/Break-Glass-avhengigheter;
- eksport av Audit-/Sign-in-logger og Change Tickets.

Microsoft anbefaler dedikerte Emergency Access Accounts, der bruk varsles og testes regelmessig ([Manage emergency access accounts](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access)).

Soft-deleted objekter har produktspesifikke gjenopprettingsvinduer og begrensninger som kan endres. Recovery-runbooks lenker derfor til den aktuelle Microsoft-dokumentasjonen og tester bruker, gruppe, app, Service Principal, Conditional-Access-feilkonfigurasjon og mistet Federation-/Domain-sti separat.

Feilsøkingen følger tokenavgjørelsen: identifiser bruker og applikasjon, kontroller påloggingsmetode og Conditional Access, les tokenclaims og evaluer til slutt autorisasjonen ved ressursen.

## Diagnose etter beslutningsfase

En Entra-feil kan avgrenses raskere hvis fasen først bestemmes: pålogging, tokenutstedelse eller målprogrammets avgjørelse. Tabellen knytter passende dokumentasjon til hver fase.

| Fase | Evidens | Typisk feilklasse |
|---|---|---|
| Discovery | Authority, Tenant-ID, OIDC Metadata, DNS/TLS | feil tenant/cloud, proxy, tid, endpoint |
| Client | Client-ID, Redirect URI, Flow, credentialtype | appdefinisjon, Reply URL, secret/cert/federation |
| Autentisering | Sign-in Log, metode, Device, risiko | credential, MFA, User/Workload disabled |
| Conditional Access | policyresultater, Target Resource, Conditions | scope, Grant Control, Device, Location, Client |
| Token | `iss`, `aud`, `tid`, `scp`/`roles`, tid | feil Resource/Authority, Consent, claim/clock |
| Resource API | Request-ID, Resource-RBAC, objektpolicy | Graph/Exchange-rolle, App Access, objektomfang |
| Provisioning/Sync | Connector/Agent, Mapping, Export/Provisioning Log | Source Anchor, filter, duplicate, karantene |

En Entra-feil løses ikke ved vilkårlig ny pålogging. Først fasen avgjør om nettverk, klientkonfigurasjon, identitet, policy, token eller ressursautorisasjon må korrigeres.

## Teknisk historikk

Microsoft utviklet Windows Azure Active Directory som en skykatalog- og federationstjeneste for flere leietakere for Microsoft Online Services og tredjepartsapplikasjoner. Azure AD tok i bruk moderne tokenprotokoller og var arkitektonisk aldri en driftet AD-DS-Domain-Controller. Microsoft Graph erstattet flere eldre API-er som felles cloud-control plane.

Hybrid Identity oppstod fra synkroniserings- og federationverktøy som DirSync, Azure AD Sync, AD FS og senere Azure AD Connect. Password Hash Sync, Pass-through Authentication, Seamless SSO, Cloud Sync og Workload Federation supplerte ulike driftsmodeller. Managed Identities flyttet credentialrotasjon for Azure-workloads inn i plattformen.

I 2023 endret Microsoft navnet Azure Active Directory til **Microsoft Entra ID**; protokollendepunkter, API-er, tenant- og lisensgrunnlag forble en del av samme tjenestekontinuitet ([Microsoft – New name for Azure Active Directory](https://learn.microsoft.com/en-us/entra/fundamentals/new-name)). Historikken forklarer dagens blanding av `login.microsoftonline.com`, `AzureAD`-eldre betegnelser, Microsoft Graph, AD-DS-hybridbegreper og Entra-produktnavn.

## Administrator-sjekkliste på et øyeblikk

Til slutt vurderes identiteter, applikasjoner, roller, policyer og gjenoppretting samlet. Sjekklisten fungerer som kort driftsdokumentasjon for disse sammenhengende kontrollene.

| Spørsmål | Driftsdokumentasjon |
|---|---|
| Hvilken tenant? | Tenant-ID, Cloud/Authority, verifisert domene, Home-/Resource-Tenant |
| Hvilket objekt? | Object-ID, type, Source of Authority, UPN/Mail kun som attributter, slettetilstand |
| Hvilken klient? | App-/Client-ID, Application Object, Service Principal i Resource Tenant, Redirect/Flow |
| Hvilken identitet? | Bruker/gjest/enhet/Service Principal/Managed Identity, credential- eller federationskilde |
| Hvilket token? | Type, Issuer, Audience, Tenant, Subject/Object, `scp`/`roles`, tid og signatur-Key-ID |
| Hvilken autorisasjon? | Consent Grant/App-Role, Directory-/Resource-RBAC, objekt-/postboksscope |
| Hvilken policy? | Conditional Access, Auth Strength, risiko, Device, Location, Session/CAE |
| Hvilken hybridkilde? | Connect/Cloud Sync, Source Anchor, filter, mapping, eksportstatus, autentiseringsmetode |
| Hvilken evidens? | UTC, Correlation-/Request-ID, Sign-in/Audit/Provisioning Log, Graphstatus |
| Hvordan gjenopprettes det? | Emergency Accounts, policyer/roller/apper/grants, hybridkonfigurasjon, domener/federation, eksterne nøkler og logger |

## Kilder

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
