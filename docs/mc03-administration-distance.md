# MC03 : administration à distance sécurisée

[Retour au README](../README.md)

## Objectif

Permettre aux administrateurs d'intervenir à distance **sans ouvrir la porte aux attaquants** : SSH durci côté Linux, PowerShell Remoting côté Windows.

> Besoins client : *« Mes informaticiens doivent pouvoir intervenir de chez eux, sans ouvrir la porte aux pirates »* et *« savoir qui a fait quoi ».*

### Consignes du TP

1. **Écrire** un rôle `ssh_hardening` : clé déposée, fichier de durcissement, validation, handler.
2. **Tester** : `ssh root@…` refusé ; `ssh -o PubkeyAuthentication=no …` refusé ; connexion par clé acceptée.
3. **Lire** les tentatives refusées dans `journalctl -u ssh`.
4. **Windows** : `Enter-PSSession` puis `Invoke-Command` sur le Windows Server Core.

### Résultat attendu

- Les 3 tests SSH donnent le bon résultat.
- Le rôle passe deux fois : `changed=0`.
- Une commande lancée à distance sur Windows.

## Partie Linux : le rôle `ssh_hardening`

Appliqué au groupe `linux` (les 4 Debian) par un nouveau play dans `site.yml`.

### Ce qu'il fait, dans cet ordre

| # | Tâche | Pourquoi |
|---|---|---|
| 1 | Crée le groupe `ssh-admins` | Seul ce groupe aura le droit d'entrer |
| 2 | Y ajoute `jawad` (le compte d'Ansible) et `sysadmin` | **Avant** le durcissement, sinon Ansible s'enferme dehors |
| 3 | Dépose la clé publique du contrôleur chez ces comptes | Une clé par personne : on sait qui se connecte |
| 4 | Écrit `/etc/ssh/sshd_config.d/10-durcissement.conf` | Le durcissement lui-même |
| 5 | Handler : `sshd -t`, puis `reload` | Une erreur de syntaxe arrête Ansible **avant** le rechargement ; `reload` ne coupe pas les sessions ouvertes |

### Le fichier de durcissement

```
PermitRootLogin no                 # on se connecte avec son compte, puis sudo
PasswordAuthentication no          # la force brute ne sert plus à rien
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AllowGroups ssh-admins             # seul ce groupe entre (au MC6 : un groupe AD)
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
```

Pourquoi un fichier dans `sshd_config.d/` : avec sshd, **la première valeur lue gagne**, et sur Debian ce dossier est inclus en tête de `sshd_config`. Notre fichier l'emporte donc sur les valeurs par défaut.

Le groupe autorisé est une **variable** (`ssh_allowed_group`) : il changera quand l'Active Directory arrivera.

### Précaution appliquée

Une **seconde session SSH ouverte** sur `rtr-a` pendant le premier passage, et la console VirtualBox à portée de main : si le durcissement avait mal tourné, on pouvait encore réparer.

## Résultats Linux

**Idempotence** : 1er passage `changed=6` sur chaque machine durcie, 2e passage **`changed=0`** sur les quatre.

**Les 3 tests** (depuis le contrôleur) :

```
$ ssh root@192.168.56.10
root@192.168.56.10: Permission denied (publickey).

$ ssh -o PubkeyAuthentication=no jawad@192.168.56.10
jawad@192.168.56.10: Permission denied (publickey).

$ ssh jawad@192.168.56.10 hostname
rtr-a
```

Au deuxième test, aucun mot de passe n'est même demandé : le serveur n'accepte plus que les clés.

**Les journaux** (`sudo journalctl -u ssh`) :

```
Accepted publickey for jawad from 192.168.56.1 port 34510 ssh2: ED25519 SHA256:7xUN…gvW8
User root from 192.168.56.1 not allowed because none of user's groups are listed in AllowGroups
Connection closed by invalid user root 192.168.56.1 port 34468 [preauth]
```

| Ligne | Lecture |
|---|---|
| `Accepted publickey for jawad … ED25519 SHA256:…` | Connexion réussie : **qui**, **d'où**, **avec quelle clé** |
| `not allowed because none of user's groups are listed in AllowGroups` | root bloqué par `AllowGroups` |
| `[preauth]` | Coupé **avant** toute authentification |

## Partie Windows : PowerShell Remoting

Depuis **PowerShell administrateur sur le PC hôte** :

```powershell
# dc02 n'est pas dans un domaine : le client doit lui faire confiance explicitement
Start-Service WinRM
Set-Item WSMan:\localhost\Client\TrustedHosts -Value 192.168.56.20 -Force

$cred = Get-Credential        # dc02\Administrateur

# Une session interactive
Enter-PSSession -ComputerName 192.168.56.20 -Credential $cred

# Une commande lancée à distance
Invoke-Command -ComputerName 192.168.56.20 -Credential $cred -ScriptBlock { w32tm /query /source; Get-Service W32Time }
```

### Résultats

```
PS> Enter-PSSession -ComputerName 192.168.56.20 -Credential $cred
[192.168.56.20]: PS C:\Users\Administrateur\Documents> hostname
dc02
[192.168.56.20]: PS C:\Users\Administrateur\Documents> Get-TimeZone
Id          : Romance Standard Time
DisplayName : (UTC+01:00) Bruxelles, Copenhague, Madrid, Paris

PS> Invoke-Command -ComputerName 192.168.56.20 -Credential $cred -ScriptBlock { … }
Status   Name      DisplayName      PSComputerName
Running  W32Time   Temps Windows    192.168.56.20
```

Le fuseau `Romance Standard Time` est celui posé par Ansible au MC01.

- [x] les 3 tests SSH donnent le bon résultat
- [x] `changed=0` au second passage
- [x] une commande lancée à distance sur Windows

## Linux et Windows, côte à côte

| | Linux | Windows |
|---|---|---|
| Accès | SSH | PowerShell Remoting (WinRM) ou SSH |
| Clés | `~/.ssh/authorized_keys` | `C:\ProgramData\ssh\administrators_authorized_keys` (pour un admin) |
| Durcissement | `sshd_config.d/` | GPO, pare-feu Windows |
| Journaux | `journalctl -u ssh` | `Get-WinEvent` : 4624 (succès) / 4625 (échec) |

## À savoir

- **Horloge des Debian après une mise en pause des VM** : chrony ne fait de grands sauts qu'au démarrage. Recaler avec `ansible linux -b -a "chronyc makestep" -e @vault/secrets.yml`. Des journaux à la mauvaise heure ne disent plus *quand* les choses se sont passées.

## Bonus non réalisé

OpenSSH sur Windows avec la clé (`Start-Service sshd`, clé dans `administrators_authorized_keys`), et `ssh web01` via un bastion (`ProxyJump` dans `~/.ssh/config`).

## À retenir

- Une clé par personne ; la clé privée ne quitte jamais le poste.
- Ni root, ni mot de passe sur SSH.
- `sshd -t` avant de recharger, et une seconde session ouverte.
- Les journaux disent qui a essayé d'entrer.
- Windows : PowerShell Remoting, ou SSH avec le bon fichier de clés.
