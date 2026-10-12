# MISP

**Malware Information Sharing Platform** : plateforme de Threat Intelligence pour stocker, corréler et partager des IOC (IP, domaines, hash, URL…) sous forme d'événements et d'attributs.

Dépôt Docker officiel : https://github.com/MISP/misp-docker

## 1. Architecture (conteneurs)

Le dépôt officiel sépare MISP en plusieurs images :

| Conteneur | Rôle |
|---|---|
| `misp-core` | Application web MISP (Apache/PHP, workers) |
| `db` | Base de données MariaDB |
| `redis` | Cache et files de tâches |
| `misp-modules` | Modules d'enrichissement/import/export |
| `mail` | Relais SMTP (Exim4) |

## 2. Prérequis

- Docker + Compose v2 ([`docs/PREREQUISITES.md`](../docs/PREREQUISITES.md))
- Accès à Docker Hub ou au registre d'images utilisé par le dépôt (le lab a déjà été bloqué par un problème de connectivité Docker Hub : tester `docker pull hello-world` avant)
- Ports `80` et `443` libres (ou modifiés dans `.env`)

## 3. Installation

```bash
git clone https://github.com/MISP/misp-docker.git
cd misp-docker

# Configuration
cp template.env .env
nano .env
```

Variables à revoir dans `.env` (les noms exacts sont ceux du `template.env` de la version clonée, à lire dans le fichier) :

- l'URL de base (`BASE_URL`), avec l'IP ou le nom réellement utilisé pour accéder à MISP ;
- les mots de passe de la base de données ;
- l'email et le mot de passe administrateur ;
- la clé/authentification admin si vous voulez une clé API initiale fixe.

Puis :

```bash
docker compose pull          # images pré-construites (ou: docker compose build)
docker compose up -d
docker compose ps            # attendre que misp-core, db, redis, misp-modules soient "healthy"
docker compose logs -f misp-core
```

Le premier démarrage initialise la base et peut durer plusieurs minutes.

Accès : `https://<IP>` (certificat auto-signé).

Identifiants par défaut décrits par le projet : `admin@admin.test` / `admin`, sauf si vous les avez définis dans `.env`. **À changer immédiatement** (MISP force le changement au premier login).

## 4. Configuration initiale

1. Se connecter, changer le mot de passe.
2. **Organisation** : *Administration → List Organisations* → créer/renommer l'organisation locale.
3. **Clé API** : *Administration → List Auth Keys* → *Add authentication key* pour un utilisateur de service TheHive. La clé n'est affichée qu'une fois.
4. **Feeds** : *Sync Actions → List Feeds* → activer les feeds voulus (ex. CIRCL OSINT) puis *Fetch and store all feed data*.
5. **Taxonomies / Galaxies / Warninglists** : *Administration → Update* pour les mettre à jour.

## 5. Connexion à TheHive

Dans TheHive : *Administration de la plateforme → MISP → Ajouter un serveur* :

| Champ | Valeur |
|---|---|
| Nom | `MISP` |
| URL | `https://<hôte-misp>` |
| Clé API | clé d'authentification MISP |
| Certificat | ignorer/importer le certificat si auto-signé |

Fonctions : import d'événements MISP en alertes TheHive et export d'observables vers MISP.

## 6. Commandes utiles

```bash
docker compose ps
docker compose logs -f misp-core
docker compose exec misp-core bash
docker compose down             # conserve les volumes
docker compose pull && docker compose up -d   # mise à jour
```

## 7. Dépannage

| Symptôme | Action |
|---|---|
| `docker compose pull` échoue / timeout | Vérifier DNS et accès à Docker Hub depuis l'hôte, proxy éventuel |
| Login impossible / URL incorrecte | Corriger `BASE_URL` dans `.env` puis `docker compose up -d` |
| `misp-core` reste `starting` | Attendre l'initialisation ; consulter les logs, vérifier que `db` est healthy |
| Erreur certificat côté TheHive | Certificat auto-signé : le déclarer/ignorer dans la config du connecteur |

Détails : [`../docs/TROUBLESHOOTING.md`](../docs/TROUBLESHOOTING.md).
