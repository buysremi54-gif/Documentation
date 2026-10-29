Retrait d’un nœud du cluster Proxmox

## 1. Retrait des OSD

Dans l’interface Web Proxmox, sélectionner le nœud concerné, puis accéder à **Ceph → OSD**.

Pour chaque OSD du nœud :

1. Effectuer l’action **OUT** sur l’OSD.
2. Attendre que Ceph termine le rééquilibrage et que l’état du cluster revienne au vert.
3. Une fois le cluster stabilisé, effectuer **STOP** sur l’OSD.
4. Une fois l’arrêt terminé, effectuer **DESTROY** sur l’OSD.

Répéter ces opérations pour tous les OSD du nœud.

## 2. Retrait des Monitors et Managers

Dans l’interface Web Proxmox, accéder aux composants Ceph du nœud.

Pour les **Monitors**, effectuer les opérations suivantes :

1. **STOP** sur le Monitor.
2. **DESTROY** une fois l’arrêt terminé.

Faire de même pour le **Manager** du nœud.

Vérifier que les autres composants Ceph restent fonctionnels avant de poursuivre.

## 3. Mise en maintenance et arrêt du nœud

Une fois les opérations Ceph terminées :

1. Passer le nœud en **maintenance**.
2. Arrêter le serveur avec **Shutdown**.
3. Vérifier que le nœud est bien arrêté.

## 4. Suppression de la configuration du nœud

Après validation avec Thierry, se connecter sur **un autre nœud du cluster**.

### Vérifier la configuration Corosync

Consulter le fichier :

```bash
/etc/pve/corosync.conf
```

Repérer le bloc correspondant au nœud à supprimer et retirer uniquement ce bloc, après validation de la procédure.

Vérifier ensuite la configuration dans :

```bash
/etc/corosync/corosync.conf
```

S’assurer que les deux fichiers sont cohérents.

### Nettoyer le répertoire du nœud

Consulter les répertoires présents :

```bash
ls -la /etc/pve/nodes/
```

Repérer le répertoire correspondant au nœud retiré. Après vérification et validation, supprimer uniquement ce répertoire :

```bash
rm -rf /etc/pve/nodes/NOM_DU_NOEUD
```

Remplacer `NOM_DU_NOEUD` par le nom exact du nœud concerné.

**Attention :** ne jamais supprimer directement `/etc/pve/nodes` dans son intégralité. Cette opération pourrait affecter la configuration des autres nœuds du cluster.

## 5. Vérifications finales

Vérifier que :

- Le nœud a été retiré de la configuration Corosync.
- Les fichiers Corosync sont cohérents.
- Seul le répertoire de l’ancien nœud a été supprimé.
- Les autres nœuds du cluster fonctionnent normalement.






