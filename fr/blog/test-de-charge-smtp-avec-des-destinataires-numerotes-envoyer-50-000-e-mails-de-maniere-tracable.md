---
title: "Test de charge SMTP avec des destinataires numérotés : envoyer 50'000 e-mails de manière traçable"
navTitle: "Tests de charge numérotés"
description: "Un test de charge ne vaut que par son évaluation. Avec l’option -N, smtp-source numérote chaque e-mail via l’adresse du destinataire sans sacrifier le débit. Comment structurer le test avec 50'000 e-mails, combien de sessions sont pertinentes et comment détecter automatiquement les numéros manquants."
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
url: https://rafaelpfister.ch/fr/blog/test-de-charge-smtp-avec-des-destinataires-numerotes-envoyer-50-000-e-mails-de-maniere-tracable
translationSourceHash: a2ec75884c06a6d736ea9b5895211ddc4cbba252c7ddf491752e1bec5ab1a24d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:20:04.581Z
translationReview: automatic
---

# Test de charge SMTP avec des destinataires numérotés : envoyer 50'000 e-mails de manière traçable

Toute personne qui effectue un test de charge avec 50'000 e-mails veut pouvoir répondre ensuite à deux questions : sont-ils tous arrivés et, sinon, lesquels manquent ? Avec des e-mails de test identiques, il n’est possible que de compter, et un écart entre 49'987 et 50'000 ne dit ni quand ni où les 13 messages manquants ont été perdus. Si, en revanche, chaque e-mail porte un numéro séquentiel, le comptage devient une comparaison : chaque numéro peut être recherché individuellement dans les journaux du système cible, les lacunes révèlent le moment de la perte et l’ordre de livraison peut être vérifié.

La réaction instinctive courante consiste à utiliser un script qui incrémente l’objet. Cela fonctionne, mais réduit le débit, car le générateur de charge `smtp-source` du paquet Postfix fixe l’objet à chaque appel, et une boucle avec un appel par e-mail impose l’établissement d’une connexion distincte pour chaque message. Le meilleur identifiant de message est déjà intégré : l’option `-N` numérote l’adresse du destinataire pour chaque message, au sein d’un seul appel avec des sessions parallèles. Pour l’évaluation, l’adresse du destinataire est tout aussi exploitable que l’objet, car elle figure dans chaque journal de suivi.

Contrairement à un simple test fonctionnel en loopback, cette configuration de test envoie vers un autre système via le réseau. Si Postfix n’est pas installé sur le système source, l’article [smtp-source sans installation de Postfix](https://rafaelpfister.ch/blog/smtp-source-ohne-postfix-installation) explique comment extraire les outils du RPM.

## Les principales options de smtp-source

Pour vous orienter, voici les options utilisées dans cet article, traduites librement depuis la page de manuel :

| Option | Signification |
|---|---|
| `-s n` | Nombre de sessions SMTP parallèles (par défaut : 1) |
| `-m n` | Nombre total de messages à envoyer (par défaut : 1) |
| `-l n` | Taille du corps du message en octets, hors en-têtes |
| `-f adresse` | Adresse de l’expéditeur |
| `-t adresse` | Adresse du destinataire (par défaut : `foo@hostname`) |
| `-S text` | Ligne d’objet, fixe pour tous les messages de l’appel |
| `-F datei` | Envoie les en-têtes et le corps inchangés depuis un fichier ; remplace `-l` et `-S` |
| `-N` | Numérote l’adresse du destinataire pour chaque message (compteur par processus ; position et valeur de départ selon la version, voir ci-dessous) |
| `-r n` | Nombre de destinataires par message (par défaut : 1), formation d’adresse comme avec `-N` |
| `-d` | Ne pas se déconnecter après un message, envoyer le suivant via la même connexion |
| `-c` | Afficher un compteur en cours, incrémenté à chaque `DATA` terminé |
| `-w n` | Temps d’attente fixe de n secondes entre les messages (par session) |
| `-v` | Sortie détaillée pour le dépannage |
| `host:port` | Cible de l’injection via TCP ; sans port indiqué, port smtp par défaut |

La liste complète, y compris les options TLS, LMTP et de temporisation, figure dans la page de manuel de `smtp-source(1)` ; son équivalent côté réception est `smtp-sink(1)` et sera utilisé plus loin pour l’évaluation.

## Comment -N numérote les destinataires

`-N` active un compteur par processus intégré à l’adresse du destinataire. Trois caractéristiques déterminent la configuration du test ; toutes trois peuvent être vérifiées dans le code source de `smtp-source.c` :

Premièrement, le format précis de l’adresse dépend de la version de Postfix. Postfix 3.5, tel que fourni par RHEL 8, place le numéro devant l’adresse entière (`RCPT TO:<%d%s>`) : `-t test@example.com` devient `1test@example.com`, `2test@example.com` et ainsi de suite, le compteur commençant à 1. Les versions actuelles de Postfix ajoutent plutôt le numéro à la fin de la partie locale et commencent à 0 (`test0@` à `test49999@`) ; pour cette variante, la page de manuel recommande l’adressage plus (`-t 'test+@example.com'` produit `test+0@` et les suivants), afin qu’un système cible avec sous-adressage associe tout à la même boîte aux lettres. Vérifiez le format avant le grand test avec une poignée d’e-mails vers un `smtp-sink` ou dans le journal de la cible ; la quantité attendue et le motif de recherche pour l’évaluation en dépendent.

Deuxièmement, le compteur est global au processus et partagé par toutes les sessions parallèles. Avec `-s 8`, les huit sessions attribuent les numéros ensemble, chaque numéro n’apparaissant qu’une seule fois. L’ordre entre les sessions n’est pas déterministe, mais l’exhaustivité de l’ensemble des numéros est garantie.

Troisièmement, la valeur de départ n’est pas configurable : 1 avec Postfix 3.5, 0 avec les versions actuelles. Les 50'000 e-mails portent donc les numéros 1 à 50'000 ou 0 à 49'999, et l’ensemble attendu pour la comparaison doit y correspondre.

## Le test en un seul appel

```bash
smtp-source -c -d -N -s 8 -m 50000 -l 5120 \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

| Option | Effet |
|---|---|
| `-c` | Compteur en cours des livraisons terminées comme indicateur de progression sur une ligne |
| `-d` | Les connexions restent ouvertes pour tous les messages ; sans `-d`, nouvelle connexion par message |
| `-N` | Numérotation des destinataires : ajoute le compteur par processus à la partie locale |
| `-s 8` | Huit sessions SMTP parallèles |
| `-m 50000` | Nombre total de messages, répartis entre les sessions |
| `-l 5120` | Taille du message en octets (hors en-têtes), ici 5 Ko |
| `-f` | Adresse de l’expéditeur |
| `-t` | Adresse de base du destinataire ; `-N` la transforme en `1test@` à `50000test@` (Postfix 3.5) ou `test0@` à `test49999@` (versions actuelles) |
| `gateway.example.com:25` | Hôte cible et port |

`-d` est déterminant pour le profil de charge : sans cette option, `smtp-source` ferme la connexion après chaque message et en établit une nouvelle pour le suivant ; avec `-d`, les huit connexions restent ouvertes et livrent successivement tous les messages, comme le ferait un expéditeur en masse.

L’option `-v`, connue des tests fonctionnels, est volontairement absente : elle journalise chaque dialogue SMTP individuel de `HELO` à `QUIT` et génère des centaines de milliers de lignes de journal pour 50'000 e-mails, sans valeur ajoutée pour l’évaluation. `-c` fournit à la place le résumé qui permet de suivre la progression du test en direct. Un `time` placé devant la commande fournit la durée totale pour le calcul du débit.

Condition préalable à toute l’approche : le système cible accepte les adresses générées. Un `smtp-sink`, un domaine catch-all, un domaine de rejet du fournisseur ou une passerelle qui ne résout les destinataires qu’après l’acceptation remplissent cette condition. En revanche, si la cible vérifie chaque destinataire dans un annuaire, elle rejette les adresses numérotées et seule la variante avec objet reste possible.

## Définir ses propres en-têtes

Certains tests de charge nécessitent leur propre en-tête, par exemple comme marqueur permettant à la passerelle d’identifier les e-mails de test ou d’appliquer une règle. `smtp-source` ne propose pas d’option à cet effet, mais `-F` lit un message entièrement préformaté depuis un fichier, dans lequel chaque en-tête souhaité peut être défini. Le fichier se compose des lignes d’en-tête, d’une ligne vide et du corps, toutes les lignes se terminant par `\r\n` :

```bash
{ printf 'X-Lasttest: aktiv\r\n'
  printf 'Subject: Lasttest\r\n'
  printf '\r\n'
  head -c 5120 /dev/zero | tr '\0' 'x'
  printf '\r\n'; } > lasttest.eml
```

```bash
smtp-source -c -d -N -s 8 -m 50000 -F lasttest.eml \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

| Option | Effet |
|---|---|
| `-F datei` | Envoie les en-têtes et le corps inchangés depuis le fichier ; remplace le contenu de message généré |

Deux conséquences : `-F` remplace `-l` et `-S`, car la taille et l’objet proviennent désormais du fichier (les deux doivent donc y figurer). En revanche, `-N` reste actif et les destinataires continuent d’être numérotés ; l’en-tête est identique dans tous les messages puisqu’il provient du fichier fixe.

## Combien de sessions ?

La manière la plus fiable de déterminer le nombre de sessions approprié est de mesurer avec exactement les mêmes options que celles prévues pour le test principal : même source de message (le même fichier `-F` ou la même valeur `-l`), même expéditeur, même cible. Seule la quantité est réduite à 2'000 par niveau, et `-s` varie. Un court test d’étalonnage avec un nombre croissant de sessions montre à partir de quand des sessions supplémentaires n’apportent plus rien :

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

Deux détails concernant l’appel : `-c` est délibérément omis ici afin qu’aucune sortie de compteur en cours ne s’affiche entre les lignes de mesure ; la boucle produit exactement une ligne de résultat par niveau. Et la partie locale vide dans `-t` fonctionne bien avec la numérotation pour un domaine de rejet : avec le compteur placé devant par Postfix 3.5, cela crée des adresses de destinataires purement numériques (`1@blackhole.example.com`, `2@…`), ce qui rend l’évaluation dans les journaux plus claire.

Concrètement, voici ce qui se passe : la boucle externe parcourt les nombres de sessions de 1 à 32 par doublement. Avant et après chaque test, `date +%s%N` enregistre l’heure actuelle sous la forme d’un grand nombre : les secondes Unix immédiatement suivies de la partie en nanosecondes. Entre les deux, `smtp-source` envoie 2'000 messages (contenu, en-têtes et taille proviennent du fichier `-F`) via le nombre correspondant de connexions parallèles qui restent ouvertes grâce à `-d` ; la boucle attend la fin complète de l’appel. La ligne `echo` convertit la différence de temps en débit : 2'000 e-mails divisés par la durée d’exécution en secondes, la durée étant exprimée en nanosecondes. 2'000 fois 10⁹ donne ainsi la constante `2000000000000`. L’arithmétique Bash `$(( ))` travaille avec des nombres entiers et tronque les décimales, ce qui est suffisamment précis pour cette mesure.

Trois remarques pratiques : `%N` ne fournit les nanosecondes qu’avec GNU date (c’est le cas sur RHEL et la plupart des systèmes Linux ; BusyBox et macOS ne le prennent pas en charge). Le test complet envoie 6 × 2'000 = 12'000 e-mails ; eux aussi nécessitent une adresse de destinataire contrôlée, et la numérotation `-N` recommence à la valeur de départ lors de chaque appel. Enfin, si un appel à `smtp-source` s’interrompt avec un message d’erreur, le débit de cette ligne est dénué de sens ; corrigez d’abord la cause, puis mesurez à nouveau.

La sortie attendue comporte une ligne par niveau. Avec des valeurs d’exemple fictives, mais typiques, cela ressemble à ceci :

```text
1 Sessions: 11 Mails/s
2 Sessions: 21 Mails/s
4 Sessions: 40 Mails/s
8 Sessions: 71 Mails/s
16 Sessions: 79 Mails/s
32 Sessions: 80 Mails/s
```

Interprétation : tant que le débit double à peu près avec le nombre de sessions, les sessions parallèles masquent le temps d’attente des réponses de la cible ; le goulot d’étranglement est alors la latence du trajet, pas la capacité. À partir du point où la courbe s’aplatit (dans l’exemple, entre 8 et 16 sessions), le système cible est soit saturé, soit la source atteint sa limite. Prenez la plus petite valeur à laquelle le débit n’augmente plus sensiblement, soit 8 à 16 dans l’exemple ; davantage de sessions n’augmentent alors que la charge liée au parallélisme, pas le débit. Pour le test principal avec 50'000 e-mails, le débit mesuré permet également d’estimer la durée attendue : à 71 e-mails/s, environ 12 minutes.

## Évaluation côté réception

Si un destinataire de test dédié est disponible sur le système cible, `smtp-sink` assure aussi directement la journalisation :

```bash
smtp-sink -c -d "mails/%Y%m%d-%H%M%S." 0.0.0.0:2525 200
```

| Option | Effet |
|---|---|
| `-c` | Compteurs en cours au lieu du dialogue SMTP complet |
| `-d "mails/…"` | Pour le sink : dump, pas maintien de connexion. Écrit chaque message accepté dans un fichier distinct (modèle de nom via strftime), y compris un en-tête `X-Rcpt-Args` contenant l’adresse du destinataire |
| `0.0.0.0:2525` | Écoute sur toutes les interfaces au port 2525 |
| `200` | Backlog : longueur maximale de la file d’attente des connexions en attente selon listen(2) |

Après le test, extrayez les numéros reçus et comparez-les avec l’ensemble attendu. Comme les numéros ne comportent pas de zéros initiaux, les deux listes sont mises au même nombre fixe de chiffres avant la comparaison, afin que le tri alphabétique de `comm` corresponde au tri numérique. Le motif de recherche correspond au format d’adresse de Postfix 3.5 (numéro avant l’adresse) ; pour les versions actuelles, utilisez respectivement `test[0-9]+@` et `seq 0 49999` :

```bash
grep -rhoE '[0-9]+test@example\.com' mails/ | \
  grep -oE '^[0-9]+' | sort -u | \
  awk '{printf "%08d\n", $1}' | sort > empfangen.txt
```

```bash
seq 1 50000 | awk '{printf "%08d\n", $1}' | \
  comm -23 - empfangen.txt
```

`comm -23` affiche exactement les numéros présents dans l’ensemble attendu mais absents de la liste de réception : les e-mails manquants. Une sortie vide signifie une livraison complète. Si des numéros apparaissent en double (visible par la différence entre `sort` et `sort -u`), un système a dupliqué le message en chemin, ce qui constitue également un constat.

Si la cible est un système proche de la production plutôt qu’un smtp-sink, sa journalisation assume le rôle des fichiers de dump. Sur un serveur Exchange, par exemple, `Get-MessageTrackingLog -Recipients` ou un filtre sur l’adresse du destinataire fournit les numéros arrivés ; sur un système Postfix, un `grep` sur `to=` et l’adresse de base dans le journal de messagerie. C’est précisément l’avantage du numéro dans l’adresse : le destinataire figure dans chaque suivi de message, tandis que l’objet peut manquer selon le système ou devoir être activé au préalable.

## Lorsque le numéro doit figurer dans l’objet

Certaines évaluations reposent sur l’objet, par exemple lorsque le système cible réécrit les adresses des destinataires ou que les journaux n’affichent le destinataire que masqué. Il reste alors la variante en boucle : un appel à `smtp-source` par e-mail avec `-m 1` et un objet incrémenté par le shell, réparti sur plusieurs workers parallèles dotés de plages de numéros contiguës.

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

Le prix à payer est l’établissement complet d’une connexion par e-mail : handshake TCP, bannière, `HELO`, envoi, `QUIT`. Ce test ne mesure donc pas le débit maximal du système cible, mais un cas volontairement intensif en connexions. Déterminez le nombre de workers de manière analogue au test d’étalonnage ci-dessus, mais avec la boucle des workers à la place de `-s`. Les zéros initiaux dans l’objet évitent le reformatage nécessaire à la comparaison avec la variante `-N`.

## Règles pour les tests contre d’autres systèmes

Dès que le test quitte votre propre système, trois conditions s’appliquent. Premièrement : l’exploitant du système cible est informé et a accepté la fenêtre temporelle ; 50'000 e-mails ressemblent pour toute supervision à une attaque ou à une vague de spam. Deuxièmement : l’adresse du destinataire aboutit dans un environnement contrôlé, dans une boîte aux lettres de test dédiée, une règle de rejet sur la cible ou un domaine de rejet prévu à cet effet par le fournisseur ; les adresses de production n’ont pas leur place dans un test de charge. Troisièmement : un critère d’arrêt est défini avant le démarrage, par exemple une file d’attente qui augmente sur la cible ou un taux d’erreur dépassant un seuil, et une personne surveille ces valeurs pendant le test.

Avec ces trois points et la numérotation, le test ne fournit pas seulement un chiffre de débit à la fin, mais une conclusion vérifiable : lesquels des 50'000 e-mails sont arrivés, lesquels manquent et où ils ont été vus pour la dernière fois sur le trajet.

## Sources

1.  [Postfix : smtp-source(1)](https://www.postfix.org/smtp-source.1.html) : page de manuel du générateur de charge ; décrit le comportement de `-N` dans la version actuelle (compteur dans la partie locale, adressage plus).

2.  [Code source Postfix 3.5.8 : smtp-source.c](https://github.com/vdukhovni/postfix/blob/v3.5.8/postfix/src/smtpstone/smtp-source.c) : démontre pour la version RHEL 8 le préfixage du numéro (`RCPT TO:<%d%s>`) avec une valeur de départ de 1 ; dans [la version actuelle](https://github.com/vdukhovni/postfix/blob/master/postfix/src/smtpstone/smtp-source.c), le numéro est ajouté à la partie locale à partir de 0.

3.  [Postfix : smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html) : page de manuel du récepteur de test avec les options de dump et les en-têtes X enregistrés.

4.  [GNU Coreutils : comm](https://www.gnu.org/software/coreutils/manual/html_node/comm-invocation.html) : comparaison d’ensembles de deux listes triées, ici pour comparer les numéros attendus et reçus.
