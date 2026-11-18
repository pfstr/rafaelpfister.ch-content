---
title: "KB5131085 sur Windows Server 2019 : Send-MailMessage échoue après la signature NTLM"
navTitle: "KB5131085 envoi d’e-mails"
description: "Après la mise à jour de novembre KB5131085, l’envoi d’e-mails via Send-MailMessage vers des serveurs Postfix échoue sur Windows Server 2019. La nouvelle signature NTLM de KB5135557 est considérée comme la cause. Cet article explique pourquoi un client SMTP utilise NTLM, comment vérifier le cas et quels contournements existent : forcer AUTH LOGIN, désactiver NTLM sur le serveur de messagerie ou le désactiver temporairement via le Registre."
date: "2026-11-17"
kategorie: "Windows et SMTP"
timeToRead: "7 min de lecture"
themen:
  - windows-client
  - smtp-mailflow
produkte:
  - "windows-client"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
  - "releases"
slug: "kb5131085-sur-windows-server-2019-send-mailmessage-echoue-apres-la-signature-ntlm"
translationId: "article-ca3e77cefa049fca"
translationOf: kb5131085-send-mailmessage-ntlm
url: https://rafaelpfister.ch/fr/blog/kb5131085-sur-windows-server-2019-send-mailmessage-echoue-apres-la-signature-ntlm
translationSourceHash: 6416155fbbec934196848b06f3468ff46dbca040310b92cf8fd3171db1858227
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:18:59.696Z
translationReview: automatic
---

# KB5131085 sur Windows Server 2019 : Send-MailMessage échoue après la signature NTLM

Depuis le Patch Tuesday du 10 novembre 2026, des administrateurs signalent que les scripts sous Windows Server 2019 ne peuvent plus envoyer d’e-mails. Sont concernés les rapports de sauvegarde, les alertes de supervision et des tâches similaires qui envoient depuis des années des e-mails à un serveur Postfix avec la cmdlet PowerShell `Send-MailMessage`. Après la désinstallation de la mise à jour cumulative KB5131085 (build du système d’exploitation 17763.9487), l’envoi fonctionne à nouveau. Günter Born a décrit ce cas le 17 novembre 2026 sur Borns IT- und Windows-Blog, à la suite du signalement d’un lecteur.

Microsoft ne répertorie jusqu’à présent pas ce problème comme problème connu. La seule entrée de problème connu relative à KB5131085 concerne des commandes d’activation avec l’erreur `0x80041008`. La cause n’est donc pas officiellement confirmée, mais elle peut être bien déduite des modifications apportées par la mise à jour et du comportement de la classe .NET `SmtpClient`.

## Ce que KB5131085 modifie dans NTLM

La mise à jour contient des mesures de protection contre les attaques par relais NTLM, décrites par Microsoft dans KB5135557 (« NTLM signing security improvements in Windows »). Elles s’appliquent à Windows 11, Windows 10 Enterprise LTSC 2021 et Windows Server 2016 à 2025. Le journal des modifications de KB5131085 indique à ce sujet que les clients proposent désormais par défaut la signature des messages NTLM pour toutes les connexions non anonymes.

Concrètement, cela signifie qu’un client Windows définit désormais, dans la négociation NTLM, l’indicateur `NTLMSSP_NEGOTIATE_SIGN` (`0x00000010`) ainsi que le `NTLMSSP_NEGOTIATE_KEY_EXCH` (`0x40000000`) nécessaire, même lorsque l’application n’a pas demandé d’intégrité. Auparavant, ces applications ne recevaient que `NTLMSSP_NEGOTIATE_ALWAYS_SIGN`, un espace réservé sans protection cryptographique. Microsoft déploie cette modification en deux phases :

| Phase | Date | Comportement |
|---|---|---|
| 1 | 10 novembre 2026 | Les clients proposent la signature sur toutes les connexions. Les serveurs acceptent les clients signés et non signés et journalisent les connexions non signées. |
| 2 | Mai 2027 | Les serveurs Windows exigent la signature par défaut et refusent les clients sans signature avec `SEC_E_UNSUPPORTED_FUNCTION` (`0x80090302`). |

Selon Microsoft, un client Windows revient sans erreur à l’ancien comportement si le serveur ne confirme pas la signature. Tous les serveurs Windows le font. Les homologues dotés de leur propre implémentation NTLM, comme un serveur de messagerie Linux, ne sont pas pris en compte dans cette affirmation.

## Pourquoi un client SMTP utilise NTLM

`Send-MailMessage` dans Windows PowerShell 5.1 utilise en interne la classe .NET `System.Net.Mail.SmtpClient`. Celle-ci vérifie la ligne `AUTH` dans la réponse EHLO du serveur et essaie les mécanismes dans un ordre fixe : d’abord `GSSAPI` (Negotiate), puis `NTLM`, ensuite `WDigest`, et enfin `LOGIN`. Si le serveur propose NTLM, `SmtpClient` utilise NTLM, même lorsqu’un nom d’utilisateur et un mot de passe sont transmis via `-Credential`.

Postfix n’implémente pas lui-même l’authentification SMTP : il la délègue à Cyrus SASL ou Dovecot. Les deux peuvent proposer NTLM : Cyrus SASL via un plugin dédié, Dovecot jusqu’à la version 2.3 via `ntlm` dans `auth_mechanisms` (Dovecot 2.4 ne propose plus NTLM). Si NTLM est actif, Postfix annonce par exemple `AUTH PLAIN LOGIN NTLM`, et chaque client Windows s’authentifie via NTLM sans que personne ne le remarque. Avec KB5131085, le client propose en plus la signature et l’échange de clés dans cette négociation. Si l’implémentation NTLM du serveur de messagerie ne peut pas gérer cela, la connexion échoue et `Send-MailMessage` s’interrompt. Un commentateur de l’article de Born indique la signature NTLM comme cause, et le constat concorde : le contournement du lecteur fonctionne parce qu’il exclut NTLM.

## Vérifier le cas

Déterminez d’abord si le serveur de messagerie propose NTLM. Depuis une machine Linux, la réponse EHLO après STARTTLS affiche les mécanismes proposés :

```bash
openssl s_client -starttls smtp -crlf -quiet \
  -connect mail.example.com:587
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-starttls smtp` | Établit la connexion en clair et bascule vers TLS via STARTTLS |
| `-crlf` | Envoie les fins de ligne au format CRLF, comme l’exige SMTP |
| `-quiet` | Masque les détails du certificat et de la session |
| `-connect mail.example.com:587` | Hôte cible et port de soumission |

</details>

Après l’établissement de la connexion, saisissez `EHLO test`. Si la ligne `250-AUTH` contient l’entrée `NTLM`, le serveur est concerné. Sur le serveur Postfix lui-même, deux commandes indiquent quel backend SASL est actif et quels mécanismes Dovecot propose :

```bash
postconf smtpd_sasl_type
doveconf auth_mechanisms
```

Sur le serveur Windows, le journal NTLM enregistre, à partir de Windows Server 2019, les connexions avec les indicateurs négociés. L’événement 4022 est l’entrée client :

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-NTLM/Operational' -MaxEvents 50 |
  Where-Object { $_.Id -in 4022, 4023 } |
  Format-List TimeCreated, Id, Message
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-LogName 'Microsoft-Windows-NTLM/Operational'` | Lit le journal NTLM opérationnel |
| `-MaxEvents 50` | Limite la sortie aux 50 entrées les plus récentes |
| `Where-Object { $_.Id -in 4022, 4023 }` | Filtre les événements d’audit pour le client (4022) et le serveur (4023) |
| `Format-List TimeCreated, Id, Message` | Affiche l’heure, l’ID d’événement et le texte avec les indicateurs négociés |

</details>

Une entrée à l’heure de la tentative d’envoi, avec le serveur de messagerie comme destination, confirme que `SmtpClient` a utilisé NTLM.

## Contournement 1 : forcer AUTH LOGIN dans le script

`Send-MailMessage` ne fournit aucun paramètre permettant de définir le mécanisme d’authentification. La classe `SmtpClient`, appelée directement, le permet : un `CredentialCache` ne fournit les données de connexion que pour un mécanisme donné. Si seul `Login` est défini, `SmtpClient` ignore NTLM et utilise `AUTH LOGIN`. Le script suivant est basé sur l’exemple du lecteur dans l’article de Born :

```powershell
$server = 'mail.example.com'
$port   = 587
$cred   = Import-Clixml -Path 'C:\Skripte\smtp.cred'

$cache = New-Object System.Net.CredentialCache
$cache.Add($server, $port, 'Login', $cred.GetNetworkCredential())

$smtp = New-Object System.Net.Mail.SmtpClient($server, $port)
$smtp.EnableSsl   = $true
$smtp.Credentials = $cache

$msg = New-Object System.Net.Mail.MailMessage
$msg.From = 'backup@example.com'
$msg.To.Add('admin@example.com')
$msg.Subject      = 'Backup-Report'
$msg.Body         = 'Sicherung abgeschlossen.'
$msg.BodyEncoding = [System.Text.Encoding]::UTF8

try {
    $smtp.Send($msg)
}
finally {
    $msg.Dispose()
    $smtp.Dispose()
}
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Import-Clixml -Path …` | Charge un PSCredential préalablement enregistré avec `Get-Credential \| Export-Clixml`; lié à l’utilisateur et à l’ordinateur via DPAPI |
| `$cache.Add($server, $port, 'Login', …)` | Enregistre les données de connexion uniquement pour l’hôte, le port et le mécanisme `Login` |
| `SmtpClient($server, $port)` | Crée le client SMTP pour le serveur et le port de soumission |
| `EnableSsl = $true` | Bascule vers TLS via STARTTLS avant la transmission des données de connexion |
| `Credentials = $cache` | Transmet le CredentialCache au lieu d’un simple NetworkCredential |
| `BodyEncoding = [System.Text.Encoding]::UTF8` | Encode le texte en UTF-8 afin que les caractères accentués arrivent correctement |

</details>

`EnableSsl` est impératif ici : `AUTH LOGIN` ne transmet le nom d’utilisateur et le mot de passe qu’encodés en Base64, donc en clair sans TLS. Ce contournement intervient script par script, mais ne modifie pas la configuration système et laisse intacte la protection NTLM de la mise à jour.

## Contournement 2 : désactiver NTLM sur le serveur de messagerie

Si vous exploitez vous-même le serveur Postfix, vous pouvez résoudre le problème de manière centralisée : retirez `ntlm` de `auth_mechanisms` dans la configuration Dovecot (généralement `/etc/dovecot/conf.d/10-auth.conf`), afin qu’il ne reste que `plain login`, puis rechargez Dovecot :

```bash
sudo systemctl reload dovecot
```

Avec Cyrus SASL, `mech_list: plain login` dans le fichier `smtpd.conf` du répertoire de configuration SASL limite les mécanismes en conséquence. Postfix n’annonce alors plus NTLM et `Send-MailMessage` revient à `LOGIN` sans modification du script. Vérifiez au préalable si d’autres clients dépendent de NTLM, par exemple d’anciens profils Outlook avec l’option « Authentification par mot de passe sécurisé (SPA) ». Pour la soumission via TLS, NTLM n’apporte aucun gain de sécurité par rapport à `PLAIN` et `LOGIN`.

## Contournement 3 : Registre, temporairement seulement

KB5135557 documente une valeur de Registre avec laquelle le client ne propose plus la signature :

```powershell
reg add 'HKLM\SYSTEM\CurrentControlSet\Control\Lsa\MSV1_0' /v ClientAllowUnsignedNtlm /t REG_DWORD /d 1 /f
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `HKLM\…\Lsa\MSV1_0` | Clé du package d’authentification NTLM |
| `/v ClientAllowUnsignedNtlm` | Nom de la valeur ; `0` (valeur par défaut) propose la signature, `1` rétablit l’ancien comportement |
| `/t REG_DWORD` | Type de données de la valeur |
| `/d 1` | Définit la valeur sur 1 |
| `/f` | Écrase sans demander de confirmation |

</details>

Selon Microsoft, aucun redémarrage n’est nécessaire. Cette valeur s’applique toutefois à toutes les connexions NTLM du serveur, pas uniquement à l’envoi d’e-mails, et supprime donc la protection contre les relais apportée par la mise à jour pour l’ensemble du système. Microsoft la présente comme une mesure de compatibilité pure et se réserve le droit de supprimer ultérieurement les valeurs de Registre. Elle ne fait au mieux que combler le délai nécessaire à la mise en œuvre du contournement 1 ou 2. Désinstaller KB5131085 n’est pas une alternative, car toutes les corrections de sécurité du Patch Tuesday de novembre seraient alors absentes.

## À moyen terme : remplacer Send-MailMessage

Microsoft considère `Send-MailMessage` comme obsolète depuis des années et déconseille son utilisation, car la cmdlet ne garantit pas des connexions sécurisées. Pour `SmtpClient` également, la documentation .NET recommande d’autres bibliothèques telles que MailKit pour les nouveaux développements. Pour les scripts qui envoient des e-mails via Exchange Online, l’API Microsoft Graph avec `Send-MgUserMail` et l’enregistrement d’application constitue la méthode prise en charge ; la configuration est décrite dans l’article [Accéder à Exchange Online avec PowerShell : remplacer EWS par Microsoft Graph](/blog/microsoft-graph-powershell-postfach-anbindung). Au plus tard avec la phase 2 en mai 2027, il vaut la peine d’inventorier les scripts et services qui utilisent encore NTLM : les événements d’audit 4022 et 4023 fournissent cette liste.

## Sources

1.  [Windows Server 2019 : KB5131085 perturbe l’envoi d’e-mails via PowerShell (Borns IT- und Windows-Blog)](https://borncity.com/blog/2026/11/17/windows-server-2019-kb5131085-stoert-mail-versand-per-powershell/) : témoignage d’un lecteur du 17 novembre 2026 avec le symptôme, le test de désinstallation, le script CredentialCache et la mention de la signature NTLM dans les commentaires.

2.  [KB5131085 (build du système d’exploitation 17763.9487)](https://support.microsoft.com/help/5131085) : journal des modifications de la mise à jour de novembre pour Windows Server 2019 et Windows 10 Enterprise LTSC 2019, avec l’entrée NTLM et le problème connu concernant l’erreur 0x80041008.

3.  [KB5135557 : NTLM signing security improvements in Windows](https://support.microsoft.com/en-us/help/5135557) : indicateurs, calendrier des deux phases, événements d’audit 4022/4023 et valeurs de Registre ClientAllowUnsignedNtlm et ServerRequireNtlmSign.

4.  [SmtpAuthenticationManager.cs (.NET Framework Reference Source)](https://github.com/microsoft/referencesource/blob/master/System/net/System/Net/mail/SmtpAuthenticationManager.cs) : ordre d’enregistrement des modules d’authentification SMTP Negotiate, NTLM, Digest et Login dans SmtpClient.

5.  [Send-MailMessage (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage) : indication selon laquelle la cmdlet est obsolète et ne garantit pas des connexions sécurisées.

6.  [Classe SmtpClient (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/api/system.net.mail.smtpclient) : recommandation d’utiliser MailKit ou d’autres bibliothèques pour les nouveaux développements.

7.  [Postfix SASL Howto](https://www.postfix.org/SASL_README.html) : authentification SMTP dans Postfix via Cyrus SASL ou Dovecot, y compris la restriction des mécanismes proposés.

8.  [Dovecot : Authentication Mechanisms Overview](https://doc.dovecot.org/main/core/config/auth/mechanisms/overview.html) : liste actuelle des mécanismes SASL dans Dovecot, qui ne contient plus NTLM.
