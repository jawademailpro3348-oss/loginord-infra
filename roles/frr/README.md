# Rôle `frr`

Routage dynamique OSPF avec FRR (MC02). Chaque routeur annonce ses réseaux ; ses voisins les apprennent.

- Active `ospfd=yes` dans `/etc/frr/daemons`.
- Génère `/etc/frr/frr.conf` depuis l'inventaire (`templates/frr.conf.j2`).
- Redémarre FRR seulement si la configuration change.

## Variables

| Variable | Où | Exemple |
|---|---|---|
| `ospf_router_id` | `host_vars/rtr-*.yml` | `10.255.0.1` (unique dans le réseau OSPF) |
| `ospf_networks` | `host_vars/rtr-*.yml` | `[10.255.0.0/30, 10.1.10.0/24]` |
| `ospf_p2p_interfaces` | `group_vars/routers.yml` | `[wg0]` |
| `ospf_passive_interfaces` | `group_vars/routers.yml` | `["{{ lan_interface }}"]` |

Vérifier : `sudo vtysh -c "show ip ospf neighbor"` (état `Full`) et `sudo vtysh -c "show ip route ospf"`.
