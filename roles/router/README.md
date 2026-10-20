# Rôle `router`

Fait d'une Debian un routeur (MC02).

- `net.ipv4.ip_forward = 1` dans `/etc/sysctl.d/99-routeur.conf`.
- Pare-feu nftables (`templates/nftables.conf.j2`), validé par `nft -c -f` avant installation :
  - `input` en `policy drop` : établi, `lo`, ICMP, SSH sur `mgmt_interface`, UDP `wg_port` sur `wan_interface`, OSPF sur `wg0` ;
  - `forward` en `policy drop` : établi, et dans les deux sens entre `lan_interface` et `wg0`.

## Variables

| Variable | Où | Exemple |
|---|---|---|
| `mgmt_interface` | `group_vars/routers.yml` | `enp0s8` |
| `wg_port` | `group_vars/routers.yml` | `51820` |
| `lan_interface`, `wan_interface` | `host_vars/rtr-*.yml` | `enp0s9`, `enp0s10` |
