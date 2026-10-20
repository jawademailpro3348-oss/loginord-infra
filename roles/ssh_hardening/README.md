# Rôle `ssh_hardening`

Ferme l'accès SSH aux seuls administrateurs, par clé (MC03).

1. Groupe `ssh_allowed_group` créé, `ssh_admins` ajoutés dedans.
2. Clé publique du contrôleur déposée chez chaque admin.
3. `/etc/ssh/sshd_config.d/10-durcissement.conf` : pas de root, pas de mot de passe, `AllowGroups`.
4. Handler : `sshd -t` (validation), **puis** `reload` (ne coupe pas les sessions).

## Variables (`defaults/main.yml`)

| Variable | Défaut |
|---|---|
| `ssh_allowed_group` | `ssh-admins` (deviendra un groupe AD) |
| `ssh_admins` | `[ansible_user, admin_user]` (**le compte d'Ansible doit y être**) |
| `ssh_admin_pubkey` | `~/.ssh/id_ed25519.pub` du contrôleur |

## Précaution

Garder une seconde session SSH ouverte et la console de l'hyperviseur à portée de main lors du premier passage.
