# Procédure de migration PBS 3 -> 4



---
!!! Enregistrer sa configuration si aucune sauvegarde est faite


## Etapes

### 1. MAJ pour avoir la dernière version 3.x
Avant de vouloir migrer vers la version 4, il faut s'assurer que tous les derniers paquets de l'ancienne version sont bien installés pour bien être sur la dernière version.
```bash
apt update && apt upgrade
```

### 2. Check de pré-migration
Lancement de l'utilitaire pour vérifier que le système est prêt pour le passage en version 4.
```bash
pbs3to4
```

### 3. Vérification de l'amorçage (EFI)
Je vérifie que Proxmox est bien présent dans le boot EFI :
- Via efibootmgr 
```bash
efibootmgr -v
```
- Via le système de fichiers : Vérifier la présence du répertoire EFI dans `/boot/efi`.

### 4. Changements des dépôts
Je passe les dépôts sur la nouvelle version (Trixie)
- Edit du fichier : `nano /etc/apt/source.list`
- Update de la liste :
```bash
apt update
```

### 5. Nettoyage
Suppresion du paquet `systemd-boot`
```bash
apt remove systemd-boot
```

### 6. Installation sécurisée via Screen
Pour ne pas perdre la main sur le terminal pendant l'install (en cas de coupure réseau), je lance un `screen`
Et ensuite je lance la grosse mise à jour une fois dans le screen 
```bash
apt dist-upgrade
```
A partir d'un moment, plusieurs choses seront demandés où il faudra taper `N` afin de garder les configurations actuelles.

Enfin il ne reste plus qu'à reboot le PBS.


