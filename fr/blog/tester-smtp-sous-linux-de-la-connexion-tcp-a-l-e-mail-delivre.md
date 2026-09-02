---
title: "Tester SMTP sous Linux : de la connexion TCP à l’e-mail délivré"
navTitle: "Tester SMTP"
description: "Lorsqu’une appliance ne délivre plus d’e-mails, un test SMTP manuel est plus utile que n’importe quel journal. Comment vérifier couche par couche avec les outils intégrés, ce que signifient les différents symptômes d’erreur et pourquoi un équilibreur de charge fausse le diagnostic."
date: "2026-07-31"
kategorie: "SMTP et flux de messagerie"
timeToRead: "10 min de lecture"
themen:
  - smtp-mailflow
  - testing
  - e-mail-verschluesselung
slug: "tester-smtp-sous-linux-de-la-connexion-tcp-a-l-e-mail-delivre"
translationId: "article-cb44a92c03a47bc0"
translationOf: smtp-verbindung-testen-linux
translationSourceHash: af2a802f67ec6d294b1507eaf26e25704b938e8760ac6751104ce7258cc2a4b3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:15:12.039Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/tester-smtp-sous-linux-de-la-connexion-tcp-a-l-e-mail-delivre
---

# Tester SMTP sous Linux : de la connexion TCP à l’e-mail délivré

Lorsqu’une passerelle de messagerie cesse soudainement de délivrer des messages, les journaux de l’appliance n’affichent souvent que le résultat final : une livraison échoue, la file d’attente grossit, un message d’erreur indique un délai d’expiration. Seul un test manuel depuis la ligne de commande permet d’en déterminer la véritable cause. SMTP est un protocole en texte clair qui peut entièrement être utilisé à la main, ce qui en fait un outil de diagnostic disponible partout sans installation supplémentaire.

Deuxième raison de recourir à un test manuel : il est généralement impossible d’installer quoi que ce soit sur les appliances. Pas de gestionnaire de paquets, pas de droits root, pas de `swaks`. Toutes les étapes suivantes fonctionnent donc avec ce qui est déjà présent sur pratiquement tous les systèmes Linux.

## Distinguer les couches

Un échec d’envoi d’e-mail peut survenir à cinq niveaux différents, chacun produisant un symptôme distinct :

1. **Résolution de noms :** L’hôte cible ne peut pas être traduit en adresse IP.
2. **Connexion TCP :** La connexion au port ne s’établit pas ou est réinitialisée.
3. **Dialogue SMTP :** La connexion est établie, mais le serveur refuse l’expéditeur, le destinataire ou le contenu.
4. **Chiffrement du transport :** STARTTLS est absent, le certificat est invalide ou la version TLS ne convient pas.
5. **Vérification de l’expéditeur :** L’e-mail est accepté puis rejeté chez le destinataire en raison de SPF, DKIM ou DMARC.

Le diagnostic gagne énormément en efficacité lorsque vous vérifiez ces niveaux l’un après l’autre et séparément, au lieu d’envoyer immédiatement un e-mail de test complet. Un essai global échoué vous dit seulement que quelque chose ne fonctionne pas. La vérification par couches vous dit quoi.

## Étape 1 : résolution de noms

```bash
getent hosts relay.example.com
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `hosts` | Base de données NSS à interroger ; utilise les mêmes sources et le même ordre que le système lui-même, conformément à `nsswitch.conf` |
| `relay.example.com` | Nom d’hôte à résoudre |

</details>

Si la sortie reste vide, aucun serveur de noms n’est accessible depuis cet hôte ou il ne répond pas aux noms externes. Cela arrive régulièrement en pratique : les appliances situées dans des zones isolées ne reçoivent souvent qu’un résolveur interne qui ne connaît que ses propres zones.

```bash
cat /etc/resolv.conf; grep hosts: /etc/nsswitch.conf
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `/etc/resolv.conf` | Fichier contenant les serveurs de noms configurés, affiché par `cat` |
| `hosts:` | Modèle de recherche pour `grep` : la ligne définissant l’ordre des sources de résolution (fichiers, DNS) |
| `/etc/nsswitch.conf` | Fichier de configuration NSS recherché par `grep` |

</details>

Si la résolution est absente, testez directement avec l’adresse IP dans les étapes suivantes. C’est parfaitement suffisant pour le diagnostic et cela sépare clairement le problème DNS du problème de transport. En production, l’absence de résolution reste naturellement un constat distinct à corriger.

## Étape 2 : accessibilité du port

Pour un simple test TCP, bash suffit. Le pseudo-périphérique `/dev/tcp` ouvre une connexion sans nécessiter l’installation de `nc` ou de `telnet` :

```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/192.0.2.25/25' ; echo "exit=$?"
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `timeout 10` | Interrompt la commande suivante après 10 secondes et renvoie alors le code de sortie 124 |
| `bash -c '…'` | Exécute la chaîne de commandes dans bash ; nécessaire car `/dev/tcp` est une fonctionnalité de bash |
| `exec 3<>/dev/tcp/192.0.2.25/25` | Ouvre le descripteur de fichier 3 en lecture et en écriture comme connexion TCP vers 192.0.2.25, port 25 |
| `echo "exit=$?"` | Affiche le code de sortie de la commande précédente |

</details>

Le code de sortie est ici l’information essentielle :

| sortie | Signification |
|---|---|
| `0` | La connexion est établie, le port est ouvert |
| `124` | Délai d’expiration : les paquets sont rejetés, typiquement par une règle de pare-feu DROP |
| `1` | Refus immédiat (RST) ou absence de route |

En pratique, la différence entre 124 et 1 est l’indice le plus important. Un délai d’expiration signifie que quelqu’un rejette silencieusement les paquets sur le trajet, ce qui est presque toujours une règle de pare-feu. Un RST immédiat provient en revanche d’un système qui répond mais n’offre pas le service.

Vérifiez immédiatement les deux ports concernés, ainsi qu’une autre destination quelconque afin de déterminer si l’hôte est autorisé à établir des connexions sortantes :

```bash
for t in "192.0.2.25 25" "192.0.2.25 587" "1.1.1.1 443"; do
  set -- $t
  timeout 8 bash -c "exec 3<>/dev/tcp/$1/$2" 2>/dev/null
  echo "$1:$2 -> exit=$?"
done
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `set -- $t` | Décompose la paire de valeurs à l’espace en paramètres positionnels `$1` (adresse IP) et `$2` (port) |
| `timeout 8` | Interrompt la tentative de connexion après 8 secondes (code de sortie 124) |
| `bash -c "…"` | Exécute l’établissement de la connexion `/dev/tcp` dans bash |
| `2>/dev/null` | Supprime les messages d’erreur afin qu’une seule ligne de résultat apparaisse par destination |

</details>

Si le test de contrôle échoue également, le système ne dispose généralement d’aucune sortie directe et le trafic doit passer par un relais interne ou un proxy. Nous verrons plus bas pourquoi ce cas est particulièrement délicat.

Si `/dev/tcp` est absent, le shell n’est pas bash. Sous `sh`, `ash` ou `ksh`, cette fonctionnalité n’existe pas, ce qui est souvent interprété à tort comme un problème réseau :

```bash
ps -p $$ -o comm= ; echo "BASH_VERSION=${BASH_VERSION:-keine bash}"
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-p $$` | Limite la sortie au processus ayant le PID du shell actuel (`$$`) |
| `-o comm=` | Affiche uniquement le nom de la commande ; l’étiquette vide après `=` supprime l’en-tête |
| `${BASH_VERSION:-keine bash}` | Affiche la version de bash ou le texte de remplacement si la variable n’est pas définie |

</details>

## Étape 3 : écouter d’abord, ne rien envoyer

Un serveur SMTP envoie spontanément une bannière `220`. Le test isolé le plus révélateur consiste donc à ouvrir une connexion et à ne rien faire :

```bash
{ exec 3<>/dev/tcp/192.0.2.25/25 && timeout 15 cat <&3; echo "[ende exit=$?]"; }
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `exec 3<>/dev/tcp/192.0.2.25/25` | Ouvre le descripteur de fichier 3 comme connexion TCP vers la cible |
| `timeout 15 cat <&3` | Lit pendant 15 secondes tout ce que le serveur envoie spontanément et l’affiche |
| `echo "[ende exit=$?]"` | Affiche le code de sortie à la fin ; 124 signifie que plus rien n’est arrivé pendant 15 secondes |

</details>

Ces quelques caractères distinguent deux situations entièrement différentes. Si une réponse `220 mail.example.com ESMTP` arrive, l’hôte distant parle et toutes les erreurs ultérieures se situent dans le dialogue. Si rien n’arrive, ce n’est pas dû à une commande mal formulée de votre part, puisque vous n’en avez envoyé aucune.

Le descripteur de fichier reste ensuite ouvert dans le shell. Fermez-le avant de lancer le test suivant, sans quoi vous risquez de continuer à travailler avec une ancienne connexion qui n’est plus intacte :

```bash
exec 3<&- 3>&-
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `3<&-` | Ferme le côté lecture du descripteur de fichier 3 |
| `3>&-` | Ferme le côté écriture du descripteur de fichier 3 |

</details>

## Étape 4 : le dialogue SMTP à la main

Une fois la bannière affichée, effectuez le dialogue complet. Il est important qu’un processus de lecture s’exécute en parallèle afin de voir chaque réponse au moment où elle arrive. Un script qui envoie d’abord tout puis lit ensuite ne vous montrera rien si l’échange s’interrompt en plein milieu du dialogue :

```bash
{
exec 3<>/dev/tcp/192.0.2.25/25
cat <&3 & R=$!
sleep 1; printf 'EHLO host.example.com\r\n' >&3
sleep 2; printf 'MAIL FROM:<absender@example.com>\r\n' >&3
sleep 2; printf 'RCPT TO:<empfaenger@example.net>\r\n' >&3
sleep 2; printf 'DATA\r\n' >&3
sleep 2; printf 'From: absender@example.com\r\nTo: empfaenger@example.net\r\nSubject: Relay-Test\r\n' >&3
printf 'Date: %s\r\nMessage-ID: <%s@example.com>\r\n\r\nTestnachricht\r\n.\r\n' "$(date -R)" "$(date +%s).$" >&3
sleep 3; printf 'QUIT\r\n' >&3
sleep 2; kill $R 2>/dev/null
}
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `exec 3<>/dev/tcp/192.0.2.25/25` | Ouvre le descripteur de fichier 3 comme connexion TCP vers la cible |
| `cat <&3 & R=$!` | Lance un lecteur en arrière-plan pour le descripteur de fichier 3 et mémorise son PID dans `R` |
| `printf '…\r\n' >&3` | Envoie une commande SMTP avec la fin de ligne CRLF requise sur la connexion |
| `sleep n` | Attend le nombre de secondes indiqué la réponse du serveur avant d’envoyer la commande suivante |
| `date -R` | Fournit la date au format conforme à la RFC pour l’en-tête `Date:` |
| `date +%s` | Fournit l’heure Unix comme base simple et unique pour l’identifiant de message |
| `kill $R 2>/dev/null` | Arrête le lecteur en arrière-plan ; aucun message d’erreur ne s’affiche s’il est déjà arrêté |

</details>

Deux détails déterminent la réussite ou l’échec. SMTP exige CRLF comme fin de ligne, d’où `printf` avec `\r\n` et non `echo`. Et le point sur une ligne à lui seul termine la partie message ; il doit être envoyé sous la forme `\r\n.\r\n`.

Déroulement attendu : `220` à l’établissement de la connexion, `250` après EHLO, `250 2.1.0` après MAIL FROM, `250 2.1.5` après RCPT TO, `354` après DATA et enfin `250 2.0.0 Ok: queued as <id>`. Notez l’identifiant de file d’attente. Il permet au fournisseur exploitant le service de suivre le message s’il n’arrive jamais chez le destinataire.

Le nom EHLO mérite votre attention : certains relais le vérifient par rapport aux DNS direct et inverse et répondent sinon par `501` ou `504`. Utilisez le FQDN réel du système émetteur, et non son nom court.

## Étape 5 : STARTTLS et certificat

Pour la connexion chiffrée, `openssl s_client` effectue lui-même la négociation STARTTLS puis transmet le canal à l’entrée standard :

```bash
openssl s_client -connect 192.0.2.25:25 -starttls smtp -tls1_2 -brief </dev/null
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-connect 192.0.2.25:25` | Hôte cible et port de la connexion |
| `-starttls smtp` | Effectue d’abord le dialogue SMTP en texte clair, puis bascule vers TLS via STARTTLS |
| `-tls1_2` | Négocie exclusivement TLS 1.2 |
| `-brief` | Réduit la sortie à un bref résumé de la connexion négociée |
| `</dev/null` | Ferme immédiatement l’entrée standard afin que `s_client` n’attende pas en mode interactif après la négociation |

</details>

Si vous vous connectez par adresse IP parce que le DNS est indisponible, la vérification du nom d’hôte ne peut pas fonctionner. Le nom du certificat ne correspond alors pas à l’adresse numérique. SNI et le nom de vérification peuvent être définis explicitement, sans aucune requête DNS :

```bash
openssl s_client -connect 192.0.2.25:25 \
  -servername mail.example.com -verify_hostname mail.example.com \
  -starttls smtp -tls1_2 -brief </dev/null
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-servername mail.example.com` | Définit le nom SNI dans le ClientHello, indépendamment de l’adresse de connexion |
| `-verify_hostname mail.example.com` | Vérifie le certificat serveur par rapport à ce nom plutôt qu’à l’adresse numérique |

</details>

Deux symptômes d’erreur apparaissent régulièrement ici et sont souvent mal interprétés.

**« Didn't find STARTTLS in server response, trying anyway »** signifie que le serveur n’a pas proposé STARTTLS dans sa réponse EHLO. `openssl` envoie tout de même un ClientHello TLS, le serveur y voit des données de protocole invalides et la connexion se termine par `wrong version number` ou `write:errno=32` (EPIPE). Ces deux messages sont des erreurs consécutives. L’information essentielle est : pas de STARTTLS. Vérifiez avec le dialogue en texte clair de l’étape 4 quelles capacités le serveur annonce réellement.

**L’absence de STARTTLS sur un saut interne** est souvent tout à fait normale. Si un équilibreur de charge transmet la connexion à la couche 4, ce n’est pas lui qui négocie TLS, mais le système situé derrière lui avec la destination réelle. Tester en texte clair sur le segment interne n’est alors pas un défaut de sécurité, mais simplement l’architecture.

## Étape 6 : Python comme alternative

Si Python est disponible, vous évitez la gestion manuelle du temps avec `sleep`. La bibliothèque standard suffit, rien ne doit être installé :

```python
#!/usr/bin/env python3
import smtplib, ssl
from email.message import EmailMessage
from email.utils import formatdate, make_msgid

msg = EmailMessage()
msg["From"] = "absender@example.com"
msg["To"] = "empfaenger@example.net"
msg["Subject"] = "Relay-Test"
msg["Date"] = formatdate(localtime=True)
msg["Message-ID"] = make_msgid(domain="example.com")
msg.set_content("Testnachricht\n")

ctx = ssl.create_default_context()
ctx.minimum_version = ssl.TLSVersion.TLSv1_2

s = smtplib.SMTP("192.0.2.25", 25, timeout=30, local_hostname="host.example.com")
s.set_debuglevel(1)
s.ehlo()
if s.has_extn("starttls"):
    s.starttls(context=ctx, server_hostname="mail.example.com")
    s.ehlo()
    print("TLS:", s.sock.version(), s.sock.cipher()[0])
s.send_message(msg)
s.quit()
```

`set_debuglevel(1)` enregistre le dialogue complet, y compris tous les codes de réponse, et `smtplib` lit chaque réponse de manière synchrone. Une interruption apparaît sous la forme `SMTPServerDisconnected`, accompagnée de la dernière ligne reçue, plutôt que comme un Broken Pipe silencieux.

Deux points échouent souvent ici : `server_hostname` est indispensable lorsque vous vous connectez via une adresse IP, sinon Python vérifie le certificat par rapport à l’adresse numérique. Et si vous désactivez délibérément la vérification, `check_hostname = False` doit précéder `verify_mode = ssl.CERT_NONE`, sinon Python lève une `ValueError`.

## Adresse d’expéditeur, SPF et alignement

Un test échoue étonnamment souvent non pas au niveau du transport, mais à cause de l’adresse d’expéditeur choisie. Trois points doivent être vérifiés au préalable.

Le domaine de l’expéditeur doit être un FQDN. Une adresse telle que `test@meine-testdomain` sans domaine de premier niveau est déjà refusée par de nombreux MTA lors de MAIL FROM, avec `501` ou `553`.

Le domaine doit autoriser le chemin d’envoi utilisé. Un regard sur l’enregistrement SPF permet de voir si l’adresse sortante est couverte :

```bash
dig +short TXT example.com | grep spf1
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `+short` | Affiche uniquement les valeurs des enregistrements, sans en-têtes ni métadonnées |
| `TXT` | Type d’enregistrement interrogé |
| `example.com` | Nom interrogé |
| `grep spf1` | Extrait la ligne SPF parmi plusieurs enregistrements TXT |

</details>

Et lorsque DMARC est actif, c’est l’alignement qui décide. Si l’enregistrement contient `aspf=s`, le domaine dans l’enveloppe (MAIL FROM) et le domaine dans l’en-tête `From:` doivent correspondre exactement, et pas seulement être apparentés :

```bash
dig +short TXT _dmarc.example.com
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `+short` | Affiche uniquement les valeurs des enregistrements, sans en-têtes ni métadonnées |
| `TXT _dmarc.example.com` | Type d’enregistrement et nom défini pour DMARC sous le domaine |

</details>

Avec `p=reject`, un e-mail de test dont l’alignement ne convient pas disparaît silencieusement chez le destinataire, bien que votre relais l’ait accepté avec `250 queued`. C’est la cause la plus fréquente des messages considérés comme envoyés avec succès côté expéditeur mais qui n’arrivent jamais.

## Lorsqu’un équilibreur de charge s’intercale

Dans les environnements plus importants, une appliance envoie rarement directement sur Internet. Il est courant d’utiliser un serveur virtuel sur un équilibreur de charge, qui accepte la connexion, réécrit l’adresse vers une adresse définie via source NAT, puis la transmet vers l’extérieur. Cela a une conséquence fâcheuse pour le diagnostic.

Un serveur virtuel fonctionnant à la couche 4 confirme immédiatement la négociation TCP, avant même d’avoir lui-même établi une connexion vers la cible. Si cette seconde connexion échoue, le client voit une connexion établie avec succès puis immédiatement réinitialisée : `Connection reset by peer`, sans aucune bannière SMTP. L’erreur ne se situe alors ni chez vous ni chez la cible, mais dans le pool derrière le serveur virtuel, par exemple parce qu’un membre est marqué comme indisponible ou que le FQDN configuré ne peut pas être résolu.

Cela explique aussi pourquoi un test direct vers la cible Internet doit échouer lorsque la règle de transfert n’accepte que le trafic provenant de l’adresse SNAT déjà réécrite. Les connexions avec l’adresse source d’origine ne correspondent à aucune règle et sont rejetées. Dans de tels environnements, testez toujours le serveur virtuel prévu, et non la cible réelle.

Une seule ligne indique quelle adresse source votre système utilise pour une destination donnée. La valeur après `src` est précisément celle dont l’équipe réseau a besoin pour l’autorisation :

```bash
ip route get 192.0.2.25
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `route get` | Demande au noyau quelle route il choisirait pour une cible précise |
| `192.0.2.25` | Adresse cible de la connexion simulée |

</details>

Si le système se trouve derrière NAT, l’hôte distant ne voit pas cette adresse, mais l’adresse publique du périmètre. Il est impossible de la déterminer depuis l’intérieur tant qu’aucun trafic ne passe ; elle figure dans la règle NAT.

## Symptômes d’erreur en un coup d’œil

| Observation | Cause probable |
|---|---|
| `Name or service not known` | Aucune résolution de noms sur l’hôte |
| Délai d’expiration, sortie 124 | Le pare-feu rejette silencieusement (DROP) |
| `Connection refused` | Aucun service sur le port ou règle REJECT |
| Connexion établie, pas de bannière, puis RST | L’équilibreur de charge accepte, mais le backend est inaccessible |
| `Didn't find STARTTLS` | Le serveur ne propose pas de chiffrement du transport |
| `wrong version number`, `errno=32` | Erreurs consécutives après TLS forcé sans STARTTLS |
| `501` / `553` à MAIL FROM | Domaine expéditeur non FQDN ou non autorisé |
| `554 relay access denied` | Adresse IP source non autorisée sur le relais |
| `250 queued`, mais aucune livraison | Alignement SPF, DKIM ou DMARC chez le destinataire |

## Tests de charge et limites de débit

Pour les tests de volume, une règle est souvent négligée au quotidien : le problème n’est pas le nombre de messages, mais le nombre de connexions. Les relais typiques autorisent quelques centaines de connexions par minute, mais des dizaines de milliers de messages. Gardez donc une session ouverte et envoyez-y de nombreuses enveloppes, plutôt que de vous reconnecter pour chaque message.

Dans `smtplib`, cela signifie simplement réutiliser plusieurs fois le même objet de connexion et rétablir la session de façon contrôlée après un nombre fixe de messages. En revanche, celui qui ouvre une nouvelle connexion pour chaque e-mail dépasse la limite de connexions bien avant la limite de messages et provoque des refus qui ressemblent à un problème du côté distant.

## Conclusion

Le test SMTP manuel n’est pas une solution de secours pour les environnements dépourvus d’outils, mais le diagnostic le plus précis disponible dans l’exploitation de la messagerie. Il distingue clairement la résolution de noms, l’accessibilité, le dialogue protocolaire et le chiffrement, et fournit un résultat univoque à chaque niveau. En écoutant d’abord, puis en menant le dialogue à la main et en prenant les codes de réponse au sérieux, vous obtenez en quelques minutes un constat permettant d’étayer un ticket auprès de l’équipe réseau ou du fournisseur : avec l’adresse source, le port cible, le comportement observé et le code de sortie.

## Sources

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Définit le dialogue SMTP, l’ordre des commandes et la signification des codes de réponse.

2.  [RFC 3207: SMTP Service Extension for Secure SMTP over TLS](https://www.rfc-editor.org/rfc/rfc3207): Décrit STARTTLS comme extension, y compris le comportement lorsque le serveur ne le propose pas.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Structure et évaluation de l’enregistrement SPF pour l’autorisation des systèmes émetteurs.

4.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Régit l’alignement entre les expéditeurs de l’enveloppe et de l’en-tête, ainsi que l’évaluation de la politique.

5.  [OpenSSL: s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Référence des options utilisées, notamment `-starttls`, `-servername` et `-verify_hostname`.

6.  [Documentation Python : smtplib](https://docs.python.org/3/library/smtplib.html): Bibliothèque standard pour les sessions SMTP, y compris STARTTLS et la sortie de débogage.

7.  [Manuel de référence Bash : redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections): Documente `/dev/tcp` comme pseudo-périphérique propre à bash pour les connexions réseau.
