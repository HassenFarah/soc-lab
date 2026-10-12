# Réseau et interconnexion (Tailscale)

Les machines du lab sont sur des hôtes et réseaux différents. Elles communiquent via un réseau maillé **Tailscale**.

## Plan d'adressage (à adapter)

| Machine | Rôle | Adresse LAN | Adresse Tailscale |
|---|---|---|---|
| WINDS | Contrôleur de domaine `LAB.LOCAL` | 192.168.50.10 | `100.x.y.z` |
| LinuxSRV | TheHive, Cortex, MISP (Docker) | variable | `100.x.y.z` |
| Serveur Wazuh | Wazuh manager/indexer/dashboard | variable | `100.x.y.z` |

> Recommandation : pour la communication inter-services, utiliser **uniquement les adresses Tailscale**. Les adresses LAN peuvent disparaître ou changer (cas rencontré dans le lab).

## Installer Tailscale (Linux)

Suivre https://tailscale.com/download/linux puis :

```bash
sudo tailscale up
tailscale ip -4      # affiche l'adresse 100.x.y.z de la machine
tailscale status     # liste des pairs
```

Windows : installer le client depuis https://tailscale.com/download/windows et se connecter avec le même compte/tailnet.

## Ports à autoriser entre les machines

| Flux | Source → Destination | Port |
|---|---|---|
| Agent Wazuh → manager | WINDS → Wazuh | 1514 (TCP/UDP selon config), 1515 (enrôlement) |
| Wazuh → TheHive | Wazuh → LinuxSRV | 9000/TCP |
| TheHive → Cortex | LinuxSRV (interne) | 9001/TCP |
| TheHive → MISP | LinuxSRV | 443/TCP (ou 80) |
| Accès admin (navigateur) | poste admin → services | 443 (Wazuh, MISP), 9000, 9001 |

## Test de connectivité

```bash
# Depuis la machine source
ping -c 3 100.x.y.z
curl -sv http://100.x.y.z:9000/api/status
nc -zv 100.x.y.z 1514
```
