# Prérequis communs

## Matériel (indicatif, lab)

| Composant | RAM minimale conseillée | Remarque |
|---|---|---|
| Wazuh (stack complète) | 4 Go (8 Go confortable) | L'indexer (OpenSearch) est gourmand |
| TheHive + Cassandra + Elasticsearch | 6 à 8 Go | Cassandra + ES sont les plus lourds |
| Cortex + Elasticsearch | 2 à 3 Go | + mémoire des conteneurs d'analyzers |
| MISP (core, db, redis, modules, mail) | 4 Go | |

Ces valeurs sont des ordres de grandeur pour un lab : vérifiez les exigences officielles de chaque version avant un déploiement durable.

## Logiciels

- Ubuntu Server 22.04 (utilisé dans ce lab) ou équivalent
- Docker Engine + plugin Compose v2 (`docker compose`, sans tiret)
- `git`, `curl`, `jq`

### Installer Docker (dépôt officiel Docker)

Suivre la procédure officielle : https://docs.docker.com/engine/install/ubuntu/

Puis autoriser votre utilisateur (évite « permission denied » sur `/var/run/docker.sock`) :

```bash
sudo usermod -aG docker $USER
```

Déconnectez-vous puis reconnectez-vous pour que le groupe soit pris en compte.

Vérification :

```bash
docker version
docker compose version
docker run --rm hello-world
```

## Paramètre noyau obligatoire (Elasticsearch / OpenSearch)

Sans lui, les conteneurs Elasticsearch/OpenSearch (TheHive, Cortex, Wazuh indexer) plantent au démarrage.

```bash
# Application immédiate
sudo sysctl -w vm.max_map_count=262144

# Persistance après redémarrage
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Attention à la syntaxe : la clé est `vm.max_map_count` (avec des points), pas `vm_max_map_count`.

## Espace disque

Si `/` est petit, déplacez le `data-root` de Docker sur un autre disque (`/etc/docker/daemon.json`) :

```json
{ "data-root": "/mnt/docker-data" }
```

Puis `sudo systemctl restart docker`. Après ce changement, les anciens conteneurs et volumes ne sont plus visibles tant qu'ils n'ont pas été copiés vers le nouveau `data-root`.

## Réseau

Voir [NETWORK.md](NETWORK.md).
