# Ajout des remotes sur PDM

## Table des matières
- [Ajout des remotes PVE/PBS distants sur PDM](#ajout-des-remotes-pve/pbs-distants-sur-pdm)
  

---


## Ajout des remotes PVE/PBS distants sur PDM

### 1. Méthode via l'interface graphique

Cette méthode permet d'ajouter un remote manuellement depuis l'interface web du Proxmox Datacenter Manager.

1.  Aller dans **Remotes**\
2.  Cliquer sur **Add**\
3.  Renseigner :
    -   **Server Address** : IP x port \
    -   **Fingerprint** : PBS -> Dashboard -> Show Fingerprint | PVE -> Certificates -> View : pve-ssl.pem -> Fingerprint\
    
Pour continuer, faire **Login and create new token** car PDM gére entierement ces tokens.
Ensuite pour le créer, il faut renseigner les informations d'un utilisateur du PVE ou PBS

4.  Finish → le remote est ajoutée

------------------------------------------------------------------------




