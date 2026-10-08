# Intégrations

Ce dossier documente les liaisons entre composants.

```
Wazuh ──(1)──► TheHive ◄──(2)──► Cortex
                  ▲
                  └────(3)────► MISP
```

| # | Liaison | Sens | Configuré où |
|---|---|---|---|
| 1 | Wazuh → TheHive | Alertes → *alerts* TheHive | Wazuh (`ossec.conf` + script) |
| 2 | TheHive ↔ Cortex | Lancer des analyzers, récupérer les rapports | TheHive (interface) |
| 3 | TheHive ↔ MISP | Import d'événements / export d'observables | TheHive (interface) |

## 1. Wazuh → TheHive

Wazuh exécute un script d'intégration personnalisé via son démon `integratord`. Le script (généralement nommé `custom-w2thive` avec un wrapper shell et un script Python) est fourni par la communauté / la documentation Wazuh ; **placez ici la copie exacte que vous utilisez** (`integrations/wazuh-thehive/custom-w2thive*`) après avoir retiré toute clé API.

### Principe

1. Créer dans TheHive un utilisateur de service, **dans une organisation** (pas le compte admin de la plateforme), avec un profil autorisant la création d'alertes, puis générer sa clé API.
2. Copier le script d'intégration dans `/var/ossec/integrations/` du **conteneur manager** (ou via un volume monté) avec les bons droits :

```bash
chown root:wazuh /var/ossec/integrations/custom-w2thive*
chmod 750 /var/ossec/integrations/custom-w2thive*
```

3. Déclarer l'intégration dans la configuration du manager (`wazuh_manager.conf` en Docker) :

```xml
<integration>
  <name>custom-w2thive</name>
  <hook_url>http://<IP_THEHIVE>:9000</hook_url>
  <api_key>VOTRE_CLE_API</api_key>
  <level>8</level>
  <alert_format>json</alert_format>
</integration>
```

`<level>` définit le niveau de règle minimum envoyé (à ajuster ; un niveau trop élevé « cache » des alertes).

4. Redémarrer le manager : `docker compose restart wazuh.manager`.

### Vérification

```bash
# Journal du démon d'intégration
docker compose exec wazuh.manager tail -f /var/ossec/logs/integrations.log
docker compose exec wazuh.manager tail -f /var/ossec/logs/ossec.log | grep -i integrat

# Connectivité vers TheHive depuis le manager
docker compose exec wazuh.manager curl -s -o /dev/null -w "%{http_code}\n" http://<IP_THEHIVE>:9000/api/status
```

### Checklist si les alertes n'arrivent plus dans TheHive

Constaté dans ce lab : les alertes restent visibles dans Wazuh mais plus dans TheHive après reconstruction de TheHive. Points à contrôler, dans cet ordre :

1. **Clé API** : après un reset de TheHive, les anciens utilisateurs et clés n'existent plus ; en régénérer une et mettre à jour `ossec.conf`.
2. **Organisation et profil** : l'utilisateur de la clé doit appartenir à une organisation existante et avoir le droit de créer des alertes.
3. **Connectivité** : le manager (dans son conteneur) atteint-il l'IP:9000 de TheHive ? (`curl` ci-dessus)
4. **Script** : présent, exécutable, propriétaire `root:wazuh` dans le conteneur, et toujours présent après recréation du conteneur (utiliser un volume).
5. **Niveau** : `<level>` inférieur ou égal au niveau des règles attendues.
6. **Logs** : `integrations.log` et `ossec.log` donnent l'erreur exacte (HTTP 401/403 = clé/profil, connexion refusée = réseau/URL).
7. **Dépendances Python** du script (module client TheHive) toujours installées dans le conteneur.