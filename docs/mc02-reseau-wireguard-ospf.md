# MC02 : le réseau côté système

[Retour au README](../README.md)

## Objectif

Relier les deux sites de LogiNord : **un routeur Debian par site**, **un tunnel WireGuard chiffré** entre eux, et **des routes qui s'apprennent seules** grâce à OSPF.

> Besoins client : *« Nos deux sites doivent se parler »* et *« On pense ouvrir un troisième site. »*

### Consignes du TP

1. **Préparer** : 4 Debian (2 routeurs, 2 clients) sur 3 réseaux internes (LAN A, WAN, LAN B).
2. **Router** : un rôle routeur : IP, routage activé, nftables.
3. **Relier** : WireGuard entre `rtr-a` et `rtr-b`, clés dans le vault.
4. **Annoncer** : FRR et OSPF, chaque routeur annonce son LAN.

### Résultat attendu

- `traceroute` de client-a vers client-b passe par les deux routeurs.
- `show ip route ospf` affiche le LAN distant.
- Tout survit à un redémarrage.

## Le lab

```
client-a ── LAN A ── rtr-a ══ wg0 (dans le WAN) ══ rtr-b ── LAN B ── client-b
10.1.10.10  10.1.10.0/24      10.255.0.1   10.255.0.2      10.1.20.0/24  10.1.20.10
```

| | rtr-a | rtr-b | client-a | client-b |
|---|---|---|---|---|
| LAN (`enp0s9`) | 10.1.10.1/24 | 10.1.20.1/24 | 10.1.10.10/24 | 10.1.20.10/24 |
| WAN (`enp0s10`) | 192.0.2.1/24 | 192.0.2.2/24 |  |  |
| Tunnel `wg0` | 10.255.0.1/30 | 10.255.0.2/30 |  |  |
| router-id OSPF | 10.255.0.1 | 10.255.0.2 |  |  |

La création des VM (modèle Debian + 4 clones) est décrite dans [00-environnement.md](00-environnement.md#4-créer-les-debian--un-modèle-quatre-clones).

## Ce qui a été construit

`site.yml` gagne deux plays :

```yaml
- name: Adresses IP des machines du lab (LAN, WAN)
  hosts: routers:clients
  roles: [network]

- name: Routeurs (routage, pare-feu, tunnel, OSPF)
  hosts: routers
  roles: [router, wireguard, frr]
```

L'inventaire gagne les groupes `routers` et `clients` (et `linux` devient leur parent).

### Rôle `network` : une IP qui survit au redémarrage

Chaque interface est décrite dans `host_vars/` et devient **un fichier** `/etc/network/interfaces.d/<interface>` (ifupdown) :

```yaml
# host_vars/rtr-a.yml
interfaces:
  - name: enp0s9
    address: 10.1.10.1/24
  - name: enp0s10
    address: 192.0.2.1/24
```

Seules les interfaces **dont le fichier a changé** sont relancées (`ifdown` / `ifup`) : au second passage, rien n'est touché.
La carte Host-Only (`enp0s8`), par laquelle Ansible parle, n'est **jamais** gérée par ce rôle.

### Rôle `router` : faire d'une Debian un routeur

1. **Routage activé** : `net.ipv4.ip_forward = 1` dans `/etc/sysctl.d/99-routeur.conf`.
2. **Pare-feu nftables**, tout ce qui n'est pas autorisé est refusé :

| Chaîne | Politique | Autorisé |
|---|---|---|
| `input` (vers le routeur) | `drop` | connexions établies, `lo`, ICMP, **SSH sur la carte d'administration**, **UDP 51820 sur le WAN** (WireGuard), **OSPF (protocole 89) sur `wg0`** |
| `forward` (à travers le routeur) | `drop` | connexions établies, **du LAN vers `wg0`** et **de `wg0` vers le LAN** |

Le fichier est validé par `nft -c -f` **avant** d'être installé : une erreur de syntaxe ne peut pas casser le pare-feu.

### Rôle `wireguard` : le tunnel

```ini
[Interface]
Address    = 10.255.0.1/30
ListenPort = 51820
PrivateKey = <dans le vault>
Table      = off                              # c'est OSPF qui pose les routes, pas wg-quick
PostUp     = ip link set dev %i multicast on  # OSPF parle en multicast

[Peer]
PublicKey  = <clé publique de rtr-b>          # lue dans host_vars/rtr-b.yml
Endpoint   = 192.0.2.2:51820
AllowedIPs = 0.0.0.0/0                        # un seul pair : il accepte tout ce qu'OSPF lui envoie
```

- **Clés privées** : dans le vault (`vault_wg_private_key.rtr-a`, `.rtr-b`).
- **Clés publiques** : dans `host_vars/` (non secrètes). Le template va chercher celle du pair avec `hostvars[wg_peer].wg_public_key`.
- Le tunnel démarre au boot : `wg-quick@wg0` activé.

Génération des clés (sur le contrôleur) :

```bash
for r in rtr-a rtr-b; do k=$(wg genkey); echo "$r privée : $k"; echo "$r publique : $(echo $k | wg pubkey)"; done
```

### Rôle `frr` : OSPF en quelques lignes

`ospfd=yes` dans `/etc/frr/daemons`, puis `/etc/frr/frr.conf` généré depuis l'inventaire :

```
interface wg0
 ip ospf network point-to-point     ! un seul voisin : pas d'élection de routeur désigné
!
interface enp0s9
 ip ospf passive                    ! on annonce le LAN, mais on n'y cherche pas de voisin
!
router ospf
 ospf router-id 10.255.0.1
 network 10.255.0.0/30 area 0
 network 10.1.10.0/24 area 0
```

### Le choix pour les clients (à justifier, demandé par le cours)

L'interface NAT de VirtualBox donne une route par défaut aux clients. Deux options : la supprimer, ou ajouter des routes précises.

**Choix retenu** : les clients **gardent leur route par défaut sur la NAT** (pour `apt`), et **tout le réseau LogiNord `10.1.0.0/16` passe par le routeur du site**, grâce à une route posée par Ansible :

```yaml
# host_vars/client-a.yml
interfaces:
  - name: enp0s9
    address: 10.1.10.10/24
    routes:
      - to: 10.1.0.0/16
        via: 10.1.10.1
```

## Résultats

Second passage : `changed=0` sur les 4 machines.

**Tunnel** (sur rtr-a) :

```
$ sudo wg show
interface: wg0
  listening port: 51820
peer: p6mw…TRM=
  endpoint: 192.0.2.2:51820
  allowed ips: 0.0.0.0/0
  latest handshake: 30 seconds ago
```

**Voisinage OSPF**, à l'état `Full` :

```
$ sudo vtysh -c "show ip ospf neighbor"
Neighbor ID     Pri State     Up Time   Dead Time Address      Interface
10.255.0.2        1 Full/-    4m26s     36.040s   10.255.0.2   wg0:10.255.0.1
```

**Route apprise** :

```
$ sudo vtysh -c "show ip route ospf"
O   10.1.10.0/24 [110/100] is directly connected, enp0s9
O>* 10.1.20.0/24 [110/110] via 10.255.0.2, wg0
O   10.255.0.0/30 [110/10] is directly connected, wg0
```

**Test final** (sur client-a) :

```
$ traceroute -n 10.1.20.10
 1  10.1.10.1    (rtr-a)
 2  10.255.0.2   (rtr-b, à travers le tunnel)
 3  10.1.20.10   (client-b)
```

**Après redémarrage des 4 VM** : tunnel, voisinage OSPF, route et traceroute reviennent seuls.

- [x] le traceroute passe par les deux routeurs
- [x] le LAN distant est appris par OSPF
- [x] tout survit à un redémarrage

## Vérifier et dépanner

| Commande (sur un routeur) | Ce qu'on regarde |
|---|---|
| `ip -br a` | Les IP sont là, `wg0` existe |
| `ip route` | `10.1.20.0/24 via 10.255.0.2 dev wg0 proto ospf` |
| `sudo wg show` | Un « latest handshake » récent |
| `sudo vtysh -c "show ip ospf neighbor"` | État `Full` |
| `sudo nft list ruleset` | Les règles chargées |
| `sudo tcpdump -ni wg0 ip proto 89` | Les messages OSPF passent dans le tunnel |

Le tableau de dépannage complet est dans le tuto du cours (`TUTO_FRR_OSPF.md`, section 5) et dans [depannage.md](depannage.md).

## À retenir

- Une IP se configure par template, jamais à la main.
- Un routeur Debian : `ip_forward` et une chaîne `forward` en `policy drop`.
- `Table = off` : c'est OSPF qui pose les routes.
- Vérifier : `ip route`, `wg show`, `vtysh`.

## Bonus non réalisé : un 3e site

Ajouter `rtr-c` sans toucher à la configuration OSPF de `rtr-a` ni de `rtr-b` :

1. un nouveau tunnel (`wg1`, réseau `10.255.0.4/30`) entre `rtr-c` et le routeur auquel il se raccroche ;
2. un `host_vars/rtr-c.yml` avec ses réseaux et un `router-id` unique ;
3. sur le routeur de raccordement : la nouvelle interface WireGuard et sa ligne `network`.

Le LAN du 3e site apparaît alors tout seul dans la table de routage des autres. Le rôle `wireguard` actuel gère un seul tunnel par routeur (`wg0`) : il faudrait le généraliser à une liste de tunnels.
