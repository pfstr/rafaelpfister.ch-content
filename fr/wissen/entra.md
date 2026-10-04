---
title: "Microsoft Entra ID : tenant, jetons et identité hybride"
blatt: "entra"
description: "Microsoft Entra ID pour les administrateurs d’infrastructure et de messagerie : objets de tenant et d’annuaire, Security Token Service, OAuth/OIDC et SAML, applications et Service Principals, identités de charge de travail, consentement et autorisation, Conditional Access, appareils, synchronisation hybride, Domain Services, Microsoft Graph, journalisation, récupération et histoire technique."
fakten:
  - label: Rôle système
    wert: service cloud mutualisé d’annuaire et d’identité avec plan de contrôle des jetons, des politiques et de l’administration
    href: https://learn.microsoft.com/en-us/entra/fundamentals/whatis
  - label: Tenant
    wert: frontière autonome d’annuaire et de politiques avec ID de tenant, domaines vérifiés et objets
    href: https://learn.microsoft.com/en-us/entra/identity-platform/single-and-multi-tenant-apps
  - label: Protocoles
    wert: OAuth 2.0 · OpenID Connect · SAML 2.0 · WS-Federation via HTTPS
    href: https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols
  - label: API d’administration
    wert: Microsoft Graph pour les objets d’annuaire, d’identité, de politiques, d’audit et d’application
    href: https://learn.microsoft.com/en-us/graph/overview
  - label: Modèle d’application
    wert: Application Object comme définition · Service Principal comme instance locale dans le tenant
    href: https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals
  - label: Jetons utilisateur
    wert: ID Token pour la session client · Access Token pour l’API de ressource · Refresh Token pour le renouvellement
    href: https://learn.microsoft.com/en-us/entra/identity-platform/security-tokens
  - label: Identité de charge de travail
    wert: Service Principal ou Managed Identity ; certificat, secret ou identifiant fédéré
    href: https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-overview
  - label: Autorisation
    wert: scopes délégués · rôles d’application · rôles d’annuaire · politique de ressource/d’objet
    href: https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview
  - label: Politique d’accès
    wert: Conditional Access évalue les signaux et applique des contrôles lors des décisions relatives aux jetons
    href: https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview
  - label: Pont hybride
    wert: Connect Sync ou Cloud Sync réplique des objets et attributs AD DS sélectionnés
    href: https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync
  - label: Services de domaine hérités
    wert: Entra Domain Services fournit Kerberos, NTLM, LDAP et Domain Join gérés
    href: https://learn.microsoft.com/en-us/entra/identity/domain-services/overview
  - label: État opérationnel
    wert: émission de jetons · résultat des politiques · synchronisation/provisionnement · expiration des identifiants · connexion/audit · Break Glass
    href: https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-sign-ins
werbung:
  - newsletter
ctaThemen:
  - active-directory-entra
  - microsoft-entra
  - powershell
translationSourceHash: 6d1a58b89bfeb1ae9f9b24608e56afa198b91d6e6681bb8692d749a36e5e1ef3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-04T10:43:42.723Z
translationReview: automatic
---

# Microsoft Entra ID : tenant, jetons et identité hybride

Microsoft Entra ID est un annuaire cloud, un fournisseur d’identité et un point d’application des politiques pour les applications Microsoft et tierces. Il stocke des utilisateurs, des groupes, des appareils, des applications, des Service Principals et des rôles ; son Security Token Service émet des jetons signés après une authentification et une vérification des politiques réussies. Microsoft Graph constitue le plan de contrôle de l’administration. Ces rôles doivent être considérés séparément : l’existence d’un objet utilisateur ne garantit pas une connexion réussie, un jeton valide ne garantit pas une autorisation fonctionnelle sur un objet et un attribut synchronisé n’a pas un effet immédiat dans chaque service cible.

Pour les administrateurs de messagerie, Entra se situe sur plusieurs chemins critiques : les utilisateurs et `proxyAddresses` alimentent Exchange Online, les groupes contrôlent les listes de distribution et les accès, les autorisations d’application permettent l’accès automatisé aux boîtes aux lettres ou à Graph, Conditional Access influence les connexions des administrateurs et des clients, et la synchronisation hybride relie Active Directory Domain Services au cloud. Le tenant n’est donc pas simplement « AD sur Internet », mais une plateforme autonome d’identité et d’autorisation.

L’explication suit une identité depuis la connexion jusqu’à l’accès à une ressource concrète. Elle aborde d’abord le tenant, l’annuaire et les jetons, puis les rôles, les applications, les appareils, la connexion hybride, les protocoles et la récupération.

Microsoft Entra ID associe un annuaire à l’émission de jetons et à la vérification des politiques. Un utilisateur n’accède pas « à Entra », mais demande un jeton pour une ressource précise ; Entra vérifie alors l’identité, l’application, l’appareil et la politique.

## Architecture : annuaire, STS, politiques et ressources

Microsoft décrit Entra ID comme un service cloud de gestion des identités et des accès. Ses principales surfaces sont :

1. **Annuaire :** objets, attributs, relations, rôles, domaines et configuration locale au tenant.
2. **Security Token Service (STS) :** points de terminaison de protocole, authentification, émission de jetons et publication de clés.
3. **Moteurs de politiques :** Conditional Access, Identity Protection, Authentication Methods, consentement et autres décisions d’accès.
4. **Provisioning/Sync :** réplication ou provisionnement vers et depuis AD DS, SaaS et des sources RH.
5. **Microsoft Graph :** API pour la gestion de l’annuaire, des identités, des audits et des politiques.
6. **Services de ressources :** Exchange Online, Graph, API propriétaires et SaaS valident les jetons et appliquent leur autorisation.

Entra prend en charge plusieurs tenants. Un tenant représente une limite administrative et de politiques, mais pas automatiquement une isolation complète des données ou du réseau pour tous les services SaaS consommés ([Microsoft Entra fundamentals – What is Microsoft Entra ID?](https://learn.microsoft.com/en-us/entra/fundamentals/whatis)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-entra.svg?v=20260813" title="Interaktive Infografik: Microsoft-Entra-Aufruf von Benutzer oder Workload über Authentisierung, Conditional Access und STS bis Token, Resource API und Microsoft Graph sowie Tenant-, App-, Hybrid-, Geräte- und Recoverygrenzen" loading="lazy">
  <a href="/images/kb-interaktiv-entra.svg?v=20260813">Ouvrir le graphique interactif directement</a>.
</iframe>

## Pile technologique du point de vue de l’administrateur

Pour un service SaaS mutualisé, la pile interne de langages de programmation, de bases de données et d’orchestration n’est pas une caractéristique de produit contrôlable par le client. La déduire à partir de noms d’hôtes, de bibliothèques clientes ou d’offres d’emploi serait sans valeur pour l’exploitation et la récupération. La **pile technologique du point de vue de l’administrateur** démontrable se compose des contrats publiés : HTTPS comme transport, OAuth 2.0, OpenID Connect, SAML et WS-Federation pour l’identité, JWT/JWS et JWKS pour les jetons et les clés, Microsoft Graph comme plan de contrôle REST/OData ainsi que la synchronisation par agent pour les identités hybrides. Microsoft documente explicitement ces protocoles et Graph comme interfaces prises en charge ([Microsoft identity platform protocols](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols), [Microsoft Graph overview](https://learn.microsoft.com/en-us/graph/overview)).

Pour l’administration, ce qui compte n’est donc pas un langage serveur supposé, mais la combinaison concrète du tenant, de l’autorité, de la version du protocole, du format de jeton, du point de terminaison Graph, du SDK ou module PowerShell, de l’agent de synchronisation et du service de ressource. Chacune de ces couches possède ses propres limites de versionnement, d’autorisations, de journalisation et d’erreurs.

## Tenant, domaines et ancres d’objet

Chaque tenant possède un ID de tenant immuable. Les noms de domaine vérifiés offrent des noms de connexion lisibles et une référence de routage, mais ne remplacent pas l’ID de tenant. Un changement de domaine ou une configuration multitenant ne doit donc pas être identifié uniquement à partir du suffixe UPN.

Les objets possèdent des ID locaux au tenant. Les utilisateurs peuvent être membres ou invités ; un invité B2B est un objet dans le tenant de ressource avec une référence à une identité externe. Les Application Objects possèdent un ID global de client/d’application, les Service Principals ont en plus leur propre Object ID dans le tenant concerné. Pour la jonction et la synchronisation, d’autres ancres telles que `onPremisesImmutableId`, les ID d’appareils et les attributs source sont pertinentes.

Les administrateurs documentent au minimum :

- l’ID de tenant, les domaines principaux et vérifiés ;
- l’Object ID plutôt que seulement le nom d’affichage ou l’UPN ;
- le tenant d’origine par rapport au tenant de ressource ;
- le système source et la Source of Authority pour chaque attribut ;
- l’état de suppression réversible/restauration et le cycle de vie ;
- les relations de licence, de rôle et de groupe en tant qu’objets distincts.

Microsoft distingue les applications monotenantes et multitenantes selon les annuaires autorisés à utiliser des comptes et des Service Principals ([Microsoft identity platform – Single- and multi-tenant apps](https://learn.microsoft.com/en-us/entra/identity-platform/single-and-multi-tenant-apps)).

## Entra ID n’est pas Active Directory Domain Services

AD DS utilise des domaines, des forêts, des contrôleurs de domaine, LDAP, Kerberos, NTLM, l’intégration DNS, les stratégies de groupe et la réplication. Entra ID utilise des objets de tenant, des points de terminaison HTTPS, des protocoles modernes de fédération et de jetons ainsi que des politiques cloud. Il n’expose aucun point de terminaison LDAP ou Kerberos général aux applications.

| Caractéristique | AD DS | Entra ID |
|---|---|---|
| Topologie | Forêt, domaine, sites, contrôleurs de domaine | Tenant et service cloud mondial |
| Protocoles principaux | Kerberos, LDAP, DNS, SMB/RPC | OAuth 2.0, OpenID Connect, SAML, HTTPS/Graph |
| Liaison des appareils | Domain Join et objet ordinateur | Registered, Entra Joined, Hybrid Joined |
| Politique | GPO, ACL, configuration Kerberos/LDAP | Conditional Access, rôles, consentement, politique de jeton/application |
| Application | Service Account/SPN, LDAP Bind, Kerberos | App Registration, Service Principal, Managed Identity |

La comparaison de Microsoft souligne expressément la différence entre les modèles de protocole et de gestion ([Microsoft Learn – Compare Active Directory to Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/fundamentals/compare)). Une appliance avec une liaison [LDAP](/kb/ldap) ne peut pas s’authentifier directement auprès d’Entra ID sans passerelle ou Domain Services ; inversement, une API OAuth ne comprend pas un ticket [Kerberos](/kb/kerberos).

## Points de terminaison et métadonnées de protocole

Le point de terminaison de découverte OpenID Connect publie l’émetteur, le point de terminaison d’autorisation, le point de terminaison de jeton, l’URI JWKS et les fonctions prises en charge. Les clients utilisent des autorités spécifiques au tenant, comme un ID de tenant concret ; `common`, `organizations` ou `consumers` autorisent des types de comptes plus larges et modifient la vérification de l’émetteur et l’admission du tenant.

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

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod), [`curl`](https://curl.se/docs/manpage.html) et [`jq`](https://jqlang.org/manual/) lisent des métadonnées publiques. Ils n’effectuent aucune validation de jeton. La découverte OIDC et les formats JWKS sont standardisés ([OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html), [RFC 7517 – JSON Web Key](https://www.rfc-editor.org/rfc/rfc7517.html)).

Une fois le tenant, le domaine et les ancres d’objet clarifiés, vient le véritable chemin de connexion. OAuth 2.0, OpenID Connect et SAML remplissent alors des rôles différents et fournissent des artefacts différents.

## OAuth 2.0, OpenID Connect et SAML

OAuth 2.0 délègue l’autorisation aux API de ressources. OpenID Connect ajoute une couche d’authentification avec ID Token et UserInfo. SAML transporte des assertions signées entre un fournisseur d’identité et un fournisseur de services. Microsoft publie les variantes de protocole prises en charge par Identity Platform ([Microsoft identity platform protocols](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols), [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html), [OASIS SAML 2.0 Technical Overview](https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html)).

Les flux OAuth courants distinguent l’identité et le type de client :

- **Authorization Code avec PKCE :** utilisateur interactif, client public ou confidentiel.
- **Client Credentials :** charge de travail sans utilisateur ; Application Permissions/App Roles.
- **On-Behalf-Of :** une API intermédiaire échange un jeton utilisateur contre un jeton destiné à une API en aval.
- **Device Code :** appareil sans navigateur pratique ; le code est confirmé sur un second appareil.
- **Refresh Token :** renouvelle l’accès sans connexion interactive complète, mais reste soumis aux politiques et aux événements de révocation.

Microsoft documente chaque flux avec ses propres limites de requête, d’identifiants et de sécurité ([Authorization Code Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow), [Client Credentials Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow), [On-Behalf-Of Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow), [Device Authorization Grant](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-device-code)).

## Types de jetons et claims

Un **ID Token** est destiné au client et atteste une authentification. Un **Access Token** est destiné à une API de ressource ; seule cette ressource doit le valider. Un **Refresh Token** est un identifiant vis-à-vis du serveur d’autorisation et n’est jamais envoyé à une API de ressource ([Microsoft identity platform – Security tokens](https://learn.microsoft.com/en-us/entra/identity-platform/security-tokens)).

Pour les jetons basés sur JWT, les éléments suivants sont notamment pertinents :

| Claim | Signification opérationnelle |
|---|---|
| `iss` | émetteur attendu, y compris la sémantique du tenant/point de terminaison |
| `aud` | API de ressource à laquelle le jeton est destiné |
| `tid` | contexte du tenant |
| `oid` / `sub` | ancre liée respectivement à l’objet ou au sujet, avec une portée différente |
| `scp` | scopes délégués d’un jeton utilisateur |
| `roles` | rôles d’application ou claims de rôle |
| `exp`, `nbf`, `iat` | limites temporelles ; l’heure du serveur et la tolérance sont pertinentes |
| `amr`, `acr` | informations d’authentification ; ne pas les interpréter universellement comme remplacement des politiques |
| `groups` | claim de groupe ou signal d’excédent lorsque l’ensemble ne tient pas dans le jeton |

La validation vérifie la signature, l’algorithme, l’émetteur, l’audience, l’heure et les claims spécifiques à l’application. Les bonnes pratiques actuelles JWT mettent en garde contre la confusion d’algorithmes, la confusion Cross-JWT et la confiance aveugle dans les claims reçus ([RFC 8725 – JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html), [Microsoft – Validate tokens](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens#validate-tokens)).

Un jeton peut être décodé localement à des fins de diagnostic ; les vrais jetons ne sont toutefois jamais copiés dans des sites web tiers ou des tickets. Le décodage n’est pas une vérification de signature.

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

[`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json), [`cut`](https://www.gnu.org/software/coreutils/manual/html_node/cut-invocation.html), [`tr`](https://www.gnu.org/software/coreutils/manual/html_node/tr-invocation.html) et [`base64`](https://www.gnu.org/software/coreutils/manual/html_node/base64-invocation.html) ne traitent qu’une copie locale. Après l’analyse, elle est supprimée ; les journaux et l’historique du shell ne doivent pas stocker le jeton.

## Application Object et Service Principal

Une App Registration crée un **Application Object** dans le tenant d’origine. Il décrit notamment l’ID client, les URI de redirection, les App Roles, les autorisations d’API demandées et les identifiants. Lorsque l’application est utilisée ou fait l’objet d’un consentement dans un tenant, un **Service Principal** y existe comme instance locale au tenant. Les Managed Identities sont des Service Principals particuliers dont Azure gère le cycle de vie des identifiants ([Application objects and service principals](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals)).

Ces ID ne doivent pas être confondus :

- `appId`/ID client identifie la définition de l’application au niveau protocolaire ;
- l’Object ID de l’Application Object identifie l’objet d’inscription dans le tenant d’origine ;
- l’Object ID du Service Principal identifie l’instance dans le tenant de ressource ;
- l’ID d’application de ressource/API ou audience identifie la cible du jeton.

Un nom d’affichage n’est pas unique et ne constitue pas une ancre d’automatisation. La suppression et la recréation ne peuvent pas reproduire arbitrairement le même ID client et créent de nouvelles relations entre objets.

Un jeton valide ne répond pas encore à ce qu’une application peut faire. Les autorisations déléguées et les autorisations d’application pures se distinguent selon qu’un contexte utilisateur est impliqué et selon qui approuve les droits.

## Delegated Permissions, Application Permissions et consentement

Les Delegated Permissions agissent dans le contexte d’un utilisateur connecté et apparaissent généralement sous la forme de `scp`. L’application ne peut pas automatiquement en faire plus que ce que l’utilisateur et la politique du tenant autorisent. Les Application Permissions agissent sans utilisateur en tant qu’App Roles dans le claim `roles` et peuvent permettre un accès étendu ([Permissions and consent overview](https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview)).

Le consentement crée des grants locaux au tenant et des attributions d’App Roles. L’Admin Consent n’est pas une simple confirmation de boîte de dialogue, mais une modification d’autorisation. L’inventaire et la revue comprennent :

- le Service Principal du client et de la ressource ;
- les grants de scopes délégués et les Application App-Role Assignments ;
- qui a donné son consentement, quand et via quel processus ;
- l’utilisation effective, Publisher Verification et provenance ;
- la restriction d’objet/boîte aux lettres dans la ressource, lorsqu’elle est disponible ;
- l’effet de la révocation et de la panne.

Dans Exchange Online, une Application Permission peut être limitée en plus par une politique d’accès spécifique à Exchange ou RBAC for Applications. Le seul objet de consentement Graph/Entra ne reflète pas entièrement la portée fonctionnelle des boîtes aux lettres ([Exchange Online – Role Based Access Control for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)).

## Identités de charge de travail et identifiants

Une identité de charge de travail est une identité logicielle, généralement un Service Principal ou une Managed Identity. Les identifiants peuvent être un Client Secret, un certificat/une clé privée, un identifiant de plateforme Managed Identity ou un Federated Identity Credential. Les secrets sont des identifiants porteurs ; les certificats et la fédération améliorent la possession des clés ou évitent le stockage de secrets statiques à longue durée de vie, mais exigent leur propre exploitation de la confiance ([Microsoft Entra Workload ID overview](https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-overview)).

**Workload Identity Federation** fait confiance à un émetteur OIDC externe et lie l’émetteur, le sujet et l’audience à un Service Principal ou à une User-Assigned Managed Identity. Un système CI, un compte de service Kubernetes ou une charge de travail cloud échange son jeton externe de courte durée contre un jeton Entra sans stocker de secret statique ([Workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)).

L’exploitation des identifiants comprend l’émetteur, le stockage, l’accès à la clé privée, NotBefore/Expiry, le chevauchement, la rotation, la dernière utilisation et la révocation. La date d’expiration dans l’Application Object ne prouve pas qu’une intégration utilise réellement cet identifiant.

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

[`Connect-MgGraph`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/connect-mggraph), [`Get-MgContext`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/get-mgcontext), [`Get-MgApplication`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.applications/get-mgapplication) et [`Get-MgServicePrincipal`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.applications/get-mgserviceprincipal) affichent le contexte Graph et les deux types d’objets. [`az account show`](https://learn.microsoft.com/en-us/cli/azure/account#az-account-show), [`az ad app`](https://learn.microsoft.com/en-us/cli/azure/ad/app) et [`az ad sp`](https://learn.microsoft.com/en-us/cli/azure/ad/sp) présentent la vue Azure CLI. Les sorties peuvent contenir des métadonnées d’identifiants et l’inventaire du tenant.

## Conditional Access et Continuous Access Evaluation

Conditional Access traite des signaux tels que l’utilisateur/la charge de travail, la ressource cible, l’appareil, l’emplacement, le risque, le type de client et le niveau d’authentification, puis applique des contrôles d’octroi ou de session. Les politiques agissent ensemble ; la réussite d’une politique individuelle ne constitue pas un résultat global ([Conditional Access overview](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview)).

Report-only et What If facilitent la planification, mais ne remplacent pas un groupe pilote avec de vrais clients. Parmi les cas particuliers pertinents pour la messagerie figurent Legacy Authentication, SMTP AUTH, les clients mobiles, les Service Accounts, les comptes Break-Glass, les portails d’administration et les connexions non interactives.

Les Access Tokens sont normalement valides jusqu’à leur expiration. Continuous Access Evaluation permet aux ressources et clients pris en charge de prendre en compte plus tôt les événements critiques et les modifications de politiques ([Continuous Access Evaluation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation)). CAE n’est pas une révocation immédiate globale pour chaque application ; la ressource et le client doivent prendre en charge le protocole.

## MFA, Authentication Methods et Authentication Strengths

MFA est une affirmation de politique, pas une méthode de produit unique. Les politiques Authentication Methods contrôlent quelles méthodes peuvent être enregistrées et utilisées ; les Conditional Access Authentication Strengths peuvent exiger des combinaisons de méthodes concrètes ([Authentication methods overview](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-methods), [Conditional Access authentication strengths](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-strengths)).

Les méthodes résistantes au phishing, telles que FIDO2/Passkeys ou l’authentification basée sur certificat, modifient les processus d’inscription, de récupération et d’appareils. Temporary Access Pass peut permettre l’amorçage et constitue lui-même un identifiant limité dans le temps. La réinitialisation par le support, le nouvel enregistrement d’un téléphone et les appareils perdus sont des flux de travail à haut risque et font partie de la piste d’audit.

## Directory Roles, PIM et Administrative Units

Les Directory Roles autorisent les fonctions de gestion Entra et Microsoft 365. Les rôles peuvent être attribués directement, via des groupes ou temporairement via Privileged Identity Management. PIM distingue eligible et active, l’approbation, MFA, la justification, la durée et les Access Reviews ([Microsoft Entra PIM overview](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)).

Les Administrative Units peuvent limiter la portée de rôles sélectionnés à des sous-ensembles d’objets. Tous les rôles ou opérations ne prennent pas en charge cette portée ; les portails Graph, Exchange et Security possèdent en partie leurs propres modèles RBAC ([Administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/administrative-units)).

Le privilège est documenté comme une chaîne : source d’attribution → activation → jeton/rôle → RBAC du service cible → action concrète sur l’objet. « Global Administrator présent » n’explique pas automatiquement une erreur 403 dans Exchange ou Graph.

## Identité des appareils et Primary Refresh Token

Entra Registered, Entra Joined et Hybrid Entra Joined représentent différentes relations d’appareil. Registered lie généralement un appareil personnel ou géré par un tiers ; Joined utilise Entra comme liaison organisationnelle principale ; Hybrid Joined combine l’adhésion à un domaine AD DS avec l’inscription Entra ([Microsoft Entra device identity](https://learn.microsoft.com/en-us/entra/identity/devices/overview)).

Sous Windows, le Primary Refresh Token prend en charge le Single Sign-On et porte le contexte de l’appareil ainsi que celui de l’authentification. L’émission et le renouvellement du PRT dépendent de l’état de l’utilisateur, de l’appareil et des clés ; ce n’est pas un Refresh Token ordinaire que les administrateurs devraient copier ([Primary Refresh Token](https://learn.microsoft.com/en-us/entra/identity/devices/concept-primary-refresh-token)).

L’état de l’appareil, la conformité MDM, Hybrid Join, les certificats et Conditional Access peuvent évoluer à des moments différents. Le diagnostic vérifie donc ensemble les informations de jonction locales, l’objet appareil Entra, l’objet de gestion, le journal de connexion et le résultat de la politique.

Dans les environnements hybrides, l’identité commence souvent dans Active Directory local. La synchronisation transfère des objets et attributs sélectionnés, mais ne fait pas d’Entra ID un remplacement de LDAP ou Kerberos.

## Identité hybride : Connect Sync et Cloud Sync

Microsoft Entra Connect Sync exploite un moteur de synchronisation sur Windows Server et peut représenter des règles étendues et certaines fonctionnalités hybrides. Cloud Sync utilise des agents de provisionnement légers et une configuration contrôlée depuis le cloud ; les fonctionnalités et la topologie diffèrent ([Microsoft Entra Connect overview](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-azure-ad-connect), [Microsoft Entra Cloud Sync overview](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync)).

Un système de synchronisation possède trois niveaux :

1. **Connector Spaces** ou connecteur source/cible avec objets importés ;
2. **Logique Metaverse/mapping** ou configuration de provisionnement cloud ;
3. **Exportation** vers Entra avec objet cible, flux d’attributs et état d’erreur.

La correspondance et la Source Anchor empêchent les doublons. Les filtres, règles de jonction, priorité des attributs et Writeback définissent la souveraineté des données. Une exécution réussie du planificateur ne prouve pas que chaque objet a été exporté ; la quarantaine, les erreurs de provisionnement et le traitement par le service cloud sont vérifiés séparément.

### Authentification en mode hybride

- **Password Hash Synchronization (PHS) :** un hachage dérivé est stocké dans Entra ; l’authentification cloud reste possible en cas de panne sur site.
- **Pass-through Authentication (PTA) :** les agents valident les mots de passe auprès d’AD DS ; le chemin des agents devient opérationnellement pertinent.
- **Federation :** Entra redirige l’authentification vers un STS fédéré ; les certificats, les Claims Rules et la disponibilité élargissent le périmètre de panne.

Microsoft décrit la sélection et les compromis de ces méthodes de connexion ([Choose the right authentication method](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/choose-ad-authn)). Seamless SSO et PRT sont des mécanismes supplémentaires et ne sont pas synonymes de méthode d’authentification principale.

## Entra Domain Services

Microsoft Entra Domain Services fournit un domaine géré avec Domain Join, stratégies de groupe, LDAP, Kerberos et NTLM. Les utilisateurs, groupes et hachages d’identifiants sont synchronisés depuis Entra vers le domaine géré. Les clients ne reçoivent aucun droit d’administrateur de domaine ou d’entreprise et ne gèrent pas eux-mêmes les contrôleurs de domaine ([Microsoft Entra Domain Services overview](https://learn.microsoft.com/en-us/entra/identity/domain-services/overview)).

Domain Services n’est pas un pont bidirectionnel vers AD DS ni un remplacement des protocoles de jetons Entra. Les applications héritées voient les objets LDAP/Kerberos dans le domaine géré ; les applications SaaS modernes continuent d’utiliser Entra. Secure LDAP nécessite un certificat, une limite réseau publiée, NSG/pare-feu et une politique d’identifiants.

## External Identities et accès intertenant

B2B Collaboration crée des objets invités dans le tenant de ressource, tandis que l’authentification a fréquemment lieu dans le tenant d’origine. Les Cross-Tenant Access Settings contrôlent la confiance entrante et sortante, la confiance dans les claims MFA/appareil et les relations organisationnelles ([Microsoft Entra B2B collaboration overview](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b), [Cross-tenant access overview](https://learn.microsoft.com/en-us/entra/external-id/cross-tenant-access-overview)).

Le tenant de ressource continue d’autoriser ses données. Un compte d’origine supprimé ou désactivé, un objet invité existant, les Access Packages et les appartenances à des groupes peuvent avoir des cycles de vie distincts. Des Access Reviews périodiques et des processus de sponsor/propriétaire comblent cette lacune.

## Microsoft Graph comme plan de contrôle

Microsoft Graph expose des chemins de ressources tels que `/users`, `/groups`, `/applications`, `/servicePrincipals`, `/policies` et `/auditLogs`. Les autorisations, la version d’API, la pagination, la limitation et l’eventual consistency de certaines requêtes font partie du contrat ([Microsoft Graph overview](https://learn.microsoft.com/en-us/graph/overview)).

Delta Query fournit les modifications depuis un état initial via des jetons opaques ; Change Notifications envoie des webhooks, mais ne remplace pas une exécution de rapprochement ([Microsoft Graph delta query](https://learn.microsoft.com/en-us/graph/delta-query-overview), [Microsoft Graph change notifications](https://learn.microsoft.com/en-us/graph/change-notifications-overview)). Pour les [API](/kb/apis), l’idempotence, la pagination, 429/Retry-After et l’ID de requête s’appliquent également ici.

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

[`Get-MgUser`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguser) et [`Invoke-MgGraphRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/invoke-mggraphrequest) utilisent le contexte Graph affiché. [`az account get-access-token`](https://learn.microsoft.com/en-us/cli/azure/account#az-account-get-access-token) fournit un jeton CLI de courte durée ; il n’est pas journalisé ni stocké de manière persistante. Les réponses Graph contiennent des données personnelles et pertinentes pour la sécurité.

## Dépendances réseau et TLS

Entra est un service HTTPS distribué. Le proxy, le pare-feu, DNS, l’inspection TLS, l’heure et la liste d’autorisation des points de terminaison influencent l’authentification, Graph, Device Registration, Sync et la révocation. Une liste IP statique n’est pas toujours le bon modèle ; Microsoft publie des Service Tags et des catégories d’URL/IP pour les points de terminaison liés à Microsoft 365 et Entra ([Microsoft 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges), [Azure service tags overview](https://learn.microsoft.com/en-us/azure/virtual-network/service-tags-overview)).

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) vérifient d’abord la résolution de noms. [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection), [`nc`](https://man.openbsd.org/nc) et [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) distinguent TCP et [TLS](/kb/tls). [`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) vérifie HTTP, [`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings) et [`timedatectl`](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html) la source de temps.

## Sign-in Logs, Audit Logs et Provisioning Logs

Les Sign-in Logs documentent les connexions interactives, non interactives, de Service Principal et de Managed Identity, avec état, évaluation Conditional Access, informations de client, d’appareil et de risque. Les Audit Logs documentent les modifications de l’annuaire et des politiques. Les Provisioning Logs montrent les étapes de provisionnement et les erreurs de mappage ([Microsoft Entra sign-in logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-sign-ins), [Microsoft Entra audit logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-audit-logs), [Provisioning logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-provisioning-logs)).

Correlation ID, Request ID, heure en UTC, tenant, ID client, ID de ressource, ID utilisateur/Service Principal et code d’erreur constituent la preuve minimale. Les textes du portail sont complétés par le code d’erreur et la phase du jeton/protocole ; des messages d’interface identiques peuvent avoir des causes différentes.

La conservation des journaux dépend de la licence et de la configuration d’exportation et peut changer. Les preuves à long terme sont envoyées avant l’incident vers un système de journalisation contrôlé via Diagnostic Settings ou des voies d’exportation prises en charge.

## Break Glass, sauvegarde et récupération

Entra est un service SaaS ; les clients ne sauvegardent pas une base de données de contrôleur de domaine. Ils doivent toutefois protéger la configuration, les identifiants externes et la capacité de récupération administrative :

- au moins deux comptes cloud-only Emergency Access avec des méthodes fortes indépendantes ;
- des exclusions de Conditional Access uniquement dans la mesure nécessaire à la récupération et surveillées ;
- la configuration du tenant/des domaines/de la fédération/de l’accès intertenant et l’export des rôles ;
- les App Registrations, Service Principals, grants, App Roles et métadonnées d’identifiants ;
- les politiques Conditional Access, Authentication Methods, PIM et de cycle de vie ;
- les règles de synchronisation hybride, Source Anchors, configuration des agents/serveurs et chemin de mise en attente ;
- les dépendances externes CA/KMS/fédération/DNS/Break-Glass ;
- l’export des Audit/Sign-in Logs et des tickets de changement.

Microsoft recommande des comptes Emergency Access dédiés, dont l’utilisation est alertée et testée régulièrement ([Manage emergency access accounts](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access)).

Les objets supprimés de manière réversible possèdent des fenêtres et restrictions de restauration spécifiques aux produits, qui peuvent changer. Les runbooks de récupération renvoient donc à la documentation Microsoft concernée et testent séparément l’utilisateur, le groupe, l’application, le Service Principal, la mauvaise configuration Conditional Access et le chemin de fédération/domaine perdu.

Le dépannage suit la décision de jeton : identifier l’utilisateur et l’application, vérifier la méthode de connexion et Conditional Access, lire les claims du jeton et enfin évaluer l’autorisation sur la ressource.

## Diagnostic par phase de décision

Une erreur Entra peut être circonscrite plus rapidement lorsque la phase est d’abord déterminée : connexion, émission de jeton ou décision de l’application cible. Le tableau associe à chaque phase les preuves appropriées.

| Phase | Éléments probants | Classe d’erreur typique |
|---|---|---|
| Discovery | Authority, ID de tenant, métadonnées OIDC, DNS/TLS | mauvais tenant/cloud, proxy, heure, point de terminaison |
| Client | ID client, URI de redirection, flux, type d’identifiant | définition d’application, Reply URL, secret/certificat/fédération |
| Authentification | Sign-in Log, méthode, appareil, risque | identifiant, MFA, utilisateur/charge de travail désactivé |
| Conditional Access | résultats de politique, Target Resource, conditions | portée, Grant Control, appareil, emplacement, client |
| Jeton | `iss`, `aud`, `tid`, `scp`/`roles`, heure | mauvaise ressource/autorité, consentement, claim/horloge |
| API de ressource | Request ID, Resource-RBAC, politique d’objet | rôle Graph/Exchange, App Access, périmètre d’objet |
| Provisioning/Sync | connecteur/agent, mapping, journal d’exportation/provisionnement | Source Anchor, filtre, doublon, quarantaine |

Une erreur Entra ne se résout pas par des reconnexions arbitraires. Seule la phase détermine si le réseau, la configuration client, l’identité, la politique, le jeton ou l’autorisation de ressource doit être corrigé.

## Histoire technique

Microsoft a développé Windows Azure Active Directory comme service mutualisé d’annuaire cloud et de fédération pour Microsoft Online Services et les applications tierces. Azure AD a adopté des protocoles de jetons modernes et n’a jamais été, sur le plan architectural, un contrôleur de domaine AD DS hébergé. Microsoft Graph a remplacé plusieurs API plus anciennes en tant que plan de contrôle cloud commun.

Hybrid Identity est née d’outils de synchronisation et de fédération tels que DirSync, Azure AD Sync, AD FS et plus tard Azure AD Connect. Password Hash Sync, Pass-through Authentication, Seamless SSO, Cloud Sync et Workload Federation ont complété différents modèles d’exploitation. Managed Identities a déplacé la rotation des identifiants pour les charges de travail Azure dans la plateforme.

En 2023, Microsoft a renommé Azure Active Directory en **Microsoft Entra ID** ; les points de terminaison de protocole, API, principes de tenant et de licence sont restés partie de la même continuité de service ([Microsoft – New name for Azure Active Directory](https://learn.microsoft.com/en-us/entra/fundamentals/new-name)). Cette histoire explique le mélange actuel de désignations héritées `login.microsoftonline.com`, `AzureAD`, de Microsoft Graph, de termes hybrides AD DS et de noms de produit Entra.

## Liste de contrôle d’administration en un coup d’œil

Pour conclure, les identités, applications, rôles, politiques et la récupération sont considérés ensemble. La liste de contrôle sert de preuve opérationnelle concise pour ces contrôles interdépendants.

| Question | Preuve opérationnelle |
|---|---|
| Quel tenant ? | ID de tenant, cloud/authority, domaine vérifié, tenant d’origine/de ressource |
| Quel objet ? | Object ID, type, Source of Authority, UPN/mail uniquement comme attributs, état de suppression |
| Quel client ? | ID d’application/client, Application Object, Service Principal dans le tenant de ressource, redirection/flux |
| Quelle identité ? | utilisateur/invité/appareil/Service Principal/Managed Identity, source d’identifiant ou de fédération |
| Quel jeton ? | type, émetteur, audience, tenant, sujet/objet, `scp`/`roles`, heure et ID de clé de signature |
| Quelle autorisation ? | Consent Grant/App-Role, RBAC d’annuaire/de ressource, portée de l’objet/de la boîte aux lettres |
| Quelle politique ? | Conditional Access, Auth Strength, risque, appareil, emplacement, session/CAE |
| Quelle source hybride ? | Connect/Cloud Sync, Source Anchor, filtre, mapping, état d’exportation, méthode d’authentification |
| Quelle preuve ? | UTC, Correlation/Request ID, Sign-in/Audit/Provisioning Log, état Graph |
| Comment restaurer ? | comptes Emergency, politiques/rôles/applications/grants, configuration hybride, domaines/fédération, clés externes et journaux |

## Sources

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
