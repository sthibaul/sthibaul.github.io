TP3: installation Mail
======================

On continue toujours avec nos VMs serveur et client.

# Exercice 1: Préparer le terrain

Avant d'installer et configurer le serveur de mail, préparons le terrain.

## Vérifier le nom du serveur

Il faut que votre serveur porte le nom qui sera utilisé par le serveur de mail, on peut vérifier que `hostname` renvoie bien `alma-server.adsillh.local` et sinon on peut utiliser

```shell
# hostnamectl set-hostname alma-server.adsillh.local
```

et rebooter pour recharger partout dans le système.

## Configurer le DNS

Il faut déclarer à quel serveur Internet doit envoyer les mails de notre
domaine `adsillh.local`: il faut ajouter à notre zone `adsillh.local` un
enregistrement `MX`:

```zone
@       MX      10 alma-srv
```

On indique ici que pour envoyer des mail à `truc@adsillh.local`, il faut se
connecter au serveur `alma-srv` (donc `alma-srv.adsillh.local`).

À noter qu'on ne peut pas faire pointer un enregistrement `MX` vers un `CNAME`
(oui, c'est dommage, cela devrait pouvoir fonctionner, mais ce n'est pas prévu).

On déclare également des alias pour les serveurs que nos utilisateurs vont utiliser:

```zone
smtp    CNAME     alma-srv
imap    CNAME     alma-srv
pop3    CNAME     alma-srv
```

## Firewall

Il faut enfin ouvrir les ports sur le firewall vers tout Internet (et pas
seulement la zone `work`).

On commence par indiquer que notre carte réseau vers Internet est dans la zone publique: on ne fait pas confiance à ce qui en vient a priori.

```shell
# firewall-cmd --zone=public --change-interface=enp0s3 --permanent
```

On ouvre les ports pour que le reste d''Internet puisse nous envoyer des mails
(25 et 465)

```shell
# firewall-cmd --zone=public --add-service=smtp --permanent
# firewall-cmd --zone=public --add-service=smtps --permanent
```

On ouvre les ports pour que nos utilisateurs puissent récupérer leurs mails
(993 et 995):

```shell
# firewall-cmd --zone=public --add-service=imaps --permanent
# firewall-cmd --zone=public --add-service=pop3s --permanent
```

Et on ouvre les ports pour que nos utilisateurs puissent émettre des mails, que
notre serveur s'occupera d'envoyer vers le reste d'Internet (587)

```shell
# firewall-cmd --zone=public --add-service=smtp-submission --permanent
```

On ouvre également sur notre réseau:

```shell
# firewall-cmd --zone=work --add-service=smtp --permanent
# firewall-cmd --zone=work --add-service=smtps --permanent
# firewall-cmd --zone=work --add-service=pop3s --permanent
# firewall-cmd --zone=work --add-service=imaps --permanent
# firewall-cmd --zone=work --add-service=smtp-submission --permanent
```

Si on a vraiment confiance que personne sur le réseau ne s'amuse à écouter
les mots de passe, on peut ouvrir les ports non chiffrés pour récupérer les
mails (143 et 110):

```shell
# firewall-cmd --zone=work --add-service=imap --permanent
# firewall-cmd --zone=work --add-service=pop3 --permanent
```

mais surtout pas sur la zone `public`: ne jamais laisser les employés de
l'entreprise se connecter au serveur de mail sans chiffrement !

Enfin on applique les règles du firewall que l'on vient d'enregistrer:

```shell
# firewall-cmd --reload
```

# Exercice 2: SMTP avec Postfix

Commençons par installer de quoi recevoir et envoyer des mails.

```shell
# dnf install postfix
```

## Certificats

Postfix nous a déjà généré une paire de clés de chiffrement + certificat:

```
# ls -l /etc/pki/tls/private/postfix.key /etc/pki/tls/certs/postfix.pem
```

On constate que la clé privée est effectivement lisible que par `root`.

## Configuration

Dans la configuration, le mail ne fonctionne qu'à l'intérieur de notre
serveur. Il faut dire au serveur de s'ouvrir vers l'extérieur, dans
le fichier `/etc/postfix/main.cf`.

Retrouvez les lignes `mydestination`, pour activer celle qui inclut `$mydomain`
(ici, `$mydomain` sera remplacé automatiquement par `adsillh.local`)

Retrouvez les lignes `inet_interfaces`, pour activer celle qui utilise `all`,
pour que le serveur écoute vers l'extérieur.

Retrouvez les lignes `mynetworks`, définissez-le à `192.168.56.0/24,
127.0.0.0/8, [::1]/128` pour que le serveur fasse confiance aux machines
de notre réseau, pour qu'elles puissent envoyer du mail sans avoir à
s'authentifier.

Retrouvez les lignes `home_mailbox`, pour activer celle qui utilise `Maildir/`, pour
définir où déposer les mails reçus. 

Enfin, on va utiliser un mapping entre les adresses mails et les comptes
utilisateur unix de notre VM serveur: ajoutez à la fin du fichier

```
virtual_alias_maps = hash:/etc/postfix/virtual
```

et ajoutez au fichier `/etc/postfix/virtual` deux alias mail qu'on
va tous deux faire pointer vers notre compte Unix `admin`:

```
test@adsillh.local admin
admin@adsillh.local admin
```

Il faut mettre à jour le fichier `/etc/postfix/virtual.db` qui est juste à côté avec

```shell
# postmap /etc/postfix/virtual
```

## Premiers tests

On peut commencer à tester notre serveur. Ci-dessous, les lignes commençant
par `2xx` sont des réponses du serveur, il ne faut pas les taper. Le point est
bel est bien à taper tout seul sur sa ligne.

```shell
# systemctl enable --now postfix
# telnet localhost 25
220 alma-server.adsillh.local ESMTP Postfix
ehlo myname
250-alma-server.adsillh.local
250-PIPELINING
250-SIZE 10240000
250-VRFY
250-ETRN
250-STARTTLS
250-ENHANCEDSTATUSCODES
250-8BITMIME
250-DSN
250 SMTPUTF8
mail from:<toto@example.com>
rcpt to:<admin@adsillh.local>
data
Hello!  Ceci est un test !
.
250 2.1.0 Ok
250 2.1.5 Ok
354 End data with <CR><LF>.<CR><LF>
250 2.0.0 Ok: queued as 289EB10948EA
quit
221 2.0.0 Bye
```

On commence par dire bonjour (`ehlo`, version étendue de la commande
`helo`). Puis on indique qu'on a un mail pour `admin@adsillh.local`. On donne le
corps du message, et on dit au-revoir.

Vérifiez que vous pouvez également tester cela depuis votre session cremi. En
fait cela pourrait fonctionner depuis partout sur Internet, pourvu que l'adresse
destination soit en `@adsillh.local`.

On peut vérifier qu'`admin` a effectivement reçu le mail:

```shell
# ls /home/admin/Maildir/new/
# cat /home/admin/Maildir/new/1726531029.Vfd00I109a01eM984214.alma-server.adsillh.local
```

Vous pouvez lire ce mail en mode un peu hack3rs. Installez le package `mailx`. Vous pouvez alors utiliser

```shell
# mail -f Maildir
```

Dans l'exercice suivant on met en place IMAP/POP3 pour pouvoir lire les mails
depuis une autre machine et avec un client mail plus évolué.

## (bonus) Vers le reste du monde

Réessayez d'envoyer un mail, cette fois vers votre adresse `@etu.u-bordeaux.fr`. Cela devrait fonctionner.

Essayez vers une adresse qui n'est pas en `u-bordeaux.fr` (mais en utilisant dans from votre adresse `u-bordeaux.fr`).

Cela ne fonctionne pas, vous ne recevez rien ! Vérifiez la file des mails en
attente avec

```shell
# mailq
```

Il nous indique par exemple:

```
       (connect to mail.aquilenet.fr[2a0c:e300::1]:25: Network is unreachable)
```

Il a essayé de se connecter au MX pour `@aquilenet.fr`, mais le Cremi ne laisse pas faire, il ne laisse les connexion vers le port 25 qu'à l'intérieur de l'université... Il nous faut utiliser le `smarthost` de l'université.

Retrouvez les lignes `relayhost`, et indiquez-y `smtp.u-bordeaux.fr`

Redémarrez postfix, constatez que le mail est bien parti.

Regardez à la fin de `/var/log/maillog` pour avoir le détail de ce qui s'est passé: il est passé par le smarthost. Il est très utile pour pouvoir débugguer !


# Exercice 3: IMAP/POP3 avec Dovecot

Pour que les utilisateurs puissent récupérer leurs mails depuis leur
ordinateur n'importe où sur Internet, il faut ouvrir des services IMAP/POP3. On
utilise ici `dovecot`, qui a la bonne idée d'arriver tout pré-configuré:

```shell
# systemctl enable --now dovecot
```

## Premiers tests

On peut tester à la main:

```shell
# telnet localhost 110
Trying ::1...
Connected to localhost.
Escape character is '^]'.
+OK Dovecot ready.
user admin
+OK
pass toto
+OK Logged in.
stat
+OK 1 465
list
+OK 1 messages:
1 465
.
retr 1
+OK 465 octets
Return-Path: <toto@example.com>
X-Original-To: admin@adsillh.local
[...]
Hello!  Ceci est un test !
.
dele 1
+OK Marked to be deleted.
quit
+OK Logging out, messages deleted.
Connection closed by foreign host.
```

Avec `stat` on voit qu'il y a un mail.

Avec `list` on récupère la liste des mails.

Avec `retr` on récupère un mail.

Avec `dele` on supprime un mail.

Vous pouvez voir que le mail a effectivement disparu de `Maildir`

On pourrait faire de même en IMAP, qui permet aussi de changer de dossier.

# Exercice 4 (bonus) : un vrai client mail

Sur votre VM client, installez les packages `mutt` et `nano`

Repassez en tant que simple `admin`, et créez le fichier `.muttrc` expliquant
à mutt votre situation:

```
# Adresse électronique de l'expéditeur
set from = "admin@adsillh.local"

# Nom complet de l'expéditeur
set realname = "Moi-même"

# Comment éditer les mails
set editor = "nano"

# On s'identifie dès le lancement de Mutt
set spoolfile="imaps://admin@imap.adsillh.local/INBOX"
# On fixe la boite de réception
set folder="imaps://imap.adsillh.local/INBOX"

# On envoie les mails via notre serveur aussi
set smtp_url = "smtp://smtp.adsillh.local:25"

# Activer le chiffrement autant que possible
set ssl_starttls = yes
```

## Lecture de mails

Lancez `mutt`. Il vous dit qu'il ne connait pas le certificat de votre
serveur. On applique le principe `TOFU` (Trust On First Use, on en reparlera en
cours de réseau) en acceptant toujours.

Une fois le mot de passe saisi, la liste des mails apparait. Si vous n'en avez
plus (parce que supprimé précédemment), renvoyez-en un de nouveau, et tapez
`c!` `entrée` pour recharger la liste des mails.

## Écriture de mails

Tapez `m`

Il vous demande l'adresse destination, indiquez `admin@adsillh.local`, le
sujet importe peu. Tapez un peu de blabla dans nano et quittez-le. Mutt vous
redonne le détail, tapez `y` pour valider l'envoi. Tapez `c!` `entrée` pour
rafraîchir, le mail est arrivé !

Mutt a envoyé le mail sur le port 25 du serveur de mail.

## Autres clients

Vous pouvez aussi essayer de configurer thunderbird dans votre session Cremi.

# Exercice 5 (bonus): SASL pour authentification auprès du serveur de mails

Quand mutt a envoyé son mail, il a pu le faire car il est sur le réseau du serveur de mail, qu'on avait indiqué dans `mynetworks`. Quand vous utilisez thunderbird depuis votre session Cremi, c'est encore le cas (il utilise la adresse `192.158.56.1`).

Mais si vous êtes complètement ailleurs dans le monde, il vous faut utiliser
le port 587 et vous authentifier auprès de votre serveur de mail.

Pour se connecter en pop3/imap, on a utilisé notre login/password unix `admin`,
en effet dovecot a cela configuré par défaut.  postfix n'a cependant pas cela par défaut. Le port 587 n'est même pas ouvert par défaut en fait.

## Ouvrir le port 587

Dans `master.cf`, il faut décommencer les lignes suivantes:

```
submission inet n       -       n       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_auth_only=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=$mua_client_restrictions
  -o smtpd_helo_restrictions=$mua_helo_restrictions
  -o smtpd_sender_restrictions=$mua_sender_restrictions
  -o smtpd_recipient_restrictions=
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

et on avait déjà ouvert le port dans le firewall

## Configurer l'authentification

Le plus simple est de dire à postfix d'utiliser dovecot, qui sait déjà faire l'authentification.

Il faut commencer par configurer dovecot: dans
`/etc/dovecot/conf.d/10-master.conf`, décommentez les lignes

```
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
  }
```

et dans `/etc/dovecot/conf.d/10-auth.conf`, complétez la ligne

```
auth_mechanisms = plain login
```

et relancez dovecot.

On peut maintenant configurer postfix: dans `main.cf`, ajoutez à la fin

```
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
```

et relancez postfix.

## Côté mutt

Sur la VM client il faut installer le package `cyrus-sasl-plain`

et corriger l'url dans `.muttrc` pour utiliser le port 587 et indiquer l'utilisateur à utiliser:

```
set smtp_url = "smtp://admin@smtp.adsillh.local:587"
```

Réessayer d'envoyer un mail, il vous demandera juste le mot de passe pour s'authentifier.

## À la main !

On pourrait vouloir tester avec `telnet`, mais le chiffrement à la main, c'est dur :)

Sur la VM client, installez le package `gnutls-utils`

Vous pouvez alors vous connecter sur le port 587 ainsi:

```shell
$ gnutls-cli --no-ca-verification --starttls-proto=smtp --port 587 192.168.56.10
```

Il s'occupe de voir le début de la négociation SMTP et d'utiliser STARTTLS et
négocier les clés. On désactive ici la vérification du CA, on en reparlera
plus tard en réseau.

On peut s'authentifier, mais il faut se préparer un peu: il faut encode en base64 login et mot de passe:

```shell
$ echo -n admin | base64
YWRtaW4=
$ echo -n toto | base64
dG90bw==
```

On peut alors discuter avec le serveur:

```
ehlo myname
250-alma-server.adsillh.local
250-PIPELINING
250-SIZE 10240000
250-VRFY
250-ETRN
250-AUTH PLAIN LOGIN
250-ENHANCEDSTATUSCODES
250-8BITMIME
250-DSN
250 SMTPUTF8
auth login
334 VXNlcm5hbWU6
YWRtaW4=
334 UGFzc3dvcmQ6
dG90bw==
235 2.7.0 Authentication successful
```

Et la suite est du smtp habituel.

Si vous vous posiez la question de ce qu'il baragouine avec 334, c'est simplement encodé en base64:

```
$ echo VXNlcm5hbWU6 | base64 -d
Username:
$ echo UGFzc3dvcmQ6 | base64 -d
Password:
```
