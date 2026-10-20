# Dépannage — les problèmes rencontrés et leurs solutions

[← Retour au README](../README.md)

Tout ce qui a bloqué pendant les TP, classé par domaine. Le format : **symptôme → cause → solution**.

---

## VirtualBox

| Symptôme | Cause | Solution |
|---|---|---|
| La touche **Maj** ne fonctionne pas dans les VM ; « MAJ » affiché en bas à droite de la fenêtre | La **touche hôte** de VirtualBox était réglée sur Maj : VirtualBox la garde pour lui | Fichier → Paramètres → Entrée → Machine virtuelle → *Combinaison touche hôte* : **Ctrl droite** |
| **Ctrl+Alt+Suppr** ne déverrouille pas Windows | Le PC hôte capture la combinaison | Menu de la VM : **Entrée → Clavier → Insérer Ctrl-Alt-Suppr** (ou Ctrl droite + Suppr) |
| L'installation se fait toute seule, sans poser de questions (bureau installé, mauvais choix) | « Skip Unattended Installation » non cochée à la création de la VM | Cocher la case ; ou rattraper à la main (voir [00-environnement](00-environnement.md#41-le-modèle-deb-template)) |
| Windows : *« Une erreur s'est produite lors de la sélection de cette partition »* | Partition laissée par une installation précédente | Supprimer toutes les partitions → « Espace non alloué » → Suivant (sinon `diskpart` → `clean`) |
| Deux réseaux Host-Only proposés (`#2`) | Un autre réseau créé par GNS3 ou un essai | Toutes les VM sur **le même**, celui en `192.168.56.1/24` |

## Réseau Debian

| Symptôme | Cause | Solution |
|---|---|---|
| `apt` : *Temporary failure resolving 'deb.debian.org'* ; `ping 8.8.8.8` : *Network is unreachable* | La carte NAT (`enp0s3`) n'a pas d'IP, pas de route par défaut | `sudo nmcli device connect enp0s3` (NetworkManager) ou `sudo ifup enp0s3` |
| L'IP fixe disparaît au redémarrage, remplacée par une IP DHCP | Avec NetworkManager : le profil « Wired connection 1 » n'était lié à aucune interface et a pris `enp0s8` au démarrage | `nmcli con mod "Wired connection 1" connection.interface-name enp0s3` |
| Ansible écrit des fichiers d'interface qui ne sont pas appliqués / conflits | NetworkManager (installé avec un bureau) gère le réseau à la place d'ifupdown | Désactiver et masquer NetworkManager, installer `ifupdown` + `dhcpcd-base` |
| `enp0s9` / `enp0s10` en `DOWN` après le clonage | Normal : pas encore configurées | Le rôle `network` leur donne leur IP |
| Les clones ne répondent pas, conflit d'IP | Tous les clones démarrent avec l'IP du modèle (`.99`) | Les démarrer **un par un** et changer l'IP avant de lancer le suivant |
| L'heure des journaux est fausse d'une heure | VM mise en pause : l'horloge reprend en retard, chrony ne la corrige que lentement | `ansible linux -b -a "chronyc makestep"` |

## Windows Server Core

| Symptôme | Cause | Solution |
|---|---|---|
| `sconfig` option 8 : l'IP fixe n'est pas appliquée | Passerelle laissée vide = **annuler** | PowerShell : `Set-NetIPInterface … -Dhcp Disabled` puis `New-NetIPAddress …` |
| `Set-NetIPInterface -InterfaceIndex 10` : objet introuvable | L'« Index NIC » de `sconfig` n'est pas l'`ifIndex` de PowerShell | Utiliser `-InterfaceAlias "Ethernet 2"` (ou lire l'ifIndex avec `Get-NetAdapter`) |
| Clavier en QWERTY malgré `Set-WinUserLanguageList fr-FR` | La liste de langues ne change pas la méthode de saisie par défaut | `Set-WinDefaultInputMethodOverride -InputTip 040C:0000040C`, puis se reconnecter |
| Le ping vers dc02 ne répond pas | Pare-feu Windows | `Get-NetFirewallRule -Name *ICMP4-ERQ* \| Enable-NetFirewallRule` |
| `w32tm /query /source` → `Local CMOS Clock` | Pas encore synchronisé sur le NTP configuré | `w32tm /config /update` puis `w32tm /resync /force` |

## Ansible

| Symptôme | Cause | Solution |
|---|---|---|
| `Incorrect sudo password` | Le vault contenait le texte d'exemple, pas le vrai mot de passe | `ansible-vault edit vault/secrets.yml` |
| `ansible-vault edit` ouvre **vim** (Ctrl+O / Ctrl+X ne marchent pas) | La variable `EDITOR` n'est pas définie dans ce terminal | Dans vim : `Échap` puis `:wq`. Puis `echo 'export EDITOR=nano' >> ~/.bashrc` |
| Une tâche échoue avec un message *censored* | `no_log: true` masque l'erreur | Retirer temporairement `no_log`, relancer sur l'hôte concerné (`--limit`), puis le remettre |
| Windows refuse le mot de passe du compte `sysadmin` | Politique de complexité Windows | Au moins 8 caractères, 3 types (majuscules, minuscules, chiffres, symboles), sans le nom du compte |
| `UNREACHABLE … Connection timed out` | La VM est éteinte (ou injoignable) — ce n'est pas un refus SSH | Démarrer la VM, `ping`, relancer |
| `Permission denied (publickey)` pour Ansible | Clé absente ou utilisateur hors de `AllowGroups` | Vérifier `ssh_admins` dans le rôle `ssh_hardening`, se reconnecter via la console |
| L'empreinte SSH a changé (`REMOTE HOST IDENTIFICATION HAS CHANGED`) | La VM a été recréée | `ssh-keygen -R <ip>` puis se reconnecter |
| `ansible.cfg` ignoré (*world writable directory*) | Dépôt placé sous `/mnt/c/…` | Mettre le dépôt dans `~/` côté WSL |

## WireGuard / OSPF

| Symptôme | Cause | Solution |
|---|---|---|
| Pas de handshake dans `wg show` | Clés incohérentes (privée et publique de deux générations différentes) ou texte d'exemple dans le vault | Régénérer les paires, privée → vault, publique → `host_vars/` |
| Aucun voisin OSPF | Pare-feu (protocole 89), multicast désactivé sur `wg0`, tunnel en panne | `wg show`, `ip link show wg0` (MULTICAST), `nft list ruleset`, `tcpdump -ni wg0 ip proto 89` |
| Voisin en `Full` mais pas de route | Ligne `network` manquante | `vtysh -c "show running-config"` |
| Routes présentes mais le ping entre clients échoue | `ip_forward`, chaîne `forward`, route des clients | `sysctl net.ipv4.ip_forward`, `nft list chain inet filter forward`, `ip route` sur le client |
