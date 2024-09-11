Cours 1: Linux, Unix
====================

[https://dept-info.labri.fr/~thibault/enseignements.html#InstallConfiglpro](https://dept-info.labri.fr/~thibault/enseignements.html#InstallConfiglpro)

Apprenez à taper sans regarder votre clavier.

# Linux ? Unix ?

Historique, origines

- Unix ~70 (e.g. AIX, Solaris, HP-UX)
  - [Unix — Wikipédia](https://fr.wikipedia.org/wiki/Unix)
- BSD ~80
  - [BSD — Wikipédia](https://fr.wikipedia.org/wiki/Berkeley_Software_Distribution)
- Linux ~90
  - [Linux — Wikipédia](https://fr.wikipedia.org/wiki/Linux)

De nombreuses distributions Linux, particularités propres

Grandes branches:

- Debian
  - `dpkg`/`apt`
- RedHat
  - `rpm`/`yum`/`dnf`
- Suse
  - `yast`
- Arch
  - `pacman`
- Linux From Scratch
  - Pas de gestionnaire de package, tout à la main :)

et successeurs (Ubuntu, Kali, Mint, Fedora, ...)

[DistroWatch.com: Put the fun back into computing. Use Linux, BSD.](https://distrowatch.com/dwres.php?resource=family-tree)

# Organisation du système

Démarrage:

- firmware/bios
- bootloader: grub
- noyau linux
- systemd
- démons
  - serveur web/dns/smtp/...
  - serveur ssh
  - bannière de login
- bureau
- shell

![](boot.svg)

# Le shell

Aussi appelé "invite de commande", `cmd.exe`, ...

Permet de taper des commandes, directement sur la console, via ssh, via un script, automatisé, etc...

Vous approfondirez son usage en cours de shell avec Olivier Delmas

## Raccourcis clavier essentiels

* `control-alt-t` pour lancer un terminal
* `tab` pour compléter les noms de fichiers
* flèche haut puis gauche/droite pour reprendre et corriger une commande précédente
* `control-r` puis taper un mot pour retrouver une commande dans l'historique

## Must have

* *`man`* : RTFM : Commande a connaitre par coeur. Permet d'avoir l'aide de n'importe quelle commande (y compris elle même)
  * Utiliser `/` pour chercher quelque chose dedans
  * Utiliser `n` pour trouver la prochaine occurrence de ce que vous avez cherché

* `cd` : change de dossier

* `nano`, `vi`, `vim`, `emacs` : Différents éditeurs, choose your poison. 

* `ls` : Liste le contenu d'un dossier.

* `tail`, `head` : Affiche respectivement la fin et le début de fichier. L'option -f de tail permet de continuer à suivre le fichier : tail -f truc.log

* `dmesg` : Afficher des logs kernel

* `| grep` : Filtre la sortie d'une autre commande

* `cat` : Affiche le contenu d'un fichier

* `cp`, `rm`, `mv`, `mkdir`, `rmdir` : Manipulation de fichier (copie, remove, move, make directory, remove directory)

* `chmod`, `chown` : Modifications des droits.

* `df` : Disk File. Lister les filesystem et leur utilisation.

* `ps` : Instantané des processus

* `free` : Voir rapidement l'utilisation de mémoire

* `top` : Voir rapidement les conso de cpu/mémoire sous forme de graph

## Des commandes qui lancent des commandes:

* `lsscsi` : Liste les périphériques dit "SCSI". Ca concerne le bus lui même, donc les disques durs aussi, pas seulement les vieilles cartes des années 50

* `nmcli`: Permet la configuration réseau. On en parlera plus tard.

* `systemctl` : "system control". On en parle un peu après aussi. Permet de "piloter" le système.

* `uname`: Récupère des infos sur l'OS, son kernel, son archi.

* `lscpu` : Récupère des infos sur le CPU. On peut aussi les avoir avec un `cat /proc/cpuinfo`

## Des commandes qui permettent de gérer les soft

* `dnf`, `apt`, `yast`, `pacam`, `pip`, etc ... :

* `make`, `cmake`, `configure`, etc ... 

## Gérer les démons

* `systemctl` `status`/`restart`/`start`/`stop`/`enable`/`list-units`

```shell
# systemctl status sshd
* sshd.service - OpenSSH server daemon
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2023-01-24 13:22:04 EST; 11min ago
...
```

* `journactl`

```shell
# journactl -u sshd
-- Logs begin at Tue 2023-01-24 13:20:14 EST, end at Tue 2023-01-24 13:29:01 EST. --
janv. 24 13:22:04 localhost.localdomain systemd[1]: Starting OpenSSH server daemon...
janv. 24 13:22:04 localhost.localdomain sshd[926]: Server listening on 0.0.0.0 port 22.
janv. 24 13:22:04 localhost.localdomain sshd[926]: Server listening on :: port 22.
janv. 24 13:22:04 localhost.localdomain systemd[1]: Started OpenSSH server daemon.
janv. 24 13:20:27 localhost.localdomain sshd[1588]: Accepted password for root from 192.168.56.200 port 51302 ssh2
janv. 24 13:20:27 localhost.localdomain sshd[1588]: pam_unix(sshd:session): session opened for user root by (uid=0)
```

# Organisation des fichiers

+ montrer en live

- `/` : la racine, contient tout le reste
- `/usr` : logiciels installés
- `/usr/local` : logiciels installés à la main
- `/etc` (Editing Text Config) : fichiers de configuration
- `/tmp` (Temporary) : fichiers stockés temporairement (effacé au reboot)
- `/home` : les données des utilisateurs
- `/root` : les données de l'administrateur
- `/var` (variable): données variables: base de données, logs, ...
- `/proc`, `/sys`, `/dev`, `/run`: données concernant le système et les processus en cours d'exécution

- `/local`: au CREMI, le disque local (*bien* plus rapide que `/home`)

Pour aller plus loin:

[https://www.linuxtricks.fr/wiki/arborescence-du-systeme-linux](https://www.linuxtricks.fr/wiki/arborescence-du-systeme-linux)

Pour aller encore plus loin:

[https://refspecs.linuxfoundation.org/fhs.shtml](https://refspecs.linuxfoundation.org/fhs.shtml)

# Gestion des packages

dnf search/install/provide/info

# Pourquoi des commandes textuelles et des fichiers texte, plutôt que des interfaces graphiques intuitives ?

* Permet d'avoir de nombreuses fonctionalités sans l'effet clickodrome
* Copier/coller facilement des commandes complexes
* Automatiser des tâches
* Automatiser le déploiement de configurations / mises à jour sur de nombreuses machines (e.g. salles de TP du CREMI)

# Début du TP1

# Systèmes de fichiers

* Disques
  * stockent juste des octets
* table de partitions
  * découpe les disques en tranches contigues
* LVM
  * Abstrait la notion de disque, pour découper en tranches de manière flexible
  * pas forcément contigues
  * pas forcément sur un seul disque
  * retaillables à la volée
* FS
  * Stocke vraiment les fichiers
  * Formattage rapide
  * FAT/vFAT
    * DOS/Windows au départ, très simple, adopté très largement (e.g. clés USB)
  * NTFS
    * Windows
  * HPFS
    * MacOS
  * ext2/3/4
    * au début de Linux, a évolué
  * jfs
  * xfs
  * btrfs
  * ...
