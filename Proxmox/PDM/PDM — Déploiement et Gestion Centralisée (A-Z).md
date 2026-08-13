# PROXMOX DATACENTER MANAGER — DÉPLOIEMENT ET GESTION CENTRALISÉE

## Table des matières

- [Présentation générale](#présentation-générale)
- [Installation de Proxmox Datacenter Manager](#partie-1--installation-de-proxmox-datacenter-manager)
  - [Prérequis](#prérequis)
  - [Installation de l'ISO](#1-installation-de-liso)
  - [Configuration Initiale](#2-configuration-initiale)
  - [Configuration Finale](#3-configuration-finale)
- [Création des Remotes](#partie-2--création-des-remotes)
  - [Prérequis](#prérequis-1)
  - [Méthode via l'interface graphique](#1-méthode-via-linterface-graphique)
  - [Méthode automatisée via script](#2-méthode-automatisée-via-script)
  - [Vérifications](#3-vérifications)
- [Tests de Migration](#partie-3--tests-de-migration)
  - [Types de migration supportés](#types-de-migration-supportés)
  - [Migration intra-cluster](#1-migration-intra-cluster-nœud-à-nœud)
  - [Migration inter-cluster](#2-migration-inter-cluster-remote-à-remote)
  - [Vérifications post-migration](#3-vérifications-post-migration)
  - [Résolution des problèmes courants](#4-résolution-des-problèmes-courants)

---

## Présentation générale

Proxmox Datacenter Manager (PDM) est une solution de gestion centralisée permettant de superviser et d'administrer plusieurs clusters Proxmox VE depuis une interface unique.

Elle offre une vue globale des ressources (nœuds, VMs, CTs) de chaque cluster distant, et intègre des fonctionnalités avancées telles que les migrations inter-clusters.

Cette documentation couvre l'ensemble du cycle de déploiement de PDM :
1. **L'installation** de l'instance PDM
2. **La création des Remotes** pour connecter les clusters PVE existants
3. **Les tests de migration** pour valider le bon fonctionnement de l'ensemble

> PDM ne remplace pas Proxmox VE. Il s'agit d'une couche de gestion supplémentaire qui se connecte aux clusters existants via des **Remotes**.

---

# PARTIE 1 — INSTALLATION DE PROXMOX DATACENTER MANAGER

## 1. Installation de l'ISO

1. Se connecter via l'interface web sur l'hyperviseur cible.
2. Récupérer l'ISO de **Proxmox Datacenter Manager** depuis le site officiel.
3. Dans Proxmox VE, créer une VM et sélectionner l'ISO dans le lecteur CD-ROM. La VM démarrera automatiquement sur l'ISO.
4. Redémarrer la VM et choisir **Install Proxmox Datacenter Manager (Graphical)**.
5. Accepter la licence (End User License Agreement).
6. Sélectionner le disque cible et cliquer sur **Options** :
   - `SWAPSIZE` = 0
   - `MINFREE` = 0
7. Configurer la zone/langue :
   - COUNTRY : France
   - TIMEZONE : Europe/Paris
   - KEYBOARD : French
8. Définir le mot de passe administrateur et l'email :
   - Email : `renseigner un mail valide` (Ne pas supprimer le `@`)
9. Configuration réseau :
   - Interface : `ens18`
   - Hostname : `pdm.domaine.extension`
   - IP : `172.16.0.10/24`
   - Gateway : `172.16.0.1`
   - DNS : `172.16.0.2`
10. Vérifier le récapitulatif puis cliquer sur **Install**.
11. Une fois l'installation terminée, retirer l'ISO et redémarrer la VM.

---

## 2. Configuration Initiale

### Dépôts APT (`/etc/apt/sources.list`)

Mise en place des dépôts officiels pour la gestion des paquets sans abonnement entreprise.

```bash
# Dépôts Debian de base
deb http://deb.debian.org/debian bookworm main contrib
deb http://deb.debian.org/debian bookworm-updates main contrib
deb http://security.debian.org/debian-security bookworm-security main contrib

# Dépôt Proxmox sans abonnement
deb http://download.proxmox.com/debian/pdm bookworm pdm-no-subscription
```

> Le dépôt `pdm-no-subscription` est destiné aux environnements sans licence entreprise. Pour un usage en production, un abonnement Proxmox est recommandé.

Désactiver le dépôt entreprise s'il est présent :

```bash
# Commenter ou supprimer la ligne enterprise dans :
nano /etc/apt/sources.list.d/pdm-enterprise.list

# Exemple de ligne à commenter :
# deb https://enterprise.proxmox.com/debian/pdm bookworm pdm-enterprise
```

---

### APT config (`/etc/apt/apt.conf`)

Désactivation du téléchargement des fichiers de traduction pour accélérer les mises à jour.

```
Acquire::Languages "none";
```

---

### Mise à jour du système

```bash
apt update
apt upgrade -y
```

---

## 3. Configuration Finale

### Accès à l'interface web

Proxmox Datacenter Manager est accessible depuis un navigateur web via :

```
https://172.16.0.10:8443
```

Se connecter avec :
- **Utilisateur** : `root`
- **Realm** : `Linux PAM standard authentication`
- **Mot de passe** : celui défini lors de l'installation

> Le port par défaut de PDM est **8443**. S'assurer que ce port est ouvert dans les règles de pare-feu.

---

### Sécurisation de l'accès

Il est recommandé de créer un utilisateur dédié à la gestion et de ne pas utiliser `root` au quotidien.

1. Aller dans **Datacenter → Users → Add**
2. Renseigner :
   - **User ID** : `admin-pdm@pve`
   - **Group** : `Administrators`
   - **Password** : mot de passe fort
3. Valider

Créer également un token API pour les automatisations :

1. Aller dans **Datacenter → API Tokens → Add**
2. Renseigner :
   - **User** : `admin-pdm@pve`
   - **Token ID** : `pdm-token`
   - **Privilege Separation** : désactiver si accès total requis
3. **Copier et sauvegarder immédiatement le secret** (il ne sera plus visible ensuite)

---

### Vérification de l'installation

```bash
# Vérifier le statut du service PDM
systemctl status proxmox-datacenter-manager

# Vérifier la version installée
proxmox-datacenter-manager version

# Voir les logs en cas de problème
journalctl -u proxmox-datacenter-manager -f
```

---

### Vérification de la connectivité réseau vers les clusters PVE

```bash
# Tester la connectivité vers un nœud PVE distant
ping 172.16.0.20

# Tester l'accès à l'API PVE (port 8006)
curl -k https://172.16.0.20:8006/api2/json/version
```

> Si la commande `curl` retourne un JSON avec la version de PVE, la connectivité est confirmée.

---

# PARTIE 2 — CRÉATION DES REMOTES

Un **Remote** dans PDM correspond à la connexion vers un cluster Proxmox VE externe.
Une fois un Remote configuré, PDM peut superviser les nœuds, les VMs et les CTs de ce cluster, et effectuer des opérations de gestion comme les migrations.

Chaque Remote est identifié par :
- Une **ID** unique (nom logique dans PDM)
- L'**IP ou DNS** du nœud PVE cible (ou du VIP du cluster)
- Un **Token API** ou des credentials utilisateur PVE
- L'**empreinte TLS (Fingerprint)** du serveur cible

## Prérequis

| Élément | Détail |
|---|---|
| PDM installé et accessible | `https://172.16.0.10:8007` |
| Nœud(s) PVE joignables | Ping OK + port 8006 ouvert |
| Token API créé sur le PVE | Avec les droits suffisants (`PVEAuditor` minimum) |
| Fingerprint TLS récupéré | Via CLI ou interface PVE |

### Récupérer le Fingerprint d'un nœud PVE

Depuis le nœud PVE cible (en SSH) :

```bash
# Afficher le fingerprint du certificat TLS de PVE
openssl x509 -noout -fingerprint -sha256 \
  -in /etc/pve/nodes/$(hostname)/pve-ssl.pem
```

---

### Créer un Token API sur le cluster PVE distant

Depuis l'interface PVE cible :

1. Aller dans **Datacenter → Permissions → API Tokens → Add**
2. Renseigner :
   - **User** : `root@pam` ou un utilisateur dédié
   - **Token ID** : `pdm-sync`
   - **Privilege Separation** : désactiver pour un accès complet
3. **Copier et sauvegarder le secret immédiatement**

Attribuer les permissions nécessaires au token :

1. Aller dans **Datacenter → Permissions → Add → API Token Permission**
2. Renseigner :
   - **Path** : `/`
   - **API Token** : `root@pam!pdm-sync`
   - **Role** : `Administrator` (ou `PVEAuditor` si lecture seule)

---

## 1. Méthode via l'interface graphique

Cette méthode est adaptée lorsqu'il y a peu de clusters PVE à ajouter (1 à 3 maximum).

1. Dans PDM, aller dans **Remotes → Add**
2. Renseigner les champs :
   - **ID** : nom logique du remote (ex : `cluster-site-a`)
   - **Host** : IP ou DNS du nœud PVE (ex : `172.16.0.20`)
   - **Port** : `8006` (port par défaut de PVE)
   - **Auth ID** : token au format `user@realm!token-id` (ex : `root@pam!pdm-sync`)
   - **Token Secret** : secret généré lors de la création du token
   - **Fingerprint** : empreinte TLS du nœud PVE
3. Cliquer sur **Test Connection** pour vérifier la connectivité
4. Valider → le Remote apparaît dans la liste

> En environnement de production, toujours renseigner le **Fingerprint** manuellement pour éviter les attaques de type man-in-the-middle.

---

## 2. Méthode automatisée via script

Pour un grand nombre de clusters PVE, la méthode manuelle n'est pas adaptée.
Le script suivant automatise l'ajout des Remotes à partir de fichiers de configuration.

### Structure des fichiers

```
auto/
  ├── add_remotes_pdm.sh
  └── remote_configs/
      ├── cluster-site-a.conf
      ├── cluster-site-b.conf
      └── cluster-site-c.conf
       ...
```

---

### Fichiers de configuration

Chaque cluster PVE dispose de son propre fichier `.conf` dans `remote_configs/`.

Exemple — `cluster-site-a.conf` :

```bash
HOST="172.16.0.20"                                    # IP ou DNS du nœud PVE (ou VIP du cluster)
PORT="8006"                                            # Port API PVE (défaut : 8006 (si PVE sinon 8007 pour PBS))
TOKEN_USER="root@pam"                                  # Utilisateur propriétaire du token
TOKEN_ID="pdm-sync"                                    # Nom du token API
TOKEN_SECRET="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"   # Secret du token (à renseigner)
FINGERPRINT="AA:BB:CC:DD:..."                          # Fingerprint TLS du nœud PVE
```

> Ne jamais versionner les fichiers `.conf` contenant des secrets dans un dépôt public. Utiliser `.gitignore` pour les exclure.

---

### Script d'ajout des Remotes

Ce script parcourt tous les fichiers `.conf` présents dans `remote_configs/`, vérifie si le Remote existe déjà, et l'ajoute si nécessaire.

```bash
#!/bin/bash

# -------------------------------------------------------------------
# Script : add_remotes_pdm.sh
# Objectif : Ajouter automatiquement les Remotes PVE dans PDM
#            à partir des fichiers de ./remote_configs
# -------------------------------------------------------------------

CONFIG_DIR="./remote_configs"

echo ""
echo "===== Démarrage de l'ajout des Remotes PDM ====="
echo ""

for CONFIG_FILE in "$CONFIG_DIR"/*.conf; do

    # Vérifier que le fichier existe bien
    [ ! -f "$CONFIG_FILE" ] && continue

    # Charger les variables du fichier de config
    source "$CONFIG_FILE"

    # Vérifications des champs obligatoires
    if [ -z "$HOST" ] || [ -z "$TOKEN_USER" ] || [ -z "$TOKEN_ID" ] || [ -z "$TOKEN_SECRET" ]; then
        echo " Fichier incomplet : $CONFIG_FILE — ignoré"
        continue
    fi

    # Nom du remote = nom du fichier sans l'extension
    REMOTE_ID=$(basename "$CONFIG_FILE" .conf)

    # Construction de l'Auth ID au format PVE : user@realm!token-id
    AUTH_ID="${TOKEN_USER}!${TOKEN_ID}"

    echo "--- Remote : $REMOTE_ID (Host: $HOST) ---"

    # Vérifier si le Remote existe déjà dans PDM
    if proxmox-datacenter-manager remote list | grep -q "^$REMOTE_ID "; then
        echo " Remote '$REMOTE_ID' déjà existant → aucune modification."
        echo ""
        continue
    fi

    # Commande d'ajout du Remote dans PDM
    proxmox-datacenter-manager remote create "$REMOTE_ID" \
        --host "$HOST" \
        --port "${PORT:-8006}" \
        --auth-id "$AUTH_ID" \
        --token "$TOKEN_SECRET" \
        --fingerprint "$FINGERPRINT"

    # Vérification du résultat
    if [ $? -eq 0 ]; then
        echo "Remote '$REMOTE_ID' ajouté avec succès."
    else
        echo "Erreur lors de l'ajout de '$REMOTE_ID'."
    fi

    echo ""

done

echo "===== Terminé ====="
```

Attribuer les droits d'exécution puis lancer le script :

```bash
chmod +x add_remotes_pdm.sh

./add_remotes_pdm.sh    # Exécution du script
```

---

## 3. Vérifications

### Via l'interface graphique

Après l'ajout, vérifier la présence et le statut des Remotes dans PDM :

1. Aller dans **Remotes**
2. Chaque Remote doit afficher le statut **Online**
3. En cliquant sur un Remote, les nœuds et ressources du cluster PVE distant doivent apparaître

---

### Via la CLI

```bash
# Lister tous les Remotes configurés dans PDM
proxmox-datacenter-manager remote list

# Tester la connectivité vers un Remote spécifique
proxmox-datacenter-manager remote scan --remote cluster-site-a

# Voir les logs en cas d'erreur de connexion
journalctl -u proxmox-datacenter-manager -f
```

---

### Résolution des problèmes courants (Remotes)

| Problème | Cause probable | Solution |
|---|---|---|
| Remote en statut `Offline` | IP/port injoignable | Vérifier le réseau et le pare-feu (port 8006) |
| Erreur `Invalid fingerprint` | Fingerprint incorrect ou certificat changé | Récupérer le nouveau fingerprint avec `openssl` |
| Erreur `401 Unauthorized` | Token invalide ou expiré | Regénérer le token sur le cluster PVE |
| Remote visible mais aucune VM | Permissions insuffisantes | Vérifier le rôle associé au token (`Administrator`) |

---

# PARTIE 3 — TESTS DE MIGRATION

## Présentation

PDM permet d'effectuer des migrations de VMs et CTs entre nœuds d'un même cluster (**intra-cluster**) ou entre deux clusters PVE différents (**inter-cluster**) via l'interface centralisée.

Ces tests ont pour objectif de valider :
- La connectivité entre les clusters via les Remotes configurés
- La bonne migration des ressources (disques, configuration réseau, mémoire)
- La continuité de service pendant et après la migration
- Les performances et délais observés

> **Toujours effectuer un snapshot ou une sauvegarde de la VM avant de démarrer un test de migration**, en particulier pour les premiers tests.

---

## Types de migration supportés

| Type | Description | Interruption de service |
|---|---|---|
| **Live Migration** | VM migrée à chaud (RAM + CPU transférés) | Aucune (quelques ms de freeze) |
| **Offline Migration** | VM éteinte avant migration | Oui (le temps de la migration) |
| **Migration intra-cluster** | Entre nœuds d'un même cluster PVE | Selon le type choisi |
| **Migration inter-cluster** | Entre deux clusters PVE via PDM | Selon le type et le stockage |

---

## 1. Migration intra-cluster (nœud à nœud)

Ce type de migration déplace une VM ou un CT d'un nœud vers un autre au sein du même cluster PVE.

### Via l'interface graphique PDM

1. Dans PDM, naviguer vers **Remotes → `nom-du-cluster` → Nœuds**
2. Localiser la VM à migrer et cliquer dessus
3. Cliquer sur **Migrate**
4. Renseigner les paramètres :
   - **Target Node** : nœud de destination dans le même cluster (ex : `pve-node02`)
   - **Migration Type** : `Live Migration` (si VM allumée) ou `Offline` (si éteinte)
   - **With local disks** : cocher si la VM utilise du stockage local (non partagé)
5. Cliquer sur **Migrate** pour démarrer

> Si le stockage est partagé (Ceph, NFS, iSCSI), la migration live est possible sans déplacement de disque. Si le stockage est local, PDM copiera les disques vers le nœud cible.

---

### Via la CLI

Depuis le nœud PVE source (en SSH) :

```bash
# Migration live d'une VM vers un autre nœud (stockage partagé)
qm migrate <VMID> <nœud-cible> --online

# Exemple :
qm migrate 101 pve-node02 --online

# Migration offline (VM éteinte)
qm migrate 101 pve-node02

# Migration avec disques locaux (copie des disques)
qm migrate 101 pve-node02 --online --with-local-disks

# Suivre le statut de la VM
qm status 101
```

Pour les **conteneurs LXC** :

```bash
# Migration live d'un CT
pct migrate <CTID> <nœud-cible> --online

# Exemple :
pct migrate 201 pve-node02 --online
```

---

## 2. Migration inter-cluster (Remote à Remote)

Ce type de migration déplace une VM d'un cluster PVE vers un autre cluster PVE entièrement différent, géré depuis PDM.

> La migration inter-cluster est une fonctionnalité avancée. Elle nécessite une configuration réseau et de stockage adaptée entre les deux sites.

---

### Prérequis spécifiques

Avant d'effectuer une migration inter-cluster, vérifier :

```bash
# Depuis PDM — tester la connectivité entre les deux clusters
proxmox-datacenter-manager remote scan --remote cluster-site-a
proxmox-datacenter-manager remote scan --remote cluster-site-b

# Depuis le nœud PVE source — vérifier l'accès réseau vers le cluster cible
ping 172.16.0.30
curl -k https://172.16.0.30:8006/api2/json/version
```

S'assurer que les **règles de pare-feu** autorisent :

| Port | Protocole | Usage |
|---|---|---|
| 8006 | TCP | API Proxmox VE |
| 60000–60050 | TCP | Tunnel de migration live |
| 111, 2049 | TCP/UDP | NFS (si stockage partagé NFS) |

---

### Via l'interface graphique PDM

1. Dans PDM, aller dans **Remotes → `cluster-site-a` → VMs**
2. Sélectionner la VM à migrer
3. Cliquer sur **Migrate**
4. Renseigner les paramètres :
   - **Target Remote** : `cluster-site-b` (cluster de destination)
   - **Target Node** : nœud de destination (ex : `pve-node03`)
   - **Target Storage** : datastore de destination pour les disques
   - **Migration Type** : `Offline` recommandé pour inter-cluster
5. Vérifier le résumé et cliquer sur **Start Migration**
6. Suivre la progression dans la fenêtre de logs PDM

> La migration inter-cluster implique le transfert des disques via le réseau entre les deux sites. La durée dépend de la taille des disques et de la bande passante disponible.

---

### Via la CLI

```bash
# Depuis PDM — lancer une migration inter-cluster
proxmox-datacenter-manager migrate \
    --source-remote cluster-site-a \
    --source-node pve-node01 \
    --vmid 101 \
    --target-remote cluster-site-b \
    --target-node pve-node03 \
    --target-storage local-lvm

# Suivre la progression
proxmox-datacenter-manager task list --remote cluster-site-a
```

---

## 3. Vérifications post-migration

### Vérification de l'état de la VM

```bash
# Sur le nœud de destination — vérifier que la VM est bien présente
qm list

# Vérifier le statut de la VM
qm status <VMID>

# Démarrer la VM si elle était éteinte (migration offline)
qm start <VMID>
```

---

### Vérification réseau

```bash
# Depuis la VM migrée — vérifier la connectivité réseau
ping 172.16.0.1         # Gateway du réseau

# Vérifier la configuration IP
ip a

# Tester la résolution DNS
nslookup google.com
```

---

### Vérification des services

```bash
# Vérifier que les services applicatifs tournent correctement
systemctl status <nom-du-service>

---

### Vérification dans PDM

1. Dans PDM, naviguer vers **Remotes → `cluster-destination`**
2. Vérifier que la VM apparaît bien dans la liste avec le statut **Running**
3. Contrôler les métriques (CPU, RAM, réseau) dans la vue détaillée de la VM

---

### Vérification des sauvegardes

Après une migration inter-cluster, vérifier que les jobs de sauvegarde sont bien configurés pour la VM sur le nouveau cluster :

```bash
# Sur le nœud de destination — lister les jobs de backup
pvesh get /nodes/<nœud-destination>/jobs/schedule-analyze

# Ou via l'interface PVE du cluster cible :
# Datacenter → Backup → vérifier que la VM est bien incluse dans un job
```

---

## 4. Résolution des problèmes courants

| Problème | Cause probable | Solution |
|---|---|---|
| Migration bloquée à 0% | Réseau inter-nœuds KO | Vérifier les ports 60000–60050 et le pare-feu |
| Erreur `storage not found` | Stockage cible inexistant ou mal nommé | Vérifier le nom exact du datastore sur le nœud cible |
| VM inaccessible après migration | Config réseau perdue (VLAN, bridge) | Vérifier que le bridge réseau existe sur le nœud cible |
| Timeout lors d'une live migration | Latence réseau trop élevée | Passer en migration offline ou optimiser la bande passante |
| Erreur de permissions PDM | Token insuffisant | Attribuer le rôle `Administrator` au token PDM sur le PVE cible |
| Disques dupliqués après annulation | Migration annulée à mi-parcours | Nettoyer manuellement les disques orphelins sur le nœud cible |

---

### Annuler une migration en cours

```bash
# Lister les tâches en cours sur le nœud source
pvesh get /nodes/<nœud-source>/tasks

# Annuler la tâche
pvesh delete /nodes/<nœud-source>/tasks/<UPID>
```

> Une annulation en cours de migration peut laisser des artefacts (fichiers de disques temporaires) sur le nœud cible. Vérifier et nettoyer manuellement si nécessaire.

---

### Tableau de suivi des tests de migration

Il est recommandé de documenter chaque test dans ce tableau :

| Date | VM / CT | Source | Destination | Type | Durée | Résultat | Observé |
|---|---|---|---|---|---|---|---|
| JJ/MM/AAAA | VM-101 | cluster-site-a / pve-node01 | cluster-site-b / pve-node03 | Offline | 8 min | OK | RAS |
| JJ/MM/AAAA | VM-205 | pve-node01 | pve-node02 | Live | 45 sec | OK | RAS |
| JJ/MM/AAAA | CT-301 | cluster-site-a / pve-node02 | cluster-site-b / pve-node03 | Offline | 3 min | KO | Erreur storage |

> Conserver ce tableau de suivi dans la documentation GitLab pour garder une traçabilité des tests réalisés.
