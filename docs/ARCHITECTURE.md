# Architecture

## Rôles des machines

| Machine | Système | Composants |
|---|---|---|
| WINDS | Windows Server | Active Directory `LAB.LOCAL`, agent Wazuh |
| LinuxSRV | Ubuntu Server 22.04 | TheHive, Cassandra, Elasticsearch, Cortex, MISP (Docker) |
| Serveur Wazuh | Linux (Docker) | Wazuh manager, indexer, dashboard |

Toutes les machines communiquent par Tailscale ([NETWORK.md](NETWORK.md)).

## Flux de données

1. **Génération** : activité dans le domaine (connexions, création de compte, ajout à un groupe) → journal Windows *Security*.
2. **Collecte** : l'agent Wazuh sur WINDS envoie les événements au manager (1514).
3. **Détection** : le manager décode, applique les règles, produit des alertes indexées dans l'indexer.
4. **Escalade** : le script d'intégration envoie les alertes de niveau suffisant vers TheHive (API 9000).
5. **Enrichissement** : l'analyste lance des analyzers Cortex sur les observables (IP, hash…).
6. **Threat Intelligence** : corrélation et partage d'IOC avec MISP.
7. **Réponse** : traitement du case, tâches, clôture.

## Ports

| Service | Port | Protocole | Usage |
|---|---|---|---|
| Wazuh dashboard | 443 | HTTPS | Interface |
| Wazuh manager | 1514 | TCP/UDP | Événements des agents |
| Wazuh manager | 1515 | TCP | Enrôlement des agents |
| Wazuh API | 55000 | HTTPS | API manager |
| Wazuh indexer | 9200 | HTTPS | API OpenSearch |
| TheHive | 9000 | HTTP | Interface + API |
| Cortex | 9001 | HTTP | Interface + API |
| MISP | 80 / 443 | HTTP(S) | Interface + API |
| Cassandra | 9042 | TCP | Interne TheHive uniquement |

Les ports internes (Cassandra, Elasticsearch, Redis, base de données) ne doivent pas être exposés hors des réseaux Docker.

## Piles Docker

| Dossier | Fichier de déploiement |
|---|---|
| `wazuh/` | Dépôt officiel `wazuh/wazuh-docker` (single-node) |
| `thehive/` | `docker-compose.yml` fourni |
| `cortex/` | `docker-compose.yml` fourni |
| `misp/` | Dépôt officiel `MISP/misp-docker` |
