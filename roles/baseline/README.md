# Rôle `baseline`

Socle commun à **toutes** les machines, Linux et Windows (MC01).

| Fonction | Linux | Windows |
|---|---|---|
| Nom d'hôte | `hostname` = nom d'inventaire, ligne `127.0.1.1` dans `/etc/hosts` | `win_hostname` (+ redémarrage si changé) |
| Fuseau horaire | `timezone_linux` | `timezone_windows` |
| NTP | `chrony` + `templates/chrony.conf.j2` | registre `W32Time\Parameters` |
| Compte admin | `admin_user` dans `sudo` | `admin_user` dans `win_admin_group` |

## Variables

| Variable | Défaut (rôle) | Valeur du projet | Où |
|---|---|---|---|
| `ntp_server` | `pool.ntp.org` | `0.fr.pool.ntp.org` (site B : `time.windows.com`) | `group_vars/all.yml`, `group_vars/site_b.yml` |
| `timezone_linux` | `Etc/UTC` | `Europe/Paris` | `group_vars/all.yml` |
| `timezone_windows` | `UTC` | `Romance Standard Time` | `group_vars/all.yml` |
| `admin_user` | `admin` | `sysadmin` | `group_vars/all.yml` |
| `admin_password` |  | `{{ vault_admin_password }}` | `group_vars/all.yml` + vault |
| `win_admin_group` |  | `Administrateurs` | `group_vars/windows.yml` |

## Notes

- Le mot de passe du compte n'est posé qu'à la création (`update_password: on_create`) : idempotence garantie.
- Les tâches de compte ont `no_log: true`.
