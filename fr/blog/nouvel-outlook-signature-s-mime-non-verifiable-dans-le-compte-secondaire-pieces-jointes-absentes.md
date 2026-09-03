---
title: "Nouvel Outlook : signature S/MIME non vérifiable dans le compte secondaire, pièces jointes absentes"
navTitle: "S/MIME dans le compte secondaire"
description: "Le nouvel Outlook indique, pour une boîte aux lettres partagée, que la signature S/MIME ne peut pas être vérifiée dans le compte secondaire et n’affiche aucune pièce jointe. Cet article explique la différence entre Clear Signing et Opaque Signing, pourquoi les pièces jointes disparaissent dans les e-mails signés de manière opaque, pourquoi le nouvel Outlook ne traite S/MIME que dans le compte principal et quelles solutions existent, y compris l’extraction du fichier smime.p7m avec PowerShell ou OpenSSL."
date: "2026-09-03"
kategorie: "Outlook"
timeToRead: "8 min de lecture"
themen:
  - outlook
  - e-mail-verschluesselung
produkte:
  - "exchange-online"
  - "outlook"
protokolle:
  - "verschluesselung"
  - "troubleshooting"
related:
  - e-mail-header-analysieren-ohne-upload
slug: "nouvel-outlook-signature-s-mime-non-verifiable-dans-le-compte-secondaire-pieces-jointes-absentes"
translationId: "article-f1e9d4ab5be67349"
aiPrompt: |
  Du bist mein Messaging-Assistent. Hilf mir, das Problem "S/MIME-Signatur kann im sekundären Konto nicht überprüft werden" in Outlook einzuordnen: Prüfe anhand der Nachrichtenquelle, ob die Mail clear-signed (multipart/signed) oder opaque-signed (application/pkcs7-mime) ist, erkläre mir, warum die Anhänge fehlen, und führe mich zu einem Ausweg (Postfach als eigenes Konto, klassisches Outlook, Outlook im Web oder Auspacken der smime.p7m mit PowerShell oder OpenSSL).
translationOf: outlook-smime-sekundaeres-konto-anhaenge-fehlen
url: https://rafaelpfister.ch/fr/blog/nouvel-outlook-signature-s-mime-non-verifiable-dans-le-compte-secondaire-pieces-jointes-absentes
translationSourceHash: ee167a56424fa3ffe1d8e79c62a748cd68c7864d7a95d3d9fdc8989a33cd4283
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:10:10.434Z
translationReview: automatic
---

# Nouvel Outlook : signature S/MIME non vérifiable dans le compte secondaire, pièces jointes absentes

Dans le nouvel Outlook pour Windows, l’ouverture d’un e-mail signé numériquement dans une boîte aux lettres partagée affiche une barre rouge : « Le signe S/MIME ne peut pas être vérifié lors de l’affichage dans le compte secondaire. » L’e-mail lui-même s’affiche, mais pas les pièces jointes, bien que l’expéditeur en ait envoyées. Les collègues qui utilisent la même boîte aux lettres comme compte principal voient les pièces jointes sans problème.

Deux éléments se renforcent mutuellement : le nouvel Outlook ne traite S/MIME que dans le compte principal, et l’expéditeur a signé l’e-mail de manière opaque. Avec cette forme de signature, l’intégralité du contenu, y compris les pièces jointes, se trouve dans un unique conteneur cryptographique. Si le client ne peut pas ouvrir ce conteneur, les pièces jointes restent invisibles. Ces deux problèmes peuvent être résolus séparément.

## Ce que signifie le message

« Compte secondaire » désigne dans le nouvel Outlook toute boîte aux lettres autre que le compte utilisé pour vous connecter. Cela concerne les boîtes aux lettres partagées (Shared Mailboxes) affichées via l’accès complet et l’automapping, mais aussi les boîtes aux lettres ajoutées via « Ajouter une boîte aux lettres partagée » ainsi que les délégations. Dans le nouvel Outlook, le traitement S/MIME est strictement lié au compte principal. Si un message signé arrive dans un autre compte, le client ne vérifie pas la signature et affiche ce message à la place.

Cela ne dit rien sur la validité de la signature et ne signale pas un problème de certificat chez l’expéditeur. Le même e-mail peut être vérifié et ouvert dans le compte principal ou dans Outlook classique.

## Clear Signing et Opaque Signing

La norme S/MIME (RFC 8551) prévoit deux formats pour les messages signés. Tous deux fournissent la même signature, mais emballent le message différemment.

| | Clear Signing | Opaque Signing |
|---|---|---|
| Type MIME | `multipart/signed` avec `protocol="application/pkcs7-signature"` | `application/pkcs7-mime` avec `smime-type=signed-data` |
| Structure | Deux parties : le texte lisible du message avec les pièces jointes et, à côté, la signature détachée | Une partie : texte du message, pièces jointes et signature réunis dans un conteneur CMS-SignedData, encodé en Base64 |
| Pièce jointe visible par un client sans S/MIME | `smime.p7s` (uniquement la signature, quelques Ko) | `smime.p7m` (l’intégralité du message) |
| Lisible sans prise en charge de S/MIME | Oui, le texte et les pièces jointes s’affichent normalement | Non, le client ne voit que le conteneur |
| Sensibilité durant le transport | La signature devient invalide si un serveur de messagerie ou une passerelle modifie les retours à la ligne, l’encodage ou les espaces | Le conteneur protège le contenu contre ces modifications |
| Section RFC 8551 | 3.5.3 | 3.5.2 |

Dans la source du message, vous reconnaissez les deux formats à l’en-tête `Content-Type`. Un e-mail signé en clair commence ainsi :

```text
Content-Type: multipart/signed; protocol="application/pkcs7-signature";
    micalg=sha-256; boundary="----=_Part_4711"
```

Un e-mail signé de manière opaque ainsi :

```text
Content-Type: application/pkcs7-mime; smime-type=signed-data;
    name="smime.p7m"
Content-Transfer-Encoding: base64
Content-Disposition: attachment; filename="smime.p7m"
```

Cette différence explique entièrement le comportement du nouvel Outlook. Pour un e-mail signé en clair, le client affiche le texte et les pièces jointes même s’il ne vérifie pas la signature ; seul l’état de la signature manque. Pour un e-mail signé de manière opaque, le client doit d’abord extraire le conteneur via le traitement S/MIME pour accéder au texte et aux pièces jointes. S’il refuse de le faire parce que le message se trouve dans un compte secondaire, le conteneur reste fermé. Si le texte reste tout de même lisible, c’est grâce à Exchange Online : le service restitue la partie texte côté serveur, mais pas les pièces jointes du conteneur.

Les deux formats ne chiffrent rien. Même la variante opaque est seulement encodée en Base64 et peut être lue par toute personne ayant accès au message. Microsoft le précise explicitement dans la documentation Exchange Online.

## Quel format choisit l’expéditeur

Dans Outlook classique, l’option « Envoyer les messages signés en texte brut » (Fichier > Options > Centre de gestion de la confidentialité > Sécurité de la messagerie) contrôle le format. Elle est activée par défaut : Outlook signe donc en clair. Si cette option est désactivée, Outlook envoie une signature opaque. Le nouvel Outlook et Outlook sur le web ne proposent pas ce réglage.

Les passerelles de messagerie qui signent de manière centralisée disposent de leur propre réglage de format de signature. Certains produits signent de manière opaque par défaut pour des raisons de robustesse, car la signature reste alors valide même après une modification par des systèmes en aval. Si vous recevez régulièrement des e-mails d’un expéditeur précis avec des pièces jointes manquantes, il vaut la peine de vérifier la configuration de sa passerelle.

## Pourquoi le nouvel Outlook ne traite S/MIME que dans le compte principal

Microsoft documente cette limitation, sans toutefois en indiquer la raison technique. Celle-ci découle de l’architecture du client.

Le nouvel Outlook est essentiellement le client web d’Outlook sur le web dans une enveloppe native. Dans le navigateur, JavaScript ne peut pas accéder au magasin de certificats Windows. C’est pourquoi Outlook sur le web a eu besoin pendant des années d’une extension de navigateur S/MIME distincte. Le nouvel Outlook remplace cette extension par un pont intégré entre l’interface web et la cryptographie Windows. Ce pont est initialisé lors de la connexion au compte principal et reçoit sa session de boîte aux lettres, ses certificats et ses paramètres S/MIME depuis Paramètres > E-mail > S/MIME.

Les boîtes aux lettres partagées et les comptes secondaires empruntent d’autres chemins de données. Les comptes secondaires disposent de leurs propres sessions, tandis que les boîtes aux lettres partagées sont chargées via la délégation du compte principal. Jusqu’à présent, le pont n’était pas connecté à ces chemins. Cela vaut également pour la simple vérification d’une signature, bien qu’aucune clé privée ne soit nécessaire : l’analyse et l’extraction de la structure PKCS#7 passent par le même composant.

Dans Outlook classique, ce problème ne se produit pas, car le traitement S/MIME s’effectue dans la pile MAPI pour chaque message, indépendamment du Store dont provient le message.

Microsoft ajoute cette connexion manquante : l’entrée de feuille de route 565861 étend S/MIME dans le nouvel Outlook aux boîtes aux lettres partagées et déléguées rattachées au compte principal. La disponibilité générale est annoncée pour juillet 2026, avec un déploiement progressif. Si le message apparaît toujours, la modification n’est pas encore arrivée dans votre tenant ou votre version du client. L’entrée ne couvre pas les comptes secondaires ajoutés séparément avec leur propre connexion.

## Solutions possibles

La solution appropriée dépend de la manière dont la boîte aux lettres est intégrée et de votre besoin de vérifier la signature ou simplement d’accéder aux pièces jointes.

| Solution | Condition | Résultat |
|---|---|---|
| Ouvrir l’e-mail dans le compte principal | Vous utilisez vous-même la boîte aux lettres comme compte principal ou l’e-mail vous a été transféré | Vérification de la signature et pièces jointes |
| Ajouter la boîte aux lettres comme compte autonome dans le nouvel Outlook (Paramètres > Comptes > Ajouter un compte) | La boîte aux lettres possède ses propres identifiants ; impossible pour les boîtes aux lettres partagées sans mot de passe | Vérification de la signature et pièces jointes dès que vous basculez vers ce compte |
| Outlook classique | Encore installé ou accessible en revenant via le bouton « Nouvel Outlook » ; intégrer la boîte aux lettres comme compte distinct (Fichier > Paramètres du compte > Nouveau) | Vérification de la signature et pièces jointes dans chaque Store |
| Outlook sur le web | Ouvrir directement la boîte aux lettres (`outlook.office.com/mail/<adresse>`), extension S/MIME pour Edge ou Chrome installée | Vérification de la signature et pièces jointes |
| Demander à l’expéditeur d’utiliser Clear Signing | L’expéditeur utilise Outlook classique ou une passerelle dont le format est sélectionnable | Pièces jointes visibles, mais état de la signature toujours indisponible dans le compte secondaire |
| Extraire manuellement le conteneur | Enregistrer l’e-mail au format `.eml` ou sauvegarder `smime.p7m` | Pièces jointes sans vérification de signature |

## Extraire manuellement le conteneur

Pour un cas ponctuel, il suffit d’ouvrir le conteneur en dehors d’Outlook. La signature est alors vérifiée mathématiquement, mais pas la chaîne de confiance du certificat. Enregistrez le message au format `.eml` ou sauvegardez la pièce jointe `smime.p7m` dans un dossier.

Sous Windows, PowerShell suffit. Le .NET Framework fournit la classe `SignedCms`, qui lit le conteneur PKCS#7 :

```powershell
Add-Type -AssemblyName System.Security
$bytes = [IO.File]::ReadAllBytes("C:\Temp\smime.p7m")
$cms = New-Object System.Security.Cryptography.Pkcs.SignedCms
$cms.Decode($bytes)
$cms.CheckSignature($true)
[IO.File]::WriteAllBytes("C:\Temp\inhalt.eml", $cms.ContentInfo.Content)
```

<details class="options-details">
<summary>Explication des options</summary>

| Instruction | Effet |
|---|---|
| `Add-Type -AssemblyName System.Security` | Charge l’assembly contenant les classes PKCS (nécessaire dans Windows PowerShell 5.1, déjà chargée dans PowerShell 7) |
| `[IO.File]::ReadAllBytes(...)` | Lit le conteneur DER binaire ; le fichier `smime.p7m` enregistré depuis Outlook est déjà décodé |
| `$cms.Decode($bytes)` | Analyse la structure CMS-SignedData |
| `$cms.CheckSignature($true)` | Vérifie uniquement la signature sur le contenu (`$true` = verifySignatureOnly) ; avec `$false`, la chaîne de certificats serait également vérifiée. En cas de signature invalide, la commande s’interrompt avec une exception |
| `$cms.ContentInfo.Content` | Le contenu signé : un message MIME complet avec texte et pièces jointes |
| `[IO.File]::WriteAllBytes(...)` | Écrit ce message MIME au format `.eml`, que vous pouvez ouvrir dans Outlook ou Thunderbird |

</details>

Sous Linux, macOS ou avec Git pour Windows, OpenSSL est disponible. Si l’e-mail entier est disponible au format `.eml`, OpenSSL effectue aussi le décodage Base64 :

```bash
openssl cms -verify -noverify \
  -in nachricht.eml \
  -out inhalt.eml
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `cms` | Outil CMS d’OpenSSL ; `smime` fonctionne de manière équivalente, `cms` est l’interface plus récente |
| `-verify` | Vérifie la signature et produit le contenu signé |
| `-noverify` | Ignore la vérification de la chaîne de certificats ; la signature elle-même est tout de même vérifiée |
| `-in nachricht.eml` | L’e-mail complet au format S/MIME (Base64 avec en-têtes MIME) ; pour un fichier `smime.p7m` sauvegardé, ajouter `-inform DER` |
| `-out inhalt.eml` | Le contenu extrait sous forme de message MIME |

</details>

Le fichier `inhalt.eml` contient le texte original du message et toutes les pièces jointes sous forme de parties MIME normales. Un double-clic l’ouvre dans Outlook, où vous pouvez enregistrer les pièces jointes comme d’habitude.

## Sources

1.  [s/mime sign cannot be verified when viewing in secondary account (Microsoft Q&A)](https://learn.microsoft.com/en-us/answers/questions/5781907/s-mime-sign-cannot-be-verified-when-viewing-in-sec): Le cas pratique avec le même message dans la boîte aux lettres partagée ; la réponse confirme que ce comportement est connu et n’indique aucune solution dans le nouvel Outlook.

2.  [RFC 8551: Secure/Multipurpose Internet Mail Extensions (S/MIME) Version 4.0 Message Specification](https://www.rfc-editor.org/rfc/rfc8551.html): Sections 3.5.2 (application/pkcs7-mime avec SignedData) et 3.5.3 (multipart/signed), avec les indications sur la lisibilité sans S/MIME et la robustesse durant le transport.

3.  [Secure messages with a digital ID in Outlook (Microsoft Support)](https://support.microsoft.com/en-us/office/secure-messages-with-a-digital-id-in-outlook-549ca2f1-a68f-4366-85fa-b3f4b5856fc6): L’option « Envoyer les messages signés en texte brut » dans Outlook classique, activée par défaut ; absente du nouvel Outlook.

4.  [Set up Outlook to use S/MIME encryption (Microsoft Support)](https://support.microsoft.com/en-us/outlook/mail/set-up-outlook-to-use-s-mime-encryption): Paramètres S/MIME du nouvel Outlook sous Paramètres > E-mail > S/MIME ; les certificats doivent être installés manuellement ou via une stratégie.

5.  [S/MIME in Exchange Online (Microsoft Learn)](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo): Indication selon laquelle les messages signés de manière opaque sont uniquement encodés en Base64 et ne sont pas confidentiels.

6.  [Microsoft 365 Roadmap, entrée 565861](https://www.microsoft.com/microsoft-365/roadmap?id=565861): S/MIME pour les boîtes aux lettres partagées et déléguées dans le nouvel Outlook pour Windows, annoncé pour juillet 2026.

7.  [Accounts in the new Outlook for Windows (Microsoft Learn)](https://learn.microsoft.com/en-us/deployoffice/outlook/get-started/supported-account-types): Types de comptes pris en charge par le nouvel Outlook et mode d’intégration des boîtes aux lettres partagées.

8.  [SignedCms Class (.NET API Reference)](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.pkcs.signedcms): Decode, CheckSignature et ContentInfo pour extraire le conteneur avec PowerShell.

9.  [openssl-cms (OpenSSL Manpage)](https://www.openssl.org/docs/man3.0/man1/openssl-cms.html): Options `-verify`, `-noverify`, `-inform` et `-out`.
