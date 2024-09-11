TP2: DHCP + DNS, redondance
===========================

On continue avec nos deux VMs serveur et client.

# Exercice 1: Baux fixes

L'attribution d'IP dynamiquement est bien pratique pour gérer un parc dynamique
de machines sans avoir à gérer leurs adresses IP. Il peut être cependant
utile pour certaines machines de toujours conserver la même adresse IP, pour
pouvoir s'y connecter en `ssh` par exemple.

On pourrait configurer à la main l'adresse IP dans la machine, comme on l'a fait
pour notre `alma-server`. Il est cependant plus malin de configurer ceci dans le
serveur DHCP en configurant des baux fixes. Cela permet en effet de gérer les
attributions d'adresses IP entièrement dans la configuration DHCP plutôt que
de manière éparpillée sur les différentes machines configurées à la
main. C'est notamment ce choix qui a été fait au CREMI par exemple.

Pour identifier la machine, c'est l'adresse MAC qui est utilisée. (Cela signifie
certes que si l'on change la carte réseau il faudra penser à corriger
l'adresse dans le bail fixe)

On va ici attribuer une IP fixe à notre VM client (ce qui permettra justement
de plus facilement s'y connecter).

## Adresse MAC

Il faut donc commencer par récupérer avec `ip a ls` l'adresse MAC de la carte
réseau `enp0s8` de la VM client.

## Configuration DHCP

Sur le serveur, on ajoute un bout de configuration DHCP pour indiquer l'adresse à
utiliser pour ce client:

```
host alma-client {
    hardware ethernet aa:bb:cc:dd:ee:ff;
    fixed-address 192.168.56.20;
}
```

Pourquoi a-t-on pris une adresse en-dehors de `192.168.56.100` - `192.168.56.150` ?

Redémarrez le serveur DHCP, relancez la configuration de l'interface `enp0s8` sur
le client, constatez que l'IP a été corrigée.

Vous pouvez désormais ajouter un autre alias dans votre `.ssh/config` pour
se connecter facilement en ssh à votre VM client.

## Configuration DNS

Ajoutez les traductions DNS pour votre VM, du nom de domaine vers l'adresse IP
et inversement. Rechargez la configuration DNS et vérifiez que c'est effectif.

Avez-vous pensé à mettre à jour le `serial` ?

# Exercice 2: Mise à jour DNS automatique

C'est fastidieux de mettre à jour le DNS à chaque fois qu'on ajoute une
machine. Il se trouve que le serveur DHCP peut s'occuper d'effectuer cette mise
à jour automatiquement. C'est notamment utile pour les machines qui n'ont pas
de bail fixe, pour lesquelles on ne pourrait de toutes façons pas enregistrer
d'IP fixe.

## Clé RNDC et DNS

Pour cela, on va faire discuter le serveur DHCP avec le serveur DNS. Pour que le
serveur DNS ait confiance en le serveur DHCP, on va utiliser une clé RNDC. Il
faut commencer par la générer:

```
rndc-confgen -a
```

Il nous dit qu'il a créé `/etc/rndc.key`. Vous pouvez regarder à quoi
ressemble le fichier, c'est un bout de configuration.

On peut l'inclure à la fin de la configuration `/etc/named.conf`:

```
include "/etc/rndc.key";
```

On peut maintenant dire au serveur DNS d'autoriser la connexion en local avec cette clé:

```
controls {
        inet 127.0.0.1 allow { localhost; } keys { rndc-key; };
};
```

Et lui dire sur quelle zone on a le droit d'écrire avec cette clé: dans
`/var/named/named.adsillh`, pour chaque `zone`, ajouter la ligne

```
  allow-update { key rndc-key; };
```

On peut faire recharger la configuration par `named`, et vérifier dans
`systemctl status named` qu'il est heureux.

## Configuration DHCP

Il reste à dire au serveur DHCP de notifier le serveur DNS, en ajoutant à la
fin de sa configuration:

```
ddns-update-style interim;
update-static-leases on; # update dns for static entries
allow client-updates;
include "/etc/rndc.key";

zone adsillh.local. {
        primary 127.0.0.1;
        key rndc-key;
}
zone 56.168.192.in-addr.arpa. {
        primary 127.0.0.1;
        key rndc-key;
}
```

Si l'on relance le serveur DHCP l'inclusion de `rndc.key` échoue... En effet,
SELinux nous embête, il faut apparemment permettre au serveur DHCP d'utiliser
mmap (bug dans AlmaLinux):

```
setsebool -P domain_can_mmap_files 1
```

Cette fois relancer le serveur DHCP devrait fonctionner.

## Côté client

Maintenant, il faut que la VM client indique quel nom de machine il souhaite utiliser, en ajoutant à son
`/etc/sysconfig/network-scripts/enp0s8` la ligne

```
DHCP_HOSTNAME=alma-client
```

Il faut recharger cette configuration

```
nmcli con reload enp0s8
```

Et re-négocier le bail DHCP

```
nmcli con down enp0s8 ; nmcli con up enp0s8
```

On peut alors voir côté serveur dans `systemctl status named` que la zone a été mise à jour:

```
adding an RR at 'alma-client.adsillh.local' A 192.168.56.20
[...]
adding an RR at '20.56.168.192.in-addr.arpa' PTR alma-client.adsillh.local.
```

On peut aussi suivre en direct les mises à jour avec 

```
tail -F /var/log/messages
```

Et l'on peut vérifier:

```
# nslookup alma-client.adsillh.local localhost
# nslookup 192.168.56.20 localhost
```

Si on fait re-négocier le bail, on constate que le serveur DHCP ne re-notifie pas le serveur DNS, il optimise. Pour pouvoir débugguer plus facilement on peut rajouter à la configuration DHCP:

```
update-optimization off;
```

pour désactiver l'optimisation.

# Exercice 3: une deuxième VM cliente

Mettons en place une deuxième VM cliente, pour vérifier que tout fonctionne comme prévu !

Pour s'éviter de réinstaller une VM, on peut simplement cloner notre VM cliente existante.

* Si elle n'a pas de nom bien explicite, on commencer par donner à notre première VM cliente un nom explicite:

```
hostnamectl set-hostname alma-client.adsillh.local
```

* Puis on l'éteint avec `poweroff`

* Dans VirtualBox, on clone la VM

  * Path: bien mettre de nouveau un chemin dans `/local/monlogin`
  * MAC Address Policy: on peut laisser "Inclure uniquement les adresses MAC de l'interface réseau NAT". La deuxième carte réseau qui est sur le vboxnet0 aura par contre une nouvelle adresse MAC.

Pourquoi est-il vraiment *primordial* de demander à générer une nouvelle adresse MAC pour le vboxnet0 ?

  * À la page suivante, demander un clone intégral pour simplifier, la copie est relativement rapide.

* Allumez la deuxième VM client, utiliser `hostnamectl` pour lui donner un nom différente, ainsi que dans le `DHCP_HOSTNAME`, et rebootez-la pour qu'elle ait bien sa configuration au propre.

* Rallumez la première VM client

Constatez que la première VM client a bien conservé son IP fixe, tandis que la
deuxième VM client a une IP dynamique.

## Attention par contre

Maintenant que la zone est mise à jour dynamiquement, il ne suffit plus d'utiliser

```
systemctl reload named
```

après avoir ajouté un nom de domaine: bind indique qu'il ne sait pas faire cela quand la zone est dynamique. Il faut désormais utiliser

```
systemctl restart named
```

ce qui implique une courte période pendant laquelle le serveur DNS est
indisponible. On verra plus loin comment redonder les serveurs DNS.

# Exercice 4: Redondance DHCP

Si le serveur DHCP est indisponible (parce que la machine est tombée en panne,
ou simplement parce qu'on est en train de la mettre à jour), les machines ne
peuvent pas récupérer d'adresse IP !

L'idée est donc de mettre en place une deuxième VM serveur dans laquelle on
fait tourner un deuxième serveur DHCP.

Le plus simple pour éviter de refaire une installation va de nouveau être de
dupliquer la VM serveur existante.

## Dupliquer la VM serveur

* Commencez par éteindre la VM serveur

* Clonez la VM comme pour la VM cliente

* Allumez la nouvelle VM serveur, utiliser `hostnamectl` pour lui donner un nom différent.

Pour l'instant, le clone répond encore à la même IP, ça va bien sûr poser problème :)

* Changez donc son IP fixe en `192.168.56.11` dans `/etc/sysconfig/network-scripts/ifcfg-enp0s8`

* Rebootez-la pour être sûr que toute la reconfiguration se fait bien.

* Vous devriez maintenant pouvoir vous y connecter avec sa nouvelle IP, ajoutez-vous un alias dans votre `.ssh/config`

* Vous pouvez maintenant rallumer la première VM serveur, les deux devraient pouvoir cohabiter.

## Corriger la configuration DHCP de la deuxième VM serveur

Les informations fixes (IP du serveur DNS, nom de domaine, etc.) peuvent rester
les mêmes, mais le subnet ne peut pas être le même (pourquoi ?)

* Mettez donc un subnet différent.

* Mettez cela à l'épreuve: arrêtez le dhcp sur la première VM serveur, et faites re-négocier le bail DHCP par la deuxième VM client, constatez quelle IP elle récupère. Essayez de faire re-négocier le bail DHCP par la première VM client, elle devrait bien garder son IP fixe.

(il est possible de configurer les deux serveur DHCP pour agir en tant que partenaires avec un failover, vous pourrez essayer si vous avez le temps)

Qu'en est-il de la mise à jour du DNS ? Cela ne fonctionne pas pour la
deuxième VM serveur !

En effet, on avait indiqué `primary 127.0.0.1`, mais du coup la deuxième
VM serveur met à jour son propre DNS, alors que c'est le serveur DNS de la
première VM serveur qui est utilisé... Il faut donc corriger la ligne `primary`

```
primary 192.168.56.10;
```

pour que le serveur DHCP tournant sur la deuxième VM serveur notifie le serveur DNS de la première VM serveur.

Il faut aussi que le serveur DNS de la première VM serveur accepte ces notifications venant du DHCP de la deuxième VM serveur. Dans `named.conf`, dans la section `controls`, il faut ajouter une deuxième ligne:

```
        inet 192.168.56.10 allow { 192.168.56.11; } keys { rndc-key; };
```

(on pourrait éventuellement utiliser une clé différente, mais simplifions-nous la vie)

Note: on avait oublié au TP précédent d'activer dns de manière permanente dans le firewall:

```shell
# firewall-cmd --zone=work --change-interface=enp0s8 --permanent
```

Rechargez la configuration du serveur DNS de la première VM serveur. Observez le log `/var/log/messages` sur les deux VMs serveur pendant la re-négociation du bail par la VM client, vous devriez voir la mise à jour DNS se faire !

# Exercice 5: Redondance DNS

La même question se pose pour le serveur DNS: s'il tombe ou est simplement en mise à jour, plus personne ne peut résoudre les noms de domaines !

On a déjà une deuxième VM serveur avec la même configuration DNS, il reste quelques détails à corriger.

## Adresse d'écoute

Dans `/etc/named.conf` on avait configuré à la main l'adresse IP sur laquelle écouter, il faut la corriger.

## Maître-esclave

Pour DHCP, on avait répliqué la configuration. Elle n'était pas très grosse,
en pratique cela ne pose pas vraiment problème.

Pour DNS par contre, répliquer la configuration c'est pénible, car les zones
peuvent rapidement contenir des dizaines d'enregistrements.

On préfère alors utiliser un fonctionnement maître-esclave: le serveur DNS
de la première VM serveur sert de maître, c'est-à-dire de référence, c'est
sur lui qu'on modifie les zones. Le serveur DNS de la deuxième VM sert
d'esclave, c'est-à-dire que le maître lui envoie automatiquement les nouvelles
versions des zones.

Pour cela, sur la première VM serveur, dans `/var/named/named.adsillh`, dans chaque section `zone` on rajoute une ligne

```
  also-notify { 192.168.56.11; };
```

Et sur la deuxième VM serveur, dans `/var/named/named.adsillh`, dans chaque section `zone`
* on passe le `type` en `slave`
* on renomme l'option `allow-update` en `allow-update-forwarding`
* et au lieu d'un `also-notify` on ajoute une ligne

```
  masters { 192.168.56.10; };
```

pour autoriser le maître à envoyer la notification.

Redémarrez le maître et l'esclave.

Observez le log `/var/log/messages` sur la deuxième VM serveur pendant que vous
ajoutez un nouveau CNAME sur le maître. Est-ce que vous avez pensé à mettre à jour le `serial` ?

Maintenant que l'esclave fonctionne, on peut le rajouter dans la liste des serveurs DNS interrogeables:

* Sur les VMs serveurs, dans `ifcfg-enp0s8` on peut ajouter une ligne

```
DNS2=192.168.56.11
```

(ou .10, de manière croisée)

* Pour les VMs clientes, on corrige simplement dans les configurations DHCP:

```
    option domain-name-servers 192.168.56.10, 192.168.56.11;
```

lors du prochain renouvellement de bail, les VMs clientes auront la mise à jour. Vous pouvez forcer le renouvellement pour constater que cela fonctionne.

## Configuration DHCP

Nos deux serveur DHCP sont actuellement configurés pour mettre à jour le DNS
maître (et il s'occupera de mettre à jour le DNS esclave). Mais si le maître
est en maintenance, il faudrait notifier au moins l'esclave.

Dans `dhcpd.conf`, en-dessous des lignes `primary`, on peut ajouter une ligne
`secondary` avec le même format, pour faire notifier vers le DNS esclave dans
le cas où le maître ne répondrait pas. Quand le DNS maître revient, le DNS
esclave s'occupe de lui envoyer la mise à jour.

Ainsi les configurations sont croisées dans tous les sens, mettez ceci à
l'épreuve en testant les différentes situations de pannes: tant qu'il y a au
moins un serveur DHCP et un serveur DNS, tout devrait bien se passer (à part la
mise à jour de l'enregistrement DNS, qui a besoin du DNS maître).
