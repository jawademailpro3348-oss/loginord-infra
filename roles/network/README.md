# Rôle `network`

Configure les interfaces du lab (LAN, WAN), avec **une interface = un fichier** : `/etc/network/interfaces.d/<nom>` (MC02).

## Variables (dans `host_vars/`)

```yaml
interfaces:
  - name: enp0s9
    address: 10.1.10.10/24
    routes:                 # facultatif
      - to: 10.1.0.0/16
        via: 10.1.10.1
```

## Comportement

- Installe `traceroute` et `tcpdump`.
- Ne relance (`ifdown` / `ifup`) **que** les interfaces dont le fichier a changé.
- Ne touche **jamais** à la carte d'administration (`enp0s8`), par laquelle Ansible se connecte.
- Prérequis : la machine utilise ifupdown (pas NetworkManager).
