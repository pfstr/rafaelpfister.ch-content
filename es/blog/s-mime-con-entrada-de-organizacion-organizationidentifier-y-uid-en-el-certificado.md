---
title: "S/MIME con entrada de organización: organizationIdentifier y UID en el certificado"
navTitle: "S/MIME con organización"
description: "Qué contiene un certificado S/MIME con entrada de organización, cómo llega la UID al certificado como organizationIdentifier y qué verifica la autoridad de certificación sobre la empresa y la persona. Incluye una comparación entre SwissSign y DigiCert, comandos para comprobar el tipo de certificado y la distinción respecto a ZertES."
date: "2026-12-15"
kategorie: "Cifrado de correo electrónico"
timeToRead: "13 min de lectura"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "powershell"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2026-12-15"
slug: "s-mime-con-entrada-de-organizacion-organizationidentifier-y-uid-en-el-certificado"
translationId: "article-e0f9bcc57f360835"
translationOf: smime-organisation-uid
translationSourceHash: 849361658c86c3e7bae12ee2d7e018cf43807a611de4911b9ad1bdbe870ea0fe
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T14:57:11.103Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/s-mime-con-entrada-de-organizacion-organizationidentifier-y-uid-en-el-certificado
---

# S/MIME con entrada de organización: organizationIdentifier y UID en el certificado

Las autoridades de certificación (CA) distinguen entre certificados S/MIME con y sin entrada de organización. Un certificado con entrada de organización acredita, además de la dirección de correo electrónico, la empresa a la que pertenece el buzón: con su nombre verificado y un número de registro. Para las empresas suizas, este número de registro es la UID, es decir, el número CHE del registro mercantil.

Conclusión breve: Solo hay una entrada de organización en los tipos de certificado `Organization-validated` y `Sponsor-validated` de los S/MIME Baseline Requirements. Consta del nombre de la empresa en el campo `O` y del campo `organizationIdentifier` (OID 2.5.4.97), en el caso de una empresa suiza, por ejemplo `NTRCH-CHE-123.456.789`. La CA verifica la empresa en un registro oficial antes de incorporar estos datos al certificado. En cambio, un certificado validado para buzón solo acredita el acceso a un buzón. En SwissSign, esto corresponde a elegir entre Silver y Gold; en DigiCert, a elegir entre «Secure Email for Individual Mailbox» y los productos «for Employee» y «for Organization». La entrada de organización no crea una firma cualificada conforme a ZertES.

**Enfoque suizo:** La clasificación jurídica sigue el Derecho suizo (ZertES, OR). Cuando las normas de la UE difieren, se incluye una nota identificada.

## Los cuatro tipos de certificado

Desde el 1 de septiembre de 2023, los certificados S/MIME de confianza pública se rigen por los `S/MIME Baseline Requirements` (BR) del CA/Browser Forum. Distinguen cuatro tipos de certificado según quién figure como titular (`Subject`) en el certificado:

| Tipo | Titular en el certificado | Entrada de organización | Qué verifica la CA |
|---|---|---|---|
| `Mailbox-validated` (MV) | solo la dirección de correo electrónico | no permitida | Control del buzón o del dominio de correo |
| `Organization-validated` (OV) | una organización, sin persona | obligatoria | Organización y buzón o dominio de correo |
| `Sponsor-validated` (SV) | una persona como integrante de una organización | obligatoria | Como OV, además la persona; la CA o la propia organización la verifica |
| `Individual-validated` (IV) | una persona sin organización | no permitida | Identidad de la persona y buzón |

Por «S/MIME con entrada de organización» se entienden los tipos OV y SV. Para ambos, los BR exigen obligatoriamente los campos `organizationName` y `organizationIdentifier`, en las tres generaciones `Legacy`, `Multipurpose` y `Strict`. Las generaciones se diferencian en los campos Subject, extensiones y usos permitidos. Los BR retiraron la generación `Legacy` el 15 de julio de 2025; desde entonces, SwissSign y DigiCert ya no la emiten.

Los certificados OV son adecuados para buzones funcionales como `info@` o `buchhaltung@`, que no están asignados a una persona. Los certificados SV están destinados a buzones personales de empleados e incluyen adicionalmente el nombre de la persona o un seudónimo.

## Estructura del organizationIdentifier

El campo `organizationIdentifier` (OID 2.5.4.97) contiene un número de registro de la organización. Los certificados conforme a eIDAS y ZertES también utilizan este mismo campo. Para los certificados S/MIME, los BR establecen la siguiente estructura:

| Parte | Ejemplo | Significado |
|---|---|---|
| Esquema de registro, 3 caracteres | `NTR` | `National Trade Register`, el registro mercantil |
| Código de país según ISO 3166-1 | `CH` | País en el que se mantiene el registro |
| Guion | `-` | Separador |
| Número de registro | `CHE-123.456.789` | Número de la organización en el registro |

En conjunto, resulta `NTRCH-CHE-123.456.789`. Dado que la propia UID contiene un guion, los BR establecen que el esquema y el código de país nunca contienen un guion. El primer guion desde la izquierda separa, por tanto, el esquema del número de registro; todos los demás pertenecen al número. Si un registro asigna números solo a nivel de un estado federado o una provincia, tras el código de país sigue un `+` con el código de la región, por ejemplo `NTRUS+CA-12345678` para California.

Los BR conocen estos esquemas de registro:

| Esquema | Procedencia del número | Ejemplo de los BR |
|---|---|---|
| `NTR` | Registro mercantil o de empresas | `NTRGB-12345678` |
| `VAT` | Autoridad fiscal (número de IVA) | `VATEL-123456789` |
| `PSD` | Número de autorización de un proveedor de servicios de pago conforme a la Directiva de la UE 2015/2366 | `PSDBE-NBB-1234.567.890` |
| `LEI` | `Legal Entity Identifier` según ISO 17442, código de país siempre `XG` | `LEIXG-` seguido del LEI |
| `GOV` | Organismos públicos sin número de uno de los demás esquemas | `GOVUS` |
| `INT` | Organizaciones internacionales | `INTXG` |

Para las empresas suizas, `NTR` con la UID es la regla general. La UID la asigna la Oficina Federal de Estadística. Los registros mercantiles la utilizan como número de empresa; también sirve como número de IVA y no cambia cuando una empresa traslada su sede a otro cantón. En cambio, un número de IVA puede dejar de existir al finalizar la obligación tributaria, y no todas las organizaciones disponen de uno. SwissSign adopta la UID con los puntos, tal como consta en el registro UID, y cita como ejemplo `NTRCH-CHE-109.357.012` para SwissSign AG.

**Nota sobre la UE:** En Alemania, los tribunales de distrito asignan los números del registro mercantil, y el mismo número aparece en varios tribunales. Por ello, desde la versión 1.0.9 de mayo de 2025, los BR permiten en el esquema `NTR` el `European Unique Identifier` (EUID) y lo recomiendan para empresas alemanas, por ejemplo `NTRDE-DER3306.HRB12345` para una empresa inscrita en el registro mercantil del tribunal de distrito de Colonia. Según los BR, el número de registro solo debe ser único cuando el registro y el ordenamiento jurídico asignan números únicos.

## Qué verifica la CA

Antes de emitir un certificado OV o SV, una CA debe verificar los siguientes datos conforme a los BR:

- **Nombre y existencia de la organización:** mediante una autoridad del país de la organización, datos LEI, una visita in situ o una confirmación con extracto del registro. Para una empresa suiza, se trata del registro mercantil y del registro UID. Si la empresa figura allí como eliminada o inactiva, la CA no puede emitir el certificado.
- **Número de registro:** La CA confirma que el número del `organizationIdentifier` pertenece a la organización indicada en el campo `O`.
- **Autorización:** Por un canal de comunicación fiable, la CA verifica que la persona solicitante puede solicitar certificados para la organización o actuar como `Enterprise RA`.
- **Buzón o dominio de correo:** La CA verifica el control sobre las direcciones de correo electrónico del certificado. Si la organización actúa como `Enterprise RA`, debe demostrar el control sobre todo el dominio de correo.
- **Persona (solo SV):** Nombre o seudónimo de la persona. La siguiente sección describe qué pruebas son admisibles para ello.

Una CA puede reutilizar los resultados de estas verificaciones: la verificación de la organización y de la autorización durante 825 días, y la verificación del control del dominio durante 398 días. Un certificado de las generaciones `Multipurpose` y `Strict` tiene una validez máxima de 825 días.

## Cómo se verifica a la persona en SV

Los BR admiten las siguientes pruebas para la identidad de la persona:

| Prueba | Procedimiento |
|---|---|
| Documento oficial de identidad | Original presencialmente o en tiempo real ante una cámara; la persona verificadora compara la foto del documento con la persona y registra el emisor, la validez y el número del documento |
| Chip del pasaporte o documento de identidad | Verificación de la firma del emisor en los datos del chip, además de la comparación fotográfica |
| eID | Solo sistemas eID notificados conforme a eIDAS con nivel de garantía «sustancial» o «alto» |
| Permiso de conducir móvil (mDL) | Consulta en tiempo real conforme a ISO/IEC 18013-5 y 18013-7 con verificación de la cadena de firmas del emisor |
| Firma digital | Firma de la solicitud con un certificado personal existente de un marco reconocido; la lista de estos marcos aún está vacía en los BR |
| Confirmación de notaría o bufete de abogados | Confirmación por escrito del país de la persona solicitante |
| Documentación del `Enterprise RA` | La empresa verifica por sí misma a sus empleados y puede basarse en datos internos existentes, por ejemplo, de recursos humanos |

Cada CA debe publicar en su CPS cuáles de estas pruebas acepta. La verificación de la persona no puede tener más de 825 días de antigüedad en el momento de la emisión. Si la empresa actúa por sí misma como `Enterprise RA`, la CA debe obligarla contractualmente y verificarla al menos una vez al año, salvo que la propia empresa sea auditada anualmente. En este caso, la vinculación con la persona es tan fiable como el proceso interno de la empresa.

## Comparación entre SwissSign y DigiCert

Ambas CA ofrecen los tipos MV, SV y OV bajo nombres de producto propios. Ninguna de las dos ofrece un certificado IV. La estructura del `organizationIdentifier` está fijada por los BR y es la misma para ambos proveedores.

| | SwissSign | DigiCert |
|---|---|---|
| MV | Personal S/MIME E-Mail ID Silver | Secure Email for Individual Mailbox |
| SV | Pro S/MIME E-Mail ID Gold | Secure Email for Employee (antes Secure Email for Business) |
| OV | Organization Validated E-Mail Certificate | Secure Email for Organization |
| IV | no disponible | no disponible |
| Obtención | Silver en la tienda web o mediante la Managed PKI; Gold y OV solo mediante la Managed PKI de la empresa | los tres mediante la cuenta CertCentral de la empresa |
| Verificación de la persona en SV | documento oficial de identidad original, presencialmente o mediante un procedimiento equivalente; otras pruebas solo con autorización de SwissSign | confirmación por la empresa; DigiCert no verifica a la persona |
| Validez de las verificaciones | datos y pruebas como máximo 398 días | organización 825 días, dominio 398 días |
| Generación en SV y OV | `Multipurpose`; una CA intermedia para SV de la generación `Strict` ya está descrita en el CP/CPS | `Strict` como estándar, `Multipurpose` seleccionable |
| Persona en SV | nombre y apellidos o seudónimo con prefijo `pseudo:` en el `CN` | nombre y apellidos o seudónimo; en el `CN`, opcionalmente el nombre o la dirección de correo electrónico |
| Usos adicionales en SV | ninguno en el perfil estándar; en el perfil con autenticación: autenticación de cliente, inicio de sesión con tarjeta inteligente y EFS | en `Strict` no repudio; en `Multipurpose` además cifrado de datos y autenticación de cliente |

DigiCert describe expresamente el producto SV conforme al modelo del `Enterprise RA`: la organización confirma que la persona indicada en el certificado pertenece a sus empleados o representantes y conserva las pruebas correspondientes. DigiCert solo verifica la organización. Quien esté autorizado para realizar pedidos en la cuenta CertCentral de la empresa puede, por tanto, introducir cualquier nombre en una dirección de los dominios propios. Para OV y SV, DigiCert exige siempre la prueba del control del dominio. La verificación de la organización es válida durante 825 días y la del dominio durante 398 días. Se puede elegir una duración de uno o dos años o una fecha de vencimiento propia de hasta 824 días. DigiCert retiró los productos anteriores Premium, Email Security Plus, Digital Signature Plus y Class 1 S/MIME el 10 de julio de 2025 y los sustituyó por estos productos.

SwissSign emite certificados Gold y OV solo mediante la Managed PKI de la empresa. La persona debe identificarse igualmente con un documento oficial de identidad original, en persona o mediante un procedimiento equivalente conforme a ETSI EN 319 411-1. La identificación la realiza el organismo de registro que procesa la solicitud o un socio de identificación formado y contratado: copiar, escanear o fotografiar el documento, comparar la foto con la persona y confirmar el proceso a SwissSign. El uso del nombre de la empresa debe ser autorizado por la dirección ejecutiva o el consejo de administración inscrito en el registro mercantil, y las personas que solicitan certificados en la Managed PKI también se identifican con un documento de identidad. Una empresa solo puede utilizar otro procedimiento, por ejemplo basado en datos de personal, si SwissSign lo ha verificado y aprobado como equivalente. Los datos y pruebas son válidos en SwissSign durante un máximo de 398 días; después, SwissSign vuelve a verificarlos.

En SwissSign, el certificado OV contiene en el `CN` el nombre de la organización tal como figura en el campo `O`. El artículo [S/MIME en bancos: situación jurídica en Suiza y la elección entre Silver y Gold](/blog/smime-silver-gold-schweiz) describe qué nivel debería elegir un banco para cada buzón y sus implicaciones jurídicas.

## Comprobar el tipo de certificado

En Outlook y otros programas de correo, los detalles del certificado de un correo firmado se pueden abrir mediante el icono de firma. En Windows, los campos se encuentran en la pestaña «Detalles», en la entrada «Sujeto». Windows no conoce un nombre de visualización para la OID 2.5.4.97 y la muestra como `OID.2.5.4.97`.

### Mostrar el titular con PowerShell

Para un certificado exportado (por ejemplo, desde los detalles del certificado de un correo firmado):

```powershell
$zert = Get-PfxCertificate -FilePath .\zertifikat.cer
$zert.SubjectName.Format($true)
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-PfxCertificate` | lee un certificado desde un archivo, también desde un archivo `.cer` sin clave privada |
| `-FilePath .\zertifikat.cer` | ruta al archivo de certificado |
| `SubjectName.Format($true)` | muestra los campos del titular línea por línea |

</details>

Salida para un certificado SV (ejemplo):

```text
CN=Max Muster
G=Max
SN=Muster
O=Muster AG
OID.2.5.4.97=NTRCH-CHE-123.456.789
S=ZH
C=CH
```

En un certificado MV, la salida solo contiene `CN=` con la dirección de correo electrónico.

### Determinar el tipo y la generación mediante la OID de política

Cada certificado conforme a los BR contiene en la extensión `Certificate Policies` una OID con el patrón `2.23.140.1.5.<Typ>.<Generation>`:

| Posición | Valor | Significado |
|---|---|---|
| Tipo | `1` | `Mailbox-validated` |
| Tipo | `2` | `Organization-validated` |
| Tipo | `3` | `Sponsor-validated` |
| Tipo | `4` | `Individual-validated` |
| Generación | `1` | `Legacy` |
| Generación | `2` | `Multipurpose` |
| Generación | `3` | `Strict` |

```powershell
$zert.Extensions |
  Where-Object { $_.Oid.Value -eq '2.5.29.32' } |
  ForEach-Object { $_.Format($true) }
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `$zert.Extensions` | extensiones del certificado leído previamente |
| `'2.5.29.32'` | OID de la extensión `Certificate Policies` |
| `Format($true)` | muestra el contenido de la extensión en varias líneas |

</details>

Salida (abreviada):

```text
[1]Certificate Policy:
     Policy Identifier=2.23.140.1.5.3.2
```

`2.23.140.1.5.3.2` corresponde a un certificado SV de la generación `Multipurpose`. Es normal que haya otras entradas con OID de la propia CA.

### Listar certificados S/MIME en el almacén de certificados propio

```powershell
Get-ChildItem Cert:\CurrentUser\My |
  Where-Object { $_.EnhancedKeyUsageList.ObjectId -contains '1.3.6.1.5.5.7.3.4' } |
  Select-Object Subject, NotAfter
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Cert:\CurrentUser\My` | almacén de certificados personal del usuario conectado |
| `EnhancedKeyUsageList.ObjectId` | usos de clave extendidos del certificado |
| `'1.3.6.1.5.5.7.3.4'` | OID para protección de correo electrónico (`id-kp-emailProtection`) |
| `Select-Object Subject, NotAfter` | muestra el titular y la fecha de vencimiento |

</details>

### Con OpenSSL

```bash
openssl x509 -in zertifikat.pem -noout -subject -nameopt multiline
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `x509` | subcomando para certificados |
| `-in zertifikat.pem` | archivo de entrada en formato PEM; para un archivo DER, añadir `-inform der` |
| `-noout` | suprime la salida del certificado codificado |
| `-subject` | muestra el titular |
| `-nameopt multiline` | un campo por línea con nombres de campo completos |

</details>

OpenSSL 3 reconoce el nombre del campo y lo muestra completo:

```text
subject=
    commonName                = Max Muster
    givenName                 = Max
    surname                   = Muster
    organizationName          = Muster AG
    organizationIdentifier    = NTRCH-CHE-123.456.789
    stateOrProvinceName       = ZH
    countryName               = CH
```

### Verificar el número de registro

Como destinatario de un correo firmado, puede verificar la entrada de organización en cuatro pasos:

1. El programa de correo indica que la firma es válida y el certificado procede de una CA en la que confía el programa de correo.
2. La OID de política comienza por `2.23.140.1.5.2` o `2.23.140.1.5.3`, y el certificado contiene los campos `O` y `organizationIdentifier`.
3. La UID del `organizationIdentifier` pertenece en el registro UID a la empresa esperada, y la empresa figura allí como activa.
4. El dominio de la dirección del remitente pertenece a esa empresa.

## Distinción respecto a ZertES

La entrada de organización no convierte un certificado S/MIME en un certificado regulado conforme a ZertES. Solo las proveedoras reconocidas conforme a ZertES emiten certificados regulados, y el certificado debe indicar que se trata de un certificado regulado (art. 2, letra g, y art. 7, apdo. 2, letra b, ZertES).

Existen paralelismos en cuanto al contenido. Un certificado regulado para una unidad UID debe contener su UID (art. 7, apdo. 2, letra e, ZertES), igual que el `organizationIdentifier` la contiene en OV y SV. Un certificado para una persona física puede incluir además la indicación de que está autorizada a representar a una determinada unidad UID (art. 7, apdo. 3, letra b, ZertES); esto corresponde en su estructura a un certificado SV.

| | S/MIME con entrada de organización | Certificado regulado conforme a ZertES |
|---|---|---|
| Base | S/MIME Baseline Requirements del CA/Browser Forum | ZertES y disposiciones de ejecución |
| Emisora | CA de confianza pública | proveedora reconocida conforme a ZertES |
| Identificador de organización | `organizationIdentifier`, para empresas suizas con la UID | UID, obligatoria para unidades UID |
| Verificación de la persona | en SV, según la CA, mediante documento de identidad o confirmación de la organización como `Enterprise RA` | en principio, comparecencia personal con prueba de identidad; para unidades UID, un representante con prueba de sus poderes de representación (art. 9 ZertES) |
| Clave | conforme a los requisitos de la CA, también en una pasarela | requisitos conforme al art. 6 ZertES |

Según el art. 14, apdo. 2bis, OR, solo la firma electrónica cualificada con sello de tiempo cualificado equivale a la firma manuscrita, y un certificado cualificado solo puede emitirse para una persona física (art. 8, apdo. 1, ZertES). Para la firma de una organización, ZertES prevé el sello electrónico regulado: una firma electrónica avanzada creada con una unidad segura de creación de sellos y basada en un certificado regulado de una unidad UID (art. 2, letra d, ZertES). Un correo firmado con S/MIME es un medio de prueba cuyo valor aprecia libremente el tribunal (art. 157 ZPO); una entrada de organización verificada respalda la atribución a la empresa remitente.

**Nota sobre la UE:** En la UE, el Reglamento eIDAS regula las firmas y sellos electrónicos. Conforme al art. 25, apdo. 2, eIDAS, también solo la firma electrónica cualificada equivale a la firma manuscrita.

## Recomendación

- **Buzones funcionales:** Certificado OV (SwissSign Organization Validated, DigiCert Secure Email for Organization). Acredita la empresa sin mencionar a una persona.
- **Buzones personales con firma frente a clientes y socios:** Certificado SV (SwissSign Gold, DigiCert Secure Email for Employee). Acredita a la persona y a la empresa.
- **Solo cifrado:** Criptográficamente, todos los tipos son equivalentes. Para el cifrado por sí solo basta un certificado MV.
- **Planificar las verificaciones:** La verificación de la organización es válida durante 825 días y la del dominio durante 398 días. Tras un cambio de nombre o sede de la empresa, los datos del certificado ya no coinciden con el registro; planifique entonces nuevos certificados.
- **Comprobar al recibir:** La OID de política y la UID muestran si una firma está atribuida a una empresa verificada. La calidad de la verificación de la persona depende en SV del procedimiento de la CA: en SwissSign se basa en la comprobación de un documento de identidad; en DigiCert, en la confirmación de la empresa.

## Fuentes

1.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): versión 1.0.17 del 20 de noviembre de 2026; tipos de certificado y OID de política (sección 1.1), `Enterprise RA` (1.3.2), verificación de la organización (3.2.3), verificación de la persona (3.2.4), verificación de la autorización (3.2.6), reutilización de resultados de verificación (4.2.1), períodos de validez (6.3.2), campos Subject (7.1.4.2) y esquemas de registro (anexo A).

2.  [cabforum/smime en GitHub: SBR.md](https://github.com/cabforum/smime/blob/main/SBR.md): versión Markdown de los S/MIME Baseline Requirements con historial de versiones, incluido EUID desde la versión 1.0.9.

3.  [SwissSign: EUID, How Your Digital Certificates Become Even More Trustworthy](https://www.swisssign.com/en/blog/european-unique-identifier-euid): denominaciones de producto Silver y Gold, ejemplo `NTRCH-CHE-109.357.012`, UID como número de registro y EUID para empresas alemanas (28.04.2025).

4.  [SwissSign CP/CPS S/MIME](https://repository.swisssign.com/SwissSign_CPS_SMIME.pdf): normativa combinada de SwissSign, versión 1.0 del 31.08.2026, que sustituye entre otros la CPR S/MIME; identificación de la persona (3.2.3), autorización (3.2.5), validez de las pruebas (3.3.1), vías de obtención (4.1.1) y perfiles de certificado para MV, SV y OV (7.1).

5.  [DigiCert: Secure Email products](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates.html): resumen de productos con asignación a MV, OV y SV, fin de la generación `Legacy` y de los productos S/MIME anteriores el 10 de julio de 2025.

6.  [DigiCert: Order your Secure Email for Employee certificate](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates/order-your-secure-email-for-employee-certificate.html): verificación de la organización y del dominio, confirmación de empleados por la organización sin verificación propia de la persona por DigiCert, perfiles `Strict` y `Multipurpose`, períodos de validez.

7.  [Ley federal sobre la firma electrónica (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): art. 2 (conceptos, sello regulado), art. 6 (claves), art. 7 (contenido de certificados regulados, UID), art. 8 (certificados cualificados) y art. 9 (verificación de identidad).

8.  [Código de Obligaciones (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14, apdo. 2bis, sobre la equivalencia de la firma electrónica cualificada con la firma manuscrita.

9.  [Registro UID](https://www.uid.admin.ch): consulta de una UID con nombre, sede y estado, mantenido por la Oficina Federal de Estadística.

10.  [Zefix](https://www.zefix.ch): índice central de empresas de los registros mercantiles cantonales.

11.  [Código Procesal Civil (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 157 sobre la libre valoración de la prueba.

12.  [Reglamento (UE) n.º 910/2014 (eIDAS)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32014R0910): art. 25, apdo. 2, sobre la equivalencia de la firma electrónica cualificada con la firma manuscrita.
