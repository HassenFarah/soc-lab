# Cortex

Moteur d'**analyse d'observables** (IP, domaines, hash, URL…) et d'actions de réponse. Il exécute des *analyzers* (ex. AbuseIPDB, VirusTotal) et des *responders*, et est appelé depuis TheHive.

Documentation officielle :
- Cortex avec Docker : https://docs.strangebee.com/cortex/installation-and-configuration/run-cortex-with-docker
- Paramètres Docker : https://docs.strangebee.com/cortex/installation-and-configuration/parameters-docker/
- Catalogue d'analyzers : https://github.com/TheHive-Project/Cortex-Analyzers

## 1. Fonctionnement

```
TheHive ──API──► Cortex ──docker.sock──► conteneur analyzer ──► service externe (AbuseIPDB…)
                    │
                    └──► Elasticsearch (index Cortex)
```

Cortex démarre chaque analyzer dans un **conteneur Docker séparé** en utilisant le socket Docker de l'hôte. C'est pourquoi le compose monte `/var/run/docker.sock` et un dossier de jobs commun.

## 2. Prérequis

- Docker + Compose v2, `vm.max_map_count=262144` ([`docs/PREREQUISITES.md`](../docs/PREREQUISITES.md))
- Accès Internet sortant : téléchargement des catalogues d'analyzers (`download.thehive-project.org`) et des images Docker (Docker Hub)
- Un Elasticsearch **dédié** à Cortex (il ne partage pas celui de TheHive)

## 3. Installation

```bash
cd cortex
cp .env.example .env

# Créer le dossier de jobs avec les bonnes permissions
sudo mkdir -p /tmp/cortex-jobs
sudo chmod 777 /tmp/cortex-jobs

docker compose up -d
docker compose ps
docker compose logs -f cortex
```

Accès : `http://<IP>:9001`

### Persistance des permissions de `/tmp/cortex-jobs`

`/tmp` est vidé au redémarrage, ce qui provoque des `AccessDeniedException` sur le dossier de jobs. Recréer le dossier automatiquement à chaque boot avec `systemd-tmpfiles` :

```bash
echo 'd /tmp/cortex-jobs 0777 root root -' | sudo tee /etc/tmpfiles.d/cortex-jobs.conf
sudo systemd-tmpfiles --create /etc/tmpfiles.d/cortex-jobs.conf
```

Alternative plus robuste : utiliser un dossier hors de `/tmp` (ex. `/opt/cortex-jobs`) en modifiant `CORTEX_JOB_DIR` dans `.env`.

## 4. Première configuration

1. Ouvrir `http://<IP>:9001`, cliquer sur **Update Database** (migration de l'index).
2. Créer le compte **superadmin**.
3. Se connecter, créer une **organisation** (ex. `SOC-LAB`).
4. Dans cette organisation, créer un utilisateur `orgadmin` (pour gérer les analyzers) et un utilisateur **de service pour TheHive** avec le rôle `read, analyze`. Générer sa **clé API** (*Create API Key* puis *Reveal*).
5. Onglet **Analyzers** : activer les analyzers voulus (ex. `AbuseIPDB`) et renseigner leurs clés API dans la configuration de chaque analyzer.

## 5. Connexion à TheHive

Dans TheHive : *Administration de la plateforme → Cortex → Ajouter un serveur* :

| Champ | Valeur |
|---|---|
| Nom | `Cortex` |
| URL | `http://<hôte-cortex>:9001` |
| Clé API | clé de l'utilisateur de service Cortex |

Si Cortex et TheHive sont sur des stacks Compose distinctes, utilisez l'IP de l'hôte (ou l'IP Tailscale) plutôt qu'un nom de conteneur.

Test : dans un case, ouvrir un observable → *Run analyzers*.

## 6. Commandes utiles

```bash
docker compose ps
docker compose logs -f cortex
curl -s http://localhost:9001/api/status | jq .
docker ps -a --filter "ancestor=cortexneurons"   # conteneurs d'analyzers (si présents)
```

## 7. Dépannage

| Symptôme | Cause probable | Action |
|---|---|---|
| Conteneurs `Exited (255)` après reboot ou changement de `data-root` | Docker relancé sans les anciens volumes | `docker compose up -d` ; vérifier `data-root` |
| Elasticsearch ne démarre pas | `vm.max_map_count` trop bas | Voir prérequis |
| `AccessDeniedException` sur `/tmp/cortex-jobs` | Permissions perdues après redémarrage | Règle `tmpfiles.d` ci-dessus |
| Analyzer en erreur « can't be run » / pas de démarrage | Chemin de jobs différent entre hôte et conteneur, ou socket Docker inaccessible | Vérifier que `job_directory` = `docker_job_directory` = dossier monté à l'identique |
| `permission denied` sur `docker.sock` dans le conteneur | Utilisateur du conteneur sans droit sur le socket | Consulter la doc « Parameters for Docker » et l'option `--daemon-user` ; en lab, exécuter le conteneur avec un utilisateur autorisé sur le socket |

Détails : [`../docs/TROUBLESHOOTING.md`](../docs/TROUBLESHOOTING.md).

## 8. Sécurité

Monter `docker.sock` donne à Cortex un contrôle quasi total sur l'hôte. Acceptable en lab isolé, à éviter tel quel en production.
