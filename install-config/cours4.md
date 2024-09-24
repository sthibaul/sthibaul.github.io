Cours 4: LDAP
=============

# Annuaires électroniques (NIS/YP, LDAP, Active Directory)

Typiquement, gérer les utilisateurs:

- Identifier: qui est-ce (nom, prénom, ...)
- Authentifier: vérifier que c'est bien cette personne (mot de passe, clé publique ssh/gpg, ..)
- Autoriser: quel droit a-t-elle

De manière centralisée:

- Ne pas avoir à configurer chaque poste
- Pouvoir facilement sauvegarder cette information
- Sans pour autant devenir un SPOF (Single Point Of Failure)
  - Redondance

Mais pas que des utilisateurs:

- Peut être aussi des groupes d'utilisateurs, des machines, ...
- Essentiellement, un recensement

-> fonctionnement générique
-> un peu abstrait...

# Vocabulaire LDAP

* dc = Domain Component: essentiellement comme un nom de domaine
* ou = Organizational Unit: type de resource recensée
* cn = Common Name: nom d'une resource recensée

```
            dc=fr
              |
          dc=exemple
         /          \
   ou=machines    ou=personnes
       /              \      \
cn=serveur5         cn=Jean  cn=Jacques
```

En assemblant le tout, on obtient un dn (*distinguished name*) qui peut donc désigner une personne, une machine, un groupe, ...:

```ldap
cn=serveur5,ou=machines,dc=exemple,dc=fr
```

Une *entrée* (ou *object*) dans la BASE ldap, désignée par le dn, est alors une fiche
contenant les diverses informations utiles, par exemple:

```ldif
dn: cn=Jean,ou=personnes,dc=exemple,dc=fr
cn: Jean
givenName: Jean
sn: Lapin
telephoneNumber: +33 12 34 56 78
telephoneNumber: +33 12 34 56 79
manager: cn=Jacques,ou=personnes,dc=exemple,dc=fr
mail: jean@exemple.fr
uid: jean
uidNumber: 1000
gidNumber: 1000
userPassword:: JHkkajlUJFZRZk0vQ0FUYzdOLklSTFp6QUhqeTAkclZwREl5ekUveU5jM2JSRjcyQXR2T0dGVzFuRXhndFh1a0VOY3EveE1tQQoK
loginShell: /bin/zsh
homeDirectory: /home/jean
objectClass: top
objectClass: person
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: posixAccount
```

# Schémas

Beaucoup d'informations dans un objet LDAP !

Essentiellement des attributs sous forme de clé-valeur, mais comment s'y
retrouver, comment est-on censé nommer les attributs ? Grâce aux schémas.

Ajouter `objectClass: truc` indique que la fiche respecte le schéma `truc`.

Chaque schéma précise quels attributs doivent (ou peuvent) être renseignés.
Par exemple `posixAccount`:

```shell
$ grep -r posixAccount /etc/ldap/schema
/etc/ldap/schema/nis.schema:objectclass ( 1.3.6.1.1.1.2.0 NAME 'posixAccount'
$ less /etc/ldap/schema/nis.schema
[...]
objectclass ( 1.3.6.1.1.1.2.0 NAME 'posixAccount'
        DESC 'Abstraction of an account with POSIX attributes'
        SUP top AUXILIARY
        MUST ( cn $ uid $ uidNumber $ gidNumber $ homeDirectory )
        MAY ( userPassword $ loginShell $ gecos $ description ) )
```

On y voit qu'il est obligatoire d'avoir un `cn` (pas très étonnant), un
`uid`, un `uidNumbmer`, un `gidNumber`, et un `homeDirectory`, pas très
étonnant pour un compte unix.

On voit qu'il est possible de préciser le `userPassword` (pour pouvoir
authentifier), mais aussi le `loginShell` par exemple.

Un objet respecte typiquement plusieurs schéma, pour répondre à différents
besoins (compte unix, personne, organisation dans l'entreprise, ...)

# Requêtes

Une fois qu'on a créé différents objets dans notre base LDAP, on peut faire des requêtes, un peu comme SQL:

```shell
# ldapsearch -H ldapi:/// -Y EXTERNAL -b dc=exemple,dc=fr uid=jean
dn: cn=Jean,ou=personnes,dc=exemple,dc=fr
cn: Jean
[...]
```

L'option `-H` précise à quel serveur ldap on s'adresse (ici, la machine locale).

L'option `-Y` précise comment on s'authentifie (ici, on est root donc c'est immédiat avec le choix `EXTERNAL`).

L'option `-b` précise dans quelle partie de ldap on veut chercher.

`uid=jean` est le critère de recherche.

On aurait aussi pu mettre par exemple `telephoneNumber=+33 12 34 56 78" pour retrouver qui possède ce numéro de téléphone.

Ou mettre `mail=jean@exemple.fr` pour vérifier s'il y a quelqu'un qui possède cette adresse mail (et ensuite regarder `homeDirectory` pour savoir où déposer un mail reçu)

# Authentification, "bind"

Ci-dessus, on a utilisé le fait d'être root pour gagner le droit à interroger ldap. C'est un peu triché...

Et quand on aura plusieurs serveurs (mail, web, ...) qui voudront interroger voire modifier la base ldap, on voudra les mettre sur des machines différentes.

On crée donc plutôt un objet, par exemple pour permettre à postfix d'accéder à ldap:

```ldif
dn: uid=postfix,ou=services,dc=exemple,dc=fr
objectClass: inetOrgPerson
cn: Postfix
sn: Postmaster
uid: postfix
userPassword:: JHkkajlUJHZRdVJlTjhiZUZJLmtaOWdFSG5TNy8kSzBpdUZUZmtBSmRvVnh2MC5MQ0hiRGpQS1NUbWtyay5kdzlzOWp0L2hFMQo=
```

On peut essayer de l'utiliser pour faire une requête:

```shell
$ ldapsearch -H ldaps://ldap.exemple.fr -x -W -D uid=postfix,ou=services,dc=exemple,dc=fr -b dc=exemple,dc=fr mail=jean@exemple.fr
Enter LDAP Password:
dn: cn=Jean,ou=personnes,dc=exemple,dc=fr
cn: Jean
[...]
```

Ici on a utilisé avec `-H` une connexion via TCP/IP, mais chiffrée (`ldaps`).

On a demandé une authentification simple (`-x -W`) en utilisant l'objet
ci-dessus pour s'authentifier (avec `-D`), et on a tapé le mot de passe.

Et on cherchait dans le champ `mail`.

Configurer un démon pour utiliser l'annuaire ldap, c'est donc essentiellement:

* Indiquer quel serveur LDAP contacter
* Quel objet LDAP utiliser pour s'authentifier
* Le mot de passe associé
* Quel champ utiliser pour chercher l'information

# Outils de gestion

On peut faire tout à la main avec `ldapsearch`, `ldapadd`, `ldapdelete`,
`ldapmodify`, c'est très pratique pour scripter des opérations.

On peut aussi utiliser des surcouches graphiques du genre `phpLDAPadmin` (PLA).
