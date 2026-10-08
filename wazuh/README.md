# Wazuh

Plateforme open source **SIEM / XDR** : elle collecte les journaux des agents, applique des règles de détection et génère des alertes. Elle se compose de trois éléments centraux et d'un agent universel :

| Élément | Rôle |
|---|---|
| Wazuh **manager** (server) | Reçoit les événements des agents, décode, applique les règles, génère les alertes |
| Wazuh **indexer** | Stocke et indexe les alertes (basé sur OpenSearch) |
| Wazuh **dashboard** | Interface web de visualisation |
| Wazuh **agent** | Installé sur les machines surveillées (ici le DC Windows) |

Documentation officielle : https://documentation.wazuh.com/current/deployment-options/docker/

---

## 1. Prérequis

- Docker + Compose v2 ([`docs/PREREQUISITES.md`](../docs/PREREQUISITES.md))
- `vm.max_map_count=262144` (obligatoire pour l'indexer)
- Ports libres : `443`, `1514`, `1515`, `514/udp`, `55000`, `9200`

## 2. Installation Docker (single-node)

Le déploiement se fait avec le dépôt officiel `wazuh/wazuh-docker`. **Choisissez la même version (tag) que celle de vos agents** : les versions des agents ne doivent pas dépasser celle du manager.

Les tags disponibles sont listés sur https://github.com/wazuh/wazuh-docker/releases

```bash
# 1. Cloner le tag choisi (remplacer X.Y.Z par la version voulue, ex. 4.11.2)
git clone https://github.com/wazuh/wazuh-docker.git -b vX.Y.Z
cd wazuh-docker/single-node

# 2. Générer les certificats TLS entre les nœuds
docker compose -f generate-indexer-certs.yml run --rm generator

# 3. Démarrer la stack
docker compose up -d

# 4. Suivre le démarrage (l'indexer met environ 1 minute)
docker compose ps
docker compose logs -f wazuh.dashboard
```

Pendant l'attente de l'indexer, des messages du type `Wazuh dashboard server is not ready yet` sont normaux.

### Accès

- Dashboard : `https://<IP-du-serveur>` (certificat auto-signé : exception navigateur à accepter)
- Identifiants par défaut de la documentation : `admin` / `SecretPassword`

### Changer les mots de passe par défaut

Les identifiants par défaut (indexer, dashboard `kibanaserver`, API `wazuh-wui`) sont publics. La procédure de changement pour la version déployée est décrite ici :
https://documentation.wazuh.com/current/deployment-options/docker/wazuh-container.html

Après modification, redéployez avec `docker compose down` puis `docker compose up -d` (sans `-v`, qui supprimerait les volumes).

## 3. Structure des fichiers utiles (dépôt officiel)

| Élément | Rôle |
|---|---|
| `single-node/docker-compose.yml` | Définition de la stack |
| `single-node/generate-indexer-certs.yml` | Générateur de certificats |
| `single-node/config/wazuh_cluster/wazuh_manager.conf` | Configuration du manager (équivalent `ossec.conf`), montée dans le conteneur |
| `single-node/config/wazuh_indexer_ssl_certs/` | Certificats générés (**ne pas commiter**) |

Après modification de `wazuh_manager.conf` : `docker compose restart wazuh.manager`.

## 4. Installer l'agent sur le contrôleur de domaine (Windows)

PowerShell en administrateur (remplacer la version et l'adresse du manager) :

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-X.Y.Z-1.msi -OutFile $env:tmp\wazuh-agent.msi
msiexec.exe /i $env:tmp\wazuh-agent.msi /q WAZUH_MANAGER="<IP_TAILSCALE_WAZUH>" WAZUH_AGENT_NAME="WINDS"
NET START WazuhSvc
```

Vérification côté serveur : dashboard → *Agents*, ou

```bash
docker compose exec wazuh.manager /var/ossec/bin/agent_control -l
```

## 5. Collecter les journaux de sécurité Windows

Le canal `Security` est collecté par défaut. Les événements utiles (4624/4625 connexions, 4720 création de compte, 4728/4732 ajout à un groupe, 4768/4769 Kerberos) ne sont générés que si la **stratégie d'audit** est activée : voir [`../active-directory/README.md`](../active-directory/README.md).

## 6. Envoi des alertes vers TheHive

Voir [`../integrations/wazuh-thehive/README.md`](../integrations/wazuh-thehive/README.md).

## 7. Commandes utiles

```bash
docker compose ps                                   # état des conteneurs
docker compose logs -f wazuh.manager                # logs du manager
docker compose exec wazuh.manager bash              # shell dans le manager
docker compose exec wazuh.manager tail -f /var/ossec/logs/alerts/alerts.json
docker compose exec wazuh.manager tail -f /var/ossec/logs/ossec.log
```

## 8. Dépannage

| Symptôme | Piste |
|---|---|
| L'indexer redémarre en boucle | `vm.max_map_count` trop bas (65530 par défaut) |
| Agent « Never connected » | Ports 1514/1515 bloqués, mauvaise IP manager, version agent > manager |
| Dashboard inaccessible | L'indexer n'est pas encore prêt, attendre ~1 min et consulter les logs |

Voir aussi [`../docs/TROUBLESHOOTING.md`](../docs/TROUBLESHOOTING.md).
