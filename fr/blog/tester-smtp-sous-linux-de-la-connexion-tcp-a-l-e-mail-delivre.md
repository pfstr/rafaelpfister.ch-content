---
title: "Tester SMTP sous Linux : de la connexion TCP à l’e-mail délivré"
navTitle: "Tester SMTP"
description: "Lorsqu’une appliance ne délivre plus d’e-mails, un test SMTP manuel est plus utile que n’importe quel journal. Découvrez comment vérifier chaque couche avec les outils intégrés, ce que signifient les différents scénarios d’erreur et pourquoi un équilibreur de charge peut fausser le diagnostic."
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
url: https://rafaelpfister.ch/fr/blog/tester-smtp-sous-linux-de-la-connexion-tcp-a-l-e-mail-delivre
translationSourceHash: 5c8e1b19b8002fc6dc109c5471afbe91dba9302274cef0b63eebd40e01a98fe2
translationModel: gpt-5.6-terra
translatedAt: 2026-08-01T06:12:30.135Z
translationReview: automatic
---

# Tester SMTP sous Linux : de la connexion TCP à l’e-mail délivré

Lorsqu’une passerelle de messagerie cesse soudainement de délivrer des messages, les journaux de l’appliance ne fournissent souvent que la dernière étape de l’histoire : une livraison échoue, la file d’attente grossit, un message d’erreur mentionne un délai d’attente. Seul un test manuel depuis la ligne de commande permet d’en déterminer la véritable cause. SMTP est un protocole en texte clair que l’on peut entièrement utiliser à la main, ce qui en fait l’un des outils de diagnostic les plus pratiques en exploitation de messagerie.

La seconde raison de recourir à un test manuel : il est généralement impossible d’installer quoi que ce soit sur les appliances. Pas de gestionnaire de paquets, pas de droits root, pas de `swaks`. Toutes les étapes suivantes fonctionnent donc avec ce qui est déjà disponible sur pratiquement tout système Linux.

## Distinguer les couches

Un envoi d’e-mail échoué peut échouer à cinq niveaux différents, chacun produisant un scénario d’erreur distinct :

1. **Résolution de noms :** l’hôte cible ne peut pas être traduit en adresse IP.
2. **Connexion TCP :** la connexion au port ne s’établit pas ou est réinitialisée.
3. **Dialogue SMTP :** la connexion est établie, mais le serveur refuse l’expéditeur, le destinataire ou le contenu.
4. **Chiffrement du transport :** STARTTLS est absent, le certificat n’est pas valide ou la version TLS ne convient pas.
5. **Vérification de l’expéditeur :** l’e-mail est accepté puis rejeté chez le destinataire à cause de SPF, DKIM ou DMARC.

Le diagnostic gagne énormément en efficacité lorsque vous vérifiez ces niveaux l’un après l’autre et séparément, au lieu d’envoyer immédiatement un e-mail de test complet. Un essai global échoué indique seulement que quelque chose ne fonctionne pas. La vérification par couche indique quoi.

## Étape 1 : Résolution de noms

```bash
getent hosts relay.example.com
```

Si la sortie reste vide, aucun serveur de noms n’est accessible sur cet hôte ou il ne répond pas aux noms externes. Cela arrive plus souvent qu’on ne le pense : les appliances dans des zones isolées ne disposent souvent que d’un résolveur interne qui ne connaît que ses propres zones.

```bash
cat /etc/resolv.conf; grep hosts: /etc/nsswitch.conf
```

Si la résolution manque, effectuez les tests suivants directement contre l’adresse IP. Cela suffit amplement pour le diagnostic et sépare clairement le problème DNS du problème de transport. En production, l’absence de résolution reste naturellement un constat distinct qu’il convient de corriger.

## Étape 2 : Accessibilité du port

Pour vérifier uniquement TCP, bash suffit. Le pseudo-périphérique `/dev/tcp` ouvre une connexion sans qu’il soit nécessaire d’installer `nc` ou `telnet` :

```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/192.0.2.25/25' ; echo "exit=$?"
```

Le code de sortie constitue ici l’information essentielle :

| exit | Signification |
|---|---|
| `0` | La connexion est établie, le port est ouvert |
| `124` | Délai d’attente : les paquets sont abandonnés, typique d’un pare-feu avec règle DROP |
| `1` | Refus immédiat (RST) ou route absente |

En pratique, la différence entre 124 et 1 est l’indication la plus importante. Un délai d’attente signifie que quelqu’un abandonne silencieusement le trafic en chemin, ce qui est presque toujours dû à une règle de pare-feu. Un RST immédiat provient au contraire d’un système qui répond, mais ne propose pas le service.

Vérifiez immédiatement les deux ports pertinents ainsi qu’une autre destination quelconque afin de déterminer si l’hôte est autorisé à établir des connexions sortantes :

```bash
for t in "192.0.2.25 25" "192.0.2.25 587" "1.1.1.1 443"; do
  set -- $t
  timeout 8 bash -c "exec 3<>/dev/tcp/$1/$2" 2>/dev/null
  echo "$1:$2 -> exit=$?"
done
```

Si le contre-test échoue également, le système ne dispose généralement pas d’accès sortant direct et le trafic doit passer par un relais interne ou un proxy. Nous verrons plus bas pourquoi ce cas est particulièrement délicat.

Si `/dev/tcp` est absent, le shell n’est pas bash. Sous `sh`, `ash` ou `ksh`, cette fonctionnalité n’existe pas, ce qui est souvent interprété à tort comme un problème réseau :

```bash
ps -p $$ -o comm= ; echo "BASH_VERSION=${BASH_VERSION:-keine bash}"
```

## Étape 3 : Écouter d’abord, ne rien envoyer

Un serveur SMTP se présente spontanément avec une bannière `220`. Le test individuel le plus parlant consiste donc à ouvrir une connexion sans rien faire :

```bash
{ exec 3<>/dev/tcp/192.0.2.25/25 && timeout 15 cat <&3; echo "[ende exit=$?]"; }
```

Ces quelques caractères distinguent deux situations totalement différentes. Si un `220 mail.example.com ESMTP` arrive, le serveur distant répond et toutes les erreurs ultérieures se trouvent dans le dialogue. Si rien n’arrive, ce n’est pas dû à une commande mal formulée de votre part, puisque vous n’en avez envoyé aucune.

Le descripteur de fichier reste ensuite ouvert dans le shell. Fermez-le avant de lancer le test suivant, faute de quoi vous risquez de continuer à travailler avec une ancienne connexion à moitié morte :

```bash
exec 3<&- 3>&-
```

## Étape 4 : Le dialogue SMTP à la main

Si la bannière apparaît, effectuez le dialogue complet. Il est important qu’un processus de lecture s’exécute en parallèle afin que vous voyiez chaque réponse au moment où elle arrive. Un script qui envoie tout d’abord puis lit ensuite ne vous montre rien en cas d’interruption au milieu du dialogue :

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

Deux détails déterminent le succès ou la frustration. SMTP exige CRLF comme fin de ligne, donc `printf` avec `\r\n` et non `echo`. Et le point sur sa propre ligne termine la partie message ; il doit être envoyé sous la forme `\r\n.\r\n`.

Le déroulement attendu : `220` lors de l’établissement de la connexion, `250` après EHLO, `250 2.1.0` après MAIL FROM, `250 2.1.5` après RCPT TO, `354` après DATA et enfin `250 2.0.0 Ok: queued as <id>`. Notez l’identifiant de file d’attente. Il permet au fournisseur exploitant le service de suivre le message s’il n’arrive jamais chez le destinataire.

Le nom EHLO mérite une attention particulière : certains relais le vérifient par rapport au DNS direct et inverse et répondent sinon par `501` ou `504`. Utilisez le FQDN réel du système expéditeur, et non son nom court.

## Étape 5 : STARTTLS et certificat

Pour la connexion chiffrée, `openssl s_client` se charge lui-même de la négociation STARTTLS puis transmet le canal à l’entrée standard :

```bash
openssl s_client -connect 192.0.2.25:25 -starttls smtp -tls1_2 -brief </dev/null
```

Si vous vous connectez via l’adresse IP parce que DNS est indisponible, la vérification du nom d’hôte devient inopérante. Le nom du certificat ne correspond alors pas à l’adresse numérique. SNI et le nom à vérifier peuvent être définis explicitement, sans aucune requête DNS :

```bash
openssl s_client -connect 192.0.2.25:25 \
  -servername mail.example.com -verify_hostname mail.example.com \
  -starttls smtp -tls1_2 -brief </dev/null
```

Deux scénarios d’erreur apparaissent régulièrement ici et sont souvent mal interprétés.

**« Didn't find STARTTLS in server response, trying anyway »** signifie que le serveur n’a pas proposé STARTTLS dans sa réponse EHLO. `openssl` envoie malgré tout un ClientHello TLS, le serveur y voit des données de protocole incohérentes et la connexion se termine par `wrong version number` ou `write:errno=32` (EPIPE). Ces deux messages sont des erreurs consécutives. L’information essentielle est : pas de STARTTLS. Consultez le dialogue en texte clair de l’étape 4 pour voir quelles capacités le serveur annonce réellement.

**L’absence de STARTTLS sur un saut interne** est souvent parfaitement correcte. Lorsqu’un équilibreur de charge transmet la connexion en couche 4, ce n’est pas lui qui négocie TLS, mais le système situé derrière lui avec la destination réelle. Tester en texte clair sur le segment interne n’est alors pas un défaut de sécurité, mais simplement l’architecture.

## Étape 6 : Python comme alternative

Si Python est disponible, vous vous épargnez les ajustements de temporisation avec `sleep`. La bibliothèque standard suffit, aucune installation supplémentaire n’est nécessaire :

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

`set_debuglevel(1)` journalise le dialogue complet, y compris tous les codes de réponse, et `smtplib` lit chaque réponse de manière synchrone. Une interruption apparaît sous forme de `SMTPServerDisconnected` avec la dernière ligne reçue, plutôt que comme un Broken Pipe silencieux.

Deux pièges : `server_hostname` est indispensable lors d’une connexion via une adresse IP, sans quoi Python vérifie le certificat par rapport à l’adresse numérique. Et si vous désactivez volontairement la vérification, `check_hostname = False` doit précéder `verify_mode = ssl.CERT_NONE`, sinon Python lève une `ValueError`.

## Adresse d’expéditeur, SPF et alignement

Un test échoue étonnamment souvent non pas au niveau du transport, mais à cause de l’adresse d’expéditeur choisie. Trois points doivent être vérifiés au préalable.

Le domaine de l’expéditeur doit être un FQDN. Une adresse telle que `test@meine-testdomain` sans domaine de premier niveau est rejetée par de nombreux MTA dès MAIL FROM avec `501` ou `553`.

Le domaine doit autoriser le chemin d’envoi utilisé. Un regard sur l’enregistrement SPF permet de voir si l’adresse sortante est couverte :

```bash
dig +short TXT example.com | grep spf1
```

Et lorsque DMARC est actif, l’alignement est déterminant. Si l’enregistrement contient `aspf=s`, le domaine dans l’enveloppe (MAIL FROM) et le domaine dans l’en-tête `From:` doivent correspondre exactement, et pas seulement être apparentés :

```bash
dig +short TXT _dmarc.example.com
```

Avec `p=reject`, un e-mail de test dont l’alignement ne convient pas disparaît silencieusement chez le destinataire, bien que votre relais l’ait accepté avec `250 queued`. C’est la cause la plus fréquente des messages considérés comme envoyés avec succès côté expéditeur, mais qui n’arrivent jamais.

## Lorsqu’un équilibreur de charge s’interpose

Dans les environnements de grande taille, une appliance envoie rarement directement sur Internet. Il est courant de disposer d’un serveur virtuel sur un équilibreur de charge qui accepte la connexion, réécrit l’adresse en une adresse définie par Source-NAT, puis la transmet vers l’extérieur. Cela a une conséquence désagréable pour le diagnostic.

Un serveur virtuel fonctionnant en couche 4 acquitte immédiatement le handshake TCP avant même d’avoir établi lui-même une connexion vers la destination. Si cette seconde connexion échoue, vous voyez côté client une connexion établie avec succès puis immédiatement réinitialisée : `Connection reset by peer`, sans aucune bannière SMTP. L’erreur ne se situe alors ni chez vous ni à destination, mais dans le pool derrière le serveur virtuel, par exemple parce qu’un membre est marqué down ou que le FQDN configuré ne peut pas être résolu.

Cela explique également pourquoi un test directement vers la destination Internet doit échouer si la règle de transfert n’accepte que le trafic issu de l’adresse SNAT déjà réécrite. Les connexions avec l’adresse source originale ne correspondent à aucune règle et sont abandonnées. Dans de tels environnements, testez toujours contre le serveur virtuel prévu, et non contre la destination réelle.

Une seule ligne indique quelle adresse source votre système utilise pour une destination donnée. La valeur après `src` est exactement l’information dont l’équipe réseau a besoin pour l’autorisation :

```bash
ip route get 192.0.2.25
```

Si le système se trouve derrière NAT, le serveur distant ne voit pas cette adresse, mais l’adresse publique du périmètre. Il est impossible de la déterminer depuis l’intérieur tant qu’aucun trafic ne passe ; elle figure dans la règle NAT.

## Scénarios d’erreur en un coup d’œil

| Observation | Cause probable |
|---|---|
| `Name or service not known` | Aucune résolution de noms sur l’hôte |
| Délai d’attente, exit 124 | Le pare-feu abandonne silencieusement le trafic (DROP) |
| `Connection refused` | Aucun service sur le port ou règle REJECT |
| Connexion établie, pas de bannière, puis RST | L’équilibreur de charge accepte, backend inaccessible |
| `Didn't find STARTTLS` | Le serveur ne propose pas de chiffrement du transport |
| `wrong version number`, `errno=32` | Erreurs consécutives après TLS forcé sans STARTTLS |
| `501` / `553` après MAIL FROM | Domaine expéditeur non FQDN ou non autorisé |
| `554 relay access denied` | IP source non autorisée sur le relais |
| `250 queued`, mais pas de livraison | Alignement SPF, DKIM ou DMARC chez le destinataire |

## Tests de charge et limites de débit

Pour les tests de volume, une règle souvent négligée au quotidien s’applique : le problème n’est pas le nombre de messages, mais le nombre de connexions. Les relais typiques autorisent quelques centaines de connexions par minute, mais des dizaines de milliers de messages. Gardez donc une session ouverte et envoyez-y de nombreuses enveloppes, plutôt que d’ouvrir une nouvelle connexion pour chaque message.

Dans `smtplib`, cela signifie simplement réutiliser plusieurs fois le même objet de connexion et rétablir la session de manière contrôlée après un nombre fixe de messages. À l’inverse, ouvrir une nouvelle connexion par e-mail atteint la limite de connexions bien avant celle des messages et provoque des refus qui donnent l’impression d’un problème du côté distant.

## Conclusion

Le test SMTP manuel n’est pas une solution de secours pour les environnements sans outils, mais le diagnostic le plus précis disponible en exploitation de messagerie. Il distingue clairement la résolution de noms, l’accessibilité, le dialogue de protocole et le chiffrement, et fournit un résultat sans ambiguïté pour chaque niveau. En écoutant d’abord, puis en menant le dialogue à la main et en prenant les codes de réponse au sérieux, vous obtenez en quelques minutes un constat permettant d’étayer un ticket auprès de l’équipe réseau ou du fournisseur : avec l’adresse source, le port cible, le comportement observé et le code de sortie.

## Sources

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Définit le dialogue SMTP, l’ordre des commandes et la signification des codes de réponse.

2.  [RFC 3207: SMTP Service Extension for Secure SMTP over TLS](https://www.rfc-editor.org/rfc/rfc3207): Décrit STARTTLS comme extension, y compris le comportement lorsque le serveur ne le propose pas.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Structure et évaluation de l’enregistrement SPF pour l’autorisation des systèmes expéditeurs.

4.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Régit l’alignement entre l’expéditeur de l’enveloppe et celui de l’en-tête, ainsi que l’évaluation de la politique.

5.  [OpenSSL: s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Référence des options utilisées, notamment `-starttls`, `-servername` et `-verify_hostname`.

6.  [Documentation Python : smtplib](https://docs.python.org/3/library/smtplib.html): Bibliothèque standard pour les sessions SMTP, y compris STARTTLS et la sortie de débogage.

7.  [Manuel de référence Bash : Redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections): Documente `/dev/tcp` comme pseudo-périphérique propre à bash pour les connexions réseau.
