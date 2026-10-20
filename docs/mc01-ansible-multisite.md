# MC01 : Ansible pour une infra multi-site

[Retour au README](../README.md)

## Objectif

Passer de « un playbook pour deux ou trois machines » à **un projet Ansible structuré** capable de construire deux sites, Linux et Windows, et qu'un autre administrateur peut reprendre.

> Besoin client : *« Si mon informaticien part, un autre doit pouvoir tout reconstruire. »*

### Consignes du TP

1. **Préparer** : 1 Debian (site A) et 1 Windows Server Core (site B), joignables en SSH et WinRM.
2. **Structurer** : le dépôt, un inventaire par site et par rôle, `group_vars`, `ansible.cfg`.
3. **Écrire** : un rôle `baseline` (nom d'hôte, fuseau horaire, NTP, compte admin) pour Linux **et** Windows.
4. **Protéger** : le mot de passe Windows dans le vault.

### Résultat attendu

- `site.yml` passe sans erreur.
- Le 2e passage affiche `changed=0`.
- Aucun secret en clair dans Git.

## Ce qui a été construit

### Une structure standard

Pas de dossier par machine, pas de playbook par machine : la structure que tout administrateur Ansible reconnaît (voir [la structure du dépôt](../README.md#structure-du-dépôt)).

`ansible.cfg` fixe l'inventaire et le fichier du mot de passe vault :

```ini
[defaults]
inventory           = inventory/hosts.yml
roles_path          = roles
vault_password_file = ~/.vault_pass
interpreter_python  = auto_silent
playbook_dir        = .
```

### Un inventaire à deux axes

Chaque machine appartient à **un site** et à **un rôle** ; ses variables viennent des deux.

```yaml
site_a:   { hosts: { rtr-a: … } }
site_b:   { hosts: { dc02: … } }
linux:    { hosts: { rtr-a: } }
windows:  { hosts: { dc02: } }
```

```
@all
$ ansible-inventory --graph
@all:
  |--@ungrouped:
  |--@site_a:
  |  |--rtr-a
  |--@site_b:
  |  |--dc02
  |--@linux:
  |  |--rtr-a
  |--@windows:
  |  |--dc02
```

### La précédence des variables, démontrée

Le serveur NTP est défini à trois niveaux. **Le plus précis gagne.**

| Niveau | Fichier | Valeur |
|---|---|---|
| Défaut du rôle | `roles/baseline/defaults/main.yml` | `pool.ntp.org` |
| Tous | `group_vars/all.yml` | `0.fr.pool.ntp.org` |
| Site B | `group_vars/site_b.yml` | `time.windows.com` *(plus tard : le DC du site)* |

```bash
ansible-inventory --host rtr-a | grep ntp     # 0.fr.pool.ntp.org   (all)
ansible-inventory --host dc02  | grep ntp     # time.windows.com    (site_b gagne)
```

### Le rôle `baseline`

Un seul rôle, deux fichiers de tâches choisis selon le système (`ansible_os_family`) :

| Fonction | Linux (`tasks/linux.yml`) | Windows (`tasks/windows.yml`) |
|---|---|---|
| Nom d'hôte | `ansible.builtin.hostname` + ligne `127.0.1.1` dans `/etc/hosts` | `ansible.windows.win_hostname` (+ redémarrage si besoin) |
| Fuseau horaire | `community.general.timezone` (`Europe/Paris`) | `community.windows.win_timezone` (`Romance Standard Time`) |
| NTP | paquet `chrony` + template `chrony.conf.j2` | registre `W32Time\Parameters` (`NtpServer`, `Type=NTP`) |
| Compte admin | `sysadmin`, groupe `sudo` | `sysadmin`, groupe `Administrateurs` |

Choix notables :

- **Handlers** : chrony / w32time ne redémarrent que si leur configuration a changé.
- **`update_password: on_create`** : le mot de passe n'est posé qu'à la création du compte. Sans ça, `password_hash` (sel aléatoire) changerait le compte à chaque passage et casserait l'idempotence.
- **`no_log: true`** sur les tâches de compte : le mot de passe n'apparaît jamais dans la sortie.

### Parler à Windows

Les variables de connexion sont dans `group_vars/windows.yml`, donc tous les Windows en héritent :

```yaml
ansible_connection: winrm
ansible_port: 5985
ansible_winrm_transport: ntlm
ansible_user: Administrateur
ansible_password: "{{ vault_win_password }}"
```

Rien à installer sur Windows : WinRM est actif par défaut sur Windows Server, et Ansible y envoie du PowerShell. Côté contrôleur, il faut `python3-winrm`.

### Les secrets

```bash
nano ~/.vault_pass && chmod 600 ~/.vault_pass       # le mot de passe du vault, hors du dépôt
ansible-vault create vault/secrets.yml              # vault_win_password, vault_linux_become_password, vault_admin_password
```

## Résultats

Second passage :

```
PLAY RECAP
dc02    : ok=8   changed=0   unreachable=0   failed=0   skipped=2
rtr-a   : ok=9   changed=0   unreachable=0   failed=0   skipped=1
```

Secrets :

```bash
git show HEAD:vault/secrets.yml | head -1
$ANSIBLE_VAULT;1.1;AES256
```

- [x] `site.yml` passe sans erreur
- [x] `changed=0` au second passage
- [x] aucun secret en clair dans Git

## Difficultés rencontrées

| Problème | Cause | Solution |
|---|---|---|
| `Incorrect sudo password` | Le vault contenait encore le texte d'exemple | Mettre les vraies valeurs (`ansible-vault edit`) |
| Création du compte Windows en échec (message masqué) | `no_log: true` cache l'erreur | Retirer temporairement `no_log`, lancer avec `--limit dc02` : *« Ce mot de passe ne correspond pas aux critères de stratégie »* |
| Mot de passe refusé par Windows | Politique de complexité (longueur, 3 types de caractères, pas le nom du compte) | Mot de passe conforme dans `vault_admin_password` |
| Le NTP Windows changé mais pas pris en compte | Le play s'est arrêté sur dc02 avant les handlers | Redémarrer `w32time` une fois à la main : `ansible windows -m ansible.windows.win_service -a "name=w32time state=restarted"` |
| `w32tm /query /source` répondait encore `Local CMOS Clock` | Le service n'avait pas relu sa configuration | `w32tm /config /update` puis `w32tm /resync /force`, la source devient `time.windows.com,0x9` |

Voir aussi [le dépannage général](depannage.md).

## À retenir

- Une structure standard que tout le monde reconnaît.
- L'inventaire a deux axes : le site et le rôle.
- La variable la plus précise gagne.
- Les secrets dans le vault, le mot de passe du vault hors du dépôt.
- Le second passage ne change rien.

## Bonus non réalisé

*Ajouter un 3e site dans l'inventaire sans toucher au rôle* : il suffirait d'un groupe `site_c` dans `inventory/hosts.yml` et d'un `group_vars/site_c.yml`. Le professeur a préféré passer directement au MC02.
