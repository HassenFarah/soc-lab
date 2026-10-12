# TheHive 5

Plateforme de **réponse à incident** : elle reçoit les alertes (ici de Wazuh), permet de les promouvoir en *cases*, d'assigner des tâches, de stocker des observables et de lancer des analyses via Cortex.

Documentation officielle :
- Docker : https://docs.strangebee.com/thehive/installation/docker/
- Dépôt officiel d'exemples Docker Compose (profils de production) : https://github.com/StrangeBeeCorp/docker

## 1. Architecture

| Conteneur | Rôle |
|---|---|
| `thehive` | Application (port 9000) |
| `cassandra` | Base de données principale |
| `elasticsearch` | Index de recherche |

Le stockage des fichiers utilise ici le disque local (volume `thehive-data`). La documentation officielle recommande MinIO (S3) pour un cluster ou de la production.

## 2. Prérequis

- Docker Engine et plugin Compose v2 ([`docs/PREREQUISITES.md`](../docs/PREREQUISITES.md))
- `vm.max_map_count=262144`
- Environ 6 à 8 Go de RAM

## 3. Installation

```bash
cd thehive
cp .env.example .env

# Générer un secret et le placer dans .env (THEHIVE_SECRET=...)
openssl rand -base64 48

docker compose up -d
docker compose ps
docker compose logs -f thehive
```

Le premier démarrage prend plusieurs minutes (Cassandra doit être « healthy » avant TheHive).

Accès : `http://<IP>:9000`

Identifiants par défaut de la documentation officielle : `admin@thehive.local` / `secret`. **Les changer immédiatement.**

> Ce `docker-compose.yml` est un exemple minimal de lab. Pour une base de production, partez des profils du dépôt `StrangeBeeCorp/docker`. Les versions de Cassandra et Elasticsearch doivent correspondre à celles supportées par la version de TheHive choisie : vérifiez la matrice de compatibilité dans la documentation avant de modifier les tags.

## 4. Configuration initiale (interface web)

1. Se connecter avec le compte admin par défaut, changer son mot de passe.
2. **Créer une organisation** (ex. `SOC-LAB`). Le compte `admin@thehive.local` appartient à l'organisation d'administration et ne sert pas au travail quotidien.
3. **Créer des utilisateurs** dans cette organisation :
   - un compte analyste (profil `analyst`) ou `org-admin` ;
   - un compte de service dédié à l'intégration Wazuh, avec une **clé API** (Paramètres du compte → *API Key*).
4. Noter les clés API dans un gestionnaire de secrets (pas dans Git).

## 5. Connexions

| Vers | Où configurer | Voir |
|---|---|---|
| Cortex | Administration de la plateforme → *Cortex* | [`../cortex/README.md`](../cortex/README.md) |
| MISP | Administration de la plateforme → *MISP* | [`../misp/README.md`](../misp/README.md) |
| Wazuh (entrant) | Script d'intégration côté Wazuh + clé API | [`../integrations/wazuh-thehive/README.md`](../integrations/wazuh-thehive/README.md) |

## 6. Commandes utiles

```bash
docker compose ps
docker compose logs -f thehive
docker compose logs cassandra | tail -50
curl -s http://localhost:9000/api/status | jq .
docker compose down            # arrêt (données conservées)
docker compose down -v         # ATTENTION : supprime aussi les volumes (base vierge)
```

## 7. Sauvegarde

Pour une sauvegarde complète, suivre la procédure officielle « Backup & Restore » (Cassandra, index, fichiers) : https://docs.strangebee.com/thehive/operations/backup-restore/

## 8. Dépannage

| Symptôme | Cause probable | Action |
|---|---|---|
| Log répété `Waiting until Cassandra DB is up` | Le conteneur `cassandra` n'existe pas ou n'est pas prêt | `docker compose ps`, `docker compose logs cassandra`, vérifier que le service est bien dans le compose |
| Elasticsearch redémarre | `vm.max_map_count` trop bas ou mémoire insuffisante | Voir prérequis |
| Connexion `admin@thehive.local / secret` refusée | Base déjà initialisée avec un autre mot de passe | Réinitialiser (`down -v`) uniquement si aucune donnée à conserver |
| `permission denied` sur `/var/run/docker.sock` | Utilisateur hors groupe `docker` | `sudo usermod -aG docker $USER` puis reconnexion |

Détails : [`../docs/TROUBLESHOOTING.md`](../docs/TROUBLESHOOTING.md).
