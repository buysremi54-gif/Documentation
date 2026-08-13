# Tests de performances FIO

## Contexte

Dans le cadre du POC Proxmox, plusieurs solutions de stockage ont été mises en concurrence afin de comparer leurs performances avant de faire un choix pour l'hébergement des machines virtuelles.

Les technologies évaluées sont :

- Ceph (réplication classique)
- Ceph Erasure Coding
- VMware vSAN

L'objectif n'était pas uniquement de comparer les débits, mais également de mesurer le comportement de chaque solution selon plusieurs types de charges représentatives d'un environnement de virtualisation.

Les tests ont été réalisés à l'aide de **FIO (Flexible I/O Tester)**.
 
Les métriques retenues sont :

- Débit (Bandwidth - BW)
- IOPS (Input / Output Per Second)
- Latence moyenne (clat)

---

# Environnement de test

Les essais ont été réalisés sur des machines virtuelles Debian configurées de manière identique.

Les seuls paramètres qui changent sont :

- la technologie de stockage
- la taille du fichier testé

Les essais sur disque dédié ont été réalisés avec trois tailles de fichiers :

- 128 Go
- 256 Go
- 1 To

Les mêmes scénarios ont ensuite été exécutés sur le disque système (`/`) afin de comparer le comportement des différentes solutions sur un volume de capacité plus faible (14 à 16 Go).

Tous les tests utilisent exactement les mêmes paramètres FIO afin que les résultats restent comparables.

---

# Paramètres FIO

Les scénarios ont été exécutés avec des paramètres identiques.

Principaux paramètres utilisés :

| Paramètre | Description |
|-----------|-------------|
| ioengine=libaio | Moteur d'entrées/sorties Linux |
| direct=1 | Ignore le cache du système d'exploitation |
| bs=4K | Taille de bloc adaptée aux accès des machines virtuelles |
| numjobs | Plusieurs processus en parallèle |
| iodepth | Plusieurs requêtes simultanées |
| group_reporting | Regroupe les statistiques des différents jobs |

L'utilisation de `direct=1` permet de mesurer les performances réelles du stockage sans être influencé par le cache mémoire du système.

Les tests en lecture et écriture aléatoires utilisent des blocs de **4 Kio**, ce qui correspond au type d'accès rencontré sur la majorité des machines virtuelles.

Les tests séquentiels utilisent des blocs plus importants afin de mesurer le débit maximal.

---

# Test n°1 - Charge mixte (`randrw`)

## Objectif

Une machine virtuelle ne fait jamais uniquement de la lecture ou uniquement de l'écriture.

Pendant son fonctionnement, elle lit ses fichiers système, écrit des journaux, met à jour ses données et effectue des accès aléatoires au stockage.

Le scénario `randrw` est celui qui se rapproche le plus d'une utilisation réelle d'un hyperviseur.

Le ratio utilisé est composé de lectures et d'écritures simultanées.

---

## Résultats - Disque dédié

| Solution | Lecture BW | Lecture IOPS | Lecture Latence | Écriture BW | Écriture IOPS | Écriture Latence |
|-----------|------------|--------------|-----------------|-------------|---------------|------------------|
| Erasure Coding | 124 MiB/s | 31.8 K | 1415 µs | 53.3 MiB/s | 13.6 K | 6043 µs |
| Ceph | 193 MiB/s | 49.4 K | 1103 µs | 82.7 MiB/s | 21.2 K | 3435 µs |
| vSAN | 89.3 MiB/s | 22.9 K | 3378 µs | 38.4 MiB/s | 9.8 K | 5145 µs |

---

## Résultats - Disque système (/)

| Solution | Lecture BW | Lecture IOPS | Lecture Latence | Écriture BW | Écriture IOPS | Écriture Latence |
|-----------|------------|--------------|-----------------|-------------|---------------|------------------|
| Erasure Coding | 96.4 MiB/s | 24.7 K | 4.45 ms | 41.4 MiB/s | 11 K | 10 ms |
| Ceph | 209 MiB/s | 53.6 K | 0.95 ms | 89.8 MiB/s | 23 K | 3 ms |
| vSAN | 107 MiB/s | 27.3 K | 1.46 ms | 45.8 MiB/s | 11.7 K | 7.5 ms |

---

## Analyse

Ce premier scénario met rapidement en évidence une différence entre les trois solutions.

Ceph obtient les meilleurs résultats aussi bien en lecture qu'en écriture. Le débit dépasse 190 MiB/s en lecture et 82 MiB/s en écriture sur le disque dédié tout en conservant les latences les plus faibles.

L'Erasure Coding arrive en seconde position. Les performances restent correctes mais la latence en écriture est nettement plus élevée.

vSAN ferme la marche sur ce scénario avec des débits plus faibles et un nombre d'IOPS inférieur aux deux autres solutions.

Sur le disque système, la tendance reste identique.

Ceph conserve son avance avec près de 209 MiB/s en lecture et une latence inférieure à la milliseconde.

Ce scénario est probablement le plus représentatif d'un environnement de virtualisation. Les machines virtuelles réalisent constamment des lectures et des écritures simultanément. Les résultats obtenus montrent donc un avantage clair pour Ceph sur ce type de charge.

---

# Test n°2 - Écriture continue (`write`)

## Objectif

Ce scénario mesure les performances en écriture séquentielle.

Contrairement au test précédent, les données sont écrites de manière continue. Ce type de charge est rencontré lors de copies de fichiers volumineux, de restaurations de sauvegardes, d'export de machines virtuelles ou encore lors de certaines opérations de migration.

L'objectif est de mesurer le débit maximal que chaque solution est capable de maintenir sur une longue phase d'écriture.

---

## Résultats - Disque dédié

| Solution | Taille | Débit (BW) | IOPS | Latence |
|-----------|---------|------------|-------|----------|
| Erasure Coding | 128 Go | 777 MiB/s | 7.77 K | 164 ms |
| Erasure Coding | 256 Go | 919 MiB/s | 9.19 K | 139 ms |
| Erasure Coding | 1 To | 1705 MiB/s | 1.705 K | 75 ms |
| Ceph | 128 Go | 1024 MiB/s | 1.024 K | 125 ms |
| Ceph | 256 Go | 680 MiB/s | 6.80 K | 188 ms |
| Ceph | 1 To | 590 MiB/s | 5.90 K | 216.5 ms |
| vSAN | 128 Go | 232 MiB/s | 2.32 K | 550 ms |
| vSAN | 256 Go | 217 MiB/s | 2.16 K | 590 ms |
| vSAN | 1 To | 229 MiB/s | 2.28 K | 558 ms |

---

## Résultats - Disque système (/)

| Solution | Taille | Débit (BW) | IOPS | Latence |
|-----------|---------|------------|-------|----------|
| Erasure Coding | 14 Go | 38.2 MiB/s | 9.8 K | 26 ms |
| Ceph | 14 Go | 110 MiB/s | 28.2 K | 9 ms |
| vSAN | 14 Go | 57.2 MiB/s | 14.6 K | 17.4 ms |

---

## Analyse

Les résultats montrent des différences importantes entre les trois technologies.

Sur le disque dédié, l'Erasure Coding obtient les meilleurs débits séquentiels, notamment sur le fichier de 1 To où il dépasse 1,7 Gio/s.

Ceph reste performant mais les résultats sont moins homogènes selon la taille du fichier testé. Malgré cela, les performances restent largement supérieures à celles obtenues avec vSAN.

vSAN est la solution la moins performante sur ce scénario. Les débits restent autour de 220 MiB/s quelle que soit la taille du fichier et la latence dépasse régulièrement les 500 ms.

Sur le disque système, Ceph reprend l'avantage avec près de 110 MiB/s, soit presque trois fois le débit obtenu avec l'Erasure Coding. Il conserve également la latence la plus faible.

Ce test montre que les performances en écriture séquentielle peuvent varier selon la technologie utilisée et la taille des données écrites. Les résultats obtenus sur le disque système restent néanmoins favorables à Ceph.

---

# Test n°3 - Lecture continue (`read`)

## Objectif

Ce test mesure les performances en lecture séquentielle.

Il reproduit des opérations où de gros volumes de données sont lus de manière continue, comme la lecture d'images ISO, la copie de fichiers volumineux, certaines restaurations ou les sauvegardes.

Contrairement aux accès aléatoires, les données sont lues de façon continue afin d'obtenir le débit maximal offert par chaque solution.

---

## Résultats - Disque dédié

| Solution | Taille | Débit (BW) | IOPS | Latence |
|-----------|---------|------------|-------|----------|
| Erasure Coding | 128 Go | 3056 MiB/s | 3 K | 42 ms |
| Erasure Coding | 256 Go | 3056 MiB/s | 3 K | 42 ms |
| Erasure Coding | 1 To | 6601 MiB/s | 6.601 K | 19 ms |
| Ceph | 128 Go | 4079 MiB/s | 4 K | 31 ms |
| Ceph | 256 Go | 5943 MiB/s | 5.9 K | 21 ms |
| Ceph | 1 To | 9773 MiB/s | 9.774 K | 13 ms |
| vSAN | 128 Go | 6083 MiB/s | 6 K | 21 ms |
| vSAN | 256 Go | 7839 MiB/s | 7.8 K | 16 ms |
| vSAN | 1 To | 13500 MiB/s | 13.8 K | 9 ms |

---

## Résultats - Disque système (/)

| Solution | Taille | Débit (BW) | IOPS | Latence |
|-----------|---------|------------|-------|----------|
| Erasure Coding | 14 Go | 167 MiB/s | 42.7 K | 6 ms |
| Ceph | 14 Go | 107 MiB/s | 27.3 K | 9.3 ms |
| vSAN | 14 Go | 348 MiB/s | 89.1 K | 2.8 ms |

---

## Analyse

La lecture séquentielle est le domaine où vSAN se démarque le plus.

Sur le disque dédié, les performances augmentent avec la taille des fichiers pour atteindre plus de 13 Gio/s sur le test de 1 To. Les IOPS suivent la même évolution tandis que la latence reste la plus faible des trois solutions.

Ceph arrive en deuxième position avec des performances élevées, notamment sur les volumes de 256 Go et 1 To. Les débits dépassent 9,7 Gio/s sur le plus gros jeu de données tout en conservant une latence contenue.

L'Erasure Coding reste plus en retrait, même si les résultats progressent également lorsque la taille du fichier augmente.

Sur le disque système, vSAN confirme son avance avec un débit supérieur à 340 MiB/s et une latence inférieure à 3 ms. Ceph obtient ici les résultats les plus faibles des trois solutions.

Ces mesures montrent que vSAN est particulièrement performant lorsqu'il s'agit de lire de gros volumes de données de manière continue.

---


# Test n°4 - Lecture aléatoire (`randread`)

## Objectif

Ce test mesure les performances en lecture aléatoire.

Contrairement à une lecture séquentielle, les données sont lues à différents emplacements du disque. C'est le type d'accès que l'on retrouve le plus souvent sur un hyperviseur : démarrage de plusieurs machines virtuelles, lecture de fichiers système, accès à des bases de données ou encore consultation de nombreux petits fichiers.

Ce scénario est donc particulièrement représentatif d'une infrastructure de virtualisation.

---

## Résultats - Disque dédié

| Solution | Taille | Débit (BW) | IOPS | Latence |
|-----------|---------|------------|-------|----------|
| Erasure Coding | 128 Go | 277 MiB/s | 71 K | 3.6 ms |
| Erasure Coding | 256 Go | 305 MiB/s | 78.1 K | 3.26 ms |
| Erasure Coding | 1 To | 980 MiB/s | 251 K | 1 ms |
| Ceph | 128 Go | 423 MiB/s | 108 K | 2.35 ms |
| Ceph | 256 Go | 923 MiB/s | 236 K | 1.07 ms |
| Ceph | 1 To | 2960 MiB/s | 758 K | 0.30 ms |
| vSAN | 128 Go | 1259 MiB/s | 322 K | 0.80 ms |
| vSAN | 256 Go | 2409 MiB/s | 617 K | 0.40 ms |
| vSAN | 1 To | 3806 MiB/s | 974 K | 0.26 ms |

---

## Résultats - Disque système (/)

| Solution | Taille | Débit (BW) | IOPS | Latence |
|-----------|---------|------------|-------|----------|
| Erasure Coding | 14 Go | 280 MiB/s | 71.7 K | 35.5 ms |
| Ceph | 14 Go | 331 MiB/s | 84.8 K | 3 ms |
| vSAN | 14 Go | 335 MiB/s | 85.8 K | 2.9 ms |

---

## Analyse

Les écarts deviennent beaucoup plus marqués sur ce scénario.

vSAN obtient les meilleurs résultats sur l'ensemble des capacités testées. Les débits dépassent 3,8 Gio/s sur le fichier de 1 To et les IOPS approchent le million d'opérations par seconde. La latence reste également la plus faible.

Ceph arrive juste derrière avec des performances très proches sur les volumes de grande capacité. Les débits progressent fortement lorsque la taille du fichier augmente et les temps de réponse restent inférieurs à la milliseconde sur le test de 1 To.

L'Erasure Coding reste plus limité. Les performances augmentent également avec la capacité mais restent nettement en dessous des deux autres solutions.

Sur le disque système, Ceph et vSAN obtiennent des résultats très proches. L'écart est faible aussi bien sur le débit que sur les IOPS. En revanche, l'Erasure Coding présente une latence beaucoup plus élevée, supérieure à 35 ms.

Dans un contexte où les machines virtuelles réalisent de nombreuses lectures aléatoires, vSAN offre les meilleures performances, suivi de très près par Ceph.

---

# Test n°5 - Écriture aléatoire (`randwrite`)

## Objectif

Ce dernier scénario mesure les performances en écriture aléatoire.

Il s'agit probablement du test le plus exigeant pour un système de stockage. Les écritures sont réparties sur l'ensemble du disque, ce qui sollicite fortement les mécanismes internes de gestion des données.

Ce type de charge correspond notamment aux bases de données, aux serveurs applicatifs, aux serveurs de fichiers ou encore aux machines virtuelles qui génèrent de nombreux journaux.

---

## Résultats - Disque dédié

| Solution | Taille | Débit (BW) | IOPS | Latence |
|-----------|---------|------------|-------|----------|
| Erasure Coding | 128 Go | 54 MiB/s | 13.8 K | 18.5 ms |
| Erasure Coding | 256 Go | 60.3 MiB/s | 15.4 K | 16.5 ms |
| Erasure Coding | 1 To | 47.6 MiB/s | 12.2 K | 20 ms |
| Ceph | 128 Go | 88.4 MiB/s | 22.6 K | 11.2 ms |
| Ceph | 256 Go | 87.4 MiB/s | 22.4 K | 11.3 ms |
| Ceph | 1 To | 76.2 MiB/s | 19.5 K | 13 ms |
| vSAN | 128 Go | 48.7 MiB/s | 12.5 K | 20.4 ms |
| vSAN | 256 Go | 46.1 MiB/s | 11.8 K | 21.5 ms |
| vSAN | 1 To | 44.3 MiB/s | 11.3 K | 22.4 ms |

---

## Résultats - Disque système (/)

| Solution | Taille | Débit (BW) | IOPS | Latence |
|-----------|---------|------------|-------|----------|
| Erasure Coding | 14 Go | 65.1 MiB/s | 16.7 K | 15.3 ms |
| Ceph | 14 Go | 145 MiB/s | 37.2 K | 6.8 ms |
| vSAN | 14 Go | 55.4 MiB/s | 14.2 K | 18 ms |

---

## Analyse

Ce scénario est celui où Ceph se démarque le plus.

Quel que soit le volume testé, Ceph obtient les meilleurs débits, le plus grand nombre d'IOPS ainsi que les latences les plus faibles.

Sur le disque dédié, les performances restent relativement stables entre les différentes tailles de fichiers. Les écritures aléatoires sont traitées plus efficacement que sur les deux autres solutions.

L'Erasure Coding obtient des résultats intermédiaires. Les performances restent correctes mais la latence est sensiblement plus élevée.

vSAN est la solution la moins performante sur ce scénario. Les débits restent inférieurs à ceux de Ceph et les temps de réponse dépassent régulièrement les 20 ms.

Les résultats sur le disque système confirment cette tendance. Ceph double quasiment les performances des deux autres solutions tout en conservant une latence deux à trois fois plus faible.

Pour une plateforme qui héberge plusieurs machines virtuelles en production, ce test est particulièrement important puisque les écritures aléatoires sont omniprésentes. Les résultats obtenus placent clairement Ceph en tête sur ce type de charge.

---

# Synthèse générale

Les cinq scénarios permettent d'observer le comportement des trois solutions selon différents types de charge.

| Scénario | Solution la plus performante |
|----------|------------------------------|
| Charge mixte (`randrw`) | Ceph |
| Écriture continue (`write`) | Erasure Coding (débit) / Ceph (sur `/`) |
| Lecture continue (`read`) | vSAN |
| Lecture aléatoire (`randread`) | vSAN |
| Écriture aléatoire (`randwrite`) | Ceph |

Chaque technologie possède donc un domaine dans lequel elle est particulièrement performante.

vSAN se distingue principalement sur les charges de lecture, aussi bien séquentielles qu'aléatoires.

Ceph offre les meilleurs résultats dès que les écritures entrent en jeu et reste très régulier sur l'ensemble des scénarios.

L'Erasure Coding conserve des performances correctes mais reste globalement en retrait. Son intérêt se situe davantage dans l'optimisation de la capacité de stockage que dans la recherche de performances maximales.

---

# Conclusion

Cette campagne de tests a permis de comparer les trois solutions dans les mêmes conditions en utilisant plusieurs scénarios représentatifs d'un environnement de virtualisation.

Les résultats montrent que les performances peuvent varier fortement selon le type de charge.

vSAN est particulièrement performant en lecture et obtient les meilleurs résultats sur les scénarios séquentiels ainsi que sur les lectures aléatoires.

À l'inverse, Ceph se montre beaucoup plus équilibré. Les performances en lecture restent élevées, mais c'est surtout sur les écritures, qu'elles soient séquentielles, aléatoires ou intégrées à une charge mixte, qu'il prend l'avantage.

L'Erasure Coding reste une solution intéressante lorsque l'objectif principal est d'optimiser l'espace de stockage. En contrepartie, les performances sont plus faibles sur la majorité des scénarios.

Au regard des résultats obtenus lors de cette campagne de tests, **Ceph apparaît comme le meilleur compromis pour un cluster Proxmox destiné à héberger des machines virtuelles en production**. Les performances sont homogènes, les temps de réponse restent faibles et le comportement observé correspond davantage aux charges rencontrées quotidiennement sur un environnement virtualisé.

Ces résultats ont servi de base pour orienter le choix de la solution de stockage retenue dans le cadre du POC.
