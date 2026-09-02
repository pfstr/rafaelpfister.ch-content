---
title: "Microsoft Entra Domain Services: LDAP y Kerberos para entornos solo en la nube"
navTitle: "Entra Domain Services"
description: "Entra ID no habla LDAP ni Kerberos. Microsoft Entra Domain Services proporciona un dominio de Active Directory administrado que sincroniza usuarios desde Entra ID y ofrece protocolos clásicos. Funcionamiento, limitaciones, costes y un caso práctico con una puerta de enlace de correo electrónico."
date: "2026-07-30"
kategorie: "Active Directory / Entra"
timeToRead: "8 min de lectura"
themen:
  - active-directory-entra
slug: "microsoft-entra-domain-services-ldap-y-kerberos-para-entornos-solo-en-la-nube"
translationId: "article-9c271900a94406b8"
draft: false
translationOf: microsoft-entra-domain-services-ldap-kerberos
translationSourceHash: 6360f60ed2e92d286f0e279f487b62a86fa9a987c2f574b0a53af0d31f0d736b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:21:11.079Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/microsoft-entra-domain-services-ldap-y-kerberos-para-entornos-solo-en-la-nube
---

# Microsoft Entra Domain Services: LDAP y Kerberos para entornos solo en la nube

Quien haya trasladado por completo sus usuarios a Microsoft Entra ID (antes Azure Active Directory) lo notará, como muy tarde, con la primera appliance o aplicación heredada: Entra ID responde a consultas mediante Microsoft Graph y protocolos de autenticación modernos como OAuth y SAML, pero no mediante LDAP, Kerberos o NTLM. Un enlace LDAP contra Entra ID sencillamente no es posible. Para todo lo que espera un Active Directory clásico, Microsoft ofrece su propio servicio: Microsoft Entra Domain Services, antes Azure AD Domain Services.

## Qué proporciona el servicio

Entra Domain Services es un dominio de Active Directory administrado. Microsoft opera para ello dos controladores de dominio de Windows en una VNet de Azure, se encarga de las actualizaciones, la replicación y las copias de seguridad, y sincroniza automáticamente usuarios y grupos desde Entra ID hacia el dominio. La sincronización funciona en un único sentido, desde Entra ID al dominio administrado; los cambios realizados directamente en el dominio no se devuelven.

De cara al exterior, el dominio se comporta como un Active Directory convencional: responde a consultas LDAP y LDAPS, admite autenticación Kerberos y NTLM, permite unir VM al dominio y ofrece directivas de grupo limitadas. No es necesario adaptar aplicaciones ni dispositivos; ven un controlador de dominio.

## Para qué se necesita

El servicio está dirigido a entornos que en realidad son solo en la nube, pero operan componentes individuales con requisitos de directorio clásicos:

- **Appliances y aplicaciones especializadas con conexión LDAP:** dispositivos que buscan usuarios mediante LDAP, evalúan pertenencias a grupos o verifican inicios de sesión mediante un enlace LDAP.
- **Migraciones lift-and-shift:** cargas de trabajo de servidores que deben seguir vinculadas al dominio (Kerberos, NTLM, unión al dominio), sin tener que operar controladores de dominio propios en Azure.
- **Entornos sin AD local:** donde nunca hubo un Active Directory o se ha desmantelado, el dominio administrado sustituye la implantación de controladores de dominio propios y su carga operativa.

Es importante diferenciarlo: quien siga operando un Active Directory local con sincronización mediante Entra Connect, por norma general no necesita el servicio; la appliance consulta entonces el AD existente. Entra Domain Services cubre la brecha cuando Entra ID es la única fuente de usuarios.

## Arquitectura y configuración

El dominio administrado se proporciona en una VNet o subred propia y recibe dos direcciones fijas de controladores de dominio. Las cargas de trabajo en otras VNet lo alcanzan mediante emparejamiento de VNet; los servidores DNS de las VNet implicadas deben apuntar a los controladores de dominio para que el nombre de dominio y los objetos puedan resolverse. El acceso se restringe mediante grupos de seguridad de red a las fuentes y puertos realmente necesarios.

Algunas particularidades del dominio administrado relevantes al conectar aplicaciones:

- Los usuarios sincronizados se encuentran en la OU **AADDC Users** y, sin configuración propia, el dominio lleva el sufijo **onmicrosoft.com**. Search Base y Bind-DN deben reflejar esta estructura, por ejemplo CN=svc-ldap,OU=AADDC Users,DC=example,DC=onmicrosoft,DC=com.
- No existe un administrador de dominio. La administración se realiza mediante el grupo delegado AAD DC Administrators; no es posible ampliar el esquema.
- Para cuentas de enlace LDAP basta una cuenta dedicada sin privilegios; para consultas de directorio puras en Entra ID, el rol Directory Readers.

## El problema de los hashes de contraseña

Hay un punto que suele costar tiempo en las pruebas: los inicios de sesión Kerberos y NTLM, así como los enlaces LDAP, necesitan hashes de contraseña en el dominio administrado. Para cuentas solo en la nube, Entra ID genera estos hashes únicamente con el siguiente cambio de contraseña tras activar el servicio. Por tanto, un usuario recién sincronizado es visible en el directorio, pero solo puede iniciar sesión después de cambiar su contraseña una vez. En las cuentas híbridas, los hashes deben sincronizarse también desde el AD local mediante Entra Connect.

## Secure LDAP paso a paso

Dentro del dominio, LDAP funciona de forma predeterminada sin cifrar a través del puerto 389. Para inicios de sesión y cualquier acceso fuera de redes estrictamente aisladas, la conexión debe usar Secure LDAP (LDAPS, puerto 636); el servicio solo ofrece acceso desde fuera de la VNet de forma cifrada. La configuración consta de cuatro pasos.

**1. Obtener un certificado.** Secure LDAP necesita un certificado propio que se carga como PFX, incluido la clave privada. Subject o SAN debe cubrir el dominio administrado como comodín (por ejemplo, *.example.onmicrosoft.com), ya que las solicitudes pueden llegar a cualquiera de los dos controladores de dominio. Como origen se puede usar una CA pública, la PKI propia o un certificado autofirmado creado específicamente. Con un certificado autofirmado, la cadena debe configurarse como de confianza en cada sistema que realice consultas; no todas las appliances lo permiten. Quien pueda elegir estará más tranquilo con una PKI propia o una CA pública.

**2. Activar Secure LDAP.** En el portal, en Settings > Secure LDAP, se activa la función y se carga el PFX junto con la contraseña. Opcionalmente, allí se puede habilitar el acceso a través de Internet; el dominio administrado recibirá entonces una dirección IP pública.

**3. Red y DNS.** La dirección IP externa se encuentra en Properties. La regla de NSG correspondiente abre TCP/636 y debería limitarse a las direcciones IP de origen realmente necesarias, no a Any. Para la resolución de nombres, una entrada DNS (por ejemplo, ldaps.example.com) apunta a esa IP; debe coincidir con el certificado. Los accesos internos continúan realizándose directamente contra las direcciones de los controladores de dominio.

**4. Probar la conexión.** Antes de cambiar la aplicación, conviene probar con un navegador LDAP, ldp.exe o ldapsearch contra el puerto 636: enlace con la cuenta de servicio y, después, una búsqueda en la OU AADDC Users. Solo cuando el enlace y la búsqueda funcionen correctamente, llega el turno de la aplicación.

Para configurar el propio servicio, la cuenta del portal necesita los roles Application Administrator, Domain Services Contributor y Groups Administrator; el despliegue del dominio administrado tarda algo más de una hora. Además, en la configuración de seguridad se puede imponer TLS 1.2 como mínimo.

## Costes

Entra Domain Services es un coste operativo permanente: el servicio se factura por hora según la SKU, a lo que se suman la VNet, el emparejamiento y las posibles VM de prueba. Para un único caso de uso pequeño de LDAP, es un precio elevado; sin embargo, la alternativa de operar controladores de dominio propios como VM compensa el ahorro con la responsabilidad de actualizaciones, copias de seguridad y disponibilidad.

## Caso práctico: puerta de enlace de correo electrónico con conexión LDAP

Un ejemplo concreto de la categoría de appliances es SEPPmail Secure E-Mail Gateway. Utiliza LDAP para la creación de usuarios y las consultas de autorizaciones, y desde el firmware 15.0.6 también para el [inicio de sesión en la GUI de administración](/blog/seppmail-admin-gui-ldap-authentifizierung). Una appliance en la VNet de Azure alcanza el dominio administrado mediante emparejamiento de VNet con una cuenta de enlace dedicada (Directory Readers), protegida mediante NSG. A más tardar para el inicio de sesión en la GUI de administración, cuya opción TLS está activa de forma predeterminada, la conexión debe usar Secure LDAP.

## Conclusión

Entra Domain Services no sustituye a Entra ID, sino que actúa como puente: el servicio traduce una base de usuarios en la nube a un dominio AD clásico para todo lo que requiera LDAP, Kerberos o unión al dominio. Quien solo necesite conectar una única aplicación debería sopesar los costes recurrentes frente a una modernización de la aplicación. Una vez que el servicio está operativo, las appliances y aplicaciones heredadas se comportan como en un entorno AD habitual, incluidas las particularidades descritas en cuanto a estructura de OU, permisos y hashes de contraseña.

## Fuentes

1.  [Microsoft Learn – «¿Qué es Microsoft Entra Domain Services?»](https://learn.microsoft.com/de-de/entra/identity/domain-services/overview): alcance funcional del dominio administrado, protocolos compatibles y diferenciación respecto a Entra ID y controladores de dominio operados por cuenta propia.

2.  [Microsoft Learn – «Sincronización en Entra Domain Services»](https://learn.microsoft.com/de-de/entra/identity/domain-services/synchronization): sincronización unidireccional, estructura de OU y comportamiento de los hashes de contraseña para cuentas solo en la nube e híbridas.

3.  [Microsoft Learn – «Configurar Secure LDAP»](https://learn.microsoft.com/de-de/entra/identity/domain-services/tutorial-configure-ldaps): LDAPS con certificado propio para accesos LDAP cifrados.

4.  [Conectar la GUI de administración de SEPPmail a Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung): configuración del inicio de sesión LDAP en la GUI de administración a partir del firmware 15.0.6.
