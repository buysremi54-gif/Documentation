# Documentation de création d'un réseau SDN Proxmox

## Table des matières

1. [Vue d'ensemble](#vue-densemble)
2. [Prérequis](#prérequis)
3. [Création d'une Zone SDN](#création-dune-zone-sdn)
4. [Création d'un VNet](#création-dun-vnet)
5. [Application de la configuration SDN](#application-de-la-configuration-sdn)
6. [Association d'un VNet à une VM](#association-dun-vnet-à-une-vm)
7. [Vérification de la configuration](#vérification-de-la-configuration)

---

# Vue d'ensemble

Le SDN (*Software Defined Network*) de Proxmox permet de centraliser la gestion des réseaux virtuels au niveau du cluster.

Cette fonctionnalité permet de créer des réseaux virtuels (**VNet**) associés à des VLAN existants afin de simplifier l'affectation réseau des machines virtuelles.

Dans cette architecture, une **Zone SDN** représente un regroupement logique de réseaux virtuels. Chaque Zone contient un ou plusieurs **VNets**, eux-mêmes associés à un identifiant VLAN.

La hiérarchie utilisée est la suivante :

```
Zone SDN
│
└── VNet
    │
    └── VLAN associé
```

Exemple d'organisation :

```
Zone DCSTI
│
└── VNet DCSTI
        VLAN 2820


Zone HPAN
│
└── VNet HPAN
        VLAN 1318
```
---

# Prérequis

Avant de créer une configuration SDN, les éléments suivants doivent être disponibles :

- Accès à l'interface Web Proxmox avec des droits administrateur
- Un cluster Proxmox opérationnel
- Un bridge réseau existant sur les nœuds (exemple : `vmbr0`)
- Flux autorisés
- Une correspondance définie entre les VLANs et les réseaux utilisés

Exemple :

```
Bridge Proxmox : vmbr0

VLAN 2820 → Réseau DCSTI
VLAN 1318 → Réseau HPAN
```

---

# Création d'une Zone SDN

La première étape consiste à créer une Zone SDN.

Depuis l'interface Proxmox :

```
Datacenter
   > SDN
      > Zones
         > Add
```

Sélectionner le type :

```
VLAN
```

Renseigner les paramètres :

| Champ | Exemple |
|---|---|
| ID | DCSTI |
| Bridge | vmbr0 |

Valider avec :

```
Create
```

Une zone SDN nommée **DCSTI** est alors créée.

Architecture obtenue :

```
Zone DCSTI
```

---

Pour créer une seconde zone, répéter l'opération :

```
Datacenter
   > SDN
      > Zones
         > Add
```

Avec les paramètres :

| Champ | Exemple |
|---|---|
| ID | HPAN |
| Bridge | vmbr0 |

Résultat :

```
Zones SDN

├── DCSTI
└── HPAN
```

---

# Création d'un VNet

Une fois la Zone créée, il est nécessaire d'ajouter un VNet associé.

Depuis l'interface Proxmox :

```
Datacenter
   > SDN
      > VNets
         > Add
```

Renseigner les champs :

| Champ | Exemple |
|---|---|
| Name | DCSTI |
| Zone | DCSTI |
| VLAN Tag | 2820 |

Valider avec :

```
Create
```

Le VNet est maintenant associé à la zone correspondante.

Architecture obtenue :

```
Zone DCSTI
│
└── VNet DCSTI
        VLAN 2820
```

---

Créer ensuite le VNet HPAN :

| Champ | Exemple |
|---|---|
| Name | HPAN |
| Zone | HPAN |
| VLAN Tag | 1318 |

Résultat :

```
Zone DCSTI
│
└── VNet DCSTI
        VLAN 2820


Zone HPAN
│
└── VNet HPAN
        VLAN 1318
```

---

# Application de la configuration SDN

Après la création des Zones et VNets, les modifications apparaissent avec l'état :

```
pending
```

La configuration doit être appliquée manuellement.

Depuis :

```
Datacenter
   > SDN
```

Cliquer sur :

```
Apply
```

Proxmox déploie alors la configuration sur l'ensemble des nœuds du cluster.

Une fois terminée, l'état passe :

```
pending
```

à :

```
ok
```

La configuration SDN est alors active.

---

# Association d'un VNet à une VM

Pour connecter une machine virtuelle à un réseau SDN :

Depuis la VM :

```
Hardware
   > Network Device
```

Modifier ou ajouter une interface réseau.

Dans le champ :

```
Bridge
```

sélectionner le VNet correspondant.

Exemple :

```
VM DCSTI-01

Network Device
       |
       └── Bridge : DCSTI
```

Le VLAN associé au VNet est automatiquement appliqué.

Il n'est donc pas nécessaire de renseigner manuellement un VLAN Tag dans la configuration de la VM.

---

Exemple de répartition :

```
Zone DCSTI
│
└── VNet DCSTI
        VLAN 2820
        │
        ├── VM DCSTI-01
        └── VM DCSTI-02


Zone HPAN
│
└── VNet HPAN
        VLAN 1318
        │
        ├── VM HPAN-01
        └── VM HPAN-02
```

---

# Vérification de la configuration

Après application du SDN, la configuration peut être vérifiée directement depuis les nœuds Proxmox.

Afficher les interfaces réseau :

```bash
ip link show
```

Vérifier la présence des réseaux virtuels créés :

```
vnet-dcsti
vnet-hpan
```

Vérifier les VLAN associés :

```bash
bridge vlan show
```

Les VLAN configurés doivent apparaître sur les interfaces concernées.

---

# Résumé de l'architecture

```
Datacenter Proxmox

        SDN

         │

 ┌───────────────┐
 │ Zone DCSTI    │
 └───────────────┘
         │
         │
   VNet DCSTI
   VLAN 2820


 ┌───────────────┐
 │ Zone HPAN     │
 └───────────────┘
         │
         │
   VNet HPAN
   VLAN 1318
```
