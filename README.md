# LogiNord – infra Ansible (MC01)

## Prérequis manuels (avant Ansible)
- rtr-a (Debian 13) : IP 192.168.56.10 sur la carte Host-Only (nmcli, connexion "mgmt"),
  openssh-server installé, clé SSH déposée avec `ssh-copy-id jawad@192.168.56.10`
- dc02 (Windows Server 2025 Core) : IP 192.168.56.20 sur "Ethernet 2",
  compte Administrateur activé, WinRM actif (par défaut)
- Contrôleur : WSL Ubuntu avec ansible, python3-winrm, python3-passlib
- Mot de passe du vault dans ~/.vault_pass (hors du dépôt)

## Lancer
ansible-playbook site.yml

## MC02 : choix réseau
- Les clients gardent leur route par défaut sur la carte NAT (pour apt).
  Le réseau LogiNord (10.1.0.0/16) passe par le routeur du site, via une route posée par Ansible.
- Les clés privées WireGuard sont dans le vault ; les clés publiques dans host_vars.
- Seul geste manuel par VM : l'IP Host-Only (fichier mgmt) et la clé SSH (ssh-copy-id).
