# INSTALLATION PBS CENTRAL

## Table des matières
- [Installation d’un serveur Proxmox Backup Server](#installation-dun-serveur-proxmox-backup-server)
  - [1. Installation de l’ISO Proxmox Backup Server](#1-installation-de-liso-proxmox-backup-server)
  - [2. Configuration Initiale](#2-configuration-initiale)
  - [3. Configuration Finale](#3-configuration-finale)
- [Ajout des remotes PBS distants](#ajout-des-remotes-pbs-distants)
  - [1. Méthode via l'interface graphique](#1-méthode-via-linterface-graphique)
  - [2. Méthode automatisée via script](#2-méthode-automatisée-via-script)
  - [3. Vérifications](#3-vérifications)
- [Gestion des Sync Jobs](#gestion-des-sync-jobs)
  - [1. Contexte](#1-contexte)
  - [2. Création automatique des Sync Jobs](#2-création-automatique-des-sync-jobs)
  - [3. Ajout automatique des paramètres des Sync Jobs](#3-ajout-automatique-des-paramètres-des-sync-jobs)
  - [4. Explication des paramètres ajoutés automatiquement aux Sync Jobs](#4-explication-des-paramètres-ajoutés-automatiquement-aux-sync-jobs)


---

## Installation d’un serveur Proxmox Backup Server

### 1. Installation de l’ISO Proxmox Backup Server
1. Se connecter via l’interface web sur l’hyperviseur.  
2. Récupérer l’ISO de **PBS 4.0**.  
3. Dans Proxmox VE, créer une VM et sélectionner l’ISO dans le lecteur CD-ROM. La VM démarrera automatiquement sur l’ISO.  
4. Redémarrer la VM et choisir **Installation Proxmox VE Graphique**.  
5. Accepter la licence.  
6. Sélectionner le disque (RAID1) et cliquer sur **Options** :  
   - SWAPSIZE = 0  
   - MINFREE = 0  
7. Configurer la zone/langue :  
   - COUNTRY : France  
   - TIMEZONE : Europe/Paris  
   - KEYBOARD : French  
8. Mot de passe et Email :  
   - Email : `renseigner un mail` (⚠ Ne pas effacer l’@)  
9. Configuration réseau :  
   - Interface : ens18  
   - Hostname : `nom-du-serveur.domaine.extension`  
   - IP : `...`  
   - Gateway : `...`  
   - DNS : `...`  
10. Cliquer sur **INSTALL**.

### 2. Configuration Initiale

#### Dépôts APT (`/etc/apt/sources.list`)
Ajout des bons dépôts pour la gestion des paquets logiciels
```text
deb http://... trixie main contrib
deb http://... trixie-updates main contrib
deb http://... trixie-security/updates main contrib
deb http://... trixie pbs-no-subscription
```
> ⚠ PBS 4.x est basé sur Debian Trixie et non plus sur Bookworm.

#### APT config (`/etc/apt/apt.conf`)
```text
Acquire::Languages "none";
```

#### Mise à jour du système
```bash
apt update
apt upgrade
```

### 3. Configuration Finale

#### Préparation du disque de sauvegarde (LVM + EXT4 + fstab)
Ajout de deux disques 1T (exemple) sur la VM 
1. **Lister les disques**

On vérifie bien la présence des deux nouveaux disques
```bash
lsblk
```
2. **Créer volumes LVM**

On initialise les disques physiques, on crée un groupe de volumes et un volume logique qui utilise tout l'espace disponible
```bash
pvcreate /dev/sdb /dev/sdc           
vgcreate backup /dev/sdb /dev/sdc   
lvcreate -l 100%FREE -n dr backup   
```
3. **Formater en EXT4**

On prépare le volume pour stocker les sauvegardes (format EXT4 est recommandé pour le stockage de sauvegardes sur PBS)
```bash
mkfs.ext4 /dev/backup/dr
```
4. **Montage temporaire**

On vérifie le volume et on le monte sur un répertoire temporaire
```bash
mkdir /mnt/temp
mount /dev/mapper/backup-dr /mnt/temp
dfc
```
5. **Optimisation EXT4**

On retire la réservation par défaut de 5% pour root afin de maximiser l'espace dispo
```bash
tune2fs -m0 /dev/mapper/backup-dr
dfc
```
6. **Démontage et création point final**

On prépare le point de montage permanent
```bash
umount /mnt/temp
mkdir /mnt/backup-dr
```
7. **Récupérer UUID & config fstab**

On utilise l'UUID pour un montage automatique au démarrage
```bash
blkid /dev/mapper/backup-dr
nano /etc/fstab
```
Ajouter :
```text
UUID=<uuid_du_volume>    /mnt/backup-dr    ext4    defaults    0 0
```
8. **Recharger et monter**

Enfin on applique la configuration et on vérifie que tout est monté correctement
```bash
systemctl daemon-reload
mount /mnt/backup-dr
dfc
```


## Ajout des remotes PBS distants

### 1. Méthode via l'interface graphique

Cette méthode permet d'ajouter un remote manuellement depuis l'interface web du PBS central.
Elle n'est pas adapté dans notre situation car elle est utile seulement lorsque l'on doit ajouter un ou deux serveurs distants et qu'une simple intervention manuelle est suffisante.
Mais il est important de rappeler que l'ajout de remote peut se faire depuis l'interface web.

1.  Aller dans **Configuration → Remotes**\
2.  Cliquer sur **Add**\
3.  Renseigner :
    -   **ID** : nom du remote \
    -   **Host** : IP ou DNS du PBS distant\
    -   **Auth ID** : `User ou Token`\
    -   **Password / Token Secret** : secret associé\
    -   **Fingerprint** : empreinte TLS du PBS distant\
4.  Valider → la remote est ajoutée

------------------------------------------------------------------------

### 2. Méthode automatisée via script
S'il y a plus de 5 PBS, la solution la plus adéquate est d'automatiser avec un script comme ici avec ansible.
Pour ce faire on va créer un dossier "auto" comprenant le script et un dossier contenant toutes les configs de chaque PBS :
``` bash
auto/
     ├─ add_remotes.sh
     └─ remote_configs/
        ├─ pbs-1.conf
        └─ pbs-2.conf
         ...
``` 

Exemple de fichier de configuration :

    HOST="100.100.100.100" 			IP à renseigner
    TOKEN_NAME="token_name"			Token à reneigner
    TOKEN_SECRET="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    FINGERPRINT="aa:bb:cc:..." 		Fingerprint à renseigner


Ensuite pour ce qui est du script, il va récupérer dans les fichiers de configuration les différents paramètres et créer les remotes avec une commande.

``` bash
#!/bin/bash

#Chemin du dossier contenant les configs
CONFIG_DIR="./remote_configs"

#Utilisateur  utilisé pour les tokens de synchronisation
AUTH_USER="..."		à renseigner

echo "--- Démarrage de l'ajout des remotes ---"

#Parcours tous les fichiers .conf dans le dossier remote_configs
for CONFIG_FILE in "$CONFIG_DIR"/*.conf
do
    #Nom du remote = nom du fichier de config sans l'extension
    REMOTE_ID=$(basename "$CONFIG_FILE" .conf)

    #Vérifie la présence du fichier
    if [ -f "$CONFIG_FILE" ]; then
        #Charge les variables du fichier de conf    
        source "$CONFIG_FILE"
    else
        echo "Fichier introuvable : $CONFIG_FILE"
        continue
    fi

    echo "=== Ajout de $REMOTE_ID (Host: $HOST) ==="

    #Construction de l'auth ID requis par PBS 4.0
    #Format : user@realm!token-name
    AUTH_ID="${AUTH_USER}!${TOKEN_NAME}"

    #Commande PBS 4.0 pour créer un remote
    proxmox-backup-manager remote create "$REMOTE_ID"         --host "$HOST"         --auth-id "$AUTH_ID"         --password "$TOKEN_SECRET"         --fingerprint "$FINGERPRINT"

    #Vérification de la réussite de la commande 
    if [ $? -eq 0 ]; then
        echo " Remote '$REMOTE_ID' ajoutée."
    else
        echo "  Erreur pour '$REMOTE_ID'."
    fi

done

echo "--- Terminé ---"
```

Maintenant pour que le script puisse être éxécuter, il faut lui attribuer les droits d'éxécution

    chmod +x add_remotes.sh

    ./add_remotes.sh    #Exécution du script


------------------------------------------------------------------------

### 3. Vérifications
Enfin pour vérifier que les remotes ont bien étaient ajouter correctement on peut voir la liste dans l'interface web dans **Configuration → Remotes**\
Ou on peut les lister et voir leur statut via les lignes suivantes :

    proxmox-backup-manager remote list

    proxmox-backup-client status --repository <remote>:<datastore>

------------------------------------------------------------------------


# Gestion des Sync Jobs

## 1. Contexte

Le but est de gérer automatiquement tous les *Pull Sync Jobs* entre le PBS central et les PBS distants.  
Un Sync Job permet de synchroniser les sauvegardes d’un serveur distant vers le serveur central.

Deux scripts sont donc utilisés :

1. **Un script qui crée automatiquement tous les Sync Jobs** à partir des fichiers `.conf` présents dans `remote_configs/`.
2. **Un script qui ajoute automatiquement les paramètres nécessaires à tous les Sync Jobs**, car certains ne peuvent pas être ajoutés directement lors de la création.

Ces deux scripts simplifient et automatisent toute la gestion.

---

## 2. Création automatique des Sync Jobs

Chaque PBS distant possède un fichier `.conf` dans `remote_configs/`.  
Le script ci-dessous lit chaque fichier, récupère le hostname, le token, génère l’ID du job, et crée automatiquement un Pull Sync Job.

Il vérifie aussi si le Sync Job existe déjà.  
Si oui, il ne fait rien (cela évite les doublons).

```bash
#!/bin/bash

# -------------------------------------------------------------------
# Script : create_pull_sync_jobs.sh
# Objectif : Créer des pull sync jobs depuis PBS central 
# vers PBS distants, en utilisant les fichiers de ./remote_configs
# -------------------------------------------------------------------

CONFIG_DIR="./remote_configs"
LOCAL_STORE="backup-dr"   # Nom du datastore local sur le PBS central

echo ""
echo "===== Création des pull Sync Jobs ====="
echo ""

for CONF in "$CONFIG_DIR"/*.conf; do
    [ ! -f "$CONF" ] && continue

    source "$CONF"

    if [ -z "$HOST" ]; then
        echo "Aucun HOST dans $CONF, ignoré"
        continue
    fi

    if [ -z "$TOKEN_NAME" ]; then
        echo "Aucun TOKEN_NAME dans $CONF, ignoré"
        continue
    fi

    # Extraire le Remote ID complet depuis token_s44proxmoxbs
    REMOTE_ID="${TOKEN_NAME#token_}"     # -> s44proxmoxbs
    SHORT=$(echo "$REMOTE_ID" | cut -c1-3)
    NS="$SHORT"
    JOB_ID="sync_${SHORT}"
    REMOTE_DS="${SHORT}backup"

    echo ""
    echo "PBS distant : $HOST"
    echo "Remote ID : $REMOTE_ID"
    echo "Namespace central : $NS"
    echo "ID du job : $JOB_ID"
    echo "Datastore distant : $REMOTE_DS"

    # ---------------------------------------------------------------
    # NOUVELLE CONDITION : si le Job existe déjà → ne rien faire
    # ---------------------------------------------------------------
    if proxmox-backup-manager sync-job list | grep -q "^$JOB_ID "; then
        echo "Job $JOB_ID déjà existant -> aucune modification."
        continue
    fi

    echo "Création du pull Sync Job..."
    proxmox-backup-manager sync-job create "$JOB_ID" \
        --remote "$REMOTE_ID" \
        --remote-store "$REMOTE_DS" \
        --store "$LOCAL_STORE" \
        --ns "$NS" \
        --sync-direction pull

    if [ $? -eq 0 ]; then
        echo "Sync Job $JOB_ID créé avec succès"
    else
        echo "Échec lors de la création du Sync Job"
    fi
done

echo ""
echo "===== Terminé ====="
```


## 3. Ajout automatique des paramètres des Sync Jobs

Après la création des Sync Jobs, il faut leur ajouter plusieurs paramètres pour qu’ils fonctionnent correctement et de manière uniforme.  
Comme ces options ne peuvent pas toutes être ajoutées lors de la création, un second script s’occupe de les insérer directement dans le fichier `/etc/proxmox-backup/sync.cfg`.

Ce script fait deux choses :

1. il supprime d’anciennes lignes si elles existent (cela évite les doublons),  
2. il ajoute les paramètres nécessaires juste après chaque ligne `sync:` dans le fichier.

Voici le script :

```bash
#!/bin/bash

CONFIG_FILE="/etc/proxmox-backup/sync.cfg"

# Supprimer d'abord toute ancienne version des lignes concernées
sed -i '
/^\s*owner /d
/^\s*transfer-last /d
/^\s*encrypted-only /d
/^\s*remove-vanished /d
/^\s*resync-corrupt /d
' "$CONFIG_FILE"

# Ajouter les nouvelles lignes juste après chaque "sync:"
sed -i "/^sync:/a\\
	owner pve-backup@pbs\\
	transfer-last 1\\
	encrypted-only false\\
	remove-vanished false\\
	resync-corrupt true" "$CONFIG_FILE"

echo "✔️ Les paramètres ont été ajoutés à tous les jobs."
```


## 4. Explication des paramètres ajoutés automatiquement aux Sync Jobs

Après la création des Sync Jobs, un second script ajoute plusieurs paramètres importants dans le fichier `sync.cfg`.  
Ces paramètres permettent d’assurer un bon fonctionnement de la synchronisation et d’éviter des problèmes de cohérence ou de performance.  
Voici une explication simple et claire de chacun d’eux.

### ● owner pve-backup@pbs
Ce paramètre définit l’utilisateur propriétaire du Sync Job.  
Cela permet d’assurer que tous les jobs utilisent le même owner, ce qui facilite la gestion, les permissions et les contrôles administratifs.

---

### ● transfer-last 1
Ce réglage permet de ne transférer que la sauvegarde la plus récente lors du premier lancement du job.  
L’avantage est de réduire fortement le temps et la bande passante nécessaires à la première synchronisation, surtout si le site distant possède un grand nombre d’archives.

---

### ● encrypted-only false
Autorise la synchronisation de sauvegardes non chiffrées.  
Ce paramètre est important car si les PBS distants sont en version 3.4.x ou inférieur ils n’utilisent pas toujours le chiffrement.  
Sans lui, certains jobs pourraient refuser de s’exécuter.

---

### ● remove-vanished false
Empêche la suppression automatique de sauvegardes absentes sur le site distant.  
C’est utile pour éviter la perte de données si une machine ou une sauvegarde disparaît temporairement sur le PBS source.

---

### ● resync-corrupt true
Autorise la resynchronisation automatique d’une sauvegarde si celle-ci est détectée comme corrompue.  
Ce paramètre évite qu’un Sync Job reste bloqué à cause d’un fichier endommagé, et améliore la fiabilité globale de la synchronisation.

---

Grâce à ces paramètres, tous les Sync Jobs se comportent de manière identique et optimisée, ce qui améliore la stabilité et la cohérence de la réplication entre le PBS central et les PBS distants.
