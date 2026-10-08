# SOC Home Lab : Wazuh · TheHive · Cortex · MISP · Active Directory

Dépôt de documentation et de déploiement d'un **SOC de laboratoire** : collecte des journaux Windows/Linux, détection, gestion d'incidents, enrichissement automatique et Threat Intelligence.

> Projet académique réalisé à la FST (Université de Tunis El Manar) par l'équipe Securinets FST / groupe projet SOC.

---

## 1. Architecture en un coup d'œil

```
                    ┌───────────────────────────┐
                    │  Active Directory (WINDS) │  LAB.LOCAL
                    │  Windows Server + agent   │
                    └────────────┬──────────────┘
                                 │ Événements Windows (Security, Sysmon…)
                                 ▼
                    ┌───────────────────────────┐
                    │  Wazuh (manager+indexer   │
                    │  +dashboard)              │
                    └────────────┬──────────────┘
                                 │ Alertes (intégration custom-w2thive)
                                 ▼
   ┌────────────┐   analyse   ┌───────────────┐   IOC / événements   ┌────────┐
   │   Cortex   │◄────────────│   TheHive 5   │◄────────────────────►│  MISP  │
   │ analyzers  │  résultats  │ (cases/alerts)│     connecteur       │  TI    │
   └────────────┘────────────►└───────────────┘                      └────────┘
```

Réseau d'interconnexion : **Tailscale** (voir [`docs/NETWORK.md`](docs/NETWORK.md)).

## 2. Structure du dépôt

```
soc-lab/
├── README.md                     ← vous êtes ici
├── .gitignore
├── docs/
│   ├── ARCHITECTURE.md           ← flux de données, ports, rôles des machines
│   ├── NETWORK.md                ← Tailscale, adressage, pare-feu
│   ├── PREREQUISITES.md          ← Docker, sysctl, ressources
│   └── TROUBLESHOOTING.md        ← problèmes rencontrés + solutions
├── wazuh/
│   ├── README.md                 ← installation Docker + agents + ossec.conf
│   └── config/
├── thehive/
│   ├── README.md
│   ├── docker-compose.yml        ← TheHive + Cassandra + Elasticsearch
│   └── .env.example
├── cortex/
│   ├── README.md
│   ├── docker-compose.yml        ← Cortex + Elasticsearch dédié
│   └── .env.example
├── misp/
│   └── README.md                 ← basé sur le dépôt officiel MISP/misp-docker
├── active-directory/
│   ├── README.md                 ← promotion DC, audit, agent Wazuh
│   └── scripts/
│       └── setup-audit-policy.ps1
├── integrations/
│   └── wazuh-thehive/README.md   ← Wazuh → TheHive, TheHive ↔ Cortex/MISP
└── assets/screenshots/
```

## 3. Ordre de déploiement recommandé

| Étape | Composant | Pourquoi dans cet ordre |
|---|---|---|
| 0 | [Prérequis](docs/PREREQUISITES.md) + [réseau](docs/NETWORK.md) | Docker, `vm.max_map_count`, Tailscale |
| 1 | [Active Directory](active-directory/README.md) | Source des événements à détecter |
| 2 | [Wazuh](wazuh/README.md) | SIEM/XDR, agent sur le DC |
| 3 | [TheHive](thehive/README.md) | Gestion des alertes et cases |
| 4 | [Cortex](cortex/README.md) | Enrichissement (analyzers) |
| 5 | [MISP](misp/README.md) | Threat Intelligence |
| 6 | [Intégrations](integrations/wazuh-thehive/README.md) | Relier le tout |

## 4. Composants

| Composant | Rôle | Port(s) principaux | Docker |
|---|---|---|---|
| Wazuh | SIEM / XDR / détection | 443 (dashboard), 1514, 1515, 55000, 9200 | Oui (dépôt officiel) |
| TheHive 5 | Gestion d'incidents | 9000 | Oui |
| Cortex 3 | Analyse d'observables | 9001 | Oui |
| MISP | Partage d'IOC | 80 / 443 | Oui (dépôt officiel) |
| Active Directory | Domaine Windows | 53, 88, 389, 445… | **Non** (Windows Server natif) |

## 5. Avertissements de sécurité

- Les mots de passe par défaut cités dans les README (Wazuh, TheHive, MISP) sont **publics** : à changer avant toute exposition.
- Ne jamais commiter de fichiers `.env`, de clés API ou de certificats (voir `.gitignore`).
- Ce lab est prévu pour un usage pédagogique : ne pas l'exposer directement sur Internet.

## 6. Équipe

.....
