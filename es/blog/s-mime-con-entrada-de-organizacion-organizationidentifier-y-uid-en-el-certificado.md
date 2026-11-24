---
title: "S/MIME con entrada de organización: organizationIdentifier y UID en el certificado"
navTitle: "S/MIME con organización"
description: "Qué contiene un certificado S/MIME con entrada de organización, cómo se incorpora la UID como organizationIdentifier en el certificado y qué verifica la autoridad de certificación para ello. Incluye una comparación de los productos de SwissSign y DigiCert, comandos para comprobar el tipo de certificado y la distinción respecto a ZertES."
date: "2026-12-15"
kategorie: "Cifrado de correo electrónico"
timeToRead: "11 min de lectura"
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
url: https://rafaelpfister.ch/es/blog/s-mime-con-entrada-de-organizacion-organizationidentifier-y-uid-en-el-certificado
translationSourceHash: f628a8f86635449d2a9e43f78f1a960ce5c8bc20af519eac110b77b2dad385a9
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T13:53:59.447Z
translationReview: automatic
---

# S/MIME con entrada de organización: organizationIdentifier y UID en el certificado

Las autoridades de certificación (CA) distinguen entre certificados S/MIME con y sin entrada de organización. Un certificado con entrada de organización identifica, además de la dirección de correo electrónico, a la empresa a la que pertenece el buzón: con su nombre verificado y un número de registro. Para las empresas suizas, este número de registro es la UID, es decir, el número CHE del registro mercantil.

Conclusión breve: Solo existe una entrada de organización para los tipos de certificado `Organization-validated` y `Sponsor-validated` de los requisitos básicos de S/MIME. Consta del nombre de la empresa en el campo `O` y del campo `organizationIdentifier` (OID 2.5.4.97), en el caso de una empresa suiza, por ejemplo, `NTRCH-CHE-123.456.789`. La CA verifica la empresa en un registro oficial antes de incorporar estos datos al certificado. En cambio, un certificado validado para buzón solo acredita el acceso a un buzón. En SwissSign, esto corresponde a elegir entre Silver y Gold; en DigiCert, a elegir entre «Secure Email for Individual Mailbox» y los productos «for Employee» y «for Organization». Una firma cualificada conforme a ZertES no surge de la entrada de organización.

**Enfoque en Suiza:** La clasificación jurídica sigue el derecho suizo (ZertES, OR). Cuando las normas de la UE difieren, se indica mediante una nota señalada.

## Los cuatro tipos de certificado

Desde el 1 de septiembre de 2023, los certificados S/MIME de confianza pública se rigen por los `S/MIME Baseline Requirements` (BR) del CA/Browser Forum. Distinguen cuatro tipos de certificado según quién figure como titular (`Subject`) en el certificado:

| Tipo | Titular en el certificado | Entrada de organización | Qué verifica la CA |
|---|---|---|---|
| `Mailbox-validated` (MV) | solo la dirección de correo electrónico | no permitida | control del buzón o del dominio de correo |
| `Organization-validated` (OV) | una organización, ninguna persona | obligatoria | organización y buzón o dominio de correo |
| `Sponsor-validated` (SV) | una persona como integrante de una organización | obligatoria | como OV, además la persona; la CA o la propia organización la verifica |
| `Individual-validated` (IV) | una persona sin organización | no permitida | identidad de la persona y buzón |

Por «S/MIME con entrada de organización» se entienden los tipos OV y SV. Para ambos, los BR exigen obligatoriamente los campos `organizationName` y `organizationIdentifier`, en las tres generaciones: `Legacy`, `Multipurpose` y `Strict`. Las generaciones se diferencian por los campos Subject permitidos, las extensiones y los usos previstos. Los BR retiraron la generación `Legacy` el 15 de julio de 2025; desde entonces, SwissSign y DigiCert ya no la emiten.

Los certificados OV son adecuados para buzones funcionales como `info@` o `buchhaltung@`, que no están asignados a una persona. Los certificados SV están pensados para buzones personales de empleados e incluyen además el nombre de la persona o un seudónimo.

## Estructura del organizationIdentifier

El campo `organizationIdentifier` (OID 2.5.4.97) contiene un número de registro de la organización. Los certificados conforme a eIDAS y ZertES utilizan el mismo campo. Para los certificados S/MIME, los BR prescriben la siguiente estructura:

| Parte | Ejemplo | Significado |
|---|---|---|
| Esquema de registro, 3 caracteres | `NTR` | `National Trade Register`, el registro mercantil |
| Código de país según ISO 3166-1 | `CH` | país en el que se lleva el registro |
| Guion | `-` | separador |
| Número de registro | `CHE-123.456.789` | número de la organización en el registro |

Combinado, el resultado es `NTRCH-CHE-123.456.789`. Puesto que la propia UID contiene un guion, los BR establecen que el esquema y el código de país nunca contienen un guion. Por tanto, el primer guion de la izquierda separa el esquema del número de registro; todos los demás pertenecen al número. Si un registro solo asigna números a nivel de un estado federado o de una provincia, tras el código de país sigue un `+` con el código de la región, por ejemplo `NTRUS+CA-12345678` para California.

Los BR contemplan estos esquemas de registro:

| Esquema | Origen del número | Ejemplo de los BR |
|---|---|---|
| `NTR` | registro mercantil o empresarial | `NTRGB-12345678` |
| `VAT` | autoridad fiscal (número de IVA) | `VATEL-123456789` |
| `PSD` | número de autorización de un proveedor de servicios de pago conforme a la Directiva de la UE 2015/2366 | `PSDBE-NBB-1234.567.890` |
| `LEI` | `Legal Entity Identifier` conforme a ISO 17442, código de país siempre `XG` | `LEIXG-` seguido del LEI |
| `GOV` | autoridades públicas sin número de uno de los demás esquemas | `GOVUS` |
| `INT` | organizaciones internacionales | `INTXG` |

Para las empresas suizas, `NTR` con la UID es el caso habitual. La UID la asigna la Oficina Federal de Estadística. Los registros mercantiles la utilizan como número de empresa, también sirve como número de IVA y no cambia cuando una empresa traslada su sede a otro cantón. En cambio, un número de IVA puede dejar de existir cuando termina la obligación tributaria, y no todas las organizaciones tienen uno. SwissSign adopta la UID con los puntos, tal como figura en el registro UID, y cita como ejemplo `NTRCH-CHE-109.357.012` para SwissSign AG.

**Nota sobre la UE:** En Alemania, los tribunales de distrito asignan los números del registro mercantil, y el mismo número aparece en varios tribunales. Por ello, desde la versión 1.0.9 de mayo de 2025, los BR permiten en el esquema `NTR` el `European Unique Identifier` (EUID) y lo recomiendan para las empresas alemanas, por ejemplo `NTRDE-DER3306.HRB12345` para una empresa en el registro mercantil del tribunal de distrito de Colonia. Según los BR, el número de registro solo debe ser inequívoco allí donde el registro y el ordenamiento jurídico asignen números inequívocos.

## Qué verifica la CA

Antes de que una CA emita un certificado OV o SV, debe verificar los siguientes datos conforme a los BR:

- **Nombre y existencia de la organización:** mediante una autoridad del país de la organización, datos LEI, una visita in situ o una confirmación con extracto del registro. Para una empresa suiza, se trata del registro mercantil y del registro UID. Si la empresa figura allí como eliminada o inactiva, la CA no puede emitir el certificado.
- **Número de registro:** La CA confirma que el número del `organizationIdentifier` pertenece a la organización del campo `O`.
- **Poder de representación:** Por un canal de comunicación fiable, la CA verifica que la persona solicitante puede solicitar certificados para la organización o actuar como `Enterprise RA`.
- **Buzón o dominio de correo:** La CA verifica el control de las direcciones de correo electrónico del certificado. Si la organización actúa como `Enterprise RA`, debe demostrar el control de todo el dominio de correo.
- **Persona (solo SV):** Los BR permiten que la organización, como `Enterprise RA`, confirme por sí misma a sus empleados. En ese caso, la CA verifica la organización y no a cada persona individualmente.

Una CA puede reutilizar los resultados de estas verificaciones: la verificación de la organización y del poder de representación durante 825 días, y la verificación del control del dominio durante 398 días. Un certificado de las generaciones `Multipurpose` y `Strict` tiene una validez máxima de 825 días.

## Comparación entre SwissSign y DigiCert

Ambas CA ofrecen los tipos MV, SV y OV con sus propios nombres de producto. Ninguna de las dos ofrece un certificado IV. Los BR definen la estructura del `organizationIdentifier`, que es igual para ambos proveedores.

| | SwissSign | DigiCert |
|---|---|---|
| MV | Personal S/MIME E-Mail ID Silver | Secure Email for Individual Mailbox |
| SV | Pro S/MIME E-Mail ID Gold | Secure Email for Employee (antes Secure Email for Business) |
| OV | Organization Validated E-Mail Certificate | Secure Email for Organization |
| IV | no disponible | no disponible |
| Generación en SV y OV | `Multipurpose` según los perfiles de certificado | `Strict` como estándar, `Multipurpose` seleccionable |
| Persona en SV | nombre y apellidos o seudónimo con el prefijo `pseudo:` en el `CN` | nombre y apellidos o seudónimo; en el `CN`, opcionalmente el nombre o la dirección de correo electrónico |
| Usos adicionales en SV | en el perfil con autenticación: autenticación de cliente, inicio de sesión con tarjeta inteligente, EFS | en `Strict` no repudio; en `Multipurpose`, además cifrado de datos y autenticación de cliente |

DigiCert describe expresamente el producto SV según el modelo del `Enterprise RA`: La organización confirma que la persona del certificado pertenece a sus empleados o representantes y conserva las pruebas correspondientes. DigiCert solo verifica la organización. Para OV y SV, DigiCert exige siempre la prueba del control del dominio. La verificación de la organización es válida durante 825 días y la del dominio durante 398 días. Se puede elegir una duración de uno o dos años, o una fecha de vencimiento propia de hasta 824 días. DigiCert retiró el 10 de julio de 2025 los productos anteriores Premium, Email Security Plus, Digital Signature Plus y Class 1 S/MIME, sustituyéndolos por estos productos.

En SwissSign, el certificado OV contiene en el `CN` el nombre de la organización, como en el campo `O`. El artículo [S/MIME en bancos: situación jurídica en Suiza y elección entre Silver y Gold](/blog/smime-silver-gold-schweiz) describe qué nivel debería elegir un banco para cada buzón y qué implica jurídicamente.

## Comprobar el tipo de certificado

En Outlook y otros programas de correo, los detalles del certificado de un correo firmado se pueden abrir mediante el icono de firma. En Windows, los campos se encuentran en la pestaña «Detalles», en la entrada «Sujeto». Windows no conoce un nombre visible para la OID 2.5.4.97 y la muestra como `OID.2.5.4.97`.

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
| `Get-PfxCertificate` | lee un certificado desde un archivo, incluso desde un archivo `.cer` sin clave privada |
| `-FilePath .\zertifikat.cer` | ruta al archivo de certificado |
| `SubjectName.Format($true)` | muestra los campos del titular, uno por línea |

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

Todo certificado conforme a los BR contiene, en la extensión `Certificate Policies`, una OID con el patrón `2.23.140.1.5.<Typ>.<Generation>`:

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
| `Cert:\CurrentUser\My` | almacén personal de certificados del usuario que ha iniciado sesión |
| `EnhancedKeyUsageList.ObjectId` | usos mejorados de clave del certificado |
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
| `-nameopt multiline` | un campo por línea con nombres de campo desarrollados |

</details>

OpenSSL 3 conoce el nombre del campo y lo muestra desarrollado:

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
2. La OID de política comienza con `2.23.140.1.5.2` o `2.23.140.1.5.3`, y el certificado contiene los campos `O` y `organizationIdentifier`.
3. La UID del `organizationIdentifier` pertenece, en el registro UID, a la empresa esperada y la empresa está activa allí.
4. El dominio de la dirección del remitente pertenece a esta empresa.

## Distinción respecto a ZertES

La entrada de organización no convierte un certificado S/MIME en un certificado regulado conforme a ZertES. Solo los proveedores reconocidos conforme a ZertES emiten certificados regulados, y el certificado debe incluir la indicación de que se trata de un certificado regulado (art. 2, letra g, y art. 7, apdo. 2, letra b, ZertES).

En cuanto al contenido, existen paralelismos. Un certificado regulado para una unidad UID debe contener su UID (art. 7, apdo. 2, letra e, ZertES), igual que el `organizationIdentifier` la contiene en OV y SV. Un certificado para una persona física también puede contener la indicación de que está autorizada para representar una unidad UID determinada (art. 7, apdo. 3, letra b, ZertES); estructuralmente, esto corresponde a un certificado SV.

| | S/MIME con entrada de organización | certificado regulado conforme a ZertES |
|---|---|---|
| Base | Requisitos básicos de S/MIME del CA/Browser Forum | ZertES y disposiciones de ejecución |
| Emisora | CA de confianza pública | proveedora reconocida conforme a ZertES |
| Identificador de organización | `organizationIdentifier`, en empresas suizas con la UID | UID, obligatoria para unidades UID |
| Verificación de la persona | en SV, por la CA o por la organización como `Enterprise RA` | por regla general, comparecencia personal con prueba de identidad; en unidades UID, un representante con prueba de poder de representación (art. 9 ZertES) |
| Clave | conforme a los requisitos de la CA, también en una pasarela | requisitos conforme al art. 6 ZertES |

Según el art. 14, apdo. 2bis, OR, solo la firma electrónica cualificada con sello de tiempo cualificado equivale a la firma manuscrita, y un certificado cualificado solo puede expedirse a una persona física (art. 8, apdo. 1, ZertES). Para la firma de una organización, ZertES prevé el sello electrónico regulado: una firma electrónica avanzada creada con una unidad segura de creación de sellos y basada en un certificado regulado de una unidad UID (art. 2, letra d, ZertES). Un correo firmado con S/MIME es un medio de prueba cuyo valor el tribunal aprecia libremente (art. 157 ZPO); una entrada de organización verificada respalda así la atribución a la empresa remitente.

**Nota sobre la UE:** En la UE, el Reglamento eIDAS regula las firmas y los sellos electrónicos. Conforme al art. 25, apdo. 2, eIDAS, también solo la firma electrónica cualificada equivale a la firma manuscrita.

## Recomendación

- **Buzones funcionales:** Certificado OV (SwissSign Organization Validated, DigiCert Secure Email for Organization). Identifica a la empresa sin mencionar a una persona.
- **Buzones personales con firma ante clientes y socios:** Certificado SV (SwissSign Gold, DigiCert Secure Email for Employee). Identifica a la persona y a la empresa.
- **Solo cifrado:** Criptográficamente, todos los tipos son equivalentes. Para el cifrado por sí solo basta un certificado MV.
- **Planificar las verificaciones:** La verificación de la organización es válida durante 825 días y la del dominio durante 398 días. Tras un cambio de nombre o sede de la empresa, los datos del certificado dejan de coincidir con el registro; planifique entonces nuevos certificados.
- **Comprobar al recibir:** La OID de política y la UID muestran si una firma está realmente asignada a una empresa verificada.

## Fuentes

1.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): versión 1.0.17 del 20 de noviembre de 2026; tipos de certificado y OID de política (sección 1.1), verificación de la organización (3.2.3), verificación del poder de representación (3.2.6), reutilización de resultados de verificación (4.2.1), periodos de validez (6.3.2), campos Subject (7.1.4.2) y esquemas de registro (anexo A).

2.  [cabforum/smime en GitHub: SBR.md](https://github.com/cabforum/smime/blob/main/SBR.md): versión Markdown de los requisitos básicos de S/MIME con historial de versiones, entre otros EUID desde la versión 1.0.9.

3.  [SwissSign: EUID, How Your Digital Certificates Become Even More Trustworthy](https://www.swisssign.com/en/blog/european-unique-identifier-euid): denominaciones de producto Silver y Gold, ejemplo `NTRCH-CHE-109.357.012`, UID como número de registro y EUID para empresas alemanas (28.04.2025).

4.  [SwissSign CPR S/MIME](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): perfiles de certificado de SwissSign para MV, SV y OV con campos Subject, usos previstos y OID de política, versión 14 del 18.09.2025.

5.  [DigiCert: Secure Email products](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates.html): visión general de productos con asignación a MV, OV y SV, fin de la generación `Legacy` y de los productos S/MIME anteriores el 10 de julio de 2025.

6.  [DigiCert: Order your Secure Email for Employee certificate](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates/order-your-secure-email-for-employee-certificate.html): verificación de organización y dominio, confirmación de los empleados por la organización, perfiles `Strict` y `Multipurpose`, periodos de validez.

7.  [Ley federal sobre la firma electrónica (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): art. 2 (conceptos, sello regulado), art. 6 (claves), art. 7 (contenido de certificados regulados, UID), art. 8 (certificados cualificados) y art. 9 (verificación de identidad).

8.  [Código de Obligaciones (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14, apdo. 2bis, sobre la equivalencia de la firma electrónica cualificada con la firma manuscrita.

9.  [Registro UID](https://www.uid.admin.ch): consulta de una UID con nombre, sede y estado, gestionado por la Oficina Federal de Estadística.

10.  [Zefix](https://www.zefix.ch): índice central de empresas de los registros mercantiles cantonales.

11.  [Código de Procedimiento Civil (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 157 sobre la libre valoración de la prueba.

12.  [Reglamento (UE) n.º 910/2014 (eIDAS)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32014R0910): art. 25, apdo. 2, sobre la equivalencia de la firma electrónica cualificada con la firma manuscrita.
