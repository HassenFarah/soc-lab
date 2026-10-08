# Active Directory (contrôleur de domaine WINDS)

Le domaine **LAB.LOCAL** est la source principale d'événements de sécurité du lab (authentifications, Kerberos, gestion des comptes et groupes).

## Pourquoi pas de Docker ici ?

Un contrôleur de domaine Active Directory Microsoft **ne peut pas tourner dans un conteneur Linux** ni dans Docker sur Linux : il nécessite Windows Server, installé sur une VM ou une machine physique. Ce composant est donc documenté en installation native.

(Une alternative existe avec Samba en mode AD DC, mais elle n'est pas équivalente à un AD Windows et n'est pas utilisée dans ce projet.)

## 1. Caractéristiques

| Élément | Valeur |
|---|---|
| Nom de la machine | `WINDS` |
| Domaine | `LAB.LOCAL` |
| Adresse LAN | `192.168.50.10` |
| Agent | Wazuh agent (nom `WINDS`) |

## 2. Installation du rôle AD DS et promotion en contrôleur de domaine

PowerShell en administrateur, sur Windows Server avec IP fixe :

```powershell
# Rôle AD DS + outils de gestion
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# Création de la forêt (redémarrage automatique à la fin)
Install-ADDSForest `
  -DomainName "lab.local" `
  -DomainNetbiosName "LAB" `
  -InstallDns `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "Mot de passe DSRM")
```

Vérification après redémarrage :

```powershell
Get-ADDomain
Get-ADDomainController
dcdiag /q
```

## 3. Structure de test (OU, utilisateurs, groupes)

```powershell
New-ADOrganizationalUnit -Name "SOC-LAB" -Path "DC=lab,DC=local"
New-ADUser -Name "Test User" -SamAccountName "tuser" `
  -AccountPassword (Read-Host -AsSecureString "Mot de passe") `
  -Enabled $true -Path "OU=SOC-LAB,DC=lab,DC=local"
New-ADGroup -Name "SOC-Analysts" -GroupScope Global -Path "OU=SOC-LAB,DC=lab,DC=local"
```

## 4. Activer l'audit (indispensable pour la détection)

Sans audit avancé, les événements 4624/4625/4720/4728… ne sont pas produits.

Script fourni : [`scripts/setup-audit-policy.ps1`](scripts/setup-audit-policy.ps1)

```powershell
Set-ExecutionPolicy -Scope Process Bypass
.\scripts\setup-audit-policy.ps1
```

Vérification :

```powershell
auditpol /get /category:*
```

Pour un DC, la configuration se fait idéalement par GPO (*Default Domain Controllers Policy → Advanced Audit Policy Configuration*).

### Événements Windows utiles

| ID | Signification |
|---|---|
| 4624 / 4625 | Connexion réussie / échouée |
| 4672 | Privilèges spéciaux attribués |
| 4720 | Compte utilisateur créé |
| 4728 / 4732 / 4756 | Ajout à un groupe (global / local / universel) |
| 4768 / 4769 / 4771 | Kerberos : TGT, ticket de service, échec de pré-authentification |
| 4740 | Compte verrouillé |
| 1102 | Journal d'audit effacé |

## 5. Agent Wazuh

Installation : voir [`../wazuh/README.md`](../wazuh/README.md#4-installer-lagent-sur-le-contrôleur-de-domaine-windows).

Vérifier ensuite le service :

```powershell
Get-Service WazuhSvc
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 30
```

## 6. Génération d'événements pour tester la chaîne complète

```powershell
# 4625 : échecs de connexion (depuis un poste du domaine)
runas /user:LAB\tuser cmd     # avec un mauvais mot de passe

# 4720 + 4728 : création de compte et ajout à un groupe
New-ADUser -Name "Alerte Test" -SamAccountName "atest" -Enabled $false
Add-ADGroupMember -Identity "SOC-Analysts" -Members "atest"
```

Contrôle : Wazuh dashboard → *Threat Hunting / Events* → filtrer sur l'agent `WINDS`.
