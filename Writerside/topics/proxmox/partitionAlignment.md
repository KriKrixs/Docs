# Alignement des partitions

**Écrit par [@krikrixs](https://github.com/KriKrixs)**

## Introduction

Après avoir installé Proxmox sur un SSD NVMe de la marque SK Hynix, j'ai remarqué que le serveur mettait beaucoup de temps à démarrer.

Puis un jour, le démarrage n'a pas abouti :

<img src="crashEFI.png" alt="Image crash boot proxmox" width="600"/>

Après un redémarrage ayant abouti cette fois-ci, par curiosité, j'ai exécuté la commande suivante :

```bash
fdisk -l
```

J'ai pu constater que les partitions n'était pas correctement alignées.

```
Disk /dev/sde: 238.47 GiB, 256060514304 bytes, 500118192 sectors
Disk model: SSD-240G V01    
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: <masqué>

Device       Start       End   Sectors   Size Type
/dev/sde1       34      2047      2014  1007K BIOS boot
/dev/sde2     2048   2099199   2097152     1G EFI System
/dev/sde3  2099200 500118158 498018959 237.5G Linux LVM

Partition 1 does not start on physical sector boundary.
```
<sup>*Ici, la partition 1 correspond à la partition EFI, la partition qui n'arrive pas à être contacté au boot.</sup>

Sur ce résultat, 2 choses importantes sont à noter. 

- La première, c'est effectivement le message d'avertissement
`Partition 1 does not start on physical sector boundary.` qui indique un problème d'alignement avec la partition EFI. (Comme pas hasard celle qui me pose souci au boot de Proxmox)
- La deuxième, c'est `Sector size (logical/physical): 512 bytes / 4096 bytes` qui indique que mon SSD utilise le format "Advanced format" 

Passons rapidement sur les 2 formats de taille de secteurs :

- Le **format standard** est le format le plus répandu et surtout le plus utilisé depuis longtemps sur les disques durs, celui-ci défini la taille des secteurs physique comme étant de **512 octets**.

    Grâce à son âge, il est le standard assurant une compatibilité universelle entre OS, matériels, logiciel, etc.

    Cependant, des secteurs de 512 octets sont moins performant et peuvent gaspiller de l'espace si les fichiers sont trop fragmentés.

- Le **format avancé "Advanced format"** est lui utilisé principalement sur les stockages à grande capacité. Celui-ci défini la taille des secteurs physique comme étant de **4096 octets**.

    Avec une taille de secteurs multipliée par 8, le disque a une meilleure densité, des performances améliorées et optimisées pour les gros fichiers.

    Cependant, il ne garantit pas une compatibilité universelle ce qui peut engendrer des problèmes d'alignement des partitions. Il est également moins adapté pour gérer des petits fichiers.

Un problème d'alignement peut engendrer des problèmes de stabilité, de performance et la partition mal alignée peut ne pas être trouvé.

Le format physique des secteurs d'un disque **ne peut pas être modifié**. Ce qui nous laisse 2 choix dans ce cas.

- Acheter un disque dont on connait la taille des secteurs physique.
- Réaligner les partitions.

La première solution est la plus facile et rapide, mais en réalité quand on voit la faible complexité de la 2ᵉ solution ainsi que les prix des NVMe, cela revient à jeter de l'argent pas les fenêtres.

Cette documentation va donc détailler l'ensemble des commandes à réaliser afin de réaligner correctement les partitions.

## Préparation et sauvegarde

En tant que flemmard, je n'ai pas envie de réinstaller complétement Proxmox et de devoir tout restaurer après. Je vais donc réaligner les partitions puis réinjecter les données dans ces dernières.

**Le réalignement des partitions comporte des risques de perte de données.**

Pour effectuer cette manipulation, nous devrons avoir le matériel suivant :

- Un PC
- Le disque à réaligner
- Dans le cas où le disque à réaligner est celui de boot, une clé USB bootable avec Ubuntu Live
- Un autre espace de stockage ayant au minimum la taille du disque à réaligner comme espace libre voir le double si possible pour plus de sécurité.
- Le résultat `fdisk -l` actuel du disque à réaligner.

Dans le cas où le disque à réaligner est le disque de boot, pour éviter toute écriture pendant le dump, je conseille vivement d'effectuer les opérations sur un autre système linux (clé usb ou autre PC).

Pour toutes les commandes ci-dessous, remplacez `sde` par le nommage correspondant à votre disque dans le résultat `fdisk -l`

Dans un premier temps, je conseille d'effectuer un dump intégrale du disque via la commande suivante :

```bash
dd if=/dev/sde of=/path/to/backup/sde.img bs=4M status=progress
```
<sup>Plus le disque est volumineux, plus le dump sera long.</sup>

**Ce dump n'est pas obligatoire**, mais garantira l'intégrité des données, mais aussi des partitions en cas d'échec ou éventuel autre problème.

Une fois fait, effectuer la commande suivante pour chaque partition :

```bash
dd if=/dev/sde1 of=/path/to/backup/sde1.img bs=4M status=progress
dd if=/dev/sde2 of=/path/to/backup/sde2.img bs=4M status=progress
dd if=/dev/sde3 of=/path/to/backup/sde3.img bs=4M status=progress
```
**Ces dumps sont obligatoires**, Ils serviront à réinjecter les données telles qu'elles étaient dans les partitions réalignées.

## Réalignement des partitions et restauration

Pour réaligner les partitions, nous allons détruire la table de partition et la recréée manuellement.

```bash
parted /dev/sde mklabel gpt
```

Maintenant que cela est fait, nous allons recréer les partitions et les identifiées correctement.

La première partition doit commencer à 1Mo pour être correctement aligné. Ce chiffre n'est pas sorti du chapeau, il s'agit du standard pour garantir la compatibilité et l'alignement le plus optimale.
En effet, si demain le disque avec secteurs physique 4096 doit être remplacé par un disque de 2048, alors commencé à 1Mo permettra de réinjecter le dump tel quel et garder l'alignement.

Cette fois-ci, il faudra effectuer les commandes suivantes en adéquation avec le résultat `fdisk -l` que vous avez obtenu plus tôt.

```bash
parted /dev/sde
(parted) mkpart primary 1MiB 3MiB
(parted) set 1 bios_grub on
(parted) mkpart primary fat32 3MiB 1027MiB
(parted) set 2 esp on
(parted) mkpart primary ext4 1027MiB 100%
(parted) quit
```

Maintenant que notre disque a des partitions correctement aligner, nous allons réinjecter les dumps 1 par 1 des partitions pour récupérer les données.

```bash
dd if=/path/to/backup/sde1.img of=/dev/sde1 bs=4M status=progress
dd if=/path/to/backup/sde2.img of=/dev/sde2 bs=4M status=progress
dd if=/path/to/backup/sde3.img of=/dev/sde3 bs=4M status=progress
```

Il est maintenant possible de démarrer sur le disque à nouveau.

Dans mon cas, j'ai eu un message indiquant que les paramètres de boot de la carte mère allait être reconfiguré. Pas de panique, si vous avez la même chose, laissez faire ou confirmer l'action.

Proxmox affichera également un message d'avertissement au premier démarrage indiquant qu'il a détecté des changements de taille pour les partitions. Encore une fois pas de panique tout est normal.

Logiquement si tout fonctionne, alors le résultat de la commande suivante ne devrait plus indiquer de problème d'alignement.

```bash
fdisk -l
```

## En cas d'échec

Si vous n'arrivez pas à effectuer la manipulation et souhaitez revenir en arrière, alors réinjecter le dump complet du disque.

```bash
dd if=/path/to/backup/sde.img of=/dev/sde bs=4M status=progress
```

Le disque complet sera remis à l'état d'avant les manipulations.

## Conclusion

De mon côté, j'ai pu noter une claire amélioration de performance notamment au démarrage, mais aussi de stabilité notamment dans les contenurs LXC ou VM.

Pour le peu de commande et de complexité que cela demande, je recommande fortement d'aligner correctement ses partitions.
