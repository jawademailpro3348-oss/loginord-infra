# L'environnement du lab

[Retour au README](../README.md)

Ce document décrit **tout ce qui est fait à la main, avant Ansible** : le poste, l'hyperviseur, les réseaux et la création des VM.
Le cours le dit : *« Avant Ansible, la VM doit être joignable en SSH ou WinRM. À documenter. »* C'est ce document.

## 1. Le poste de travail

| Élément | Choix | Pourquoi |
|---|---|---|
| Système hôte | Windows 11 | Le PC personnel |
| Hyperviseur | Oracle VirtualBox 7 | Déjà installé, gratuit |
| Contrôleur Ansible | **WSL 2 + Ubuntu 24.04** | Ansible ne tourne pas nativement sous Windows. WSL évite une VM de plus. |
| Éditeur | nano (dans WSL) | `export EDITOR=nano` dans `~/.bashrc` pour `ansible-vault edit` |

### Installer le contrôleur

PowerShell **administrateur** sur le PC :

```powershell
wsl --install -d Ubuntu-24.04
```

Puis dans Ubuntu :

```bash
sudo apt update
sudo apt install -y ansible python3-winrm python3-passlib wireguard-tools git unzip
echo 'export EDITOR=nano' >> ~/.bashrc
ssh-keygen -t ed25519 -C "jawad@loginord"
```

| Paquet | Rôle |
|---|---|
| `ansible` | Ansible et les collections (`ansible.windows`, `community.general`, `ansible.posix`…) |
| `python3-winrm` | Permet à Ansible de parler à Windows (WinRM) |
| `python3-passlib` | Filtre `password_hash` (mot de passe du compte Linux) |
| `wireguard-tools` | `wg genkey` / `wg pubkey` pour générer les clés du tunnel |

> Le dépôt vit dans `~/loginord-infra` (système de fichiers Linux). Sous `/mnt/c/…`, Ansible refuse de lire `ansible.cfg` (dossier « world writable »).
> Accès depuis l'Explorateur Windows : `\\wsl.localhost\Ubuntu-24.04\home\jawad\loginord-infra`.

## 2. Les réseaux VirtualBox

| Réseau | Type VirtualBox | Adresses | Usage |
|---|---|---|---|
| NAT | NAT (carte 1 de chaque VM) | 10.0.2.15, passerelle 10.0.2.2 | Accès Internet pour `apt` / Windows Update |
| Host-Only | « VirtualBox Host-Only Ethernet Adapter » | **192.168.56.0/24** (PC = .1) | **Administration** : SSH et WinRM depuis WSL |
| `lan-a` | Réseau interne | 10.1.10.0/24 | LAN du site A |
| `wan` | Réseau interne | 192.0.2.0/24 | Lien entre les deux routeurs (porte le tunnel) |
| `lan-b` | Réseau interne | 10.1.20.0/24 | LAN du site B |

Règle : **on ne touche jamais à la carte Host-Only avec Ansible** (« ne coupez pas la branche ») et **on ne met pas de passerelle** sur la carte Host-Only (la route par défaut reste sur la NAT).

> Si WSL ne joint pas le réseau Host-Only : créer `C:\Users\<vous>\.wslconfig` avec `[wsl2]` / `networkingMode=mirrored`, puis `wsl --shutdown`. (Pas nécessaire ici : le mode NAT par défaut de WSL a fonctionné.)

## 3. Les machines

| VM | Système | RAM | Carte 1 | Carte 2 | Carte 3 (`enp0s9`) | Carte 4 (`enp0s10`) | IP Host-Only |
|---|---|---|---|---|---|---|---|
| `rtr-a` | Debian 13 | 512 Mo à 1 Go | NAT | Host-Only | interne `lan-a` | interne `wan` | 192.168.56.10 |
| `rtr-b` | Debian 13 | 512 Mo à 1 Go | NAT | Host-Only | interne `lan-b` | interne `wan` | 192.168.56.11 |
| `client-a` | Debian 13 | 512 Mo à 1 Go | NAT | Host-Only | interne `lan-a` |  | 192.168.56.12 |
| `client-b` | Debian 13 | 512 Mo à 1 Go | NAT | Host-Only | interne `lan-b` |  | 192.168.56.13 |
| `dc02` | Windows Server 2025 Core (éval.) | 4 Go | NAT | Host-Only |  |  | 192.168.56.20 |

Les noms d'interface Debian suivent la position de la carte : carte 1 = `enp0s3`, 2 = `enp0s8`, 3 = `enp0s9`, 4 = `enp0s10` (vérifier avec `ip -br a`).

## 4. Créer les Debian : un modèle, quatre clones

### 4.1 Le modèle `deb-template`

1. VirtualBox > Nouvelle : ISO Debian 13 netinst, **cocher « Skip Unattended Installation »**, 1 Go RAM, 10 Go disque.
   Carte 1 NAT, carte 2 Host-Only.
2. Pendant l'installation :
   - clavier **Français** ;
   - interface réseau principale : **`enp0s3`** (la NAT) ;
   - mot de passe root **vide**, l'utilisateur reçoit alors `sudo` ;
   - utilisateur `jawad` (même mot de passe que `vault_linux_become_password`) ;
   - **Sélection des logiciels** (flèches + barre d'espace) : décocher l'environnement de bureau et GNOME, cocher **serveur SSH** et **utilitaires usuels du système** ;
   - GRUB : Oui, sur `/dev/sda`.
3. Carte Host-Only en IP fixe temporaire :
   ```bash
   sudo nano /etc/network/interfaces.d/mgmt
   ```
   ```
   auto enp0s8
   iface enp0s8 inet static
       address 192.168.56.99/24
   ```
   ```bash
   sudo ifup enp0s8
   ```
4. Depuis WSL : `ssh-copy-id jawad@192.168.56.99` (la clé sera dans tous les clones), puis `sudo poweroff`.

> **Si un bureau a été installé par erreur**, on peut rattraper sans réinstaller :
> ```bash
> sudo apt install -y openssh-server ifupdown dhcpcd-base
> # ajouter à /etc/network/interfaces :  allow-hotplug enp0s3 / iface enp0s3 inet dhcp
> sudo systemctl set-default multi-user.target
> sudo systemctl disable NetworkManager && sudo systemctl mask NetworkManager
> sudo systemctl enable ssh && sudo reboot
> ```
> NetworkManager (installé avec le bureau) entre en conflit avec les fichiers d'interface qu'Ansible écrit : il faut le désactiver.

### 4.2 Les clones

1. Clic droit sur le modèle > Cloner : **clone intégral**, **nouvelles adresses MAC pour toutes les cartes**. Noms : `rtr-a`, `rtr-b`, `client-a`, `client-b`.
2. Ajouter les cartes 3 et 4 (réseaux internes) selon le tableau ci-dessus, VM éteinte.
3. Démarrer les clones **un par un** (ils ont tous `.99` au départ) : modifier `/etc/network/interfaces.d/mgmt` avec l'IP définitive, puis `sudo reboot`.
4. Depuis WSL, accepter les empreintes une fois :
   ```bash
   for ip in 10 11 12 13; do ssh jawad@192.168.56.$ip hostname; done
   ```

Le nom d'hôte (`deb-template` dans les clones) est corrigé ensuite par Ansible (rôle `baseline`).

## 5. Créer le Windows Server `dc02`

1. ISO **Windows Server 2025 Évaluation** (Microsoft Evaluation Center), en français. VirtualBox : **cocher « Skip Unattended Installation »**, 4 Go RAM, 2 CPU, 50 Go.
   Carte 1 NAT, carte 2 Host-Only (le même réseau que les Debian).
2. Édition : **Windows Server 2025 Standard Evaluation** (sans « Expérience de bureau » = Core).
3. Disque : supprimer toute partition existante pour ne garder que « Espace non alloué ».
4. Après installation, dans `sconfig` : nom **DC02** (option 2), administration à distance **Activée** (option 4, c'est le défaut).
5. IP fixe de la carte Host-Only. `sconfig` exige une passerelle (vide = annuler), donc en PowerShell (option 15) :
   ```powershell
   Set-NetIPInterface -InterfaceAlias "Ethernet 2" -Dhcp Disabled
   New-NetIPAddress -InterfaceAlias "Ethernet 2" -IPAddress 192.168.56.20 -PrefixLength 24
   Get-NetFirewallRule -Name *ICMP4-ERQ* | Enable-NetFirewallRule     # autoriser le ping
   ```
6. Activer le compte Administrateur local (utilisé par Ansible) :
   ```powershell
   net user Administrateur * /active:yes
   ```
7. Clavier AZERTY si besoin : `Set-WinDefaultInputMethodOverride -InputTip 040C:0000040C`, puis se reconnecter.

> Dans VirtualBox : **Entrée > Clavier > Insérer Ctrl-Alt-Suppr** pour déverrouiller Windows (le vrai Ctrl+Alt+Suppr est capturé par le PC).

## 6. Vérifier que tout est joignable

Depuis WSL :

```bash
for ip in 10 11 12 13 20; do ping -c 1 -W 2 192.168.56.$ip; done
nc -zv 192.168.56.20 5985                                       # WinRM
ansible linux   -m ansible.builtin.ping     -e @vault/secrets.yml
ansible windows -m ansible.windows.win_ping -e @vault/secrets.yml
```

Quatre `pong` côté Linux et un côté Windows : l'environnement est prêt, la suite se fait avec Ansible.

## Récapitulatif des gestes manuels

| Où | Geste | Pourquoi Ansible ne peut pas le faire |
|---|---|---|
| VirtualBox | Créer les VM, les cartes réseau | Hors des machines |
| Debian | IP Host-Only (`/etc/network/interfaces.d/mgmt`), `openssh-server` | Sans IP ni SSH, Ansible ne peut pas se connecter |
| Debian | `ssh-copy-id` | Première clé déposée à la main |
| Windows | IP de « Ethernet 2 », compte Administrateur activé | Sans IP ni compte, WinRM est injoignable |
| WSL | `~/.vault_pass` | Le secret qui ouvre tous les autres, jamais dans Git |
