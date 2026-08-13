# Procédure de migration PVE 8 -> 9



---

## Etapes

### 1. MAJ pour avoir la dernière version 8.x
Avant de vouloir migrer vers la version 9, il faut s'assurer que tous les derniers paquets de l'ancienne version sont bien installés pour bien être sur la dernière version.
```bash
apt update && apt upgrade
```

### 2. Check de pré-migration
Lancement de l'utilitaire pour vérifier que le système est prêt pour le passage en version 9.
```bash
pve8to9
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
- Edit du fichier : `nano /etc/apt/source.list` -> remplacement de bookworm par trixie
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
apt upgrade
```
A partir d'un moment, plusieurs choses seront demandés où il faudra taper `N` afin de garder les configurations actuelles.

Une fois que cette mise à jour est faite, dans le fichier `/opt/ansible/hosts` il va falloir ajouté le PVE mis à jour dans le groupe des PVE v9.
Enfin, il faut lancer le script de configuration des PVE v9.x (conf_proxmox_9.yml) présent sur s57proxmox01 dans `/opt/ansible`.
```bash
ansible-playbook conf_proxmox_9.yml --limit nom du PVE mis à jour
```

Enfin il ne reste plus qu'à reboot le PVE.
