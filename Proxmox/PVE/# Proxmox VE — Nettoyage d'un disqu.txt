# Proxmox VE — Nettoyage d'un disque Ceph après retrait d'un OSD

## 1. Objectif

Cette procédure décrit les étapes à suivre lorsqu'un disque destiné à la création d'un nouvel OSD Ceph ne peut pas être nettoyé depuis l'interface Web de Proxmox VE.

L'échec de l'opération **Wipe Disk** peut être lié à la présence d'anciennes signatures Ceph, de métadonnées LVM ou de partitions résiduelles laissées après le retrait d'un OSD.

L'objectif est de vérifier l'état du disque, d'identifier les métadonnées restantes, de procéder à son nettoyage si nécessaire, puis de confirmer qu'il peut être réutilisé.

## 2. Prérequis et précautions

Avant toute intervention, vérifier les points suivants :

- L'ancien OSD a été correctement retiré du cluster Ceph.
- Le disque ne contient aucune donnée à conserver.
- Le disque n'est plus utilisé par un OSD actif.
- Le disque système et les autres disques de données ont été identifiés.
- L'état du cluster Ceph a été contrôlé.

Vérifier l'état général du cluster :

```bash
ceph -s
ceph health detail
ceph osd tree
```

Ne pas poursuivre si le disque appartient encore à un OSD actif ou si le retrait de l'OSD n'est pas terminé.

> **Attention :** les opérations de nettoyage peuvent supprimer définitivement les métadonnées et rendre les données du disque inaccessibles. Toujours vérifier le disque ciblé avant d'exécuter une commande destructive.

## 3. Identifier le disque à nettoyer

Se connecter au Shell du nœud Proxmox concerné.

Afficher les disques présents :

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL,SERIAL
```

Afficher les chemins persistants des disques :

```bash
ls -l /dev/disk/by-id/
```

Identifier le disque à partir de sa capacité, de son modèle et de son numéro de série.

Privilégier un chemin stable sous `/dev/disk/by-id/` plutôt qu'un nom comme `/dev/sdb`, susceptible de changer après un redémarrage.

Dans la suite de cette procédure, remplacer `ID_DU_DISQUE` par le chemin réel du disque concerné.

Exemple de chemin :

```text
/dev/disk/by-id/ID_DU_DISQUE
```

Ne pas utiliser cet exemple tel quel : il s'agit d'un emplacement à remplacer.

## 4. Vérifier les métadonnées restantes

### 4.1 Vérifier les anciens OSD Ceph

```bash
ceph-volume lvm list
```

Cette commande permet d'identifier les OSD associés aux volumes LVM connus du nœud.

Vérifier que le disque ciblé ne correspond pas à un OSD encore utilisé.

### 4.2 Vérifier les volumes LVM

```bash
pvs
vgs
lvs
```

Ces commandes affichent respectivement les volumes physiques, les groupes de volumes et les volumes logiques présents sur le nœud.

Si des éléments associés à Ceph apparaissent, vérifier leur correspondance avec le disque concerné avant toute suppression.

Ne pas supprimer un groupe ou un volume LVM uniquement parce que son nom contient `ceph`.

### 4.3 Vérifier les signatures du disque

```bash
wipefs -n /dev/disk/by-id/ID_DU_DISQUE
```

L'option `-n` affiche les signatures détectées sans les effacer.

La présence d'anciennes signatures peut expliquer pourquoi Proxmox refuse de nettoyer ou de réutiliser le disque.

## 5. Nettoyer les métadonnées de l'ancien OSD

Si le retrait de l'OSD est terminé, que le disque est correctement identifié et qu'aucune donnée n'est à conserver, utiliser `ceph-volume` pour nettoyer les métadonnées de l'ancien OSD.

```bash
ceph-volume lvm zap --destroy /dev/disk/by-id/ID_DU_DISQUE
```

Cette commande demande à `ceph-volume` de nettoyer le disque et de supprimer les structures LVM associées à l'OSD.

Vérifier une dernière fois le chemin du disque avant de valider l'opération.

### 5.1 Si l'option `--destroy` n'est pas reconnue

Si la commande renvoie une erreur indiquant que l'option n'est pas reconnue, ne pas essayer d'autres commandes destructives au hasard.

Vérifier la version installée :

```bash
ceph-volume --version
```

Consulter l'aide de la commande :

```bash
ceph-volume lvm zap --help
```

Comparer ensuite la syntaxe disponible avec la documentation correspondant à la version de Ceph installée.

Ne pas remplacer automatiquement cette commande par `wipefs -a`, `sgdisk --zap-all` ou une suppression manuelle de volumes LVM.

Si la syntaxe reste incompatible, conserver le message d'erreur complet afin d'adapter la procédure à la version installée.

## 6. Vérifier le résultat du nettoyage

Après l'opération, relancer les contrôles suivants :

```bash
lsblk -f
wipefs -n /dev/disk/by-id/ID_DU_DISQUE
ceph-volume lvm list
pvs
vgs
lvs
```

Vérifier les éléments suivants :

- Le disque ciblé ne présente plus les anciennes signatures attendues.
- Aucune ancienne structure LVM liée à l'OSD ne subsiste sur ce disque.
- Aucun volume utilisé par un autre service n'a été supprimé.
- Le disque est disponible pour une nouvelle utilisation.

Les commandes `pvs`, `vgs` et `lvs` affichent les ressources LVM du nœud entier. La présence d'autres volumes Ceph ne signifie pas nécessairement que le nettoyage a échoué.

## 7. Relancer Wipe Disk depuis l'interface Proxmox

Lorsque les contrôles sont satisfaisants :

1. Ouvrir l'interface Web de Proxmox VE.
2. Sélectionner le nœud concerné.
3. Ouvrir **Disks**.
4. Sélectionner le disque identifié.
5. Cliquer sur **Wipe Disk**.
6. Attendre la fin de l'opération et vérifier son résultat.

Si le nettoyage réussit, le disque peut être réutilisé pour la création d'un nouvel OSD, sous réserve des contrôles habituels de disponibilité et d'éligibilité.

## 8. En cas de nouvel échec

Si la création de l'OSD ou l'opération Wipe Disk échoue encore, récupérer le message complet de la tâche dans **Node → Task History**.

Contrôler à nouveau :

```bash
ceph -s
ceph health detail
ceph osd tree
ceph-volume lvm list
lsblk -f
```

Ne pas répéter les opérations destructives sans avoir identifié la cause de l'échec.

Si le problème persiste, conserver les sorties des commandes, le message d'erreur et la version de Proxmox VE/Ceph pour poursuivre le diagnostic.

## 9. Bonnes pratiques

- Toujours identifier un disque par son numéro de série avant de le nettoyer.
- Vérifier que l'ancien OSD a été retiré avant d'effacer le disque.
- Ne jamais lancer une commande de nettoyage sur un disque système ou un OSD actif.
- Ne pas supprimer manuellement des volumes LVM sans avoir identifié leur rôle.
- Adapter les commandes à la version de Ceph installée.
- Contrôler l'état du cluster avant et après toute opération sur un OSD.

## 10. Documentation de référence

- Proxmox VE : https://pve.proxmox.com/pve-docs/
- Documentation Ceph — `ceph-volume` : https://docs.ceph.com/en/latest/ceph-volume/
- Documentation Ceph — gestion des OSD : https://docs.ceph.com/en/latest/rados/operations/add-or-rm-osds/
