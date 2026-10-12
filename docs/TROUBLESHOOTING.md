# Dépannage : problèmes rencontrés dans le lab

Chaque entrée : symptôme → cause → solution.

## Docker / système

### `permission denied` sur `/var/run/docker.sock`
- **Cause** : l'utilisateur n'est pas dans le groupe `docker` (fréquent après une réinstallation ou un changement de `data-root`).
- **Solution** : `sudo usermod -aG docker $USER`, puis se reconnecter (ou `newgrp docker`).

### Elasticsearch / OpenSearch redémarre en boucle
- **Cause** : `vm.max_map_count` à 65530 (valeur par défaut).
- **Solution** : `sudo sysctl -w vm.max_map_count=262144` et l'écrire dans `/etc/sysctl.conf` sous la forme exacte `vm.max_map_count=262144`. Une ligne mal écrite (`vm_max_map_count`) est ignorée.

### Conteneurs `Exited (255)` après plusieurs jours
- **Cause probable** : le daemon Docker a redémarré avec un autre `data-root`, ou la machine a été arrêtée.
- **Solution** : `docker ps -a`, `docker compose up -d` dans le dossier de la stack ; vérifier `docker info | grep "Docker Root Dir"`.

### `docker compose pull` échoue (MISP)
- **Cause** : connectivité vers Docker Hub.
- **Solution** : tester `docker pull hello-world`, DNS, proxy, pare-feu.

## TheHive

### Boucle `Waiting until Cassandra DB is up`
- **Cause** : conteneur Cassandra absent, arrêté ou non prêt.
- **Solution** : `docker compose ps` ; `docker compose logs cassandra` ; relancer la stack avec le compose complet.

### Connexion admin refusée
- **Cause** : la base contient un autre mot de passe que celui par défaut.
- **Solution** : si aucune donnée à conserver, réinitialiser les volumes (`docker compose down -v`) ; sinon restaurer depuis une sauvegarde. Après réinitialisation, refaire : organisation, utilisateurs, clés API, connecteurs Cortex/MISP, clé API dans Wazuh.

## Cortex

### `AccessDeniedException` sur `/tmp/cortex-jobs`
- **Cause** : `/tmp` recréé au démarrage sans les permissions.
- **Solution** : règle `systemd-tmpfiles` ([`cortex/README.md`](../cortex/README.md#persistance-des-permissions-de-tmpcortex-jobs)) ou dossier hors `/tmp`.

### Cortex ne démarre pas / erreurs d'index
- **Cause** : Elasticsearch incompatible ou indisponible.
- **Solution** : utiliser un ES 7.x dédié, vérifier `vm.max_map_count`, cliquer sur *Update Database* au premier accès.

## Wazuh → TheHive

Voir la checklist : [`integrations/wazuh-thehive/README.md`](../integrations/wazuh-thehive/README.md#checklist-si-les-alertes-narrivent-plus-dans-thehive).

## Réseau

### Un service ne joint plus une adresse LAN
- **Cause** : l'interface / l'IP a disparu.
- **Solution** : utiliser l'adresse Tailscale (`tailscale ip -4`) pour toutes les liaisons inter-services.