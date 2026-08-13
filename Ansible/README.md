
# Ansible

Ansible est utilisé pour automatiser la configuration, la maintenance et les mises à jour des serveurs.

## Arborescence

```text
/opt/ansible/
├── hosts -> /etc/ansible/hosts
├── conf_proxmox_9/
├── playbooks/
└── scripts/
```

### `conf_proxmox_9/`

Contient les fichiers de configuration propres à l'environnement **Proxmox VE 9**.

### `playbooks/`

Contient les playbooks Ansible utilisés pour les différentes opérations d'administration.

### `scripts/`

Contient les scripts complémentaires utilisés par Ansible, notamment les scripts Bash de maintenance.

### `hosts`

Le fichier d'inventaire réel se trouve dans :

```text
/etc/ansible/hosts
```

Un lien symbolique est présent dans `/opt/ansible/` :

```text
/opt/ansible/hosts -> /etc/ansible/hosts
```

Cela permet de conserver un seul inventaire tout en le gardant accessible depuis le répertoire Ansible.

## Utilisation

Tester la connexion aux serveurs :

```bash
ansible all -m ping
```

Lancer un playbook :

```bash
ansible-playbook /opt/ansible/playbooks/<playbook>.yml
```

Limiter l'exécution à un groupe :

```bash
ansible-playbook /opt/ansible/playbooks/<playbook>.yml --limit PROXMOX_9
```
