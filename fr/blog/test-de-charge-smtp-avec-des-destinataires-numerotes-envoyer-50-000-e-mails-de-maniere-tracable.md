---
title: "Test de charge SMTP avec des destinataires numérotés : envoyer chaque e-mail de manière traçable"
navTitle: "Tests de charge numérotés"
description: "Un test de charge ne vaut que par son évaluation. Avec l’option -N, smtp-source numérote chaque e-mail via l’adresse du destinataire sans sacrifier le débit. Comment structurer l’exécution, combien de sessions sont pertinentes et comment détecter automatiquement les numéros manquants."
date: "2026-08-27"
kategorie: "SMTP et flux de messagerie"
timeToRead: "8 min de lecture"
themen:
  - smtp-lasttests
  - smtp-mailflow
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "testing"
slug: "test-de-charge-smtp-avec-des-destinataires-numerotes-envoyer-50-000-e-mails-de-maniere-tracable"
translationId: "article-57f09c758baf6e1e"
translationOf: smtp-lasttest-nummerierte-empfaenger
translationSourceHash: 7145f2b49fb0b141d9c74d009d7c480ce4d119b4c97236e2ed7d92a39f65a1c5
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:46:35.239Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/test-de-charge-smtp-avec-des-destinataires-numerotes-envoyer-50-000-e-mails-de-maniere-tracable
---

# Test de charge SMTP avec des destinataires numérotés : envoyer chaque e-mail de manière traçable

Quiconque réalise un test de charge veut pouvoir répondre ensuite à deux questions : tous les e-mails sont-ils arrivés et, sinon, lesquels manquent ? Avec des e-mails de test identiques, il n’est possible que de compter, et un compteur indiquant 13 messages manquants ne dit ni quand ni où ils ont été perdus. En revanche, si chaque e-mail porte un numéro séquentiel, le comptage devient une comparaison : chaque numéro peut être retrouvé individuellement dans les journaux du système cible, les lacunes indiquent le moment de la perte et l’ordre de distribution peut être vérifié.

La réaction réflexe la plus répandue consiste à utiliser un script qui incrémente l’objet. Cela fonctionne, mais coûte en débit, car le générateur de charge `smtp-source` du paquet Postfix fixe l’objet à chaque invocation, et une boucle avec une invocation par e-mail impose l’établissement d’une connexion distincte pour chaque message. La meilleure identification de message est déjà intégrée : l’option `-N` numérote l’adresse du destinataire pour chaque message, au sein d’une seule invocation avec des sessions parallèles. Pour l’évaluation, l’adresse du destinataire est aussi exploitable que l’objet, car elle figure dans chaque journal de suivi.

Cette configuration de test envoie, contrairement à un simple test fonctionnel de bouclage local, vers un autre système via le réseau. Si aucun Postfix n’est installé sur le système source, l’article [smtp-source sans installation de Postfix](https://rafaelpfister.ch/blog/smtp-source-ohne-postfix-installation) explique comment extraire les outils du RPM.

## Les principales options de smtp-source

Pour vous orienter, voici les options utilisées dans cet article, traduites librement depuis la page de manuel :

<details class="options-details">
<summary>Aperçu des options</summary>

| Option | Signification |
|---|---|
| `-s n` | Nombre de sessions SMTP parallèles (par défaut : 1) |
| `-m n` | Nombre total de messages à envoyer (par défaut : 1) |
| `-l n` | Taille du corps du message en octets, hors en-têtes |
| `-f adresse` | Adresse de l’expéditeur |
| `-t adresse` | Adresse du destinataire (par défaut : `foo@hostname`) |
| `-S text` | Ligne d’objet, fixe pour tous les messages de l’invocation |
| `-F datei` | Envoie les en-têtes et le corps inchangés depuis un fichier ; remplace `-l` et `-S` |
| `-N` | Numérote l’adresse du destinataire pour chaque message (compteur par processus ; position et valeur de départ selon la version, voir ci-dessous) |
| `-r n` | Nombre de destinataires par message (par défaut : 1), génération d’adresses comme avec `-N` |
| `-d` | Ne pas se déconnecter après un message, envoyer le suivant via la même connexion |
| `-c` | Afficher le compteur en cours, qui augmente avec chaque `DATA` terminé |
| `-w n` | Temps d’attente fixe de n secondes entre les messages (par session) |
| `-v` | Sortie détaillée pour le dépannage |
| `host:port` | Cible de remise via TCP ; sans indication de port, le port SMTP standard est utilisé |

</details>

La liste complète, y compris les options TLS, LMTP et de temporisation, figure dans la page de manuel de `smtp-source(1)`; son équivalent pour le côté réception est `smtp-sink(1)` et sera utilisé plus bas lors de l’évaluation.

## Comment -N numérote les destinataires

`-N` active un compteur par processus intégré dans l’adresse du destinataire. Trois propriétés déterminent la configuration du test ; toutes trois peuvent être consultées dans le code source de `smtp-source.c` :

Premièrement, le format exact de l’adresse dépend de la version de Postfix. Postfix 3.5, tel que fourni par RHEL 8, place le numéro devant l’adresse complète (`RCPT TO:<%d%s>`) : à partir de `-t test@example.com`, on obtient `1test@example.com`, `2test@example.com` et ainsi de suite, le compteur commençant à 1. Les versions actuelles de Postfix ajoutent au contraire le numéro à la fin de la partie locale et commencent à 0 (`test0@` à `test49999@`) ; pour cette variante, la page de manuel recommande l’adressage avec plus (`-t 'test+@example.com'` devient `test+0@` et suivants), afin qu’un système cible prenant en charge le sous-adressage associe tout au même compte de messagerie. Vérifiez le format avant l’exécution importante avec une poignée d’e-mails contre un `smtp-sink` ou dans le journal de la cible ; la quantité attendue et le motif de recherche pour l’évaluation en dépendent.

Deuxièmement, le compteur est global au processus et partagé par toutes les sessions parallèles. Avec `-s 8`, les huit sessions attribuent les numéros conjointement, chaque numéro n’apparaissant qu’une seule fois. L’ordre entre les sessions n’est pas déterministe, mais l’exhaustivité de l’ensemble des numéros est garantie.

Troisièmement, la valeur de départ n’est pas configurable : 1 avec Postfix 3.5, 0 avec les versions actuelles. Les e-mails portent donc les numéros 1 jusqu’au nombre total défini par `-m`, ou 0 jusqu’au nombre total moins 1, et l’ensemble attendu pour la comparaison doit être adapté en conséquence.

## L’exécution du test en une seule invocation

Le nombre d’e-mails couvert par l’exécution ne change rien à la méthode ; `-m` détermine le nombre total, et les exemples de cet article utilisent 50'000 comme valeur de substitution arbitraire.

```bash
smtp-source -c -d -N -s 8 -m 50000 -l 5120 \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-c` | Compteur en cours des remises terminées sous forme d’indicateur de progression sur une ligne |
| `-d` | Les connexions restent ouvertes pour tous les messages ; sans `-d`, une nouvelle connexion par message |
| `-N` | Numérotation des destinataires : ajoute le compteur par processus à la partie locale |
| `-s 8` | Huit sessions SMTP parallèles |
| `-m 50000` | Nombre total de messages, répartis entre les sessions |
| `-l 5120` | Taille du message en octets (hors en-têtes), ici 5 Ko |
| `-f` | Adresse de l’expéditeur |
| `-t` | Adresse de base du destinataire ; `-N` la transforme en `1test@`, `2test@` et ainsi de suite (Postfix 3.5), ou `test0@`, `test1@` et ainsi de suite (versions actuelles) |
| `gateway.example.com:25` | Hôte cible et port |

</details>

`-d` est déterminant pour le profil de charge : sans cette option, `smtp-source` ferme la connexion après chaque message et en établit une nouvelle pour le suivant ; avec `-d`, les huit connexions restent ouvertes et remettent successivement tous les messages, comme le ferait un expéditeur en masse.

L’option `-v`, connue des tests fonctionnels, est volontairement absente : elle journalise chaque dialogue SMTP individuel, de `HELO` à `QUIT`, et produit des centaines de milliers de lignes de journal lors d’une grande exécution, sans valeur ajoutée pour l’évaluation. `-c` fournit à la place le récapitulatif permettant de suivre la progression en direct. La durée totale nécessaire au calcul du débit est fournie par un `time` placé devant.

Condition préalable à toute l’approche : le système cible doit accepter les adresses générées. Un `smtp-sink`, un domaine catch-all, un domaine de rejet du fournisseur ou une passerelle qui ne résout les destinataires qu’après acceptation répondent à cette condition. En revanche, si la cible vérifie chaque destinataire dans un annuaire, elle rejettera les adresses numérotées et seule la variante avec l’objet reste possible.

## Définir ses propres en-têtes

Certains tests de charge nécessitent un en-tête spécifique, par exemple comme marqueur permettant à la passerelle de reconnaître les e-mails de test ou de déclencher une règle. `smtp-source` ne propose pas d’option à cet effet, mais `-F` lit un message entièrement préformaté depuis un fichier, dans lequel chaque en-tête souhaité peut être défini. Le fichier se compose des lignes d’en-tête, d’une ligne vide et du corps ; toutes les lignes se terminent par `\r\n` :

```bash
{ printf 'X-Lasttest: aktiv\r\n'
  printf 'Subject: Lasttest\r\n'
  printf '\r\n'
  head -c 5120 /dev/zero | tr '\0' 'x'
  printf '\r\n'; } > lasttest.eml
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `head -c 5120` | Affiche les 5120 premiers octets de l’entrée, ici depuis `/dev/zero` |
| `tr '\0' 'x'` | Remplace chaque octet nul par le caractère `x` et génère ainsi le texte de remplissage de 5 Ko |
| `> lasttest.eml` | Écrit le message assemblé dans le fichier destiné à `-F` |

</details>

```bash
smtp-source -c -d -N -s 8 -m 50000 -F lasttest.eml \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-F datei` | Envoie les en-têtes et le corps inchangés depuis le fichier ; remplace le contenu généré du message |

</details>

Deux conséquences : `-F` remplace `-l` et `-S`, car la taille et l’objet proviennent désormais du fichier (ils doivent donc tous deux y figurer). En revanche, `-N` reste actif, les destinataires continuent d’être numérotés ; l’en-tête est identique dans tous les messages puisqu’il provient du fichier fixe.

## Combien de sessions ?

Le moyen le plus fiable de déterminer le nombre de sessions approprié est de mesurer, avec exactement les mêmes options que celles prévues pour l’exécution principale : même source de messages (le même fichier `-F` ou la même option `-l`), même expéditeur, même cible. Seule la quantité est réduite à 2'000 par niveau, et `-s` varie. Une courte exécution d’étalonnage avec un nombre croissant de sessions montre à partir de quand les sessions supplémentaires ne rapportent plus rien :

```bash
for s in 1 2 4 8 16 32; do
  t0=$(date +%s%N)
  smtp-source -d -N -s "$s" -m 2000 -F lasttest.eml \
    -f lasttest@example.com -t '@blackhole.example.com' \
    gateway.example.com:25
  t1=$(date +%s%N)
  echo "$s Sessions: $(( 2000000000000 / (t1 - t0) )) Mails/s"
done
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `date +%s%N` | Affiche les secondes Unix immédiatement suivies de la partie nanoseconde sous forme d’un seul nombre |
| `-d` | Les connexions restent ouvertes pour tous les messages du niveau |
| `-N` | Numérotation des destinataires via le compteur par processus |
| `-s "$s"` | Nombre de sessions parallèles, de 1 à 32 à chaque itération de la boucle |
| `-m 2000` | 2'000 messages par niveau de mesure |
| `-F lasttest.eml` | Même fichier de message que pour l’exécution principale prévue |
| `-f` | Adresse de l’expéditeur |
| `-t '@blackhole.example.com'` | Adresse de base du destinataire avec partie locale vide sur un domaine de rejet |
| `gateway.example.com:25` | Hôte cible et port |

</details>

Deux détails concernant l’invocation : `-c` est ici volontairement omis, afin qu’aucune sortie de compteur ne s’affiche entre les lignes de mesure ; la boucle produit exactement une ligne de résultat par niveau. Et la partie locale vide dans `-t` fonctionne bien avec la numérotation sur un domaine de rejet : avec le compteur préfixé de Postfix 3.5, cela produit des adresses de destinataires purement numériques (`1@blackhole.example.com`, `2@…`), ce qui rend l’évaluation dans les journaux plus claire.

Concrètement, voici ce qui se passe : la boucle externe parcourt les nombres de sessions de 1 à 32 par paliers de doublement. Avant et après chaque exécution, `date +%s%N` enregistre l’heure actuelle sous la forme d’un grand nombre, à savoir les secondes Unix immédiatement suivies de la partie nanoseconde. Entre les deux, `smtp-source` envoie 2'000 messages (contenu, en-têtes et taille proviennent du fichier `-F`) via le nombre correspondant de connexions parallèles qui restent ouvertes grâce à `-d` ; la boucle attend que l’invocation soit entièrement terminée. La ligne `echo` convertit la différence de temps en débit : 2'000 e-mails divisés par la durée d’exécution en secondes, alors que cette durée est exprimée en nanosecondes. 2'000 fois 10⁹ donne ainsi la constante `2000000000000`. L’arithmétique Bash `$(( ))` effectue un calcul entier et tronque les décimales, ce qui est suffisamment précis pour cette mesure.

Trois indications pratiques : `%N` ne fournit des nanosecondes qu’avec GNU date (c’est le cas sur RHEL et la plupart des systèmes Linux ; BusyBox et macOS ne le prennent pas en charge). L’exécution complète envoie 6 × 2'000 = 12'000 e-mails ; eux aussi nécessitent une adresse de destinataire contrôlée, et la numérotation `-N` recommence à la valeur initiale à chaque invocation. Enfin, si une invocation de `smtp-source` s’interrompt avec un message d’erreur, le débit de cette ligne est sans valeur ; corrigez d’abord la cause, puis mesurez à nouveau.

La sortie attendue comprend une ligne par niveau. Avec des valeurs d’exemple fictives, mais typiques, cela ressemble à ceci :

```text
1 Sessions: 11 Mails/s
2 Sessions: 21 Mails/s
4 Sessions: 40 Mails/s
8 Sessions: 71 Mails/s
16 Sessions: 79 Mails/s
32 Sessions: 80 Mails/s
```

Interprétation : tant que le débit double à peu près avec le nombre de sessions, les sessions parallèles masquent le temps d’attente des réponses de la cible ; le goulot d’étranglement est alors la latence du trajet, non la capacité. À partir du point où la courbe s’aplatit (dans l’exemple, entre 8 et 16 sessions), soit le système cible est saturé, soit la source atteint sa limite. Choisissez la plus petite valeur à partir de laquelle le débit n’augmente plus sensiblement, soit 8 à 16 dans l’exemple ; davantage de sessions n’augmentent alors que la charge induite par le parallélisme, pas le débit. Pour l’exécution principale, le débit mesuré permet également d’estimer la durée attendue : le nombre total défini par `-m` divisé par le débit.

## Évaluation côté réception

Si un récepteur de test dédié est disponible sur le système cible, `smtp-sink` assure également la journalisation :

```bash
smtp-sink -c -d "mails/%Y%m%d-%H%M%S." 0.0.0.0:2525 200
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-c` | Compteurs en cours au lieu du dialogue SMTP complet |
| `-d "mails/…"` | Avec le sink : dump, et non maintien de la connexion. Écrit chaque message accepté dans un fichier distinct (modèle de nom via strftime), y compris un en-tête `X-Rcpt-Args` contenant l’adresse du destinataire |
| `0.0.0.0:2525` | Écoute sur toutes les interfaces au port 2525 |
| `200` | Backlog : longueur maximale de la file d’attente des connexions en attente selon listen(2) |

</details>

Après l’exécution, extrayez les numéros reçus et comparez-les à l’ensemble attendu. Comme les numéros ne comportent pas de zéros initiaux, les deux listes sont complétées à un nombre fixe de chiffres avant comparaison, afin que le tri alphabétique de `comm` corresponde au tri numérique. Le motif de recherche correspond au format d’adresse de Postfix 3.5 (numéro avant l’adresse) ; pour les versions actuelles, utilisez respectivement `test[0-9]+@` et `seq` à partir de 0 :

```bash
grep -rhoE '[0-9]+test@example\.com' mails/ | \
  grep -oE '^[0-9]+' | sort -u | \
  awk '{printf "%08d\n", $1}' | sort > empfangen.txt
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `grep -r` | Recherche récursivement dans le répertoire `mails/` |
| `grep -h` | Supprime les noms de fichiers devant les correspondances |
| `grep -o` | N’affiche que la partie correspondante de l’adresse, et non toute la ligne |
| `grep -E` | Expressions régulières étendues, ici pour `[0-9]+` |
| `sort -u` | Trie et supprime les doublons (chaque numéro une seule fois) |
| `awk '{printf "%08d\n", $1}'` | Complète chaque numéro avec des zéros initiaux jusqu’à huit chiffres |
| `sort` | Trie les numéros complétés pour la comparaison avec `comm` |

</details>

```bash
seq 1 50000 | awk '{printf "%08d\n", $1}' | \
  comm -23 - empfangen.txt
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `seq 1 50000` | Génère l’ensemble attendu des numéros ; la valeur finale correspond au total envoyé défini par `-m` |
| `comm -23` | Supprime la colonne 2 (uniquement dans le fichier 2) et la colonne 3 (dans les deux) ; restent les lignes présentes uniquement dans l’ensemble attendu |
| `-` | Lit la première liste de comparaison depuis le pipe plutôt que depuis un fichier |
| `empfangen.txt` | Deuxième liste de comparaison : les numéros effectivement reçus |

</details>

`comm -23` affiche exactement les numéros présents dans l’ensemble attendu mais absents de la liste de réception : les e-mails manquants. Une sortie vide signifie que la remise est complète. Si des numéros apparaissent en double (visible par la différence entre `sort` et `sort -u`), un système a dupliqué le message en cours de route, ce qui constitue également un constat.

Si la cible est un système proche de la production plutôt qu’un smtp-sink, sa journalisation joue le rôle des fichiers de dump. Sur un serveur Exchange, par exemple, `Get-MessageTrackingLog -Recipients` ou un filtre sur l’adresse du destinataire fournit les numéros arrivés ; sur un système Postfix, un `grep` sur `to=` et l’adresse de base dans le journal de messagerie. C’est précisément l’avantage du numéro dans l’adresse : le destinataire figure dans chaque suivi de message, alors que l’objet peut manquer selon le système ou devoir être activé au préalable.

## Lorsque le numéro doit figurer dans l’objet

Certaines évaluations dépendent de l’objet, par exemple lorsque le système cible réécrit les adresses des destinataires ou que les journaux ne montrent le destinataire que masqué. Il reste alors la variante avec boucle : une invocation de `smtp-source` par e-mail avec `-m 1` et un objet incrémenté par le shell, répartie entre plusieurs workers parallèles avec des plages de numéros contiguës.

```bash
worker() {
  local i
  for ((i = $1; i <= $2; i++)); do
    smtp-source -s 1 -m 1 -l 5120 \
      -S "$(printf 'Lasttest %05d' "$i")" \
      -f lasttest@example.com -t test@example.com \
      gateway.example.com:25 || echo "$i" >> fehlend.log
  done
}
for w in 0 1 2 3; do
  worker $(( w * 12500 + 1 )) $(( (w + 1) * 12500 )) &
done
wait
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-s 1` | Une session par invocation ; les quatre workers assurent le parallélisme |
| `-m 1` | Exactement un message par invocation, afin de pouvoir définir l’objet pour chaque e-mail |
| `-l 5120` | Taille du message en octets (hors en-têtes), ici 5 Ko |
| `-S "$(printf 'Lasttest %05d' "$i")"` | Objet contenant le numéro séquentiel complété à cinq chiffres |
| `-f` / `-t` | Adresses de l’expéditeur et du destinataire |
| `gateway.example.com:25` | Hôte cible et port |

</details>

Le prix à payer est l’établissement complet d’une connexion par e-mail : négociation TCP, bannière, `HELO`, envoi, `QUIT`. Cette exécution ne mesure donc pas le débit maximal du système cible, mais un cas volontairement intensif en connexions. Déterminez le nombre de workers comme pour l’exécution d’étalonnage ci-dessus, mais avec la boucle des workers au lieu de `-s`. Les zéros initiaux dans l’objet évitent le reformatage nécessaire à la variante `-N` lors de la comparaison.

## Règles pour les tests contre d’autres systèmes

Dès que le test quitte votre propre système, trois conditions s’appliquent. Premièrement : l’exploitant du système cible est informé et a approuvé le créneau horaire ; pour toute supervision, un test de charge ressemble à une attaque ou à une vague de spam. Deuxièmement : l’adresse du destinataire se termine dans un environnement contrôlé, dans une boîte aux lettres de test dédiée, une règle de rejet sur la cible ou un domaine de rejet prévu à cet effet par le fournisseur ; les adresses de production n’ont pas leur place dans un test de charge. Troisièmement : un critère d’arrêt est défini avant le démarrage, par exemple une file d’attente qui augmente sur la cible ou un taux d’erreurs supérieur à un seuil, et quelqu’un surveille ces valeurs pendant l’exécution.

Avec ces trois points et la numérotation, l’exécution ne fournit pas seulement un chiffre de débit à la fin, mais une constatation vérifiable : quels e-mails sont arrivés, lesquels manquent et où ils ont été vus pour la dernière fois sur le trajet.

## Sources

1.  [Postfix : smtp-source(1)](https://www.postfix.org/smtp-source.1.html) : page de manuel du générateur de charge ; décrit le comportement de `-N` dans la version actuelle (compteur dans la partie locale, adressage avec plus).

2.  [Code source Postfix 3.5.8 : smtp-source.c](https://github.com/vdukhovni/postfix/blob/v3.5.8/postfix/src/smtpstone/smtp-source.c) : démontre, pour la version RHEL 8, le préfixage du numéro (`RCPT TO:<%d%s>`) avec une valeur de départ de 1 ; dans [la version actuelle](https://github.com/vdukhovni/postfix/blob/master/postfix/src/smtpstone/smtp-source.c), le numéro est au contraire ajouté à la partie locale, à partir de 0.

3.  [Postfix : smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html) : page de manuel du récepteur de test avec les options de dump et les en-têtes X enregistrés.

4.  [GNU Coreutils : comm](https://www.gnu.org/software/coreutils/manual/html_node/comm-invocation.html) : comparaison d’ensembles de deux listes triées, ici pour comparer les numéros attendus et reçus.
