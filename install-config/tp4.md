TP4: installation LDAP
======================

On continue toujours avec nos VMs serveur et client.

Pour l'instant on travaille seulement sur la VM serveur.

# Exercice 1: Préparer le terrain

Avant d'installer et configurer le serveur de mail, préparons le terrain.

## Configurer le DNS

Ajoutez un alias DNS `ldap` pour votre serveur, qu'on utilisera pour indiquer le
serveur ldap à utiliser.

Avez-vous pensé à mettre à jour le serial ?

Vérifiez que l'alias fonctionne.

## Firewall

Dans notre cas on va commencer par interroger en local, donc le firewall ne
gênera pas au début.

## Ajouter le dépôt EPEL

LDAP n'est pas fourni dans les packages de base d'AlmaLinux, il faut activer le dépôt EPEL et activer la partie powertools:

```shell
# dnf install epel-release
# dnf config-manager --set-enabled powertools
```

Essentiellement, cela dépose dans `/etc/yum.repos.d` des bouts de configuration
pour ajouter l'adresse du dépôt EPEL, ainsi qu'une clé GPG pour vérifier les
signatures des packages récupérés depuis ce dépôt.

# Exercice 2: openldap

## Installation de base

Commençons par installer le serveur et les clients LDAP:

```shell
# dnf install openldap-servers openldap-clients
```

Et activons le serveur (qui s'appelle en fait slapd)

```shell
# systemctl enable --now slapd
```

La configuration d'openldap ne se fait plus dans des fichiers, il faut utiliser
les outils ldap (vous trouverez peut-être encore des tutoriels sur Internet qui
font modifier des fichiers de configuration de slapd, ce n'est plus valide)

Commençons par injecter les classes que l'on utilisera:

```shell
# ldapadd -H ldapi:/// -Y EXTERNAL -f /etc/openldap/schema/cosine.ldif
# ldapadd -H ldapi:/// -Y EXTERNAL -f /etc/openldap/schema/nis.ldif
# ldapadd -H ldapi:/// -Y EXTERNAL -f /etc/openldap/schema/inetorgperson.ldif
```

## Configuration

Regardons la configuration actuelle, qui est placée dans une partie `cn=config`
qui n'est pas affichée par défaut:

```shell
# ldapsearch -H ldapi:/// -Y EXTERNAL -b 'olcDatabase={2}mdb,cn=config' 
```

On peut y voir que le domaine utilisé est actuellement `my-domain.com`, on
voudrait changer cela. Pour cela il faut modifier les attributs correspondant de
l'objet `'olcDatabase={2}mdb,cn=config'`. On pourrait taper tout en interactif,
mais le risque de typo est grand. Il vaut donc mieux préparer un fichier
`todo.ldif` contenant les modifications voulues:

```ldif
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=adsillh,dc=local

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=Manager,dc=adsillh,dc=local
```

Le format est un peu verbeux...

Avec `dn` on indique l'objet à modifier.

Avec `changetype` on indique si on veut modifier (`modify`) un objet existant ou en ajouter (`add`) ou en enlever (`delete`). Ici on modifie un objet existant.

Avec `replace` on indique qu'on veut remplacer un attribut existant (on pourrait aussi en ajouter (`add`) ou enlever (`delete`)).

Et enfin on donne la nouvelle valeur de l'attribut.

Ici on fait cela deux fois. On pourrait aussi faire une seule opération
contenant deux modifications du même objet en séparant avec `-` plutôt qu'une
ligne vide:

```ldif
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=adsillh,dc=local
-
replace: olcRootDN
olcRootDN: cn=Manager,dc=adsillh,dc=local
```

Une fois le fichier `todo.ldif` préparé au propre, on peut l'appliquer à la base:

```shell
# ldapmodify -H ldapi:/// -Y EXTERNAL -f todo.ldif
```

On peut revérifier la configuration avec la commande `ldapsearch` pour constater que cela a bien eu effet.

Un autre point intéressant à noter dans la configuration, c'est l'attribut
`olcDbIndex` qui indique sur quels attributs les objets sont indexés,
c'est-à-dire que les requêtes de recherche seront très efficaces sur ces
attributs. De manière peu étonnante, on retrouve ce qui est typiquement
cherché: `cn` et `mail`.

## Permissions

Pour se simplifier la vie, on va permettre à root de modifier la base, et aux
autres de juste lire, sauf le mot de passe:

```ldif
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to attrs=userPassword by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" write by anonymous auth by * none
olcAccess: {1}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" write by * read
```

La première règle (`{0}`) précise d'abord les droits pour l'attribut
`userPassword`: pour root (uid 0) il a droit d'écriture (et donc lecture
aussi), pour les clients non encore connectés il peut servir à s'authentifier,
et pour les autres ils ne peuvent rien en faire.

La deuxième règle (`{1}`) préciser les droits pour les autres attributs: root
a le droit d'écriture, les autres ont le droit de lecture.

On peut injecter, et l'on peut vérifier que cela a bien été mis à jour.

# Exercice 3: Créer une hiérarchie de base

## Base de la hiérarchie

Maintenant que l'on peut écrire dans la base LDAP, créons une simple
hiérarchie `dc=adsillh,dc=local` avec juste des utilisateurs dedans. On
prépare le fichier `todo.ldif` pour ajouter les objets conteneurs:

```ldif
dn: dc=adsillh,dc=local
objectClass: top
objectClass: dcObject
objectclass: organization
o: LPro ADSILLH
dc: adsillh

dn: ou=Etudiants,dc=adsillh,dc=local
objectClass: organizationalUnit
ou: Etudiants
```

Ici on n'a pas mis de `changetype`, car l'on injecte plutôt avec `ldapadd` qui est fait pour créer des objets:

```shell
# ldapadd -H ldapi:/// -Y EXTERNAL -f todo.ldif
```

## Un utilisateur

On peut ajouter un objet pour représenter un étudiant:

```ldif
dn: cn=Toto Lapin,ou=Etudiants,dc=adsillh,dc=local
cn: Toto Lapin
givenName: Toto
sn: Lapin
uid: toto
uidNumber: 10000
gidNumber: 10000
homeDirectory: /home/toto
mail: toto@adsillh.local
objectClass: top
objectClass: person
objectClass: inetOrgPerson
objectClass: posixAccount
```

On a donné à l'étudiant la classe `posixAccount` pour remplir tous les champs
utiles à ce qu'il ait un compte unix, et la classe `inetOrgPerson` pour remplir
le champ `mail` (i.e. qu'il "existe" sur Internet (`inet organization`)).

Retrouvez la définition de la classe `inetOrgPerson` dans
`/etc/openldap/schema/*.schema`, constatez que la classe permet d'inclure
beaucoup d'informations, mais n'en exige aucune :)

On peut vérifier l'ajout en regardant toute la base:

```shell
# ldapsearch -H ldapi:/// -Y EXTERNAL -b dc=adsillh,dc=local
```

On peut voir les opérations dans le log:

```shell
# journalctl -u slapd
```

Dans l'exercice suivant, on va vouloir retrouver les étudiants à partir de
leur `uid`, ce sera donc intéressant d'indexer la base sur l'attribut `uid`.

Retournez voir la configuration de LDAP, on y voit l'indexation activée actuellement:

```
olcDbIndex: objectClass eq,pres
olcDbIndex: ou,cn,mail,surname,givenname eq,pres,sub
```

Modifiez donc cet attribut `olcDbIndex` pour ajouter `uid` à côté de `givenname`.

# Exercice 4: authentification par LDAP

On veut maintenant utiliser LDAP pour gérer les utilisateurs Unix

RedHat utilise SSSD (System Security Services Daemon) pour gérer les
utilisateurs. Il faut installer la partie ldap:

```shell
# dnf install sssd-ldap oddjob-mkhomedir
```

Et créer un fichier de configuration `/etc/sssd/sssd.conf` pour configurer l'utilisation de LDAP (oui, les `%2f` sont faits exprès) :

```
[domain/default]
id_provider = ldap
autofs_provider = ldap
auth_provider = ldap
chpass_provider = ldap
ldap_uri = ldapi://%2fvar%2frun%2fldapi
ldap_search_base = dc=adsillh,dc=local

[sssd]
services = nss, pam, autofs
domains = default

[nss]
homedir_substring = /home
```

Il faut protéger le fichier contre la lecture sinon `sssd` n'en veut pas:

```shell
# chmod 600 /etc/sssd/sssd.conf
```

On peut démarrer sssd et activer l'authentification via sssd:

```shell
# systemctl restart sssd
# authselect select sssd with-mkhomedir --force 
```

On peut vérifier que notre utilisateur est reconnu désormais, avec les uid/gid
que l'on avait choisis:

```shell
# id toto
```

On peut lui donner un mot de passe

```shell
# ldappasswd -H ldapi:/// -Y EXTERNAL -S cn="Toto Lapin",ou=Etudiants,dc=adsillh,dc=local
```

Et l'on peut se logguer en tant que `toto` ! Sauf qu'il n'a pas encore de
`/home/toto`, activons le service qui peut le créer à la volée:

```shell
# systemctl enable --now oddjobd
```

En se reloggant, cette fois on a bien un home !

L'utilisateur peut maintenant aussi lui-même se connecter à ldap pour lire des informations:

```shell
$ ldapsearch -H ldapi:/// -x -W -D cn="Toto Lapin",ou=Etudiants,dc=adsillh,dc=local -b dc=adsillh,dc=local
```

Ajoutez un autre utilisateur `tata` dans LDAP, ayant le même `gidNumber` mais
un `uidNumber` différent (et avec le login et prénom corrigés), constatez que
son compte Unix `tata` est disponible immédiatement.

Constatez que l'on peut récupérer l'un des deux utilisateurs seulement en utilisant un filtre:

```shell
# ldapsearch -H ldapi:/// -Y EXTERNAL -b dc=adsillh,dc=local uid=toto
```

Essayez un autre filtre, essayez de filtrer sur `gidNumber` pour constater que
vous récupérez bien les deux.

Essayons de changer la fiche de l'utilisateur `toto`: modifiez l'attribut
gidNumber pour y mettre 20000. Relancez `id toto`, constatez que cela n'a pas
changé ! En effet `sssd` utilise un cache pour éviter de solliciter le
serveur LDAP en permanence. On peut vider le cache avec `sss_cache -E` et `id
toto` devrait désormais donner la réponse mise à jour.

Appelez `id tata` , puis supprimez l'objet LDAP de `tata`, et appelez `id tata`
de nouveau. Essayez de vous logguer avec. Le cache ne voit pas la suppression
non plus, flushez-le pour forcer la disparition du compte unix. Mais son home
lui n'est pas supprimé (c'est rarement une bonne idée de supprimer des
données automatiquement de toutes façons)

# Exercice 5 (bonus): TLS

On a utilisé une connexion locale `ldapi` pour se simplifier la vie. Mais pour
que l'utilisateur puisse être reconnu sur la VM client aussi, il faut utiliser
une connexion TCP/IP, et donc chiffrer pour éviter que les mots de passe soient
communiqués en clair.

## Configurer des clés

Contrairement à postfix/dovecot, les clés ne sont pas faites automatiquement.
Il faut les générer, les poser à un endroit, et s'assurer que `slapd` peut
lire la clé privée.

Pour les questions que pose `openssl` on peut laisser les valeurs par défaut
en validant tout avec juste `entrée`, *sauf* pour la question
`Common Name` qui *doit* être répondue avec `ldap.adsillh.local`

```shell
# cd
# openssl req -x509 -nodes -newkey rsa:2048 -keyout ldap.key -out ldap.crt -days 365
# mv ldap.key /etc/openldap/certs/
# mv ldap.crt /etc/openldap/certs/
# chown ldap:ldap /etc/openldap/certs/ldap.key
# restorecon /etc/openldap/certs/*
```

Et l'on peut modifier la configuration ldap:

```
dn: cn=config
changetype: modify
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/openldap/certs/ldap.crt
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/ldap.key
-
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/ldap.crt
```

Vérifiez bien que le `ldapmodify` ne produit pas d'erreur.

Dans `/etc/openldap/ldap.conf`, on configure la partie client LDAP. Sur votre VM serveur
mettez-le à jour pour pointer sur votre serveur ldap:

```
BASE    dc=adsillh,dc=local
URI     ldaps://ldap.adsillh.local
TLS_CACERT      /etc/openldap/certs/ldap.crt
```

## Tester

Vous devriez désormais pouvoir vous connecter en `ldap://` avec l'utilisateur
`toto`:

```shell
$ ldapsearch -H ldap://ldap.adsillh.local -x -W -D cn="Toto Lapin",ou=Etudiants,dc=adsillh,dc=local -b dc=adsillh,dc=local
```

Mais aussi en `ldaps://`

```shell
$ ldapsearch -H ldaps://ldap.adsillh.local -x -W -D cn="Toto Lapin",ou=Etudiants,dc=adsillh,dc=local -b dc=adsillh,dc=local
```

Il ne reste qu'à ouvrir le firewall:

```shell
# firewall-cmd --zone=work --add-service=ldaps --permanent
# firewall-cmd --reload
```

Passez sur la VM client,

* copiez-y la clé publique `ldap.crt` du serveur,
* Lancez `restorecon /etc/openldap/certs/*`
* configurez `ldap.conf` de la même façon que sur le serveur
* et testez-y `ldapsearch` (en ldaps seulement).

## Authentification par LDAP

Configurez l'authentification des utilisateurs Unix par LDAP sur la VM client aussi.

Cette fois pour la configuration `sssd.conf` il faut utiliser l'url
`ldaps://ldap.adsillh.local`

Vérifiez `id toto` et essayez de vous logguer en tant que toto

## Un nouveau venu

Ajoutez un nouvel utilisateur dans LDAP. Vérifiez qu'immédiatement il obtient
un compte Unix partout, et une adresse email.

# Exercice 6: Dovecot + LDAP

We will now connect Dovecot to our LDAP server, so that users can have an email
address without necessarily having a Unix account (that's the typical case of a
mail hosting service). In that case, we can configure `postfix` to give mails to
`dovecot` via `virtual_transport`, and configure `dovecot` to connect to LDAP
and translate LDAP attributes into mail configuration (where to put mails, which
password use for IMAP/POP3 authentication, etc.)

## Create LDAP users for daemons

In general one prefers to create a separate LDAP object in a separate `ou`, with
its own password, and add permissions for it to read the LDAP database.

Create a `ou=daemons,dc=adsillh,dc=local` LDAP object (like you created `ou=Etudiants`).

Create a `uid=dovecot,ou=daemons,dc=adsillh,dc=local` LDAP object (like we
created `Toto Lapin`), with only the `inetOrgPerson` class (so that it can just
have a `userPassword` field). You can use `Dovecot` as `cn` and `Mailbox` as
`sn`. Note that we have used `uid=dovecot` and not `cn=dovecot`, and
`ou=daemons` and not `ou=Etudiants`

Give a password to this object (like we gave one to `Toto Lapin`)

Note that this is a LDAP user, not a Unix user. To test that this LDAP user can indeed connect to the LDAP server, you can use

```shell
$ ldapsearch -H ldapi:/// -x -W -D uid=dovecot,ou=daemons,dc=adsillh,dc=local -b dc=adsillh,dc=local
```

You will notice that it cannot read the `userPassword` field, it's indeed a different LDAP user than the root user, with different privileges.

## Create vmail hosting

Even if we don't want to have a Unix account for each mail account, we need a
Unix account for hosting the mails, which we will call `vmail`. Let's create
this user and check that its home is completely protected:

```shell
# adduser --uid 5000 vmail
# ls -ld /home/vmail
```

## Add virtual mail support in postfix

Let's now tell postfix in `main.cf` to pass mails to dovecot instead of storing
them itself.

Look for the definition of `mydestination`, to enable the version that does
*not* include `$mydomain`

And add this at the end to tell postfix to just relay `@adsillh.local` mails to dovecot:

```
## Virtual mailbox settings
virtual_mailbox_domains = adsillh.local
virtual_transport = lmtp:unix:/run/dovecot/lmtp
```

## Add virtual mail support in dovecot

Let's now explain dovecot how to check LDAP and where to store incoming mails.

In `conf.d/10-auth.conf` we can simply enable including `auth-ldap.conf.ext` 

Read that last file, to see that it refers to `/etc/dovecot/dovecot-ldap.conf.ext`
which we can now create, to tell the dovecot ldap driver how to access LDAP:

```
# Version to use
ldap_version = 3
uris = ldapi:/// # Which server to connect to, here local

# which LDAP object & password to use
dn = uid=dovecot,ou=daemons,dc=adsillh,dc=local
dnpass = toto

# which part of LDAP to look into
base = ou=Etudiants,dc=adsillh,dc=local
scope = subtree

# What to check and how to translate
user_filter = (&(objectClass=inetOrgPerson)(mail=%u))
user_attrs = =home=/home/vmail/%{ldap:mail}

pass_filter = (&(objectClass=inetOrgPerson)(mail=%u))
pass_attrs = =password=%{ldap:userPassword}
```

In the filters, we have told that we look for objects that have the `MailUser`
class, and (`&`) that have the email address (`mail` attribute) that is being looked
for. We then tell dovecot to use the `mail` attribute from ldap
(`%{ldap:mail}`), and prepend `/home/vmail/` to obtain the `home` for the
user, where mails will be put into.  The password is directly taken from the
`userPassword` LDAP attribute.

In `conf.d/10-mail.conf` look for `mail_location`, and uncomment to force using `Maildir`:

```
mail_location = maildir:~/Maildir
```

also, set these to force using the `vmail` user for the maildir:

```
mail_uid = 5000
mail_gid = 5000
```

For security we also prefer to restrict dovecot to read/writing mails from there:

```
valid_chroot_dirs = /home/vmail
```

Last but not least, we also need to make SELinux allow `dovecot` to access
`/var/run/ldapi` to connect to LDAP (we could also use ldaps, but ldapi is more efficient and does not require setting up TLS):

```shell
# setsebool -P authlogin_nsswitch_use_ldap 1
```

## Test with a user with mail but no unix account

Create a `cn=Titi Lapin,ou=Etudiants,dc=adsillh,dc=local` LDAP object similarly
to what we did for other students, but *without* the `posixAccount` class, and
thus not the `homeDirectory`, `uid`, `uidNumber`, `gidNumber` attributes, but
give it `titi@adsillh.local` as `mail` attribute.

Try to send a mail to it, and look in `/home/vmail/titi@adsillh.local/`

If it is not there, you can check `mailq` to see it in the queue,
`journalctl -u postfix` to see if it's postfix that has troubles,
and `journalctl -u dovecot` to see if it's dovecot that has troubles.

After fixing something in the config, remember to restart the corresponding
daemon, and use `postqueue -f` to tell postfix to retry delivering the mail.

## Reading mail (Bonus)

If you try to run

```shell
# ldapsearch -H ldapi:/// -x -W -D uid=dovecot,ou=daemons,dc=adsillh,dc=local
```

You will see that we do not get the `userPassword` attributes. Indeed, we had
only permitted root to access them.

Modify the `olcAccess` attribute of `{2}mdb,cn=config` , to add

`by dn.base=uid=dovecot,ou=daemons,dc=adsillh,dc=local read`


Check `ldapsearch` again, now you should see the `userPassword` attributes.

Set a password for `cn="Titi Lapin"`

On the client VM, change `.muttrc` to use the `titi@adsillh.local` login instead
of `admin` (yes, there are now two `@`).

You should be able to read the mail you have sent.

Try to send a mail, that should work too!

## Aliases (Bonus)

Some users may want to have several email addresses that all end up in the same
mailbox. We can tell postfix to perform the rewriting through virtual mapping
before giving the mail to dovecot.

Let's first inject the definition for the `inetLocalMailRecipient` class that we
can use for this:

```
# ldapadd -H ldapi:/// -Y EXTERNAL -f /etc/openldap/schema/misc.ldif
```

Add to `cn=Titi Lapin` this:

```
objectClass: inetLocalMailRecipient
mailLocalAddress: tititi@adsillh.local
mailLocalAddress: titititi@adsillh.local
```

Create a `uid=postfix,ou=daemons,dc=adsillh,dc=local` LDAP object and give it a password.

We can now make postfix look for these aliases in addition to what we already
defined in `main.cf`:

```
virtual_alias_maps = hash:/etc/postfix/virtual, ldap:/etc/postfix/virtual_aliases.cf
```

And in `virtual_aliases.cf`:

```
## Postfix-LDAP settings - aliases
version = 3
server_host = ldapi:///

bind = yes
bind_dn = uid=postfix,ou=daemons,dc=adsillh,dc=local
bind_pw = toto

search_base = ou=Etudiants,dc=adsillh,dc=local
scope = sub

query_filter = (&(objectClass=inetLocalMailRecipient)(mailLocalAddress=%s))
result_attribute = mail
```

The meanings of the fields should now be very familiar :)

Note that we do not need to give postfix access to user passwords, that's even
better for security.

Restart postfix, try to send a mail to `tititi@adsillh.local`, you should see in
`maillog` that it is first rewritten to `titi@adsillh.local` before giving it to
dovecot for delivery. And `titititi@adsillh.local` should be working at well.

Change the `mail` attribute of `cn=Titi Latin` to your personal address. Try
to send a mail to `tititi@adsillh.local`, you should be receiving it :) That
allows to automatically forward mails to another email domain. You will however
of course need to add a `mailLocalAddress: titi@adsillh.local` for that address
to become valid again.

# Note: redondance

Il est possible d'installer un deuxième serveur LDAP. La mise en place de la
synchronisation est un peu délicate car il y a des parties de configuration
propres à chaque serveur, et le reste doit être répliqué. Une fois le
deuxième serveur LDAP, on peut pointer les clients vers les deux serveurs LDAP
pour obtenir la redondance.
