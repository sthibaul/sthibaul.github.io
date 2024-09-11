Cours 2: DHCP & DNS
===================

# DHCP

## Dynamic Host Configuration Protocol.

Configuration automatique des machines du réseau

* Attribuer une adresse IP

Mais aussi d'autres paramètres

* IP de la passerelle vers Internet
* IP d'un serveur DNS
* Nom de domaine local
* Le fuseau horaire
* IP d'un serveur d'impression
* Nom d'un serveur TFTP pour démarrage automatique PXE (installation automatique de machine)
* Serveur kickstart
* ...

En général ces paramètres sont fixes (enregistrés en dur dans `dhcpd.conf`)

Pour l'attribution d'IP par contre non :)

* Une IP piochée dans un pool d'adresses
* Avec un bail de durée limitée

## Principe

* Le client envoie un `discover` en broadcast
* Un serveur répond
* Le client lui demande une IP
* Le serveur confirme

## Quid si le serveur DHCP tombe en panne ?

-> redonder avec un deuxième serveur

*Attention*: avoir des pools séparés

# DNS

## Domain Name System

Traduire les noms de domaine en adresses IP et inversement.

La traduction se fait de droite à gauche: pour `fr.wikipedia.org.`

* On demande à `.` le serveur DNS pour `.org.`
* On demande au serveur DNS de `.org` le serveur DNS pour `wikipedia.org.`
* On demande au serveur DNS de `wikipedia.org` l'adresse de `fr.wikipedia.org.`
* Il répond `185.15.58.224` et `2a02:ec80:600:ed1a::1`

On en reparlera plus précisément en cours de réseau

## Zone

Conséquence: pour notre domaine `adsillh.local.`, on est responsable de résoudre tous les `*.adsillh.local.`

D'où notre fichier de zone `adsillh.local`:

```





; date file for zone adsillh.local
$TTL 86400      ; 1 day
@           IN SOA  adsillh.local. root.adsillh.local. (
                                2024090101 ; serial
                                10800      ; refresh (3 hours)
                                3600       ; retry (1 hour)
                                604800     ; expire (1 week)
                                38400      ; minimum (10 hours 40 minutes)
                                )
$TTL 3600       ; 1 hour
@                    NS      alma-server
alma-server          A       192.168.56.10
```

Attention au `serial`: il faut le mettre à jour quand on change la zone !

Après le SOA, une ligne = un enregistrement (record)

* Une traduction nom de domaine -> IPv4

```alma-server          A       192.168.56.10```

* Une traduction nom de domaine -> IPv6

```alma-server          AAAA       fec0::1```

* Un alias (appelé Canonical Name)

```myserver             CNAME       alma-server```

* L'adresse d'un serveur de mail pour le domaine

```@                    MX          10 alma-server```

Nota:

* `@` désigne la zone elle-même (ici `adsillh.local.` donc)
* Un `.` à la fin indique que le nom de domaine est complet (plutôt que un sous-domaine de la zone courante):
  * `truc.bidule` désigne `truc.bidule.adsillh.local.`
  * `truc.bidule.` désigne `truc.bidule.`

Avez-vous pensé à mettre à jour le `serial` ?

Il y a des tas d'autres types d'enregistrements:

* La position géographique

```alma-server          LOC         44 52 N 0 33 W 20m```

* Le serveur DNS pour un sous-domaine

```perso                NS          ns.perso```

* Le type SPF pour la validation des mails (on en reparle plus tard)

```@                    SPF         "v=spf1 a -all"```

Au fait, je vous ai parlé du champ `serial` ?

* Le type TXT, complètement fourre-tout.

```@                    TXT         "printer=lpr5"```

Tous les admins ont un jour oublié de mettre à jour le champ `serial`.

## Un exemple

```
@                    NS      10 alma-server
@                    MX      10 alma-server2
alma-server          A       192.168.56.10
alma-server          AAAA    fec0::10
alma-server2         A       192.168.56.11
alma-server2         AAAA    fec0::11
webmail              CNAME   mail-server2
```

## Zone inverse

On est aussi responsable de la traduction d'IP en nom de domaine.

On renseigne la résolution dans le domaine dédié `in-addr.arpa.`

On fait aussi la résolution de droite à gauche (du plus général au plus spécifique)

-> il faut retourner l'adresse IP !

-> pour `192.168.56.10` il faut renseigner `10.56.168.192.in-addr.arpa.`

Dans notre cas en TP on a configuré une zone `56.168.192.in-addr.arpa.`, et l'on a mis les enregistrements

```
$TTL 86400      ; 1 day
@       IN SOA  adsillh.local. root.adsillh.local. (
                                14         ; serial
                                10800      ; refresh (3 hours)
                                3600       ; retry (1 hour)
                                604800     ; expire (1 week)
                                38400      ; minimum (10 hours 40 minutes)
                                )
$TTL 3600       ; 1 hour
@          NS    alma-server.adsillh.local.
10         PTR   alma-server.adsillh.local.
11         PTR   alma-server2.adsillh.local.
```

Là aussi il faut mettre à jour le `serial` quand on modifie la zone.

## TTL, mise à jour, ~~propagation~~

### TTL

`$TTL` (Time To Live) indique la durée de vie des enregistrements, en secondes.

Pour que votre serveur DNS ne soit pas interrogé en permanence, les clients DNS
gardent en cache l'information pendant la durée du TTL

TTL grand = DNS moins chargé.

TTL grand = si on change un enregistrement, les clients DNS vont quand même encore garder l'ancien enregistrement pendant longtemps.

### Mise à jour

-> Si on prévoit de migrer un site web vers un autre serveur, commencer par diminuer le TTL.

-> Le jour où l'on fait la migration, les clients basculeront progressivement pendant le temps de la TTL.

### Nouveau nom de domaine

Si on ajoute un nouveau nom de domaine, tester d'abord en interrogeant directement son DNS.

Si on teste via un cache DNS et que l'on s'était trompé, ce cache va rester
"pollué" avec la mauvaise réponse pendant tout le temps du TTL.

### ~~propagation~~

L'information DNS ne se "propage" *pas* sur Internet: l'information DNS se *périme* toute seule au bout du TTL.

## Quid si le serveur DNS tombe en panne ?

Utiliser un Maître et un ou plusieurs Esclaves

* On configure sur le maître
* Le maître envoie la nouvelle configuration aux esclaves
* Les deux peuvent répondre

Avez-vous pensé à mettre à jour le `serial` ? Sinon les esclaves ne seront pas mis à jour.

# DHCP + DNS

Mettre à jour DNS avec la bonne IP est un peu fastidieux.

Quand une machine obtient une IP dynamiquement, il faut mettre à jour le DNS dynamiquement !

-> DHCP notifie DNS des baux
