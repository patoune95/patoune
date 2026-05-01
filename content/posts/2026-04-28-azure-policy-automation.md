---
title: "Gérer les Azure Policies à l'échelle : une approche via Terraform et azapi"
date: 2026-04-28T10:00:00+02:00
draft: true
tags:
  - azure
  - policy
  - governance
  - iac
  - automation
cover:
  image: assets/images/azure-policy-terraform.jpeg
  alt: "Azure Policy Automation"
---

<!-- Introduction -->

## Qu'est-ce qu'une Azure Policy ?

Une Azure Policy est une règle de gouvernance appliquée sur des ressources Azure. Elle permet de s'assurer qu'un environnement reste conforme à des standards définis : sécurité, nommage, régions autorisées, chiffrement...

Il y a trois notions clés à distinguer.

**La Policy Definition** est la règle elle-même. Elle définit ce qui est évalué et l'effet déclenché en cas de non-conformité. Les effets principaux sont :

| Effet | Comportement |
|---|---|
| `Audit` | Log la non-conformité, ne bloque pas |
| `AuditIfNotExists` | Audite une ressource *associée* absente ou non-conforme (ex : diagnostic settings, Defender for Cloud) |
| `Deny` | Bloque la création ou modification de la ressource |
| `DenyAction` | Bloque une action spécifique sur une ressource (actuellement uniquement `DELETE`) — protège les ressources critiques contre la suppression accidentelle |
| `DeployIfNotExists` | Déploie automatiquement une ressource associée si absente |
| `Modify` | Modifie une propriété lors de la création/mise à jour |
| `Append` | Ajoute des champs à la ressource |
| `Disabled` | Désactive complètement l'évaluation de la policy |

**L'Initiative** (ou `PolicySetDefinition`) est un regroupement de plusieurs definitions. Plutôt que d'assigner chaque policy une par une, on les regroupe dans une initiative cohérente, par exemple une baseline sécurité ou conformité CIS.

**L'Assignment** est ce qui lie une définition ou une initiative à un **scope** : Management Group, Subscription ou Resource Group. Sans assignment, une policy n'a aucun effet.

![Définitions, Initiative et Assignments Azure Policy](/assets/images/azure-policy-automation/azure-policy-definitions-initiative-assignment.svg)

Une policy peut être assignée directement ou via une initiative, mais en pratique, passer par des initiatives facilite grandement la gestion à l'échelle.

## Les contraintes du terrain

Sur le papier, une policy s'applique uniformément à tout son scope. En pratique, c'est rarement aussi simple : une équipe a besoin d'une exception temporaire, un environnement de dev ne doit pas être bloqué par une règle pensée pour la prod, une ressource legacy ne peut pas encore être mise en conformité.

La solution la plus directe est d'utiliser le mode `DoNotEnforce` sur l'assignment. Quand il est activé, la policy est toujours évaluée (elle remonte des résultats de conformité) mais elle ne produit aucun effet bloquant. Un `Deny` devient silencieux, un `DeployIfNotExists` ne se déclenche pas automatiquement lors de la création ou mise à jour d'une ressource. À noter : les tâches de remédiation *manuelles* peuvent toujours être déclenchées en `DoNotEnforce`, ce qui permet de tester le comportement d'un `DeployIfNotExists` pendant la phase de validation sans risquer de bloquer quoi que ce soit.

```json
{
  "enforcementMode": "DoNotEnforce"
}
```

C'est souvent utilisé comme **filet de sécurité lors d'un déploiement** : on assigne l'initiative en mode `DoNotEnforce` pour observer les résultats de conformité sans risquer de casser quoi que ce soit, puis on bascule en `Default` une fois qu'on est confiant sur la couverture.

Il y a un autre risque terrain moins évident : si on assigne une policy en mode `Default` et qu'on lance ensuite un script pour basculer en `DoNotEnforce`, il existe une fenêtre de quelques secondes à quelques minutes pendant laquelle la policy est active. Un `Deny` dans cette fenêtre peut bloquer la création d'une ressource en cours de déploiement et créer une interruption de service.

La bonne nouvelle : **Azure permet de déployer un assignment directement en `DoNotEnforce` dès la création**, sans passer par une phase `Default`. Il n'y a donc aucune raison d'accepter ce risque. La [documentation Microsoft](https://learn.microsoft.com/en-us/azure/governance/policy/how-to/policy-safe-deployment-practices) recommande d'ailleurs explicitement cette approche dans ses *safe deployment practices* pour les policies avec effets `DeployIfNotExists` ou `Modify` : assigner en `DoNotEnforce`, valider la conformité, puis basculer en `Default`.

`DoNotEnforce` fonctionne aussi bien pour une policy seule que pour une initiative entière : c'est une propriété de l'assignment, pas de la définition.

### Couper seulement certaines policies dans une initiative

Le `DoNotEnforce` agit sur tout l'assignment. Quand on veut désactiver seulement une ou plusieurs policies au sein d'une initiative, sans toucher aux autres, il faut utiliser les **overrides**.

Les overrides permettent de surcharger l'effet d'une policy spécifique dans l'initiative via son `policyDefinitionReferenceId`, sans modifier la définition elle-même :

```json
{
  "properties": {
    "policyDefinitionId": "/subscriptions/{subId}/providers/Microsoft.Authorization/policySetDefinitions/BaselineSecurite",
    "overrides": [
      {
        "kind": "policyEffect",
        "value": "Disabled",
        "selectors": [
          {
            "kind": "policyDefinitionReferenceId",
            "in": [
              "noPublicIpPolicy",
              "diagnosticSettingsPolicy"
            ]
          }
        ]
      }
    ]
  }
}
```

Dans cet exemple, l'initiative `BaselineSecurite` reste assignée et active, mais les deux policies ciblées sont désactivées. Les autres policies de l'initiative continuent de s'appliquer normalement.

> Un assignment peut contenir jusqu'à 10 overrides, chacun pouvant cibler jusqu'à 50 `policyDefinitionReferenceId`. C'est suffisant pour couvrir la grande majorité des cas terrain.

## Organisation des scopes : un exemple concret

Avant de rentrer dans l'automatisation, voici un exemple d'organisation typique en entreprise. Les **definitions et initiatives** sont portées au niveau des management groups d'environnement (DEV / HML / PRD), ce qui permet de définir des règles différentes par environnement tout en gardant une structure commune par BU. Les **assignments** sont appliqués au niveau des souscriptions.

![Organisation des scopes Azure Policy](/assets/images/azure-policy-automation/azure-policy-scope-organisation.svg)

> **Bonne pratique** : le [Cloud Adoption Framework Microsoft](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/resource-org-management-groups) recommande de ne jamais utiliser le Tenant Root Group comme point d'attache direct pour les BU ou les policies. Il faut intercaler un **Intermediate Root Management Group** (nommé d'après l'organisation, ex: `Contoso`) entre le Tenant Root et le reste de la hiérarchie. Cela permet d'appliquer des policies globales à ce niveau sans toucher au Root, et de garder le Root comme ancre neutre, ce qui facilite les évolutions futures et le debug des héritages de policies.

Ce découpage offre une isolation claire : les règles DEV peuvent être plus souples (Audit) pendant que PRD applique des effets stricts (Deny). Chaque BU est autonome dans ses definitions tout en héritant des règles globales posées au niveau de l'Intermediate Root.

## Pourquoi automatiser la gestion des policies ?

À petite échelle, gérer des policies à la main dans le portail Azure est faisable. Dès qu'on monte en charge (plusieurs BUs, plusieurs environnements, des dizaines de policies), ça devient ingérable rapidement.

Les problèmes concrets qu'on rencontre :

**Traçabilité :** impossible de savoir facilement quelle version d'une policy est déployée sur quel scope, par qui, et depuis quand. Le portail ne remplace pas un historique git.

**Drift de configuration :** sans source de vérité commune, les environnements divergent. Une policy modifiée à la main en PRD ne sera pas répercutée en HML, et vice versa. On finit par ne plus savoir ce qui est vraiment en place.

**Gestion du cycle de vie des overrides :** comme on l'a vu, le `DoNotEnforce` et les overrides sont des états temporaires. Sans automatisation, on n'a aucune garantie qu'un override posé pour un déploiement d'urgence sera bien retiré ensuite. Il faut un outil qui puisse suivre ces états et piloter la transition vers `Default` de façon contrôlée.

**Cohérence inter-BU :** quand plusieurs BUs partagent des policies communes (sécurité, nommage, régions autorisées), toute modification doit être propagée de façon consistante. À la main, c'est une source d'erreur permanente.

**Audit et review :** les changements de policies ont un impact direct sur la sécurité et la conformité. Il faut pouvoir les soumettre à une revue (pull request), les tester avant déploiement, et avoir un historique clair de ce qui a changé et pourquoi.

## Mon approche : un catalogue versionné

L'idée est de traiter les policies comme du code : un repository git comme source de vérité, une structure de catalogue claire, et un pipeline CI/CD qui gère le déploiement.

### Structure du repository

On opte pour un **monorepo** avec deux couches distinctes.

La première couche est le **catalogue** : il contient les définitions de policies et d'initiatives, organisées par service Azure. Ce n'est pas ce qui est déployé directement, c'est une bibliothèque de modules Terraform.

Chaque service dispose de sa propre initiative qui regroupe toutes les policies le concernant. C'est le niveau de granularité le plus adapté : on assigne une initiative par service sur un scope, et on joue sur les overrides pour désactiver des policies précises selon le contexte. Pas besoin d'une initiative "fourre-tout". Chaque service est autonome et versionneable indépendamment.

```
catalog/
├── storage/
│   ├── policies/
│   │   ├── deny-public-access/     # policy : pas d'accès public sur les Storage Accounts
│   │   │   └── main.tf
│   │   └── require-secure-transfer/
│   │       └── main.tf
│   └── initiative/                 # initiative "storage" regroupant les policies du service
│       └── main.tf
├── keyvault/
│   ├── policies/
│   │   ├── deny-public-access/     # policy : pas d'accès public sur les Key Vaults
│   │   │   └── main.tf
│   │   └── require-soft-delete/
│   │       └── main.tf
│   └── initiative/                 # initiative "keyvault"
│       └── main.tf
└── networking/
    ├── policies/
    └── initiative/
```

La deuxième couche est la couche **assignments** : elle décrit ce qui est réellement déployé, pour quelle BU, sur quel environnement, et en référençant une version précise du catalogue.

```
assignments/
├── bu1/
│   ├── dev/
│   ├── hml/
│   └── prd/
└── bu2/
    ├── dev/
    ├── hml/
    └── prd/
```

### Le problème du versioning

C'est le point le plus délicat de cette architecture. Si le catalogue est partagé, une modification d'une policy peut impacter toutes les BUs et tous les environnements en même temps, ce qui est exactement ce qu'on veut éviter.

## Mise en pratique

### Pourquoi le provider azapi ?

Le provider `azurerm` expose des ressources Terraform dédiées pour les policies (`azurerm_policy_definition`, `azurerm_management_group_policy_assignment`...), mais il a un défaut : il rattrape toujours l'API ARM avec du retard. Les `overrides`, les `resourceSelectors`, ou certains champs d'`enforcementMode` n'y sont pas toujours disponibles au moment où on en a besoin.

Le provider **`azapi`** résout ça proprement. Il prend le body ARM directement en HCL et Terraform gère le cycle de vie de la ressource (création, update, suppression, state), sans passer par un `azurerm_resource_group_template_deployment` qui déploierait un template ARM entier comme une boîte noire.

```hcl
terraform {
  required_providers {
    azapi = {
      source  = "Azure/azapi"
      version = "~> 2.0"
    }
  }
}
```

### azapi & le versionning

Puisqu'on part sur Terraform + `azapi`, la réponse naturelle est de traiter chaque service du catalogue comme un **module Terraform versionné**. Chaque assignment dans la couche `assignments/` déclare une source de module avec un ref git précis :

```hcl
module "initiative_storage" {
  source = "git::https://github.com/org/policy-catalog.git//catalog/storage/initiative?ref=storage-v1.2.0"
}
```

La mise à jour d'un service pour une BU ou un environnement est donc un changement de `ref` dans un fichier Terraform, visible en PR, ciblé et sans effet de bord sur les autres BUs. BU1 peut rester sur `storage-v1.2.0` pendant que BU2 passe à `storage-v2.0.0`, le tout dans le même monorepo.


### Déployer une définition de policy

Dans le catalogue, chaque policy est définie dans son propre fichier. Terraform déploie la définition sur le management group cible et en gère l'état.

```hcl
resource "azapi_resource" "policy_no_public_ip" {
  type      = "Microsoft.Authorization/policyDefinitions@2024-05-01"
  name      = "deny-public-ip"
  parent_id = "/providers/Microsoft.Management/managementGroups/bu1-dev"

  body = {
    properties = {
      displayName = "Deny Public IP addresses"
      policyType  = "Custom"
      mode        = "All"
      policyRule = {
        if = {
          field  = "type"
          equals = "Microsoft.Network/publicIPAddresses"
        }
        then = {
          effect = "Deny"
        }
      }
    }
  }
}
```

### Gérer les overrides

C'est là que la granularité devient importante. Les deux scénarios décrits n'ont pas la même solution.

#### Scénario 1 : Override au niveau du management group (BU1-DEV, `deny-storage-public-access` désactivée)

L'initiative `storage` est assignée au MG `bu1-dev`. En DEV, on ne veut pas bloquer l'accès public aux Storage Accounts, trop contraignant pour les développeurs. On désactive la policy `deny-storage-public-access` via un override sur l'assignment du MG, sans toucher à la définition de l'initiative.

```hcl
resource "azapi_resource" "assignment_bu1_dev_storage" {
  type      = "Microsoft.Authorization/policyAssignments@2025-01-01"
  name      = "initiative-storage-bu1-dev"
  parent_id = "/providers/Microsoft.Management/managementGroups/bu1-dev"

  body = {
    properties = {
      policyDefinitionId = "/providers/Microsoft.Management/managementGroups/contoso/providers/Microsoft.Authorization/policySetDefinitions/initiative-storage"
      enforcementMode    = "Default"
      overrides = [
        {
          kind  = "policyEffect"
          value = "Disabled"
          selectors = [
            {
              kind = "policyDefinitionReferenceId"
              in   = ["deny-storage-public-access"]
            }
          ]
        }
      ]
    }
  }
}
```

#### Scénario 2 : Override au niveau d'une souscription spécifique (`sampleSubscription` en PRD BU2, `deny-keyvault-public-access` désactivée)

L'initiative `keyvault` est assignée au MG `bu2-prd` et couvre toutes les souscriptions dessous. La souscription `sampleSubscription` héberge un workload legacy dont les Key Vaults ne peuvent pas encore être mis en conformité. La policy `deny-keyvault-public-access` doit être désactivée uniquement pour cette souscription, sans impacter le reste de `bu2-prd`.

Les `overrides` ARM ne permettent pas de cibler une souscription spécifique dans leurs selectors (uniquement `resourceLocation` et `policyDefinitionReferenceId`). La solution est en deux temps :

1. Exclure `sampleSubscription` de l'assignment MG via `notScopes`
2. Créer un assignment dédié à `sampleSubscription` avec l'override sur `deny-keyvault-public-access`

```hcl
# Assignment de base au niveau MG — sampleSubscription exclu
resource "azapi_resource" "assignment_bu2_prd_keyvault" {
  type      = "Microsoft.Authorization/policyAssignments@2025-01-01"
  name      = "initiative-keyvault-bu2-prd"
  parent_id = "/providers/Microsoft.Management/managementGroups/bu2-prd"

  body = {
    properties = {
      policyDefinitionId = "/providers/Microsoft.Management/managementGroups/contoso/providers/Microsoft.Authorization/policySetDefinitions/initiative-keyvault"
      enforcementMode    = "Default"
      notScopes = [
        "/subscriptions/sampleSubscription-id"
      ]
    }
  }
}

# Assignment spécifique pour sampleSubscription avec override sur deny-keyvault-public-access
resource "azapi_resource" "assignment_bu2_prd_keyvault_sample" {
  type      = "Microsoft.Authorization/policyAssignments@2025-01-01"
  name      = "initiative-keyvault-sampleSubscription"
  parent_id = "/subscriptions/sampleSubscription-id"

  body = {
    properties = {
      policyDefinitionId = "/providers/Microsoft.Management/managementGroups/contoso/providers/Microsoft.Authorization/policySetDefinitions/initiative-keyvault"
      enforcementMode    = "Default"
      overrides = [
        {
          kind  = "policyEffect"
          value = "Disabled"
          selectors = [
            {
              kind = "policyDefinitionReferenceId"
              in   = ["deny-keyvault-public-access"]
            }
          ]
        }
      ]
    }
  }
}
```

La souscription `sampleSubscription` reçoit bien toutes les policies de l'initiative `keyvault`, à l'exception de `deny-keyvault-public-access`. Le reste de `bu2-prd` n'est pas impacté.

> Ce pattern `notScopes` + assignment dédié à la souscription est une façon valide de gérer des exceptions sub-scope. L'inconvénient : il faut suivre ces assignments d'exception explicitement dans le repository, d'où l'importance d'une structure `assignments/bu2/prd/sampleSubscription/` pour les cas particuliers.

### Alternative : les Policy Exemptions

Azure Policy propose une alternative officielle pour les exceptions ciblées : les **Policy Exemptions** (`Microsoft.Authorization/policyExemptions`). Elles présentent une différence fondamentale avec `notScopes` :

| Mécanisme | Visibilité conformité | Expiration | Usage recommandé |
|---|---|---|---|
| `notScopes` | Invisible dans les rapports | Aucune | Exclusions permanentes et larges (ex : sandbox entier) |
| Exemptions | Visible comme "Exempt" | `expiresOn` configurable | Exceptions temporaires ou ciblées avec traçabilité |

Pour le cas de `sampleSubscription`, une exemption serait plus appropriée que le pattern `notScopes` si l'exception est temporaire (migration prévue, workload legacy en cours de mise en conformité) : elle reste visible dans les rapports de conformité et impose une date de révision, ce qui évite qu'elle dure indéfiniment. Microsoft recommande d'ailleurs les exemptions pour les scénarios temporaires ou spécifiques, et `notScopes` uniquement pour les exclusions permanentes et larges.

## Bonnes pratiques

### Ne jamais assigner directement sur le Tenant Root Group

Comme vu dans la section architecture, le Tenant Root Group doit rester une ancre neutre. Toute policy ou initiative posée à ce niveau hérite sur l'intégralité du tenant, y compris les souscriptions de plateforme, de sandbox et de management. Un `Deny` mal calibré à ce niveau peut bloquer des opérations critiques difficiles à diagnostiquer.

La règle est simple : le premier niveau d'assignment opérationnel commence à partir de l'Intermediate Root Management Group.

### Conventions de nommage

Un nommage cohérent est indispensable pour naviguer dans un catalogue qui grossit. Voici une convention qui fonctionne bien en pratique :

| Élément | Format | Exemple |
|---|---|---|
| Policy definition | `[service]-[effet]-[description]` | `storage-deny-public-access` |
| Initiative | `initiative-[service]` | `initiative-keyvault` |
| Assignment (MG) | `[initiative]-[bu]-[env]` | `initiative-storage-bu1-dev` |
| Assignment (subscription) | `[initiative]-[subscription]` | `initiative-keyvault-sampleSubscription` |
| Tag de version | `[service]-v[major].[minor].[patch]` | `storage-v1.2.0` |

Le `policyDefinitionReferenceId` utilisé dans les overrides doit correspondre exactement à la valeur définie dans l'initiative (`policySetDefinition`) pour ce champ — ce n'est pas nécessairement le nom ARM de la policy definition. Si ce champ n'est pas renseigné explicitement dans l'initiative, il prend par défaut le dernier segment de l'ID ARM de la définition. C'est ce référenceId qui fait le lien entre l'initiative et les overrides dans les assignments.

### Gestion des versions

Le versioning des modules du catalogue suit un semver strict avec une signification précise pour chaque niveau :

- **Patch** (`v1.0.x`) : correction d'un libellé, d'un paramètre non bloquant, d'une metadata. Aucun impact fonctionnel.
- **Minor** (`v1.x.0`) : ajout d'une nouvelle policy dans l'initiative. Les assignments existants ne sont pas affectés tant qu'ils ne mettent pas à jour leur `ref`.
- **Major** (`vx.0.0`) : changement d'effet (`Audit` → `Deny`, ajout d'un `DeployIfNotExists`...). Potentiellement bloquant. La mise à jour vers une version majeure doit passer par le cycle `DoNotEnforce` → validation → `Default`.

Chaque BU met à jour sa `ref` de module indépendamment, via une PR dédiée. On évite ainsi les mises à jour en masse non contrôlées.

### Toujours déployer en `DoNotEnforce` d'abord

Toute nouvelle initiative ou toute montée de version majeure doit être déployée en `enforcementMode: "DoNotEnforce"` dans un premier temps, y compris en PRD. On laisse tourner quelques jours, on observe les résultats de conformité dans le portail Azure, et on bascule en `Default` uniquement une fois que le niveau de non-conformité est compris et maîtrisé.

Ne jamais déployer directement en `Default` sur un scope de production sans avoir validé l'impact au préalable.

### Managed Identity pour les effets actifs

Les effets `DeployIfNotExists` et `Modify` nécessitent que l'assignment dispose d'une **Managed Identity** avec les permissions suffisantes pour effectuer les opérations de remédiation. Sans elle, la policy est évaluée mais la remédiation échoue silencieusement. C'est une source fréquente d'erreur lors de la mise en place d'une baseline DINE.

En Terraform avec `azapi`, cela se traduit par un bloc `identity` sur le resource d'assignment :

```hcl
resource "azapi_resource" "assignment_dine" {
  type      = "Microsoft.Authorization/policyAssignments@2025-01-01"
  name      = "initiative-diagnostics-bu1-prd"
  parent_id = "/providers/Microsoft.Management/managementGroups/bu1-prd"

  identity {
    type = "SystemAssigned"
  }

  body = {
    properties = {
      policyDefinitionId = "..."
      enforcementMode    = "DoNotEnforce"
    }
  }
}
```

Le rôle attribué à cette identité (ex : `Monitoring Contributor`) doit être défini au niveau du scope de l'assignment ou en dessous.

### Documenter les exceptions

Tout `notScopes` ou override posé dans un assignment d'exception doit être accompagné d'un commentaire dans le code Terraform précisant la raison et si possible une date de révision prévue. Sans ça, ces exceptions deviennent invisibles et persistent indéfiniment.

```hcl
# Exception temporaire : legacy workload non compatible avec firewall KV
# À revoir après migration prévue en Q3 2026
resource "azapi_resource" "assignment_bu2_prd_keyvault_sample" {
  ...
}
```

## Conclusion

L'approche décrite dans cet article pose des bases solides pour industrialiser la gestion des Azure Policies : un catalogue versionné par service, des assignments isolés par BU et environnement, et un provider `azapi` qui donne accès à l'intégralité des fonctionnalités ARM sans friction.

Elle résout les problèmes les plus concrets du terrain : traçabilité via git, isolation des cycles de vie, gestion des exceptions par overrides ou `notScopes`, et déploiement sécurisé via `DoNotEnforce`.

Cela dit, cette architecture a ses limites.

**Ce qui manque ou reste fragile :**

- **Le suivi des overrides dans le temps.** Rien dans cette solution ne garantit qu'un override posé en urgence sera bien retiré. Il n'y a pas de mécanisme natif pour suivre les transitions `DoNotEnforce` → `Default` ou pour alerter quand un override dépasse une durée acceptable.
- **La visibilité globale.** Pour savoir ce qui est en `DoNotEnforce` sur l'ensemble du tenant, il faut aller lire le state Terraform de chaque workspace. Il n'y a pas de vue consolidée des états d'enforcement à l'échelle.
- **La complexité du state Terraform.** Avec plusieurs BUs, plusieurs environnements et des assignments d'exception par souscription, le nombre de workspaces Terraform à maintenir croît rapidement. Sans une organisation rigoureuse du backend, ça devient difficile à opérer.
- **L'absence de pipeline de validation.** On peut tagger une version et l'appliquer, mais rien n'automatise la vérification de conformité après déploiement ni ne déclenche automatiquement la bascule vers `Default` une fois le seuil atteint.
- **Les mises à jour cross-BU restent manuelles.** Quand une politique de sécurité doit être appliquée à toutes les BUs simultanément (incident, nouvelle exigence réglementaire), il faut ouvrir autant de PRs qu'il y a de BUs. C'est lent et source d'oubli.

Pour aller plus loin, il manque un outil capable de piloter ce cycle de vie de bout en bout : visualiser l'état d'enforcement de chaque assignment à travers tout le tenant, déclencher les transitions de mode de façon contrôlée, et alerter sur les overrides qui traînent. ça sera possiblement le sujet d'un futur article.

## Liens

- [Effets Azure Policy — Microsoft Learn](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effect-basics)
- [Effet AuditIfNotExists — Microsoft Learn](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effect-audit-if-not-exists)
- [Effet DenyAction — Microsoft Learn](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effect-deny-action)
- [Structure d'un assignment Azure Policy — Microsoft Learn](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/assignment-structure)
- [Structure d'une exemption Azure Policy — Microsoft Learn](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/exemption-structure)
- [Safe deployment practices for Azure Policy — Microsoft Learn](https://learn.microsoft.com/en-us/azure/governance/policy/how-to/policy-safe-deployment-practices)
- [Remédier les ressources non-conformes — Microsoft Learn](https://learn.microsoft.com/en-us/azure/governance/policy/how-to/remediate-resources)
- [CAF — Management groups design area — Microsoft Learn](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/resource-org-management-groups)
- [Provider azapi — Terraform Registry](https://registry.terraform.io/providers/Azure/azapi/latest/docs)
- [Référence API Microsoft.Authorization/policyDefinitions — Microsoft Learn](https://learn.microsoft.com/en-us/azure/templates/microsoft.authorization/policydefinitions)
- [Référence API Microsoft.Authorization/policyAssignments — Microsoft Learn](https://learn.microsoft.com/en-us/azure/templates/microsoft.authorization/policyassignments)
