---
title: "S/MIME avec mention de l’organisation : organizationIdentifier et UID dans le certificat"
navTitle: "S/MIME avec organisation"
description: "Ce que contient un certificat S/MIME avec mention de l’organisation, comment l’UID est intégrée au certificat sous forme d’organizationIdentifier et ce que l’autorité de certification vérifie pour l’entreprise et la personne. Avec une comparaison entre SwissSign et DigiCert, des commandes pour vérifier le type de certificat et la distinction avec ZertES."
date: "2026-12-15"
kategorie: "Chiffrement des e-mails"
timeToRead: "13 min de lecture"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "powershell"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2026-12-15"
slug: "s-mime-avec-mention-de-l-organisation-organizationidentifier-et-uid-dans-le-certificat"
translationId: "article-e0f9bcc57f360835"
translationOf: smime-organisation-uid
translationSourceHash: 849361658c86c3e7bae12ee2d7e018cf43807a611de4911b9ad1bdbe870ea0fe
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T14:55:07.202Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/s-mime-avec-mention-de-l-organisation-organizationidentifier-et-uid-dans-le-certificat
---

# S/MIME avec mention de l’organisation : organizationIdentifier et UID dans le certificat

Les autorités de certification (CA) distinguent les certificats S/MIME avec et sans mention de l’organisation. Un certificat avec mention de l’organisation indique, outre l’adresse e-mail, l’entreprise à laquelle appartient la boîte aux lettres : avec son nom vérifié et un numéro de registre. Pour les entreprises suisses, ce numéro de registre est l’UID, c’est-à-dire le numéro CHE du registre du commerce.

En bref : une mention de l’organisation n’existe que pour les types de certificats `Organization-validated` et `Sponsor-validated` des S/MIME Baseline Requirements. Elle se compose du nom de l’entreprise dans le champ `O` et du champ `organizationIdentifier` (OID 2.5.4.97), par exemple `NTRCH-CHE-123.456.789` pour une entreprise suisse. La CA vérifie l’entreprise dans un registre officiel avant d’inscrire ces informations dans le certificat. Un certificat validé par boîte aux lettres prouve en revanche uniquement l’accès à une boîte aux lettres. Chez SwissSign, cela correspond au choix entre Silver et Gold ; chez DigiCert, au choix entre « Secure Email for Individual Mailbox » et les produits « for Employee » et « for Organization ». Une signature qualifiée selon ZertES ne résulte pas de la mention de l’organisation.

**Focus Suisse :** la qualification juridique suit le droit suisse (ZertES, CO). Lorsque les règles de l’UE diffèrent, une remarque signalée l’indique.

## Les quatre types de certificats

Depuis le 1er septembre 2023, les `S/MIME Baseline Requirements` (BR) du CA/Browser Forum s’appliquent aux certificats S/MIME publiquement reconnus. Elles distinguent quatre types de certificats selon le titulaire (`Subject`) figurant dans le certificat :

| Type | Titulaire dans le certificat | Mention de l’organisation | Ce que la CA vérifie |
|---|---|---|---|
| `Mailbox-validated` (MV) | uniquement l’adresse e-mail | non autorisée | contrôle de la boîte aux lettres ou du domaine de messagerie |
| `Organization-validated` (OV) | une organisation, sans personne | obligatoire | organisation ainsi que boîte aux lettres ou domaine de messagerie |
| `Sponsor-validated` (SV) | une personne membre d’une organisation | obligatoire | comme OV, plus la personne ; la CA ou l’organisation elle-même vérifie cette dernière |
| `Individual-validated` (IV) | une personne sans organisation | non autorisée | identité de la personne et boîte aux lettres |

Par « S/MIME avec mention de l’organisation », on entend les types OV et SV. Pour les deux, les BR prescrivent obligatoirement les champs `organizationName` et `organizationIdentifier`, et ce dans les trois générations `Legacy`, `Multipurpose` et `Strict`. Les générations se distinguent par les champs Subject, extensions et usages autorisés. Les BR ont abandonné la génération `Legacy` au 15 juillet 2025 ; SwissSign et DigiCert ne l’émettent plus depuis lors.

Les certificats OV conviennent aux boîtes aux lettres fonctionnelles telles que `info@` ou `buchhaltung@`, qui ne sont attribuées à aucune personne. Les certificats SV sont destinés aux boîtes aux lettres personnelles des collaborateurs et contiennent en plus le nom de la personne ou un pseudonyme.

## Structure de l’organizationIdentifier

Le champ `organizationIdentifier` (OID 2.5.4.97) contient un numéro de registre de l’organisation. Les certificats selon eIDAS et ZertES utilisent également ce champ. Pour les certificats S/MIME, les BR prescrivent la structure suivante :

| Partie | Exemple | Signification |
|---|---|---|
| Schéma d’enregistrement, 3 caractères | `NTR` | `National Trade Register`, le registre du commerce |
| Code pays selon ISO 3166-1 | `CH` | pays dans lequel le registre est tenu |
| Tiret | `-` | séparateur |
| Numéro de registre | `CHE-123.456.789` | numéro de l’organisation dans le registre |

L’ensemble donne `NTRCH-CHE-123.456.789`. Comme l’UID contient elle-même un tiret, les BR stipulent que le schéma et le code pays ne contiennent jamais de tiret. Le premier tiret à partir de la gauche sépare donc le schéma du numéro de registre ; tous les suivants font partie du numéro. Si un registre attribue des numéros uniquement au niveau d’un État fédéré ou d’une province, un `+` suivi du code de la région s’ajoute après le code pays, par exemple `NTRUS+CA-12345678` pour la Californie.

Les BR connaissent ces schémas d’enregistrement :

| Schéma | Origine du numéro | Exemple des BR |
|---|---|---|
| `NTR` | registre du commerce ou des entreprises | `NTRGB-12345678` |
| `VAT` | administration fiscale (numéro de TVA) | `VATEL-123456789` |
| `PSD` | numéro d’agrément d’un prestataire de services de paiement selon la directive UE 2015/2366 | `PSDBE-NBB-1234.567.890` |
| `LEI` | `Legal Entity Identifier` selon ISO 17442, code pays toujours `XG` | `LEIXG-` suivi du LEI |
| `GOV` | autorités sans numéro relevant de l’un des autres schémas | `GOVUS` |
| `INT` | organisations internationales | `INTXG` |

Pour les entreprises suisses, `NTR` avec l’UID est le cas habituel. L’UID est attribué par l’Office fédéral de la statistique. Les registres du commerce l’utilisent comme numéro d’entreprise ; il sert également de numéro de TVA et reste identique lorsqu’une entreprise transfère son siège dans un autre canton. En revanche, un numéro de TVA peut disparaître lorsque l’assujettissement fiscal prend fin, et toute organisation n’en possède pas nécessairement. SwissSign reprend l’UID avec les points, tel qu’il figure dans le registre UID, et cite `NTRCH-CHE-109.357.012` comme exemple pour SwissSign AG.

**Remarque UE :** en Allemagne, les tribunaux d’instance attribuent les numéros du registre du commerce, et le même numéro peut apparaître auprès de plusieurs tribunaux. Depuis la version 1.0.9 de mai 2025, les BR autorisent donc, dans le schéma `NTR`, le `European Unique Identifier` (EUID) et le recommandent pour les entreprises allemandes, par exemple `NTRDE-DER3306.HRB12345` pour une entreprise inscrite au registre du commerce du tribunal d’instance de Cologne. Selon les BR, le numéro de registre ne doit être unique que là où le registre et l’ordre juridique attribuent des numéros uniques.

## Ce que la CA vérifie

Avant d’émettre un certificat OV ou SV, une CA doit vérifier les informations suivantes selon les BR :

- **Nom et existence de l’organisation :** par une autorité du pays de l’organisation, des données LEI, une visite sur place ou une confirmation accompagnée d’un extrait de registre. Pour une entreprise suisse, il s’agit du registre du commerce et du registre UID. Si l’entreprise y est enregistrée comme radiée ou inactive, la CA ne peut pas émettre le certificat.
- **Numéro de registre :** la CA confirme que le numéro dans `organizationIdentifier` appartient à l’organisation figurant dans le champ `O`.
- **Pouvoir :** par un canal de communication fiable, la CA vérifie que la personne requérante est autorisée à demander des certificats pour l’organisation ou à agir en tant qu’`Enterprise RA`.
- **Boîte aux lettres ou domaine de messagerie :** la CA vérifie le contrôle des adresses e-mail figurant dans le certificat. Si l’organisation agit en tant qu’`Enterprise RA`, elle doit prouver le contrôle de l’ensemble du domaine de messagerie.
- **Personne (SV uniquement) :** nom ou pseudonyme de la personne. La section suivante décrit les justificatifs autorisés à cette fin.

Une CA peut réutiliser les résultats de ces vérifications : la vérification de l’organisation et du pouvoir pendant 825 jours, celle du contrôle du domaine pendant 398 jours. Un certificat des générations `Multipurpose` et `Strict` est valable au maximum 825 jours.

## Comment la personne est vérifiée pour SV

Les BR autorisent les justificatifs suivants pour l’identité de la personne :

| Justificatif | Procédure |
|---|---|
| pièce d’identité officielle | original sur place ou en temps réel devant une caméra ; la personne chargée de la vérification compare la photo du document avec la personne et consigne l’émetteur, la validité et le numéro du document |
| puce du passeport ou de la carte d’identité | vérification de la signature de l’émetteur sur les données de la puce, complétée par la comparaison de la photo |
| eID | uniquement les systèmes eID notifiés selon eIDAS, avec le niveau de garantie « substantiel » ou « élevé » |
| permis de conduire mobile (mDL) | interrogation en temps réel selon ISO/IEC 18013-5 et 18013-7 avec vérification de la chaîne de signatures de l’émetteur |
| signature numérique | signature de la demande avec un certificat personnel existant issu d’un cadre reconnu ; la liste de ces cadres est encore vide dans les BR |
| confirmation par un notariat ou un cabinet d’avocats | confirmation écrite provenant du pays de la personne requérante |
| documents de l’`Enterprise RA` | l’entreprise vérifie elle-même ses collaborateurs et peut s’appuyer sur des données internes existantes, par exemple des ressources humaines |

La CA doit publier dans son CPS les justificatifs qu’elle accepte. Au moment de l’émission, la vérification de la personne ne doit pas dater de plus de 825 jours. Si l’entreprise vérifie elle-même en tant qu’`Enterprise RA`, la CA doit l’y obliger contractuellement et la contrôler au moins une fois par an, sauf si l’entreprise est elle-même auditée annuellement. Dans ce cas, le rattachement à la personne est aussi fiable que le processus interne de l’entreprise.

## Comparaison entre SwissSign et DigiCert

Les deux CA proposent les types MV, SV et OV sous leurs propres noms de produits. Aucune des deux ne propose de certificat IV. La structure de `organizationIdentifier` est définie par les BR et s’applique de la même manière aux deux fournisseurs.

| | SwissSign | DigiCert |
|---|---|---|
| MV | Personal S/MIME E-Mail ID Silver | Secure Email for Individual Mailbox |
| SV | Pro S/MIME E-Mail ID Gold | Secure Email for Employee (anciennement Secure Email for Business) |
| OV | Organization Validated E-Mail Certificate | Secure Email for Organization |
| IV | non proposé | non proposé |
| Obtention | Silver dans la boutique en ligne ou via la Managed PKI, Gold et OV uniquement via la Managed PKI de l’entreprise | les trois via le compte CertCentral de l’entreprise |
| Vérification de la personne pour SV | pièce d’identité officielle en original, sur place ou selon une procédure équivalente ; autres justificatifs uniquement avec l’autorisation de SwissSign | confirmation par l’entreprise ; DigiCert ne vérifie pas la personne |
| Validité des vérifications | informations et justificatifs au maximum 398 jours | organisation 825 jours, domaine 398 jours |
| Génération pour SV et OV | `Multipurpose`; une CA intermédiaire pour SV dans la génération `Strict` est déjà décrite dans le CP/CPS | `Strict` par défaut, `Multipurpose` au choix |
| Personne pour SV | prénom et nom ou pseudonyme avec le préfixe `pseudo:` dans le `CN` | prénom et nom ou pseudonyme ; dans le `CN`, au choix le nom ou l’adresse e-mail |
| Usages supplémentaires pour SV | aucun dans le profil standard ; dans le profil avec authentification : authentification client, connexion par carte à puce et EFS | pour `Strict`, non-répudiation ; pour `Multipurpose`, en plus chiffrement des données et authentification client |

DigiCert décrit explicitement le produit SV selon le modèle de l’`Enterprise RA` : l’organisation confirme que la personne figurant dans le certificat fait partie de ses collaborateurs ou représentants et conserve les justificatifs correspondants. DigiCert ne vérifie elle-même que l’organisation. Toute personne autorisée à commander dans le compte CertCentral de l’entreprise peut ainsi inscrire n’importe quel nom sur une adresse de ses propres domaines. Pour OV et SV, DigiCert exige toujours la preuve du contrôle du domaine. La vérification de l’organisation est valable 825 jours, celle du domaine 398 jours. Une durée d’un ou deux ans, ou une date d’expiration personnalisée jusqu’à 824 jours au maximum, est disponible. DigiCert a abandonné les anciens produits Premium, Email Security Plus, Digital Signature Plus et Class 1 S/MIME le 10 juillet 2025 et les a remplacés par ces produits.

SwissSign n’émet les certificats Gold et OV que via la Managed PKI de l’entreprise. La personne doit néanmoins s’identifier au moyen d’une pièce d’identité officielle originale, en personne ou selon une procédure équivalente conforme à ETSI EN 319 411-1. L’identification est effectuée par le service d’enregistrement qui traite la demande, ou par un partenaire d’identification formé sous contrat : copie, numérisation ou photographie du document, comparaison de la photo avec la personne et confirmation de la procédure auprès de SwissSign. L’utilisation du nom de l’entreprise doit être approuvée par la direction inscrite au registre du commerce ou le conseil d’administration, et les personnes qui demandent des certificats dans la Managed PKI sont également identifiées au moyen d’une pièce d’identité. Une entreprise ne peut appliquer une autre procédure, par exemple fondée sur des données du personnel, que si SwissSign l’a examinée et approuvée comme équivalente. Chez SwissSign, les informations et justificatifs sont valables au maximum 398 jours, après quoi SwissSign effectue une nouvelle vérification.

Chez SwissSign, le certificat OV contient dans le `CN` le nom de l’organisation tel qu’il figure dans le champ `O`. L’article [S/MIME dans les banques : cadre juridique en Suisse et choix entre Silver et Gold](/blog/smime-silver-gold-schweiz) décrit le niveau qu’une banque devrait choisir pour quelle boîte aux lettres et ses implications juridiques.

## Vérifier le type de certificat

Dans Outlook et d’autres clients de messagerie, les détails du certificat d’un e-mail signé peuvent être ouverts via l’icône de signature. Sous Windows, les champs se trouvent dans l’onglet « Détails », à l’entrée « Sujet ». Pour l’OID 2.5.4.97, Windows ne connaît pas de nom d’affichage et l’indique comme `OID.2.5.4.97`.

### Afficher le titulaire avec PowerShell

Pour un certificat exporté (par exemple depuis les détails du certificat d’un e-mail signé) :

```powershell
$zert = Get-PfxCertificate -FilePath .\zertifikat.cer
$zert.SubjectName.Format($true)
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Get-PfxCertificate` | lit un certificat depuis un fichier, y compris un fichier `.cer` sans clé privée |
| `-FilePath .\zertifikat.cer` | chemin vers le fichier de certificat |
| `SubjectName.Format($true)` | affiche les champs du titulaire ligne par ligne |

</details>

Sortie pour un certificat SV (exemple) :

```text
CN=Max Muster
G=Max
SN=Muster
O=Muster AG
OID.2.5.4.97=NTRCH-CHE-123.456.789
S=ZH
C=CH
```

Pour un certificat MV, la sortie ne contient que `CN=` avec l’adresse e-mail.

### Déterminer le type et la génération via l’OID de politique

Chaque certificat conforme aux BR contient dans l’extension `Certificate Policies` une OID suivant le modèle `2.23.140.1.5.<Typ>.<Generation>` :

| Position | Valeur | Signification |
|---|---|---|
| Type | `1` | `Mailbox-validated` |
| Type | `2` | `Organization-validated` |
| Type | `3` | `Sponsor-validated` |
| Type | `4` | `Individual-validated` |
| Génération | `1` | `Legacy` |
| Génération | `2` | `Multipurpose` |
| Génération | `3` | `Strict` |

```powershell
$zert.Extensions |
  Where-Object { $_.Oid.Value -eq '2.5.29.32' } |
  ForEach-Object { $_.Format($true) }
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `$zert.Extensions` | extensions du certificat lu précédemment |
| `'2.5.29.32'` | OID de l’extension `Certificate Policies` |
| `Format($true)` | affiche le contenu de l’extension sur plusieurs lignes |

</details>

Sortie (abrégée) :

```text
[1]Certificate Policy:
     Policy Identifier=2.23.140.1.5.3.2
```

`2.23.140.1.5.3.2` correspond à un certificat SV de la génération `Multipurpose`. D’autres entrées avec des OID de la CA elle-même sont normales.

### Répertorier les certificats S/MIME dans son propre magasin de certificats

```powershell
Get-ChildItem Cert:\CurrentUser\My |
  Where-Object { $_.EnhancedKeyUsageList.ObjectId -contains '1.3.6.1.5.5.7.3.4' } |
  Select-Object Subject, NotAfter
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Cert:\CurrentUser\My` | magasin de certificats personnel de l’utilisateur connecté |
| `EnhancedKeyUsageList.ObjectId` | usages de clé étendus du certificat |
| `'1.3.6.1.5.5.7.3.4'` | OID pour la protection des e-mails (`id-kp-emailProtection`) |
| `Select-Object Subject, NotAfter` | affiche le titulaire et la date d’expiration |

</details>

### Avec OpenSSL

```bash
openssl x509 -in zertifikat.pem -noout -subject -nameopt multiline
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `x509` | sous-commande pour les certificats |
| `-in zertifikat.pem` | fichier d’entrée au format PEM ; pour un fichier DER, ajouter `-inform der` |
| `-noout` | supprime l’affichage du certificat encodé |
| `-subject` | affiche le titulaire |
| `-nameopt multiline` | un champ par ligne, avec les noms de champs développés |

</details>

OpenSSL 3 connaît le nom du champ et l’affiche en toutes lettres :

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

### Vérifier le numéro de registre

En tant que destinataire d’un e-mail signé, vous pouvez vérifier la mention de l’organisation en quatre étapes :

1. Le client de messagerie indique que la signature est valide et le certificat provient d’une CA à laquelle il fait confiance.
2. L’OID de politique commence par `2.23.140.1.5.2` ou `2.23.140.1.5.3`, et le certificat contient les champs `O` et `organizationIdentifier`.
3. L’UID figurant dans `organizationIdentifier` appartient, dans le registre UID, à l’entreprise attendue et cette entreprise y est active.
4. Le domaine de l’adresse de l’expéditeur appartient à cette entreprise.

## Distinction avec ZertES

La mention de l’organisation ne transforme pas un certificat S/MIME en certificat réglementé selon ZertES. Seuls les prestataires reconnus selon ZertES émettent des certificats réglementés, et le certificat doit contenir l’indication qu’il s’agit d’un certificat réglementé (art. 2, let. g et art. 7, al. 2, let. b ZertES).

Sur le fond, il existe des parallèles. Un certificat réglementé pour une unité UID doit contenir son UID (art. 7, al. 2, let. e ZertES), tout comme `organizationIdentifier` le contient pour OV et SV. Un certificat pour une personne physique peut en outre contenir l’indication qu’elle est autorisée à représenter une unité UID déterminée (art. 7, al. 3, let. b ZertES) ; cela correspond, dans sa structure, à un certificat SV.

| | S/MIME avec mention de l’organisation | certificat réglementé selon ZertES |
|---|---|---|
| Base | S/MIME Baseline Requirements du CA/Browser Forum | ZertES et dispositions d’exécution |
| Émettrice | CA publiquement reconnue | prestataire reconnu selon ZertES |
| Identifiant de l’organisation | `organizationIdentifier`, pour les entreprises suisses avec l’UID | UID, obligatoire pour les unités UID |
| Vérification de la personne | pour SV, selon la CA, avec pièce d’identité ou par confirmation de l’organisation en tant qu’`Enterprise RA` | en principe comparution personnelle avec preuve d’identité ; pour les unités UID, une représentation avec preuve du pouvoir de représentation (art. 9 ZertES) |
| Clés | selon les exigences de la CA, également sur une passerelle | exigences selon l’art. 6 ZertES |

Seule la signature électronique qualifiée avec horodatage qualifié est assimilée à la signature manuscrite selon l’art. 14, al. 2bis CO, et un certificat qualifié ne peut être émis qu’à une personne physique (art. 8, al. 1 ZertES). Pour la signature d’une organisation, ZertES prévoit le sceau électronique réglementé : une signature électronique avancée créée au moyen d’un dispositif de création de sceau sécurisé et reposant sur un certificat réglementé d’une unité UID (art. 2, let. d ZertES). Un e-mail signé S/MIME est un moyen de preuve dont le tribunal apprécie librement la valeur (art. 157 CPC) ; une mention d’organisation vérifiée soutient alors l’attribution à l’entreprise expéditrice.

**Remarque UE :** dans l’UE, le règlement eIDAS régit les signatures et sceaux électroniques. Selon l’art. 25, al. 2 eIDAS, seule la signature électronique qualifiée est également assimilée à la signature manuscrite.

## Recommandation

- **Boîtes aux lettres fonctionnelles :** certificat OV (SwissSign Organization Validated, DigiCert Secure Email for Organization). Il identifie l’entreprise sans nommer de personne.
- **Boîtes aux lettres personnelles avec signature vis-à-vis des clients et partenaires :** certificat SV (SwissSign Gold, DigiCert Secure Email for Employee). Il identifie la personne et l’entreprise.
- **Chiffrement uniquement :** sur le plan cryptographique, tous les types sont équivalents. Pour le seul chiffrement, un certificat MV suffit.
- **Planifier les vérifications :** la vérification de l’organisation est valable 825 jours, celle du domaine 398 jours. Après une modification de la raison sociale ou du siège, les informations du certificat ne correspondent plus au registre ; prévoyez alors de nouveaux certificats.
- **Vérifier à la réception :** l’OID de politique et l’UID indiquent si une signature est attribuée à une entreprise vérifiée. La qualité de la vérification de la personne dépend, pour SV, de la procédure de la CA : SwissSign procède à une vérification de pièce d’identité, DigiCert s’appuie sur la confirmation de l’entreprise.

## Sources

1.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): version 1.0.17 du 20 novembre 2026 ; types de certificats et OID de politique (section 1.1), `Enterprise RA` (1.3.2), vérification de l’organisation (3.2.3), vérification de la personne (3.2.4), vérification du pouvoir (3.2.6), réutilisation des résultats de vérification (4.2.1), durées de validité (6.3.2), champs Subject (7.1.4.2) et schémas d’enregistrement (annexe A).

2.  [cabforum/smime sur GitHub: SBR.md](https://github.com/cabforum/smime/blob/main/SBR.md): version Markdown des S/MIME Baseline Requirements avec historique des versions, notamment l’EUID à partir de la version 1.0.9.

3.  [SwissSign: EUID, How Your Digital Certificates Become Even More Trustworthy](https://www.swisssign.com/en/blog/european-unique-identifier-euid): noms des produits Silver et Gold, exemple `NTRCH-CHE-109.357.012`, UID comme numéro de registre et EUID pour les entreprises allemandes (28.04.2025).

4.  [SwissSign CP/CPS S/MIME](https://repository.swisssign.com/SwissSign_CPS_SMIME.pdf): règlement combiné de SwissSign, version 1.0 du 31.08.2026, remplaçant notamment la CPR S/MIME ; identification de la personne (3.2.3), pouvoir (3.2.5), validité des justificatifs (3.3.1), modalités d’obtention (4.1.1) et profils de certificats pour MV, SV et OV (7.1).

5.  [DigiCert: Secure Email products](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates.html): aperçu des produits avec attribution à MV, OV et SV, fin de la génération `Legacy` et des anciens produits S/MIME le 10 juillet 2025.

6.  [DigiCert: Order your Secure Email for Employee certificate](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates/order-your-secure-email-for-employee-certificate.html): vérification de l’organisation et du domaine, confirmation des collaborateurs par l’organisation sans vérification propre de la personne par DigiCert, profils `Strict` et `Multipurpose`, durées de validité.

7.  [Loi fédérale sur la signature électronique (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): art. 2 (notions, sceau réglementé), art. 6 (clés), art. 7 (contenu des certificats réglementés, UID), art. 8 (certificats qualifiés) et art. 9 (vérification de l’identité).

8.  [Code des obligations (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14, al. 2bis sur l’assimilation de la signature électronique qualifiée à la signature manuscrite.

9.  [Registre UID](https://www.uid.admin.ch): consultation d’un UID avec nom, siège et statut, tenu par l’Office fédéral de la statistique.

10.  [Zefix](https://www.zefix.ch): index central des entreprises des registres cantonaux du commerce.

11.  [Code de procédure civile (CPC)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 157 relatif à la libre appréciation des preuves.

12.  [Règlement (UE) n° 910/2014 (eIDAS)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32014R0910): art. 25, al. 2 sur l’assimilation de la signature électronique qualifiée à la signature manuscrite.
