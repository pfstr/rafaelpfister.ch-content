---
title: "Tester SMTP sous Linux : de la connexion TCP à l’e-mail distribué"
navTitle: "Tester SMTP"
description: "Lorsqu’une appliance ne distribue plus d’e-mails, un test SMTP manuel est plus utile que n’importe quel journal. Découvrez comment vérifier, couche par couche et avec les outils intégrés, ce que signifient les différents symptômes d’erreur et pourquoi un Load Balancer peut fausser le diagnostic."
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
translationSourceHash: 17f985224e55b45d0395c6f5f6a100918ad6718f2bd22191bc29f30e204e5e11
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:48:43.015Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/tester-smtp-sous-linux-de-la-connexion-tcp-a-l-e-mail-delivre
---

# Tester SMTP sous Linux : de la connexion TCP à l’e-mail distribué

Lorsqu’une passerelle de messagerie cesse soudainement de distribuer des e-mails, les journaux de l’appliance n’affichent souvent que le résultat final : une distribution échoue, la file d’attente grossit, un message d’erreur mentionne un délai d’attente. Seul un test manuel depuis la ligne de commande permet de déterminer la cause réelle. SMTP est un protocole en clair qui peut entièrement être utilisé à la main, ce qui en fait un outil de diagnostic disponible partout sans installation supplémentaire.

La deuxième raison du test manuel : il est généralement impossible d’installer quoi que ce soit sur les appliances. Pas de gestionnaire de paquets, pas de droits root, pas de `swaks`. Toutes les étapes suivantes fonctionnent donc avec ce qui est déjà présent sur pratiquement tous les systèmes Linux.

## Séparer les couches

Un échec d’envoi d’e-mail peut se produire à cinq niveaux différents, chacun générant un symptôme différent :

1. **Résolution de noms :** l’hôte cible ne peut pas être traduit en adresse IP.
2. **Connexion TCP :** la connexion au port ne s’établit pas ou est réinitialisée.
3. **Dialogue SMTP :** la connexion est établie, mais le serveur refuse l’expéditeur, le destinataire ou le contenu.
4. **Chiffrement de transport :** STARTTLS est absent, le certificat n’est pas valide ou la version TLS ne convient pas.
5. **Vérification de l’expéditeur :** l’e-mail est accepté puis rejeté chez le destinataire en raison de SPF, DKIM ou DMARC.

Le diagnostic gagne énormément en efficacité lorsque vous vérifiez ces niveaux successivement et séparément, plutôt que d’envoyer immédiatement un e-mail de test complet. Un essai global qui échoue indique seulement que quelque chose ne fonctionne pas. La vérification par couche indique quoi.

## Étape 1 : résolution de noms

```bash
getent hosts relay.example.com
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `hosts` | Base de données NSS interrogée ; utilise les mêmes sources et le même ordre que le système lui-même, conformément à `nsswitch.conf` |
| `relay.example.com` | Nom d’hôte à résoudre |

</details>

Si la sortie reste vide, aucun serveur de noms n’est accessible depuis cet hôte ou il ne répond pas aux noms externes. Cela arrive régulièrement en pratique : les appliances situées dans des zones isolées ne reçoivent souvent qu’un résolveur interne ne connaissant que ses propres zones.

```bash
cat /etc/resolv.conf; grep hosts: /etc/nsswitch.conf
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `/etc/resolv.conf` | Fichier contenant les serveurs de noms configurés, affiché par `cat` |
| `hosts:` | Motif de recherche pour `grep` : la ligne qui définit l’ordre des sources de résolution (fichiers, DNS) |
| `/etc/nsswitch.conf` | Fichier de configuration NSS consulté par `grep` |

</details>

En l’absence de résolution, testez directement l’adresse IP dans les étapes suivantes. Cela suffit largement pour le diagnostic et sépare clairement le problème DNS du problème de transport. En production, l’absence de résolution reste naturellement un constat distinct à corriger.

## Étape 2 : accessibilité du port

Pour le simple contrôle TCP, bash suffit. Le pseudo-périphérique `/dev/tcp` ouvre une connexion sans nécessiter l’installation de `nc` ou de `telnet` :

```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/192.0.2.25/25' ; echo "exit=$?"
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `timeout 10` | Interrompt la commande suivante après 10 secondes et renvoie alors le code de sortie 124 |
| `bash -c '…'` | Exécute la chaîne de commandes dans bash ; nécessaire car `/dev/tcp` est une fonctionnalité de bash |
| `exec 3<>/dev/tcp/192.0.2.25/25` | Ouvre le descripteur de fichier 3 en lecture et écriture comme connexion TCP vers 192.0.2.25, port 25 |
| `echo "exit=$?"` | Affiche le code de sortie de la commande précédente |

</details>

Le code de sortie est ici l’information essentielle :

| exit | Signification |
|---|---|
| `0` | La connexion est établie, le port est ouvert |
| `124` | Délai d’attente : les paquets sont supprimés, typique d’un pare-feu avec règle DROP |
| `1` | Refus immédiat (RST) ou route absente |

En pratique, la différence entre 124 et 1 est l’indice le plus important. Un délai d’attente signifie que quelqu’un supprime silencieusement les paquets en chemin, ce qui est presque toujours une règle de pare-feu. Un RST immédiat provient en revanche d’un système qui répond, mais ne propose pas le service.

Vérifiez immédiatement les deux ports pertinents ainsi qu’une autre cible arbitraire pour voir si l’hôte est autorisé à établir des connexions sortantes :

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
| `set -- $t` | Découpe la paire de valeurs au niveau de l’espace en paramètres positionnels `$1` (adresse IP) et `$2` (port) |
| `timeout 8` | Interrompt la tentative de connexion après 8 secondes (code de sortie 124) |
| `bash -c "…"` | Exécute l’établissement de connexion `/dev/tcp` dans bash |
| `2>/dev/null` | Supprime les messages d’erreur afin qu’une seule ligne de résultat apparaisse par cible |

</details>

Si le test de contrôle échoue lui aussi, le système n’a généralement pas d’accès direct vers l’extérieur et le trafic doit passer par un relais interne ou un proxy. Vous verrez plus bas pourquoi ce cas est particulièrement délicat.

Si `/dev/tcp` est absent, le shell n’est pas bash. Cette fonctionnalité n’existe pas sous `sh`, `ash` ou `ksh`, ce qui est volontiers interprété à tort comme un problème réseau :

```bash
ps -p $$ -o comm= ; echo "BASH_VERSION=${BASH_VERSION:-keine bash}"
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-p $$` | Limite la sortie au processus portant le PID du shell actuel (`$$`) |
| `-o comm=` | N’affiche que le nom de la commande ; l’étiquette vide après `=` supprime l’en-tête |
| `${BASH_VERSION:-keine bash}` | Affiche la version de bash ou le texte de remplacement si la variable n’est pas définie |

</details>

## Étape 3 : d’abord écouter, ne pas envoyer

Un serveur SMTP se présente spontanément par une bannière `220`. Le test individuel le plus significatif consiste donc à ouvrir une connexion sans rien faire :

```bash
{ exec 3<>/dev/tcp/192.0.2.25/25 && timeout 15 cat <&3; echo "[ende exit=$?]"; }
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `exec 3<>/dev/tcp/192.0.2.25/25` | Ouvre le descripteur de fichier 3 comme connexion TCP vers la cible |
| `timeout 15 cat <&3` | Lit pendant 15 secondes tout ce que le serveur envoie spontanément et l’affiche |
| `echo "[ende exit=$?]"` | Affiche le code de sortie à l’expiration ; 124 signifie : plus rien n’est arrivé pendant 15 secondes |

</details>

Ces quelques caractères distinguent deux situations totalement différentes. Si un `220 mail.example.com ESMTP` arrive, le serveur distant parle et les erreurs suivantes se trouvent dans le dialogue. Si rien n’arrive, cela ne vient pas d’une commande mal formulée de votre part puisque vous n’en avez envoyé aucune.

Le descripteur de fichier reste ensuite ouvert dans le shell. Fermez-le avant de lancer le test suivant, sans quoi vous risquez de continuer à travailler sur une ancienne connexion qui n’est plus intacte :

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

Une fois la bannière reçue, effectuez le dialogue complet. Il est important qu’un processus de lecture soit actif afin de voir chaque réponse au moment où elle arrive. Un script qui envoie d’abord tout puis lit ensuite ne vous montrera rien en cas d’interruption au milieu du dialogue :

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
| `cat <&3 & R=$!` | Démarre un lecteur en arrière-plan pour le descripteur de fichier 3 et enregistre son PID dans `R` |
| `printf '…\r\n' >&3` | Envoie une commande SMTP avec la fin de ligne CRLF requise sur la connexion |
| `sleep n` | Attend le nombre de secondes indiqué pour la réponse du serveur avant la commande suivante |
| `date -R` | Fournit la date au format conforme RFC pour l’en-tête `Date:` |
| `date +%s` | Fournit l’heure Unix comme base unique simple pour le Message-ID |
| `kill $R 2>/dev/null` | Arrête le lecteur en arrière-plan ; aucun message d’erreur n’apparaît s’il est déjà terminé |

</details>

Deux détails déterminent la réussite ou l’échec. SMTP exige CRLF comme fin de ligne, donc `printf` avec `\r\n` et non `echo`. Le point sur sa propre ligne termine la partie message ; il doit être envoyé sous la forme `\r\n.\r\n`.

Déroulement attendu : `220` lors de l’établissement de la connexion, `250` en réponse à EHLO, `250 2.1.0` à MAIL FROM, `250 2.1.5` à RCPT TO, `354` à DATA et enfin `250 2.0.0 Ok: queued as <id>`. Notez l’ID de file d’attente. Il permet au fournisseur exploitant de suivre le message s’il n’arrive jamais chez le destinataire.

Le nom EHLO mérite attention : certains relais le vérifient par rapport au DNS direct et inverse et répondent sinon par `501` ou `504`. Utilisez le FQDN réel du système expéditeur, pas le nom court.

## Étape 5 : STARTTLS et certificat

Pour la connexion chiffrée, `openssl s_client` prend lui-même en charge la négociation STARTTLS puis transmet le canal à l’entrée standard :

```bash
openssl s_client -connect 192.0.2.25:25 -starttls smtp -tls1_2 -brief </dev/null
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-connect 192.0.2.25:25` | Hôte cible et port de la connexion |
| `-starttls smtp` | Exécute d’abord le dialogue SMTP en clair, puis passe à TLS via STARTTLS |
| `-tls1_2` | Négocie exclusivement TLS 1.2 |
| `-brief` | Réduit la sortie à un bref résumé de la connexion négociée |
| `</dev/null` | Ferme immédiatement l’entrée standard afin que `s_client` n’attende pas de manière interactive après le handshake |

</details>

Si vous vous connectez via l’adresse IP parce que le DNS est absent, la vérification du nom d’hôte devient inopérante. Le nom du certificat ne correspond alors pas à l’adresse numérique. SNI et le nom de vérification peuvent être définis explicitement, sans aucune requête DNS :

```bash
openssl s_client -connect 192.0.2.25:25 \
  -servername mail.example.com -verify_hostname mail.example.com \
  -starttls smtp -tls1_2 -brief </dev/null
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-servername mail.example.com` | Définit le nom SNI dans ClientHello, indépendamment de l’adresse de connexion |
| `-verify_hostname mail.example.com` | Vérifie le certificat serveur par rapport à ce nom plutôt qu’à l’adresse numérique |

</details>

Deux symptômes d’erreur apparaissent régulièrement ici et sont souvent mal interprétés.

**« Didn't find STARTTLS in server response, trying anyway »** signifie que le serveur n’a pas proposé STARTTLS dans sa réponse EHLO. `openssl` envoie néanmoins un ClientHello TLS, le serveur y voit des données de protocole invalides et la connexion se termine par `wrong version number` ou `write:errno=32` (EPIPE). Ces deux messages sont des erreurs consécutives. L’information essentielle est : pas de STARTTLS. Consultez le dialogue en clair de l’étape 4 pour voir quelles capacités le serveur annonce réellement.

**L’absence de STARTTLS sur un saut interne** est souvent parfaitement correcte. Lorsqu’un Load Balancer transmet la connexion au niveau 4, ce n’est pas lui qui négocie TLS, mais le système placé derrière lui avec la véritable cible. Tester en clair sur le segment interne n’est alors pas une faille de sécurité, mais simplement l’architecture.

## Étape 6 : Python comme alternative

Si Python est disponible, vous vous épargnez la temporisation manuelle avec `sleep`. La bibliothèque standard suffit, il n’y a rien à installer :

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

`set_debuglevel(1)` journalise le dialogue complet avec tous les codes de réponse, et `smtplib` lit chaque réponse de manière synchrone. Une interruption apparaît comme `SMTPServerDisconnected` avec la dernière ligne reçue, plutôt que comme un Broken Pipe silencieux.

Deux points posent souvent problème : `server_hostname` est obligatoire lors d’une connexion via une adresse IP, faute de quoi Python vérifie le certificat par rapport à l’adresse numérique. Et si vous désactivez délibérément la vérification, `check_hostname = False` doit précéder `verify_mode = ssl.CERT_NONE`, sans quoi Python lève une `ValueError`.

## Adresse d’expéditeur, SPF et alignement

Un test échoue étonnamment souvent non pas au transport, mais à l’adresse d’expéditeur choisie. Trois points doivent être vérifiés au préalable.

Le domaine expéditeur doit être un FQDN. Une adresse telle que `test@meine-testdomain` sans domaine de premier niveau est déjà refusée par de nombreux MTA lors de MAIL FROM avec `501` ou `553`.

Le domaine doit autoriser le chemin d’envoi utilisé. Un coup d’œil à l’enregistrement SPF permet de voir si l’adresse sortante est couverte :

```bash
dig +short TXT example.com | grep spf1
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `+short` | N’affiche que les valeurs des enregistrements, sans en-têtes ni métadonnées |
| `TXT` | Type d’enregistrement interrogé |
| `example.com` | Nom interrogé |
| `grep spf1` | Filtre la ligne SPF parmi plusieurs enregistrements TXT |

</details>

Et lorsque DMARC est actif, l’alignement est déterminant. Si l’enregistrement contient `aspf=s`, le domaine de l’enveloppe (MAIL FROM) et le domaine de l’en-tête `From:` doivent correspondre exactement, et non simplement être apparentés :

```bash
dig +short TXT _dmarc.example.com
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `+short` | N’affiche que les valeurs des enregistrements, sans en-têtes ni métadonnées |
| `TXT _dmarc.example.com` | Type d’enregistrement et nom défini pour DMARC sous le domaine |

</details>

Avec `p=reject`, un e-mail de test dont l’alignement ne correspond pas disparaît silencieusement chez le destinataire, même si votre relais l’a accepté avec `250 queued`. C’est la cause la plus fréquente des messages considérés comme envoyés avec succès côté expéditeur, mais qui n’arrivent jamais.

## Lorsqu’un Load Balancer s’interpose

Dans les environnements plus importants, une appliance envoie rarement directement vers Internet. Un serveur virtuel sur un Load Balancer reçoit habituellement la connexion, la réécrit vers une adresse définie via Source-NAT, puis la transmet vers l’extérieur. Cela a une conséquence désagréable pour le diagnostic.

Un serveur virtuel fonctionnant au niveau 4 confirme immédiatement le handshake TCP avant d’avoir lui-même établi une connexion avec la cible. Si cette seconde connexion échoue, vous voyez côté client une connexion établie avec succès puis immédiatement réinitialisée : `Connection reset by peer`, sans aucune bannière SMTP. L’erreur ne se situe alors ni chez vous ni à la cible, mais dans le pool derrière le serveur virtuel, par exemple parce qu’un membre est marqué down ou que le FQDN configuré ne peut pas être résolu.

Cela explique aussi pourquoi un test direct vers la cible Internet doit échouer lorsque la règle de transfert n’accepte que le trafic provenant de l’adresse SNAT déjà réécrite. Les connexions ayant l’adresse source initiale ne correspondent à aucune règle et sont supprimées. Dans de tels environnements, testez toujours contre le serveur virtuel prévu, et non contre la véritable cible.

Une seule ligne indique quelle adresse source votre système utilise pour une cible donnée. La valeur après `src` est exactement l’information dont l’équipe réseau a besoin pour autoriser le trafic :

```bash
ip route get 192.0.2.25
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `route get` | Demande au noyau quelle route il choisirait pour une cible donnée |
| `192.0.2.25` | Adresse cible de la connexion simulée |

</details>

Si le système est derrière NAT, le système distant ne voit pas cette adresse, mais l’adresse publique du périmètre. Il est impossible de la déterminer depuis l’intérieur tant qu’aucun trafic ne passe ; elle figure dans la règle NAT.

## Symptômes d’erreur en un coup d’œil

| Observation | Cause probable |
|---|---|
| `Name or service not known` | Aucune résolution de noms sur l’hôte |
| Délai d’attente, exit 124 | Le pare-feu supprime silencieusement les paquets (DROP) |
| `Connection refused` | Aucun service sur le port ou règle REJECT |
| Connexion établie, pas de bannière, puis RST | Le Load Balancer accepte, backend inaccessible |
| `Didn't find STARTTLS` | Le serveur ne propose pas de chiffrement de transport |
| `wrong version number`, `errno=32` | Erreurs consécutives après TLS forcé sans STARTTLS |
| `501` / `553` sur MAIL FROM | Domaine expéditeur non FQDN ou non autorisé |
| `554 relay access denied` | IP source non autorisée sur le relais |
| `250 queued`, mais aucune distribution | Alignement SPF, DKIM ou DMARC chez le destinataire |

## Tests de charge et limites de débit

Pour les tests de volume, une règle souvent négligée au quotidien s’applique : ce n’est pas le nombre de messages qui pose problème, mais le nombre de connexions. Les relais typiques autorisent quelques centaines de connexions par minute, mais des dizaines de milliers de messages. Gardez donc une session ouverte et envoyez-y de nombreuses enveloppes plutôt que de vous reconnecter pour chaque message.

Dans `smtplib`, cela signifie simplement réutiliser plusieurs fois le même objet de connexion et rétablir la session de manière contrôlée après un nombre fixe de messages. À l’inverse, celui qui ouvre une nouvelle connexion pour chaque e-mail dépasse la limite de connexions bien avant celle des messages et provoque des refus qui ressemblent à un problème du système distant.

## Conclusion

Le test SMTP manuel n’est pas une solution de secours pour les environnements dépourvus d’outils, mais le diagnostic le plus précis disponible dans l’exploitation de la messagerie. Il sépare clairement la résolution de noms, l’accessibilité, le dialogue protocolaire et le chiffrement, et fournit un résultat sans ambiguïté pour chaque niveau. En écoutant d’abord, puis en menant le dialogue à la main et en prenant les codes de réponse au sérieux, vous obtenez en quelques minutes un constat permettant d’étayer un ticket auprès de l’équipe réseau ou du fournisseur : adresse source, port cible, comportement observé et code de sortie.

## Historique des modifications

- **25 octobre 2026 :** remplacement de la source RFC 7489 par RFC 9989, qui remplace la spécification DMARC depuis mai 2026.

## Sources

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): définit le dialogue SMTP, l’ordre des commandes et la signification des codes de réponse.

2.  [RFC 3207: SMTP Service Extension for Secure SMTP over TLS](https://www.rfc-editor.org/rfc/rfc3207): décrit STARTTLS comme extension, y compris le comportement lorsque le serveur ne le propose pas.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): structure et évaluation de l’enregistrement SPF pour l’autorisation des systèmes expéditeurs.

4.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): régit l’alignement entre les expéditeurs de l’enveloppe et de l’en-tête, la balise `aspf` ainsi que l’évaluation des politiques ; remplace RFC 7489 depuis mai 2026.

5.  [OpenSSL: s_client](https://docs.openssl.org/master/man1/openssl-s_client/): référence des options utilisées, notamment `-starttls`, `-servername` et `-verify_hostname`.

6.  [Documentation Python : smtplib](https://docs.python.org/3/library/smtplib.html): bibliothèque standard pour les sessions SMTP, y compris STARTTLS et la sortie de débogage.

7.  [Manuel de référence Bash : redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections): documente `/dev/tcp` comme pseudo-périphérique propre à bash pour les connexions réseau.
