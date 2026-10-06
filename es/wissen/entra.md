---
title: "Microsoft Entra ID: inquilino, tokens e identidad híbrida"
blatt: "entra"
description: "Microsoft Entra ID para administradores de infraestructura y mensajería: objetos de inquilino y directorio, Security Token Service, OAuth/OIDC y SAML, aplicaciones y Service Principals, identidades de carga de trabajo, consentimiento y autorización, Conditional Access, dispositivos, sincronización híbrida, Domain Services, Microsoft Graph, registros, recuperación e historia técnica."
fakten:
  - label: Función del sistema
    wert: servicio de directorio e identidad en la nube multiinquilino con plano de control de tokens, políticas y administración
    href: https://learn.microsoft.com/en-us/entra/fundamentals/whatis
  - label: Inquilino
    wert: límite independiente de directorio/políticas con ID de inquilino, dominios verificados y objetos
    href: https://learn.microsoft.com/en-us/entra/identity-platform/single-and-multi-tenant-apps
  - label: Protocolos
    wert: OAuth 2.0 · OpenID Connect · SAML 2.0 · WS-Federation mediante HTTPS
    href: https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols
  - label: API de administración
    wert: Microsoft Graph para objetos de directorio, identidad, políticas, auditoría y aplicaciones
    href: https://learn.microsoft.com/en-us/graph/overview
  - label: Modelo de aplicaciones
    wert: Application Object como definición · Service Principal como instancia local en el inquilino
    href: https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals
  - label: Tokens de usuario
    wert: ID Token para sesión de cliente · Access Token para Resource API · Refresh Token para renovación
    href: https://learn.microsoft.com/en-us/entra/identity-platform/security-tokens
  - label: Identidad de carga de trabajo
    wert: Service Principal o Managed Identity; certificado, secreto o credencial federada
    href: https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-overview
  - label: Autorización
    wert: Scopes delegados · Application Roles · Directory Roles · política de recurso/objeto
    href: https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview
  - label: Política de acceso
    wert: Conditional Access evalúa señales y aplica controles en las decisiones de tokens
    href: https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview
  - label: Puente híbrido
    wert: Connect Sync o Cloud Sync replica objetos y atributos seleccionados de AD DS
    href: https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync
  - label: Servicios de dominio heredados
    wert: Entra Domain Services proporciona Kerberos, NTLM, LDAP y Domain Join administrados
    href: https://learn.microsoft.com/en-us/entra/identity/domain-services/overview
  - label: Estado operativo
    wert: emisión de tokens · resultado de políticas · sincronización/aprovisionamiento · caducidad de credenciales · inicio de sesión/auditoría · Break Glass
    href: https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-sign-ins
werbung:
  - newsletter
ctaThemen:
  - active-directory-entra
  - microsoft-entra
  - powershell
translationSourceHash: 6d1a58b89bfeb1ae9f9b24608e56afa198b91d6e6681bb8692d749a36e5e1ef3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T11:06:12.592Z
translationReview: automatic
---

# Microsoft Entra ID: inquilino, tokens e identidad híbrida

Microsoft Entra ID es un directorio en la nube, proveedor de identidad y punto de aplicación de políticas para aplicaciones de Microsoft y de terceros. Almacena usuarios, grupos, dispositivos, aplicaciones, Service Principals y roles; su Security Token Service emite tokens firmados tras una autenticación y comprobación de políticas satisfactorias. Microsoft Graph constituye el plano de control de administración. Estas funciones deben considerarse por separado: la existencia de un objeto de usuario no garantiza un inicio de sesión correcto, un token válido no garantiza autorización funcional para un objeto y un atributo sincronizado no produce un efecto inmediato en todos los servicios de destino.

Para los administradores de mensajería, Entra interviene en varias rutas críticas: los usuarios y `proxyAddresses` alimentan Exchange Online, los grupos controlan listas de distribución y acceso, los permisos de aplicaciones permiten acceso automatizado a buzones o Graph, Conditional Access influye en los inicios de sesión de administradores y clientes, y la sincronización híbrida conecta Active Directory Domain Services con la nube. Por tanto, el inquilino no es simplemente «AD en Internet», sino una plataforma independiente de identidad y autorización.

La explicación sigue una identidad desde el inicio de sesión hasta el acceso a un recurso concreto. Primero se explican el inquilino, el directorio y los tokens; después, los roles, aplicaciones, dispositivos, conectividad híbrida, protocolos y recuperación.

Microsoft Entra ID combina un directorio con emisión de tokens y comprobación de políticas. Un usuario no accede «a Entra», sino que solicita un token para un recurso concreto; Entra comprueba la identidad, la aplicación, el dispositivo y la política.

## Arquitectura: Directory, STS, Policy y Resource

Microsoft describe Entra ID como un servicio de gestión de identidades y accesos basado en la nube. Sus principales componentes son:

1. **Directory:** objetos, atributos, relaciones, roles, dominios y configuración local del inquilino.
2. **Security Token Service (STS):** puntos de conexión de protocolos, autenticación, emisión de tokens y publicación de claves.
3. **Policy Engines:** Conditional Access, Identity Protection, Authentication Methods, consentimiento y otras decisiones de acceso.
4. **Provisioning/Sync:** replicación o aprovisionamiento hacia y desde AD DS, SaaS y fuentes de RR. HH.
5. **Microsoft Graph:** API para la administración de directorio, identidad, auditoría y políticas.
6. **Resource Services:** Exchange Online, Graph, API propias y SaaS validan tokens y aplican su autorización.

Entra es multiinquilino. Un inquilino constituye un límite administrativo y de políticas, pero no implica automáticamente un aislamiento completo de datos o red de todos los servicios SaaS consumidos ([Microsoft Entra fundamentals – What is Microsoft Entra ID?](https://learn.microsoft.com/en-us/entra/fundamentals/whatis)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-entra.svg?v=20260813" title="Interaktive Infografik: Microsoft-Entra-Aufruf von Benutzer oder Workload über Authentisierung, Conditional Access und STS bis Token, Resource API und Microsoft Graph sowie Tenant-, App-, Hybrid-, Geräte- und Recoverygrenzen" loading="lazy">
  <a href="/images/kb-interaktiv-entra.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

## Stack tecnológico desde la perspectiva del administrador

En un servicio SaaS multiinquilino, el stack interno de lenguajes de programación, bases de datos y orquestación no es una característica del producto controlable por el cliente. Deducirlo a partir de nombres de host, bibliotecas de cliente u ofertas de empleo carecería de valor para la operación y la recuperación. El **stack tecnológico desde la perspectiva del administrador** verificable consta de los contratos publicados: HTTPS como transporte, OAuth 2.0, OpenID Connect, SAML y WS-Federation para identidad, JWT/JWS y JWKS para tokens y claves, Microsoft Graph como plano de control REST/OData y sincronización basada en agentes para identidades híbridas. Microsoft documenta expresamente estos protocolos y Graph como interfaces compatibles ([Microsoft identity platform protocols](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols), [Microsoft Graph overview](https://learn.microsoft.com/en-us/graph/overview)).

Por ello, para la administración no importa un lenguaje de servidor supuesto, sino la combinación concreta de inquilino, autoridad, versión del protocolo, formato del token, punto de conexión de Graph, módulo SDK o PowerShell, agente de sincronización y Resource Service. Cada una de estas capas tiene sus propios límites de versionado, permisos, registros y errores.

## Inquilino, dominios y anclajes de objetos

Cada inquilino tiene un ID de inquilino inmutable. Los nombres de dominio verificados proporcionan nombres de inicio de sesión legibles y referencia de enrutamiento, pero no sustituyen el ID de inquilino. Por ello, un cambio de dominio o una configuración multiinquilino no debe identificarse únicamente mediante el sufijo UPN.

Los objetos tienen ID locales al inquilino. Los usuarios pueden ser Member o Guest; un invitado B2B es un objeto en el Resource Tenant con referencia a una identidad externa. Los Application Objects tienen un ID global de cliente/aplicación y los Service Principals, además, su propio Object ID en el inquilino correspondiente. Para la unión y sincronización son relevantes otros anclajes como `onPremisesImmutableId`, ID de dispositivos y atributos de origen.

Los administradores documentan como mínimo:

- ID de inquilino, dominios principales y verificados;
- ID de objeto en lugar de solo nombre para mostrar o UPN;
- Home Tenant frente a Resource Tenant;
- sistema de origen y Source of Authority por atributo;
- estado de eliminación temporal/restauración y ciclo de vida;
- relaciones de licencia, roles y grupos como objetos separados.

Microsoft distingue las aplicaciones de inquilino único y multiinquilino según qué directorios pueden utilizar cuentas y Service Principals ([Microsoft identity platform – Single- and multi-tenant apps](https://learn.microsoft.com/en-us/entra/identity-platform/single-and-multi-tenant-apps)).

## Entra ID no es Active Directory Domain Services

AD DS utiliza dominios, bosques, controladores de dominio, LDAP, Kerberos, NTLM, integración DNS, directivas de grupo y replicación. Entra ID utiliza objetos de inquilino, puntos de conexión HTTPS, protocolos modernos de federación/tokens y políticas basadas en la nube. No expone un punto de conexión LDAP o Kerberos general para aplicaciones.

| Característica | AD DS | Entra ID |
|---|---|---|
| Topología | Bosque, dominio, sitios, controlador de dominio | Inquilino y servicio global en la nube |
| Protocolos principales | Kerberos, LDAP, DNS, SMB/RPC | OAuth 2.0, OpenID Connect, SAML, HTTPS/Graph |
| Vinculación de dispositivos | Domain Join y objeto de equipo | Registered, Entra Joined, Hybrid Joined |
| Política | GPO, ACL, configuración de Kerberos/LDAP | Conditional Access, roles, consentimiento, política de tokens/aplicaciones |
| Aplicación | Service Account/SPN, LDAP Bind, Kerberos | App Registration, Service Principal, Managed Identity |

La comparación de Microsoft señala expresamente los distintos modelos de protocolos y administración ([Microsoft Learn – Compare Active Directory to Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/fundamentals/compare)). Un dispositivo con enlace [LDAP](/kb/ldap) no puede autenticarse directamente con Entra ID sin una puerta de enlace o Domain Services; a la inversa, una API OAuth no entiende un ticket [Kerberos](/kb/kerberos).

## Puntos de conexión de protocolo y metadatos

El punto de conexión de detección de OpenID Connect publica el emisor, Authorization Endpoint, Token Endpoint, URI de JWKS y funciones compatibles. Los clientes utilizan autoridades específicas del inquilino, como un ID de inquilino concreto; `common`, `organizations` o `consumers` permiten tipos de cuenta más amplios y modifican la comprobación del emisor y la admisión de inquilinos.

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

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod), [`curl`](https://curl.se/docs/manpage.html) y [`jq`](https://jqlang.org/manual/) leen metadatos públicos. No realizan validación de tokens. Los formatos OIDC Discovery y JWKS están estandarizados ([OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html), [RFC 7517 – JSON Web Key](https://www.rfc-editor.org/rfc/rfc7517.html)).

Una vez aclarados el inquilino, el dominio y los anclajes de objetos, sigue la ruta de inicio de sesión propiamente dicha. OAuth 2.0, OpenID Connect y SAML cumplen funciones distintas y proporcionan artefactos diferentes.

## OAuth 2.0, OpenID Connect y SAML

OAuth 2.0 delega la autorización para API de recursos. OpenID Connect añade una capa de autenticación con ID Token y UserInfo. SAML transporta assertions firmadas entre Identity Provider y Service Provider. Microsoft publica las variantes de protocolo compatibles de Identity Platform ([Microsoft identity platform protocols](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols), [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html), [OASIS SAML 2.0 Technical Overview](https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html)).

Los flujos OAuth habituales diferencian identidad y tipo de cliente:

- **Authorization Code con PKCE:** usuario interactivo, cliente público o confidencial.
- **Client Credentials:** carga de trabajo sin usuario; Application Permissions/App Roles.
- **On-Behalf-Of:** una API intermedia intercambia un token de usuario por otro para una API posterior.
- **Device Code:** dispositivo sin navegador cómodo; el código se confirma en un segundo dispositivo.
- **Refresh Token:** renueva el acceso sin un inicio de sesión interactivo completo, pero sigue sujeto a eventos de políticas y revocación.

Microsoft documenta cada flujo con sus propios límites de solicitud, credenciales y seguridad ([Authorization Code Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow), [Client Credentials Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow), [On-Behalf-Of Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow), [Device Authorization Grant](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-device-code)).

## Tipos de tokens y claims

Un **ID Token** está destinado al cliente y acredita una autenticación. Un **Access Token** está destinado a una API de recursos; solo ese recurso debe validarlo. Un **Refresh Token** es una credencial frente al Authorization Server y nunca se envía a una API de recursos ([Microsoft identity platform – Security tokens](https://learn.microsoft.com/en-us/entra/identity-platform/security-tokens)).

En los tokens basados en JWT, son relevantes entre otros:

| Claim | Significado operativo |
|---|---|
| `iss` | emisor esperado, incluida la semántica del inquilino/punto de conexión |
| `aud` | API de recursos a la que está destinado el token |
| `tid` | contexto del inquilino |
| `oid` / `sub` | anclaje relativo al objeto o sujeto con distinto ámbito |
| `scp` | scopes delegados de un token de usuario |
| `roles` | Application Roles o claims de roles |
| `exp`, `nbf`, `iat` | límites temporales; son relevantes la hora del servidor y la tolerancia |
| `amr`, `acr` | información de autenticación; no interpretar universalmente como sustituto de políticas |
| `groups` | claim de grupos o señal de exceso cuando la cantidad no cabe en el token |

La validación comprueba firma, algoritmo, emisor, audiencia, tiempo y claims específicos de la aplicación. Las mejores prácticas actuales de JWT advierten sobre la confusión de algoritmos, la confusión entre JWT y la confianza ciega en claims recibidos ([RFC 8725 – JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html), [Microsoft – Validate tokens](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens#validate-tokens)).

Un token puede decodificarse localmente para diagnóstico; los tokens reales no se copian en sitios web ajenos ni en tickets. Decodificar no es comprobar la firma.

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

[`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json), [`cut`](https://www.gnu.org/software/coreutils/manual/html_node/cut-invocation.html), [`tr`](https://www.gnu.org/software/coreutils/manual/html_node/tr-invocation.html) y [`base64`](https://www.gnu.org/software/coreutils/manual/html_node/base64-invocation.html) procesan únicamente una copia local. Tras el análisis se elimina; los registros y el historial de shell no deben almacenar el token.

## Application Object y Service Principal

Una App Registration crea un **Application Object** en el Home Tenant. Describe, entre otras cosas, el ID de cliente, las URI de redirección, App Roles, permisos de API solicitados y credenciales. Cuando la aplicación se utiliza o recibe consentimiento en un inquilino, existe allí un **Service Principal** como instancia local del inquilino. Las Managed Identities son Service Principals especiales cuyo ciclo de vida de credenciales administra Azure ([Application objects and service principals](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals)).

No se confunden estos ID:

- `appId`/ID de cliente identifica la definición de aplicación a nivel de protocolo;
- Object ID de Application Object identifica el objeto de registro en el Home Tenant;
- Object ID de Service Principal identifica la instancia en el Resource Tenant;
- ID de aplicación de recurso/API o audiencia identifica el destino del token.

Un nombre para mostrar no es único ni es un anclaje de automatización. Eliminar y volver a crear no puede reproducir arbitrariamente el mismo ID de cliente y genera nuevas relaciones de objetos.

Un token válido aún no responde qué puede hacer una aplicación. Los permisos delegados y los permisos de aplicación puros se diferencian en si interviene un contexto de usuario y quién aprueba los derechos.

## Delegated Permissions, Application Permissions y consentimiento

Los Delegated Permissions actúan en el contexto de un usuario autenticado y normalmente aparecen como `scp`. La aplicación no puede hacer automáticamente más de lo que permiten el usuario y la política del inquilino. Los Application Permissions actúan sin usuario como App Roles en el claim `roles` y pueden permitir un acceso amplio ([Permissions and consent overview](https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview)).

El consentimiento crea grants locales al inquilino y asignaciones de App Roles. El consentimiento de administrador no es una simple confirmación de un cuadro de diálogo, sino un cambio de autorización. El inventario y la revisión incluyen:

- Service Principal de cliente y de recurso;
- grants de scopes delegados y asignaciones de App Roles de aplicación;
- quién otorgó el consentimiento, cuándo y mediante qué proceso;
- uso real, verificación del publicador y procedencia;
- restricción de objetos/buzones en el recurso, cuando esté disponible;
- efecto de revocación e interrupción.

En Exchange Online, un Application Permission puede limitarse adicionalmente mediante una política de acceso específica de Exchange o RBAC for Applications. El objeto de consentimiento de Graph/Entra por sí solo no refleja por completo el alcance funcional de los buzones ([Exchange Online – Role Based Access Control for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)).

## Workload Identities y credenciales

Una Workload Identity es una identidad de software, normalmente un Service Principal o una Managed Identity. Las credenciales pueden ser Client Secret, certificado/clave privada, credencial de plataforma Managed Identity o Federated Identity Credential. Los secretos son credenciales bearer; los certificados y la federación mejoran la posesión de claves o evitan secretos persistentes almacenados, pero requieren su propia operación de confianza ([Microsoft Entra Workload ID overview](https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-overview)).

**Workload Identity Federation** confía en un emisor OIDC externo y vincula emisor, sujeto y audiencia a un Service Principal o una User-Assigned Managed Identity. Un sistema de CI, una cuenta de servicio de Kubernetes o una carga de trabajo en la nube intercambia su token externo de corta duración por un token de Entra, sin almacenar un secreto estático ([Workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)).

La operación de credenciales comprende emisor, almacenamiento, acceso a clave privada, NotBefore/Expiry, superposición, rotación, último uso y revocación. La fecha de caducidad en el Application Object no demuestra que una integración utilice realmente esa credencial.

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

[`Connect-MgGraph`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/connect-mggraph), [`Get-MgContext`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/get-mgcontext), [`Get-MgApplication`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.applications/get-mgapplication) y [`Get-MgServicePrincipal`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.applications/get-mgserviceprincipal) muestran el contexto de Graph y ambos tipos de objetos. [`az account show`](https://learn.microsoft.com/en-us/cli/azure/account#az-account-show), [`az ad app`](https://learn.microsoft.com/en-us/cli/azure/ad/app) y [`az ad sp`](https://learn.microsoft.com/en-us/cli/azure/ad/sp) muestran la vista de Azure CLI. Las salidas pueden contener metadatos de credenciales e inventario del inquilino.

## Conditional Access y Continuous Access Evaluation

Conditional Access procesa señales como usuario/carga de trabajo, recurso objetivo, dispositivo, ubicación, riesgo, tipo de cliente y fortaleza de autenticación, y aplica controles de concesión o sesión. Las políticas actúan conjuntamente; una política individual satisfactoria no constituye un resultado global ([Conditional Access overview](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview)).

Report-only y What If ayudan con la planificación, pero no sustituyen un grupo piloto con clientes reales. Casos especiales relevantes para mensajería son Legacy Authentication, SMTP AUTH, clientes móviles, Service Accounts, cuentas Break-Glass, portales de administración e inicios de sesión no interactivos.

Los Access Tokens normalmente son válidos hasta su caducidad. Continuous Access Evaluation permite a los recursos y clientes compatibles considerar antes los eventos críticos y cambios de políticas ([Continuous Access Evaluation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation)). CAE no es una revocación inmediata global para todas las aplicaciones; el recurso y el cliente deben ser compatibles con el protocolo.

## MFA, Authentication Methods y Authentication Strengths

MFA es una afirmación de política, no un método de producto individual. Las Authentication Methods Policies controlan qué métodos pueden registrarse y utilizarse; las Conditional Access Authentication Strengths pueden exigir combinaciones concretas de métodos ([Authentication methods overview](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-methods), [Conditional Access authentication strengths](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-strengths)).

Los métodos resistentes al phishing, como FIDO2/Passkeys o la autenticación basada en certificados, modifican los procesos de inscripción, recuperación y dispositivos. Temporary Access Pass puede permitir el arranque inicial y es en sí una credencial de duración limitada. El restablecimiento por el servicio de asistencia, un nuevo registro de teléfono y los dispositivos perdidos son flujos de trabajo de alto riesgo y pertenecen a la ruta de auditoría.

## Directory Roles, PIM y Administrative Units

Los Directory Roles autorizan funciones de administración de Entra y Microsoft 365. Los roles pueden asignarse directamente, mediante grupos o temporalmente mediante Privileged Identity Management. PIM diferencia entre eligible y active, aprobación, MFA, justificación, duración y Access Reviews ([Microsoft Entra PIM overview](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)).

Las Administrative Units pueden limitar el ámbito de determinados roles a subconjuntos de objetos. No todos los roles u operaciones admiten este ámbito; los portales de Graph, Exchange y seguridad tienen en parte sus propios modelos RBAC ([Administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/administrative-units)).

El privilegio se documenta como una cadena: origen de asignación → activación → token/rol → RBAC del servicio de destino → acción concreta sobre el objeto. «Tener Global Administrator» no explica automáticamente un 403 en Exchange o Graph.

## Identidad de dispositivo y Primary Refresh Token

Entra Registered, Entra Joined y Hybrid Entra Joined son relaciones de dispositivo diferentes. Registered suele vincular un dispositivo personal o administrado por terceros; Joined utiliza Entra como vínculo organizativo principal; Hybrid Joined combina AD-DS Domain Join con registro en Entra ([Microsoft Entra device identity](https://learn.microsoft.com/en-us/entra/identity/devices/overview)).

En Windows, el Primary Refresh Token admite el inicio de sesión único y contiene contexto de dispositivo y autenticación. La emisión y renovación de PRT dependen del estado del usuario, dispositivo y clave; no es un Refresh Token ordinario que los administradores deban copiar ([Primary Refresh Token](https://learn.microsoft.com/en-us/entra/identity/devices/concept-primary-refresh-token)).

El estado del dispositivo, el cumplimiento de MDM, Hybrid Join, los certificados y Conditional Access pueden no coincidir en el tiempo. Por ello, el diagnóstico comprueba conjuntamente la información local de unión, el objeto de dispositivo de Entra, el objeto de administración, el Sign-in Log y el resultado de la política.

En entornos híbridos, la identidad comienza a menudo en el Active Directory local. La sincronización transfiere objetos y atributos seleccionados, pero no convierte Entra ID en un sustituto de LDAP o Kerberos.

## Identidad híbrida: Connect Sync y Cloud Sync

Microsoft Entra Connect Sync ejecuta un motor de sincronización en Windows Server y puede representar reglas extensas y determinadas características híbridas. Cloud Sync utiliza agentes de aprovisionamiento ligeros y configuración controlada desde la nube; las funciones y la topología son distintas ([Microsoft Entra Connect overview](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-azure-ad-connect), [Microsoft Entra Cloud Sync overview](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync)).

Un sistema de sincronización tiene tres niveles:

1. **Connector Spaces** o conector de origen/destino con objetos importados;
2. lógica de **Metaverse/mapping** o configuración de aprovisionamiento en la nube;
3. **Exportación** a Entra con objeto de destino, flujo de atributos y estado de error.

El emparejamiento y Source Anchor evitan duplicados. Los filtros, reglas de unión, prioridad de atributos y Writeback definen la soberanía de datos. Una ejecución satisfactoria del programador no demuestra que todos los objetos se hayan exportado; la cuarentena, los errores de aprovisionamiento y el procesamiento por el servicio en la nube se verifican por separado.

### Autenticación en operación híbrida

- **Password Hash Synchronization (PHS):** se almacena en Entra un hash derivado; la autenticación en la nube sigue siendo posible si falla el entorno local.
- **Pass-through Authentication (PTA):** los agentes validan contraseñas contra AD DS; la ruta de agentes adquiere relevancia operativa.
- **Federation:** Entra redirige la autenticación a un STS federado; los certificados, Claims Rules y la disponibilidad amplían el ámbito de fallo.

Microsoft describe la selección y los compromisos de estos métodos de inicio de sesión ([Choose the right authentication method](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/choose-ad-authn)). Seamless SSO y PRT son mecanismos adicionales y no sinónimos del método de autenticación principal.

## Entra Domain Services

Microsoft Entra Domain Services proporciona un dominio administrado con Domain Join, directivas de grupo, LDAP, Kerberos y NTLM. Los usuarios, grupos y hashes de credenciales se sincronizan desde Entra al dominio administrado. Los clientes no reciben derechos de administrador de dominio o empresarial y no administran ellos mismos los controladores de dominio ([Microsoft Entra Domain Services overview](https://learn.microsoft.com/en-us/entra/identity/domain-services/overview)).

Domain Services no es un puente bidireccional hacia AD DS ni un sustituto de los protocolos de tokens de Entra. Las aplicaciones heredadas ven objetos LDAP/Kerberos en el dominio administrado; las aplicaciones SaaS modernas siguen utilizando Entra. Secure LDAP requiere certificado, límite de red publicado, NSG/firewall y política de credenciales.

## External Identities y acceso entre inquilinos

B2B Collaboration crea objetos de invitado en el Resource Tenant, mientras que la autenticación suele producirse en el Home Tenant. Cross-Tenant Access Settings controla la confianza entrante y saliente, la confianza en claims de MFA/dispositivo y las relaciones organizativas ([Microsoft Entra B2B collaboration overview](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b), [Cross-tenant access overview](https://learn.microsoft.com/en-us/entra/external-id/cross-tenant-access-overview)).

El Resource Tenant continúa autorizando sus datos. Una cuenta principal eliminada o deshabilitada, un objeto de invitado existente, Access Packages y pertenencias a grupos pueden tener ciclos de vida distintos. Las Access Reviews periódicas y los procesos de patrocinador/propietario cierran esta brecha.

## Microsoft Graph como plano de control

Microsoft Graph expone rutas de recursos como `/users`, `/groups`, `/applications`, `/servicePrincipals`, `/policies` y `/auditLogs`. Los permisos, la versión de API, la paginación, el throttling y la consistencia eventual de determinadas consultas forman parte del contrato ([Microsoft Graph overview](https://learn.microsoft.com/en-us/graph/overview)).

Delta Query proporciona cambios desde un estado inicial mediante tokens opacos; Change Notifications envía webhooks, pero no sustituye una ejecución de reconciliación ([Microsoft Graph delta query](https://learn.microsoft.com/en-us/graph/delta-query-overview), [Microsoft Graph change notifications](https://learn.microsoft.com/en-us/graph/change-notifications-overview)). Para las [API](/kb/apis) también se aplican idempotencia, paginación, 429/Retry-After e ID de solicitud.

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

[`Get-MgUser`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguser) y [`Invoke-MgGraphRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/invoke-mggraphrequest) utilizan el contexto de Graph mostrado. [`az account get-access-token`](https://learn.microsoft.com/en-us/cli/azure/account#az-account-get-access-token) proporciona un token de CLI de corta duración; no se registra ni se almacena de forma persistente. Las respuestas de Graph contienen datos personales y relevantes para la seguridad.

## Dependencias de red y TLS

Entra es un servicio HTTPS distribuido. El proxy, firewall, DNS, inspección TLS, hora y lista de permitidos de puntos de conexión influyen en la autenticación, Graph, Device Registration, Sync y revocación. Una lista estática de IP no siempre es el modelo adecuado; Microsoft publica Service Tags y categorías de URL/IP para puntos de conexión relacionados con Microsoft 365 y Entra ([Microsoft 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges), [Azure service tags overview](https://learn.microsoft.com/en-us/azure/virtual-network/service-tags-overview)).

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) comprueban primero la resolución de nombres. [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection), [`nc`](https://man.openbsd.org/nc) y [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) separan TCP y [TLS](/kb/tls). [`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) comprueba HTTP, [`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings) y [`timedatectl`](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html) la fuente de tiempo.

## Sign-in Logs, Audit Logs y Provisioning Logs

Los Sign-in Logs documentan inicios de sesión interactivos, no interactivos, de Service Principal y de Managed Identity con estado, evaluación de Conditional Access e información de cliente, dispositivo y riesgo. Los Audit Logs documentan cambios de directorio y políticas. Los Provisioning Logs muestran pasos de aprovisionamiento y errores de mapping ([Microsoft Entra sign-in logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-sign-ins), [Microsoft Entra audit logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-audit-logs), [Provisioning logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-provisioning-logs)).

Correlation ID, Request ID, hora en UTC, inquilino, ID de cliente, ID de recurso, ID de usuario/Service Principal y código de error constituyen la evidencia mínima. Los textos del portal se complementan con el código de error y la fase de token/protocolo; los mismos mensajes de interfaz pueden tener causas distintas.

La retención de registros depende de la licencia y de la configuración de exportación, y puede cambiar. La evidencia a largo plazo se envía antes del incidente mediante Diagnostic Settings o rutas de exportación compatibles a un sistema de registros controlado.

## Break Glass, copia de seguridad y recuperación

Entra es un servicio SaaS; los clientes no realizan copias de seguridad de una base de datos de controladores de dominio. Sin embargo, deben proteger la configuración, las credenciales externas y la capacidad de recuperación administrativa:

- al menos dos cuentas cloud-only Emergency Access con métodos fuertes independientes;
- exclusiones de Conditional Access solo en la medida necesaria para la recuperación y bajo supervisión;
- exportación de configuración de inquilino/dominio/federación/acceso entre inquilinos y roles;
- App Registrations, Service Principals, grants, App Roles y metadatos de credenciales;
- políticas de Conditional Access, Authentication Methods, PIM y ciclo de vida;
- reglas de sincronización híbrida, Source Anchors, configuración de agentes/servidores y ruta de staging;
- dependencias externas de CA/KMS/federación/DNS/Break-Glass;
- exportación de Audit Logs/Sign-in Logs y tickets de cambios.

Microsoft recomienda cuentas Emergency Access dedicadas, cuyo uso genere alertas y se pruebe periódicamente ([Manage emergency access accounts](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access)).

Los objetos eliminados temporalmente tienen ventanas de restauración y limitaciones específicas del producto, que pueden cambiar. Por ello, los runbooks de recuperación enlazan a la documentación correspondiente de Microsoft y prueban por separado el usuario, grupo, aplicación, Service Principal, configuración incorrecta de Conditional Access y ruta de federación/dominio perdida.

La resolución de problemas sigue la decisión de token: identificar usuario y aplicación, comprobar el método de inicio de sesión y Conditional Access, leer los claims del token y, por último, evaluar la autorización en el recurso.

## Diagnóstico por fase de decisión

Un error de Entra puede delimitarse más rápidamente si primero se determina la fase: inicio de sesión, emisión de token o decisión de la aplicación de destino. La tabla asigna las pruebas adecuadas a cada fase.

| Fase | Evidencia | Clase de error típica |
|---|---|---|
| Discovery | Authority, ID de inquilino, metadatos OIDC, DNS/TLS | inquilino/nube incorrectos, proxy, hora, punto de conexión |
| Cliente | ID de cliente, URI de redirección, flujo, tipo de credencial | definición de aplicación, Reply URL, secreto/certificado/federación |
| Autenticación | Sign-in Log, método, dispositivo, riesgo | credencial, MFA, usuario/carga de trabajo deshabilitados |
| Conditional Access | resultados de políticas, Target Resource, condiciones | ámbito, Grant Control, dispositivo, ubicación, cliente |
| Token | `iss`, `aud`, `tid`, `scp`/`roles`, hora | recurso/autoridad incorrectos, consentimiento, claim/reloj |
| Resource API | Request ID, Resource RBAC, política de objeto | rol de Graph/Exchange, acceso de aplicación, ámbito de objeto |
| Provisioning/Sync | Connector/Agent, mapping, Export/Provisioning Log | Source Anchor, filtro, duplicado, cuarentena |

Un error de Entra no se resuelve mediante inicios de sesión aleatorios. Solo la fase determina si debe corregirse la red, configuración del cliente, identidad, política, token o autorización del recurso.

## Historia técnica

Microsoft desarrolló Windows Azure Active Directory como servicio multiinquilino de directorio en la nube y federación para Microsoft Online Services y aplicaciones de terceros. Azure AD adoptó protocolos de tokens modernos y arquitectónicamente nunca fue un controlador de dominio AD DS alojado. Microsoft Graph sustituyó varias API anteriores como plano de control común de la nube.

Hybrid Identity surgió de herramientas de sincronización y federación como DirSync, Azure AD Sync, AD FS y posteriormente Azure AD Connect. Password Hash Sync, Pass-through Authentication, Seamless SSO, Cloud Sync y Workload Federation añadieron distintos modelos operativos. Managed Identities trasladó la rotación de credenciales de cargas de trabajo de Azure a la plataforma.

En 2023, Microsoft cambió el nombre de Azure Active Directory a **Microsoft Entra ID**; los puntos de conexión de protocolo, las API y las bases de inquilino y licencias siguieron formando parte de la misma continuidad del servicio ([Microsoft – New name for Azure Active Directory](https://learn.microsoft.com/en-us/entra/fundamentals/new-name)). La historia explica la actual combinación de denominaciones antiguas de `login.microsoftonline.com`, `AzureAD`, Microsoft Graph, términos híbridos de AD DS y nombres de productos Entra.

## Lista de comprobación para administradores de un vistazo

Para finalizar, se consideran conjuntamente identidades, aplicaciones, roles, políticas y recuperación. La lista de comprobación sirve como breve evidencia operativa de estos controles relacionados.

| Pregunta | Evidencia operativa |
|---|---|
| ¿Qué inquilino? | ID de inquilino, nube/Authority, dominio verificado, Home/Resource Tenant |
| ¿Qué objeto? | ID de objeto, tipo, Source of Authority, UPN/correo solo como atributos, estado de eliminación |
| ¿Qué cliente? | ID de aplicación/cliente, Application Object, Service Principal en el Resource Tenant, redirección/flujo |
| ¿Qué identidad? | usuario/invitado/dispositivo/Service Principal/Managed Identity, fuente de credencial o federación |
| ¿Qué token? | tipo, emisor, audiencia, inquilino, sujeto/objeto, `scp`/`roles`, hora e ID de clave de firma |
| ¿Qué autorización? | Consent Grant/App Role, Directory/Resource RBAC, ámbito de objeto/buzón |
| ¿Qué política? | Conditional Access, Auth Strength, riesgo, dispositivo, ubicación, sesión/CAE |
| ¿Qué fuente híbrida? | Connect/Cloud Sync, Source Anchor, filtro, mapping, estado de exportación, método de autenticación |
| ¿Qué evidencia? | UTC, Correlation/Request ID, Sign-in/Audit/Provisioning Log, estado de Graph |
| ¿Cómo se recupera? | cuentas Emergency, políticas/roles/aplicaciones/grants, configuración híbrida, dominios/federación, claves y registros externos |

## Fuentes

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
