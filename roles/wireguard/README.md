# Rôle `wireguard`

Monte le tunnel `wg0` entre les deux routeurs (MC02).

- `Table = off` : wg-quick ne pose pas de routes, c'est OSPF qui décide.
- `PostUp … multicast on` : OSPF communique en multicast.
- `AllowedIPs = 0.0.0.0/0` : un seul pair par tunnel, il accepte tous les réseaux annoncés.
- Service `wg-quick@wg0` activé au démarrage.

## Variables

| Variable | Où | Rôle |
|---|---|---|
| `wg_address` | `host_vars/rtr-*.yml` | Adresse du routeur dans le tunnel (`10.255.0.1/30`) |
| `wg_peer` | `host_vars/rtr-*.yml` | Nom d'inventaire du pair (`rtr-b`) |
| `wg_public_key` | `host_vars/rtr-*.yml` | Clé publique **de cette machine** (lue par le pair) |
| `wan_ip` | `host_vars/rtr-*.yml` | IP WAN (sert d'`Endpoint` au pair) |
| `vault_wg_private_key[<hôte>]` | vault | Clé privée, jamais en clair |

Générer une paire : `k=$(wg genkey); echo "$k"; echo "$k" | wg pubkey`.
