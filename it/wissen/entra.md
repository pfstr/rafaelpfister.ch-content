---
title: "Microsoft Entra ID: tenant, token e identità ibrida"
blatt: "entra"
description: "Microsoft Entra ID per amministratori dell'infrastruttura e della messaggistica: oggetti tenant e directory, Security Token Service, OAuth/OIDC e SAML, app e service principal, identità dei workload, consenso e autorizzazione, Conditional Access, dispositivi, sincronizzazione ibrida, Domain Services, Microsoft Graph, logging, recovery e storia tecnica."
fakten:
  - label: Ruolo di sistema
    wert: servizio cloud multi-tenant di directory e identità con control plane per token, policy e amministrazione
    href: https://learn.microsoft.com/en-us/entra/fundamentals/whatis
  - label: Tenant
    wert: confine autonomo di directory/policy con ID tenant, domini verificati e oggetti
    href: https://learn.microsoft.com/en-us/entra/identity-platform/single-and-multi-tenant-apps
  - label: Protocolli
    wert: OAuth 2.0 · OpenID Connect · SAML 2.0 · WS-Federation su HTTPS
    href: https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols
  - label: API di amministrazione
    wert: Microsoft Graph per oggetti di directory, identità, policy, audit e applicazioni
    href: https://learn.microsoft.com/en-us/graph/overview
  - label: Modello delle app
    wert: Application Object come definizione · Service Principal come istanza locale nel tenant
    href: https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals
  - label: Token utente
    wert: ID Token per la sessione client · Access Token per la Resource API · Refresh Token per il rinnovo
    href: https://learn.microsoft.com/en-us/entra/identity-platform/security-tokens
  - label: Identità del workload
    wert: Service Principal o Managed Identity; certificato, segreto o credenziale federata
    href: https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-overview
  - label: Autorizzazione
    wert: scope delegati · Application Roles · Directory Roles · policy della risorsa/dell'oggetto
    href: https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview
  - label: Policy di accesso
    wert: Conditional Access valuta i segnali e applica controlli nelle decisioni sui token
    href: https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview
  - label: Ponte ibrido
    wert: Connect Sync o Cloud Sync replica oggetti e attributi AD DS selezionati
    href: https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync
  - label: Servizi di dominio legacy
    wert: Entra Domain Services fornisce Kerberos, NTLM, LDAP e Domain Join gestiti
    href: https://learn.microsoft.com/en-us/entra/identity/domain-services/overview
  - label: Stato operativo
    wert: emissione di token · risultato delle policy · sincronizzazione/provisioning · scadenza delle credenziali · sign-in/audit · Break Glass
    href: https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-sign-ins
werbung:
  - newsletter
ctaThemen:
  - active-directory-entra
  - microsoft-entra
  - powershell
translationSourceHash: 6d1a58b89bfeb1ae9f9b24608e56afa198b91d6e6681bb8692d749a36e5e1ef3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T11:03:33.646Z
translationReview: automatic
---

# Microsoft Entra ID: tenant, token e identità ibrida

Microsoft Entra ID è una directory cloud, un identity provider e un punto di applicazione delle policy per applicazioni Microsoft e di terze parti. Memorizza utenti, gruppi, dispositivi, applicazioni, service principal e ruoli; il suo Security Token Service emette token firmati dopo l'autenticazione e la verifica delle policy. Microsoft Graph costituisce il control plane di amministrazione. Questi ruoli devono essere considerati separatamente: un oggetto utente esistente non garantisce un accesso riuscito, un token valido non garantisce l'autorizzazione funzionale all'oggetto e un attributo sincronizzato non ha effetto immediato in ogni servizio di destinazione.

Per gli amministratori della messaggistica, Entra si trova su diversi percorsi critici: utenti e `proxyAddresses` alimentano Exchange Online, i gruppi controllano distribuzione e accesso, le autorizzazioni delle app consentono l'accesso automatizzato alle cassette postali o a Graph, Conditional Access influenza gli accessi amministrativi e client e la sincronizzazione ibrida collega Active Directory Domain Services al cloud. Il tenant non è quindi semplicemente «AD su Internet», ma una piattaforma autonoma per identità e autorizzazione.

La spiegazione segue un'identità dall'accesso fino all'accesso a una risorsa specifica. Prima vengono illustrati tenant, directory e token, poi ruoli, applicazioni, dispositivi, collegamento ibrido, protocolli e recovery.

Microsoft Entra ID combina una directory con l'emissione di token e la verifica delle policy. Un utente non accede «a Entra», ma richiede un token per una risorsa specifica; Entra verifica identità, applicazione, dispositivo e policy.

## Architettura: directory, STS, policy e risorsa

Microsoft descrive Entra ID come un servizio cloud di gestione delle identità e degli accessi. Le sue superfici principali sono:

1. **Directory:** oggetti, attributi, relazioni, ruoli, domini e configurazione locale del tenant.
2. **Security Token Service (STS):** endpoint dei protocolli, autenticazione, emissione di token e pubblicazione delle chiavi.
3. **Motori delle policy:** Conditional Access, Identity Protection, Authentication Methods, consenso e ulteriori decisioni di accesso.
4. **Provisioning/sincronizzazione:** replica o provisioning verso e da AD DS, SaaS e fonti HR.
5. **Microsoft Graph:** API per la gestione di directory, identità, audit e policy.
6. **Servizi risorsa:** Exchange Online, Graph, API proprie e SaaS convalidano i token e applicano la relativa autorizzazione.

Entra supporta più tenant. Un tenant costituisce un confine amministrativo e di policy, non automaticamente un isolamento completo di dati o rete per tutti i servizi SaaS utilizzati ([Microsoft Entra fundamentals – What is Microsoft Entra ID?](https://learn.microsoft.com/en-us/entra/fundamentals/whatis)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-entra.svg?v=20260813" title="Interaktive Infografik: Microsoft-Entra-Aufruf von Benutzer oder Workload über Authentisierung, Conditional Access und STS bis Token, Resource API und Microsoft Graph sowie Tenant-, App-, Hybrid-, Geräte- und Recoverygrenzen" loading="lazy">
  <a href="/images/kb-interaktiv-entra.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

## Stack tecnologico dal punto di vista dell'amministratore

In un servizio SaaS multi-tenant, lo stack interno di linguaggi di programmazione, database e orchestrazione non è una caratteristica del prodotto controllabile dal cliente. Deducerlo da nomi host, librerie client o annunci di lavoro non avrebbe valore per operatività e recovery. Lo **stack tecnologico dal punto di vista dell'amministratore** verificabile consiste nei contratti pubblicati: HTTPS come trasporto, OAuth 2.0, OpenID Connect, SAML e WS-Federation per l'identità, JWT/JWS e JWKS per token e chiavi, Microsoft Graph come control plane REST/OData e sincronizzazione basata su agenti per le identità ibride. Microsoft documenta esplicitamente questi protocolli e Graph come interfacce supportate ([Microsoft identity platform protocols](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols), [Microsoft Graph overview](https://learn.microsoft.com/en-us/graph/overview)).

Per l'amministrazione, pertanto, non conta un presunto linguaggio server, bensì la combinazione concreta di tenant, authority, versione del protocollo, formato del token, endpoint Graph, modulo SDK o PowerShell, agente di sincronizzazione e servizio risorsa. Ciascuno di questi livelli possiede limiti propri di versionamento, autorizzazione, logging ed errore.

## Tenant, domini e ancoraggi degli oggetti

Ogni tenant possiede un ID tenant immutabile. I nomi di dominio verificati offrono nomi di accesso leggibili e un riferimento di routing, ma non sostituiscono l'ID tenant. Un cambio di dominio o una configurazione multi-tenant non deve quindi essere identificato soltanto in base al suffisso UPN.

Gli oggetti possiedono ID locali al tenant. Gli utenti possono essere Member o Guest; un ospite B2B è un oggetto nel Resource Tenant con riferimento a un'identità esterna. Gli Application Object possiedono un ID globale client/application, mentre i service principal possiedono inoltre un proprio Object ID nel rispettivo tenant. Per il join e la sincronizzazione diventano rilevanti ulteriori ancoraggi quali `onPremisesImmutableId`, ID dei dispositivi e attributi di origine.

Gli amministratori documentano almeno:

- ID tenant, domini primari e verificati;
- ID oggetto anziché solo nome visualizzato o UPN;
- Home Tenant rispetto a Resource Tenant;
- sistema di origine e Source of Authority per attributo;
- stato di eliminazione temporanea/ripristino e ciclo di vita;
- relazioni di licenza, ruolo e gruppo come oggetti separati.

Microsoft distingue le applicazioni single-tenant e multi-tenant in base alle directory che possono utilizzare account e service principal ([Microsoft identity platform – Single- and multi-tenant apps](https://learn.microsoft.com/en-us/entra/identity-platform/single-and-multi-tenant-apps)).

## Entra ID non è Active Directory Domain Services

AD DS utilizza domini, foreste, domain controller, LDAP, Kerberos, NTLM, integrazione DNS, criteri di gruppo e replica. Entra ID utilizza oggetti tenant, endpoint HTTPS, moderni protocolli di federazione/token e policy cloud. Non espone un endpoint LDAP o Kerberos generale per le applicazioni.

| Caratteristica | AD DS | Entra ID |
|---|---|---|
| Topologia | Foresta, dominio, siti, domain controller | Tenant e servizio cloud globale |
| Protocolli primari | Kerberos, LDAP, DNS, SMB/RPC | OAuth 2.0, OpenID Connect, SAML, HTTPS/Graph |
| Associazione dei dispositivi | Domain Join e oggetto computer | Registered, Entra Joined, Hybrid Joined |
| Policy | GPO, ACL, configurazione Kerberos/LDAP | Conditional Access, ruoli, consenso, policy per token/app |
| Applicazione | Service Account/SPN, LDAP Bind, Kerberos | App Registration, Service Principal, Managed Identity |

Il confronto di Microsoft evidenzia esplicitamente i diversi modelli di protocollo e gestione ([Microsoft Learn – Compare Active Directory to Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/fundamentals/compare)). Un'appliance con [LDAP](/kb/ldap) Bind non può autenticarsi direttamente contro Entra ID senza gateway o Domain Services; al contrario, un'API OAuth non comprende un ticket [Kerberos](/kb/kerberos).

## Endpoint dei protocolli e metadati

L'endpoint di discovery OpenID Connect pubblica issuer, authorization endpoint, token endpoint, URI JWKS e funzionalità supportate. I client utilizzano authority specifiche del tenant, ad esempio un ID tenant concreto; `common`, `organizations` o `consumers` consentono tipi di account più ampi e modificano la verifica dell'issuer e l'ammissione del tenant.

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

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod), [`curl`](https://curl.se/docs/manpage.html) e [`jq`](https://jqlang.org/manual/) leggono metadati pubblici. Non eseguono alcuna convalida dei token. I formati OIDC Discovery e JWKS sono standardizzati ([OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html), [RFC 7517 – JSON Web Key](https://www.rfc-editor.org/rfc/rfc7517.html)).

Dopo aver chiarito tenant, dominio e ancoraggi degli oggetti, segue il percorso di accesso effettivo. OAuth 2.0, OpenID Connect e SAML svolgono compiti diversi e forniscono artefatti differenti.

## OAuth 2.0, OpenID Connect e SAML

OAuth 2.0 delega l'autorizzazione per le Resource API. OpenID Connect aggiunge un livello di autenticazione con ID Token e UserInfo. SAML trasporta assertion firmate tra identity provider e service provider. Microsoft pubblica le varianti di protocollo supportate dalla Identity Platform ([Microsoft identity platform protocols](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols), [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html), [OASIS SAML 2.0 Technical Overview](https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html)).

I comuni flussi OAuth differenziano identità e tipo di client:

- **Authorization Code con PKCE:** utente interattivo, client public o confidential.
- **Client Credentials:** workload senza utente; Application Permissions/App Roles.
- **On-Behalf-Of:** un'API intermedia scambia un token utente per un'API a valle.
- **Device Code:** dispositivo senza browser agevole; il codice viene confermato su un secondo dispositivo.
- **Refresh Token:** rinnova l'accesso senza autenticazione interattiva completa, ma resta soggetto a eventi di policy e revoca.

Microsoft documenta ciascun flusso con propri limiti di richiesta, credenziale e sicurezza ([Authorization Code Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow), [Client Credentials Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow), [On-Behalf-Of Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow), [Device Authorization Grant](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-device-code)).

## Tipi di token e claim

Un **ID Token** è destinato al client e attesta un'autenticazione. Un **Access Token** è destinato a una Resource API; solo tale risorsa deve convalidarlo. Un **Refresh Token** è una credenziale nei confronti dell'authorization server e non viene mai inviato a una Resource API ([Microsoft identity platform – Security tokens](https://learn.microsoft.com/en-us/entra/identity-platform/security-tokens)).

Per i token basati su JWT sono rilevanti, tra gli altri:

| Claim | Significato operativo |
|---|---|
| `iss` | issuer previsto, inclusa la semantica di tenant/endpoint |
| `aud` | Resource API a cui è destinato il token |
| `tid` | contesto tenant |
| `oid` / `sub` | ancoraggio riferito rispettivamente a oggetto o soggetto con scope diverso |
| `scp` | scope delegati di un token utente |
| `roles` | Application Roles o claim di ruolo |
| `exp`, `nbf`, `iat` | limiti temporali; rilevanti l'ora del server e la tolleranza |
| `amr`, `acr` | informazioni di autenticazione; non interpretarle universalmente come sostituto delle policy |
| `groups` | claim di gruppo o segnale di overage quando la quantità non entra nel token |

La convalida verifica firma, algoritmo, issuer, audience, tempo e claim specifici dell'applicazione. Le JWT Best Current Practices avvertono contro la confusione degli algoritmi, la Cross-JWT-Confusion e la fiducia cieca nei claim ricevuti ([RFC 8725 – JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html), [Microsoft – Validate tokens](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens#validate-tokens)).

Un token può essere decodificato localmente per la diagnosi; i token reali non vengono copiati in siti web esterni né nei ticket. La decodifica non è una verifica della firma.

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

[`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json), [`cut`](https://www.gnu.org/software/coreutils/manual/html_node/cut-invocation.html), [`tr`](https://www.gnu.org/software/coreutils/manual/html_node/tr-invocation.html) e [`base64`](https://www.gnu.org/software/coreutils/manual/html_node/base64-invocation.html) elaborano soltanto una copia locale. Dopo l'analisi viene eliminata; log e cronologia della shell non devono memorizzare il token.

## Application Object e Service Principal

Una App Registration crea un **Application Object** nell'Home Tenant. Descrive, tra l'altro, ID client, URI di reindirizzamento, App Roles, autorizzazioni API richieste e credenziali. Quando l'applicazione viene utilizzata o riceve consenso in un tenant, vi esiste un **Service Principal** come istanza locale del tenant. Le Managed Identities sono service principal speciali il cui ciclo di vita delle credenziali è gestito da Azure ([Application objects and service principals](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals)).

Questi ID non vanno confusi:

- `appId`/ID client identifica la definizione dell'applicazione a livello di protocollo;
- ID dell'Application Object identifica l'oggetto di registrazione nell'Home Tenant;
- ID oggetto del Service Principal identifica l'istanza nel Resource Tenant;
- ID app della risorsa/API o audience identifica la destinazione del token.

Un nome visualizzato non è univoco né costituisce un ancoraggio per l'automazione. L'eliminazione e la ricreazione non possono riprodurre arbitrariamente lo stesso ID client e generano nuove relazioni tra oggetti.

Un token valido non risponde ancora a ciò che un'applicazione può fare. Le autorizzazioni delegate e le autorizzazioni esclusivamente applicative differiscono per il coinvolgimento di un contesto utente e per chi approva i diritti.

## Delegated Permissions, Application Permissions e consenso

Le Delegated Permissions agiscono nel contesto di un utente connesso e compaiono tipicamente come `scp`. L'app non può automaticamente fare più di quanto consentano utente e policy del tenant. Le Application Permissions agiscono senza utente come App Roles nel claim `roles` e possono consentire accessi estesi ([Permissions and consent overview](https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview)).

Il consenso crea grant locali al tenant e assegnazioni di app role. L'Admin Consent non è una mera conferma di una finestra di dialogo, ma una modifica dell'autorizzazione. Inventario e revisione includono:

- service principal client e risorsa;
- grant di scope delegati e Application App-Role-Assignments;
- chi ha dato il consenso, quando e attraverso quale processo;
- utilizzo effettivo, Publisher Verification e origine;
- restrizione dell'oggetto/cassetta postale nella risorsa, se disponibile;
- effetto della revoca e dell'interruzione.

In Exchange Online, un'Application Permission può essere ulteriormente limitata da una policy di accesso specifica di Exchange o da RBAC for Applications. Il solo oggetto di consenso Graph/Entra non rappresenta completamente la portata funzionale della cassetta postale ([Exchange Online – Role Based Access Control for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)).

## Identità dei workload e credenziali

Un'identità del workload è un'identità software, tipicamente un service principal o una managed identity. Le credenziali possono essere client secret, certificato/chiave privata, credenziale di piattaforma Managed Identity o Federated Identity Credential. I segreti sono Bearer Credentials; certificati e federazione migliorano il possesso della chiave o evitano segreti di lunga durata memorizzati, ma richiedono una gestione della fiducia propria ([Microsoft Entra Workload ID overview](https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-overview)).

**Workload Identity Federation** considera attendibile un issuer OIDC esterno e associa issuer, subject e audience a un service principal o a una User-Assigned Managed Identity. Un sistema CI, un service account Kubernetes o un workload cloud scambia il proprio token esterno di breve durata con un token Entra senza memorizzare un segreto statico ([Workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)).

La gestione delle credenziali comprende issuer, archiviazione, accesso alla chiave privata, NotBefore/Expiry, sovrapposizione, rotazione, ultimo utilizzo e revoca. La data di scadenza nell'Application Object non dimostra che un'integrazione utilizzi effettivamente tale credenziale.

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

[`Connect-MgGraph`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/connect-mggraph), [`Get-MgContext`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/get-mgcontext), [`Get-MgApplication`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.applications/get-mgapplication) e [`Get-MgServicePrincipal`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.applications/get-mgserviceprincipal) mostrano il contesto Graph e i due tipi di oggetto. [`az account show`](https://learn.microsoft.com/en-us/cli/azure/account#az-account-show), [`az ad app`](https://learn.microsoft.com/en-us/cli/azure/ad/app) e [`az ad sp`](https://learn.microsoft.com/en-us/cli/azure/ad/sp) rappresentano la vista di Azure CLI. Gli output possono contenere metadati delle credenziali e inventario del tenant.

## Conditional Access e Continuous Access Evaluation

Conditional Access elabora segnali quali utente/workload, risorsa di destinazione, dispositivo, posizione, rischio, tipo di client e forza di autenticazione, e applica Grant Controls o Session Controls. Le policy agiscono insieme; l'esito positivo di una singola policy non è un risultato complessivo ([Conditional Access overview](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview)).

Report-only e What If aiutano nella pianificazione, ma non sostituiscono un gruppo pilota con client reali. Casi speciali rilevanti per la messaggistica sono Legacy Authentication, SMTP AUTH, client mobili, service account, account Break-Glass, portali amministrativi e accessi non interattivi.

Gli access token sono normalmente validi fino alla scadenza. Continuous Access Evaluation consente a risorse e client supportati di considerare prima eventi critici e modifiche delle policy ([Continuous Access Evaluation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation)). CAE non è una revoca immediata globale per ogni applicazione; risorsa e client devono supportare il protocollo.

## MFA, Authentication Methods e Authentication Strengths

MFA è un'affermazione di policy, non un singolo metodo di prodotto. Le Authentication Methods Policies controllano quali procedure possono essere registrate e utilizzate; le Conditional Access Authentication Strengths possono richiedere combinazioni concrete di metodi ([Authentication methods overview](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-methods), [Conditional Access authentication strengths](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-strengths)).

Metodi resistenti al phishing come FIDO2/passkey o autenticazione basata su certificato modificano i processi di enrollment, recovery e gestione dei dispositivi. Temporary Access Pass può consentire il bootstrap ed è esso stesso una credenziale limitata nel tempo. Reset dell'helpdesk, nuova registrazione del telefono e dispositivi smarriti sono workflow ad alto rischio e devono rientrare nel percorso di audit.

## Directory Roles, PIM e Administrative Units

Le Directory Roles autorizzano funzioni di amministrazione di Entra e Microsoft 365. I ruoli possono essere assegnati direttamente, tramite gruppi o temporaneamente attraverso Privileged Identity Management. PIM distingue tra eligible e active, approvazione, MFA, motivazione, durata e Access Reviews ([Microsoft Entra PIM overview](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)).

Le Administrative Units possono limitare lo scope di ruoli selezionati a sottoinsiemi di oggetti. Non ogni ruolo o operazione supporta questo scope; i portali Graph, Exchange e Security dispongono in parte di modelli RBAC propri ([Administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/administrative-units)).

Il privilegio viene documentato come catena: fonte dell'assegnazione → attivazione → token/ruolo → RBAC del servizio di destinazione → azione concreta sull'oggetto. «Global Administrator presente» non spiega automaticamente un 403 in Exchange o Graph.

## Identità del dispositivo e Primary Refresh Token

Entra Registered, Entra Joined e Hybrid Entra Joined sono relazioni di dispositivo diverse. Registered associa tipicamente un dispositivo personale o gestito da terzi; Joined utilizza Entra come legame organizzativo primario; Hybrid Joined collega AD-DS Domain Join alla registrazione Entra ([Microsoft Entra device identity](https://learn.microsoft.com/en-us/entra/identity/devices/overview)).

In Windows, il Primary Refresh Token supporta il Single Sign-On e porta il contesto del dispositivo e dell'autenticazione. L'emissione e il rinnovo del PRT dipendono dallo stato di utente, dispositivo e chiavi; non è un normale Refresh Token che gli amministratori dovrebbero copiare ([Primary Refresh Token](https://learn.microsoft.com/en-us/entra/identity/devices/concept-primary-refresh-token)).

Stato del dispositivo, conformità MDM, Hybrid Join, certificati e Conditional Access possono evolvere in tempi diversi. La diagnosi verifica quindi insieme informazioni locali di join, oggetto dispositivo Entra, oggetto di gestione, Sign-in Log e risultato della policy.

Negli ambienti ibridi l'identità inizia spesso nell'Active Directory locale. La sincronizzazione trasferisce oggetti e attributi selezionati, ma non rende Entra ID un sostituto di LDAP o Kerberos.

## Identità ibrida: Connect Sync e Cloud Sync

Microsoft Entra Connect Sync esegue un motore di sincronizzazione su Windows Server e può rappresentare regole estese e determinate funzionalità ibride. Cloud Sync utilizza Provisioning Agents leggeri e una configurazione gestita dal cloud; funzionalità e topologia differiscono ([Microsoft Entra Connect overview](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-azure-ad-connect), [Microsoft Entra Cloud Sync overview](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync)).

Un sistema di sincronizzazione possiede tre livelli:

1. **Connector Spaces** oppure connettori di origine/destinazione con oggetti importati;
2. **logica Metaverse/mapping** oppure configurazione del Cloud Provisioning;
3. **esportazione** in Entra con oggetto di destinazione, flusso degli attributi e stato di errore.

Matching e Source Anchor impediscono i duplicati. Filtri, regole di join, priorità degli attributi e writeback definiscono la titolarità dei dati. Un'esecuzione riuscita dello scheduler non dimostra che ogni oggetto sia stato esportato; quarantena, errori di provisioning ed elaborazione del servizio cloud vengono verificati separatamente.

### Autenticazione in esercizio ibrido

- **Password Hash Synchronization (PHS):** un hash derivato viene memorizzato in Entra; l'autenticazione cloud resta possibile in caso di guasto on-premises.
- **Pass-through Authentication (PTA):** gli agenti convalidano le password rispetto ad AD DS; il percorso dell'agente diventa rilevante operativamente.
- **Federation:** Entra inoltra l'autenticazione a uno STS federato; certificati, Claims Rules e disponibilità estendono l'area di guasto.

Microsoft descrive la scelta e i trade-off di questi metodi di accesso ([Choose the right authentication method](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/choose-ad-authn)). Seamless SSO e PRT sono meccanismi aggiuntivi e non sinonimi del metodo di autenticazione primario.

## Entra Domain Services

Microsoft Entra Domain Services fornisce un dominio gestito con Domain Join, criteri di gruppo, LDAP, Kerberos e NTLM. Utenti, gruppi e hash delle credenziali vengono sincronizzati da Entra nel dominio gestito. I clienti non ricevono diritti di Domain Admin o Enterprise Admin e non gestiscono direttamente i domain controller ([Microsoft Entra Domain Services overview](https://learn.microsoft.com/en-us/entra/identity/domain-services/overview)).

Domain Services non è un ponte bidirezionale verso AD DS né un sostituto dei protocolli token di Entra. Le applicazioni legacy vedono oggetti LDAP/Kerberos nel Managed Domain; le applicazioni SaaS moderne continuano a utilizzare Entra. Secure LDAP richiede certificato, confine di rete pubblicato, NSG/firewall e policy delle credenziali.

## External Identities e accesso cross-tenant

B2B Collaboration crea oggetti guest nel Resource Tenant, mentre l'autenticazione avviene spesso nell'Home Tenant. Le Cross-Tenant Access Settings controllano trust inbound e outbound, fiducia nei claim MFA/dispositivo e relazioni organizzative ([Microsoft Entra B2B collaboration overview](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b), [Cross-tenant access overview](https://learn.microsoft.com/en-us/entra/external-id/cross-tenant-access-overview)).

Il Resource Tenant continua ad autorizzare i propri dati. Un account home eliminato o disabilitato, un oggetto guest esistente, gli Access Packages e le appartenenze ai gruppi possono avere cicli di vita diversi. Access Reviews periodiche e processi per sponsor/owner colmano questa lacuna.

## Microsoft Graph come Control Plane

Microsoft Graph espone percorsi di risorse quali `/users`, `/groups`, `/applications`, `/servicePrincipals`, `/policies` e `/auditLogs`. Autorizzazione, versione API, paging, throttling ed eventual consistency di determinate query fanno parte del contratto ([Microsoft Graph overview](https://learn.microsoft.com/en-us/graph/overview)).

Delta Query fornisce modifiche a partire da uno stato iniziale tramite token opachi; Change Notifications inviano webhook, ma non sostituiscono un'esecuzione di riconciliazione ([Microsoft Graph delta query](https://learn.microsoft.com/en-us/graph/delta-query-overview), [Microsoft Graph change notifications](https://learn.microsoft.com/en-us/graph/change-notifications-overview)). Per le [API](/kb/apis) valgono anche qui idempotenza, paginazione, 429/Retry-After e Request-ID.

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

[`Get-MgUser`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguser) e [`Invoke-MgGraphRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/invoke-mggraphrequest) utilizzano il contesto Graph visualizzato. [`az account get-access-token`](https://learn.microsoft.com/en-us/cli/azure/account#az-account-get-access-token) fornisce un token CLI di breve durata; non viene registrato né memorizzato in modo persistente. Le risposte Graph contengono dati personali e rilevanti per la sicurezza.

## Dipendenze di rete e TLS

Entra è un servizio HTTPS distribuito. Proxy, firewall, DNS, ispezione TLS, ora ed elenco di endpoint consentiti influenzano autenticazione, Graph, Device Registration, Sync e revoca. Un elenco IP statico non è sempre il modello corretto; Microsoft pubblica Service Tags e categorie URL/IP per endpoint correlati a Microsoft 365 ed Entra ([Microsoft 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges), [Azure service tags overview](https://learn.microsoft.com/en-us/azure/virtual-network/service-tags-overview)).

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) verificano anzitutto la risoluzione dei nomi. [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection), [`nc`](https://man.openbsd.org/nc) e [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) separano TCP e [TLS](/kb/tls). [`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) verifica HTTP, [`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings) e [`timedatectl`](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html) la fonte dell'ora.

## Sign-in Logs, Audit Logs e Provisioning Logs

I Sign-in Logs documentano accessi interattivi, non interattivi, di service principal e managed identity con stato, valutazione di Conditional Access e informazioni su client, dispositivo e rischio. Gli Audit Logs documentano modifiche a directory e policy. I Provisioning Logs mostrano passaggi di provisioning ed errori di mapping ([Microsoft Entra sign-in logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-sign-ins), [Microsoft Entra audit logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-audit-logs), [Provisioning logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-provisioning-logs)).

Correlation ID, Request ID, ora in UTC, tenant, ID client, ID risorsa, ID utente/service principal e codice di errore costituiscono la prova minima. I testi del portale vengono integrati con codice di errore e fase token/protocollo; messaggi UI identici possono avere cause differenti.

La conservazione dei log dipende dalla licenza e dalla configurazione di esportazione e può cambiare. La prova a lungo termine viene inviata, prima dell'incidente, tramite Diagnostic Settings o percorsi di esportazione supportati a un sistema di log controllato.

## Break Glass, backup e recovery

Entra è un servizio SaaS; i clienti non eseguono il backup di un database dei domain controller. Devono tuttavia proteggere configurazione, credenziali esterne e recuperabilità amministrativa:

- almeno due cloud-only Emergency Access Accounts con metodi forti indipendenti;
- esclusioni da Conditional Access soltanto nella misura necessaria per il recovery e monitorate;
- configurazione tenant/dominio/federazione/cross-tenant ed esportazione dei ruoli;
- App Registrations, Service Principals, grant, App Roles e metadati delle credenziali;
- policy di Conditional Access, Authentication Methods, PIM e Lifecycle;
- regole di sincronizzazione ibrida, Source Anchors, configurazione agent/server e percorso di staging;
- dipendenze esterne di CA/KMS/federazione/DNS/Break-Glass;
- esportazione di Audit/Sign-in Logs e ticket di modifica.

Microsoft raccomanda Emergency Access Accounts dedicati, il cui utilizzo venga segnalato e testato regolarmente ([Manage emergency access accounts](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access)).

Gli oggetti soft-deleted possiedono finestre e limitazioni di ripristino specifiche del prodotto, che possono cambiare. I runbook di recovery collegano quindi la rispettiva documentazione Microsoft e testano separatamente utente, gruppo, app, service principal, configurazione errata di Conditional Access e percorso di federazione/dominio perso.

La risoluzione dei problemi segue la decisione del token: identificare utente e applicazione, verificare metodo di accesso e Conditional Access, leggere i token claim e infine valutare l'autorizzazione sulla risorsa.

## Diagnosi per fase decisionale

Un errore Entra può essere circoscritto più rapidamente se si determina prima la fase: accesso, emissione del token o decisione dell'applicazione di destinazione. La tabella assegna a ogni fase le prove appropriate.

| Fase | Evidenza | Classe di errore tipica |
|---|---|---|
| Discovery | Authority, ID tenant, metadati OIDC, DNS/TLS | tenant/cloud errato, proxy, ora, endpoint |
| Client | ID client, URI di reindirizzamento, flow, tipo di credenziale | definizione app, Reply URL, secret/certificato/federazione |
| Autenticazione | Sign-in Log, metodo, dispositivo, rischio | credenziale, MFA, utente/workload disabilitato |
| Conditional Access | risultati delle policy, Target Resource, condizioni | scope, Grant Control, dispositivo, posizione, client |
| Token | `iss`, `aud`, `tid`, `scp`/`roles`, ora | risorsa/authority errata, consenso, claim/orologio |
| Resource API | Request-ID, Resource-RBAC, policy dell'oggetto | ruolo Graph/Exchange, accesso app, ambito dell'oggetto |
| Provisioning/sincronizzazione | connettore/agente, mapping, log di export/provisioning | Source Anchor, filtro, duplicato, quarantena |

Un errore Entra non viene risolto effettuando accessi ripetuti in modo casuale. Solo la fase determina se occorre correggere rete, configurazione client, identità, policy, token o autorizzazione della risorsa.

## Storia tecnica

Microsoft ha sviluppato Windows Azure Active Directory come servizio multi-tenant di directory cloud e federazione per Microsoft Online Services e applicazioni di terze parti. Azure AD ha adottato protocolli token moderni e architettonicamente non è mai stato un domain controller AD DS ospitato. Microsoft Graph ha sostituito diverse API precedenti come control plane cloud comune.

L'Hybrid Identity è nata da strumenti di sincronizzazione e federazione quali DirSync, Azure AD Sync, AD FS e in seguito Azure AD Connect. Password Hash Sync, Pass-through Authentication, Seamless SSO, Cloud Sync e Workload Federation hanno aggiunto diversi modelli operativi. Le Managed Identities hanno spostato nella piattaforma la rotazione delle credenziali per i workload Azure.

Nel 2023 Microsoft ha rinominato Azure Active Directory in **Microsoft Entra ID**; endpoint dei protocolli, API, tenant e basi di licenza sono rimasti parte della stessa continuità del servizio ([Microsoft – New name for Azure Active Directory](https://learn.microsoft.com/en-us/entra/fundamentals/new-name)). La storia spiega l'attuale combinazione di vecchie denominazioni `login.microsoftonline.com`, `AzureAD`, Microsoft Graph, termini ibridi AD DS e nomi dei prodotti Entra.

## Checklist per amministratori in sintesi

Per concludere, identità, applicazioni, ruoli, policy e ripristino vengono considerati congiuntamente. La checklist funge da breve prova operativa per questi controlli interconnessi.

| Domanda | Prova operativa |
|---|---|
| Quale tenant? | ID tenant, cloud/authority, dominio verificato, Home/Resource Tenant |
| Quale oggetto? | ID oggetto, tipo, Source of Authority, UPN/mail solo come attributi, stato di eliminazione |
| Quale client? | ID app/client, Application Object, Service Principal nel Resource Tenant, redirect/flow |
| Quale identità? | utente/guest/dispositivo/service principal/managed identity, fonte di credenziale o federazione |
| Quale token? | tipo, issuer, audience, tenant, subject/object, `scp`/`roles`, ora e ID chiave di firma |
| Quale autorizzazione? | consent grant/app-role, Directory/Resource-RBAC, scope dell'oggetto/cassetta postale |
| Quale policy? | Conditional Access, Auth Strength, rischio, dispositivo, posizione, sessione/CAE |
| Quale fonte ibrida? | Connect/Cloud Sync, Source Anchor, filtro, mapping, stato di esportazione, metodo di autenticazione |
| Quale evidenza? | UTC, Correlation/Request-ID, Sign-in/Audit/Provisioning Log, stato Graph |
| Come viene ripristinato? | Emergency Accounts, policy/ruoli/app/grant, configurazione ibrida, domini/federazione, chiavi e log esterni |

## Fonti

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
