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
url: https://rafaelpfister.ch/es/blog/microsoft-entra-domain-services-ldap-y-kerberos-para-entornos-solo-en-la-nube
translationSourceHash: 00f01b9fa1426d692146e27b2e15e6926e04ea3cccd4855bd0b18c8c10e36e0d
translationModel: gpt-5.6-terra
translatedAt: 2026-07-30T12:22:32.969Z
translationReview: automatic
---

# Microsoft Entra Domain Services: LDAP y Kerberos para entornos solo en la nube

Quien haya migrado por completo sus usuarios a Microsoft Entra ID (anteriormente Azure Active Directory) lo notará, como muy tarde, con la primera appliance o aplicación heredada: Entra ID responde consultas mediante Microsoft Graph y protocolos de autenticación modernos como OAuth y SAML, pero no mediante LDAP, Kerberos o NTLM. Un enlace LDAP contra Entra ID simplemente no es posible. Para todo lo que espera un Active Directory clásico, Microsoft ofrece su propio servicio: Microsoft Entra Domain Services, anteriormente Azure AD Domain Services.

## Qué proporciona el servicio

Entra Domain Services es un dominio de Active Directory administrado. Para ello, Microsoft opera dos controladores de dominio Windows en una VNet de Azure, se encarga de las actualizaciones, la replicación y las copias de seguridad, y sincroniza automáticamente usuarios y grupos desde Entra ID al dominio. La sincronización solo funciona en una dirección, desde Entra ID hacia el dominio administrado; los cambios realizados directamente en el dominio no se sincronizan de vuelta.

De cara al exterior, el dominio se comporta como un Active Directory convencional: responde consultas LDAP y LDAPS, admite autenticación Kerberos y NTLM, permite unir VM al dominio y ofrece directivas de grupo limitadas. No es necesario adaptar aplicaciones ni dispositivos para ello; ven un controlador de dominio.

## Para qué se necesita

El servicio está orientado a entornos que en realidad son solo en la nube, pero que operan componentes individuales con requisitos de directorio clásicos:

- **Appliances y aplicaciones especializadas con integración LDAP:** dispositivos que buscan usuarios mediante LDAP, evalúan pertenencias a grupos o verifican inicios de sesión mediante enlace LDAP.
- **Migraciones lift-and-shift:** cargas de trabajo de servidor que deben permanecer vinculadas al dominio (Kerberos, NTLM, unión al dominio), sin tener que operar controladores de dominio propios en Azure.
- **Entornos sin AD local:** cuando nunca existió un Active Directory o se desmanteló, el dominio administrado sustituye la creación de controladores de dominio propios y la carga operativa asociada.

Importante para diferenciar: quien todavía opera un Active Directory local con sincronización de Entra Connect normalmente no necesita el servicio; en ese caso, la appliance consulta el AD existente. Entra Domain Services cubre la brecha cuando Entra ID es la única fuente de usuarios.

## Arquitectura y configuración

El dominio administrado se aprovisiona en su propia VNet o subred y recibe dos direcciones fijas de controlador de dominio. Las cargas de trabajo de otras VNet lo alcanzan mediante VNet Peering; los servidores DNS de las VNet implicadas deben apuntar a los controladores de dominio para poder resolver el nombre de dominio y los objetos. El acceso se restringe mediante Network Security Groups a las fuentes y puertos realmente necesarios.

Algunas particularidades del dominio administrado relevantes para conectar aplicaciones:

- Los usuarios sincronizados se encuentran en la OU **AADDC Users** y el dominio lleva, sin configuración propia, el sufijo **onmicrosoft.com**. La base de búsqueda y los DN de enlace deben reflejar esta estructura, por ejemplo: CN=svc-ldap,OU=AADDC Users,DC=example,DC=onmicrosoft,DC=com.
- No existe un Domain Administrator. La administración se realiza mediante el grupo delegado AAD DC Administrators; no son posibles las extensiones de esquema.
- Para las cuentas de enlace LDAP basta una cuenta dedicada sin privilegios; para consultas de directorio puras en Entra ID se requiere el rol Directory Readers.

## La trampa de los hashes de contraseña

Hay un aspecto que regularmente consume tiempo en las pruebas: los inicios de sesión Kerberos y NTLM, así como los enlaces LDAP, necesitan hashes de contraseña en el dominio administrado. Para cuentas solo en la nube, Entra ID genera estos hashes únicamente en el siguiente cambio de contraseña tras activar el servicio. Por tanto, un usuario recién sincronizado es visible en el directorio, pero solo podrá iniciar sesión después de haber cambiado su contraseña una vez. En las cuentas híbridas, los hashes deben sincronizarse también desde el AD local mediante Entra Connect.

## Secure LDAP paso a paso

Dentro del dominio, LDAP funciona sin cifrar de forma predeterminada a través del puerto 389. Para inicios de sesión y cualquier acceso fuera de redes estrictamente aisladas, la conexión debe utilizar Secure LDAP (LDAPS, puerto 636); el servicio solo ofrece acceso desde fuera de la VNet de forma cifrada. La configuración consta de cuatro pasos.

**1. Obtener un certificado.** Secure LDAP requiere un certificado propio que se carga como PFX junto con la clave privada. El Subject o SAN debe cubrir el dominio administrado mediante un comodín (por ejemplo, *.example.onmicrosoft.com), ya que las solicitudes pueden llegar a cualquiera de los dos controladores de dominio. Como origen se puede utilizar una CA pública, la PKI propia o un certificado autofirmado creado específicamente. Con un certificado autofirmado, la cadena debe instalarse como confiable en cada sistema que realice consultas; no todas las appliances lo permiten. Quien pueda elegir estará más tranquilo con su propia PKI o una CA pública.

**2. Activar Secure LDAP.** En el portal, en Settings > Secure LDAP, se activa la función y se carga el PFX junto con la contraseña. Opcionalmente, se puede habilitar allí el acceso a través de Internet; el dominio administrado recibe entonces una dirección IP pública.

**3. Red y DNS.** La dirección IP externa aparece en Properties. La regla NSG correspondiente abre TCP/636 y debe restringirse a las direcciones IP de origen realmente necesarias, no a Any. Para la resolución de nombres, una entrada DNS (por ejemplo, ldaps.example.com) apunta a esta IP; debe coincidir con el certificado. Los accesos internos siguen realizándose directamente contra las direcciones de los controladores de dominio.

**4. Probar la conexión.** Antes de cambiar la aplicación, conviene realizar una prueba con un navegador LDAP, ldp.exe o ldapsearch contra el puerto 636: enlace con la cuenta de servicio y, a continuación, una búsqueda en la OU AADDC Users. Solo cuando el enlace y la búsqueda funcionen correctamente se debe pasar a la aplicación.

Para configurar el propio servicio, la cuenta del portal necesita los roles Application Administrator, Domain Services Contributor y Groups Administrator; el despliegue del dominio administrado tarda aproximadamente una hora. Además, en la configuración de seguridad se puede forzar TLS 1.2 como mínimo.

## Costes

Entra Domain Services es un coste operativo permanente: el servicio se factura por hora según la SKU, a lo que se suman VNet, peering y posibles VM de prueba. Para un único caso de uso pequeño de LDAP, es un precio considerable; sin embargo, la alternativa de operar controladores de dominio propios como VM compensa el ahorro con la responsabilidad de actualizaciones, copias de seguridad y disponibilidad.

## Caso práctico: puerta de enlace de correo electrónico con integración LDAP

Un ejemplo concreto de la categoría de appliances es el SEPPmail Secure E-Mail Gateway. Utiliza LDAP para la creación de usuarios y las consultas de permisos, y desde el firmware 15.0.6 también para el [inicio de sesión en la interfaz gráfica de administración](/blog/seppmail-admin-gui-ldap-authentifizierung). Una appliance en la VNet de Azure alcanza el dominio administrado mediante VNet Peering con una cuenta de enlace dedicada (Directory Readers), protegida mediante NSG. A más tardar para el inicio de sesión en la interfaz gráfica de administración, cuya opción TLS está activada de forma predeterminada, la conexión debe utilizar Secure LDAP.

## Conclusión

Entra Domain Services no es un sustituto de Entra ID, sino un puente: el servicio traduce una base de usuarios en la nube a un dominio AD clásico para todo lo que requiere LDAP, Kerberos o unión al dominio. Quien solo tenga que conectar una aplicación debería sopesar los costes continuos frente a una modernización de la aplicación. Una vez que el servicio está en funcionamiento, las appliances y las aplicaciones heredadas se comportan como en un entorno AD conocido, incluidas las particularidades descritas relativas a la estructura de OU, permisos y hashes de contraseña.

## Fuentes

1.  [Microsoft Learn – «¿Qué es Microsoft Entra Domain Services?»](https://learn.microsoft.com/de-de/entra/identity/domain-services/overview): alcance funcional del dominio administrado, protocolos compatibles y diferenciación respecto a Entra ID y controladores de dominio operados por el propio usuario.

2.  [Microsoft Learn – «Sincronización en Entra Domain Services»](https://learn.microsoft.com/de-de/entra/identity/domain-services/synchronization): sincronización unidireccional, estructura de OU y comportamiento de los hashes de contraseña para cuentas solo en la nube e híbridas.

3.  [Microsoft Learn – «Configurar Secure LDAP»](https://learn.microsoft.com/de-de/entra/identity/domain-services/tutorial-configure-ldaps): LDAPS con certificado propio para accesos LDAP cifrados.

4.  [Conectar la interfaz gráfica de administración de SEPPmail a Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung): configuración del inicio de sesión LDAP en la interfaz gráfica de administración a partir del firmware 15.0.6.
