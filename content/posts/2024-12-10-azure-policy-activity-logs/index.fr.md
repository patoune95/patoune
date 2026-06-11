---
title: "Azure Policy : lire les Activity Logs pour diagnostiquer un Deny"
date: 2024-12-10T10:00:00+02:00
description: "Comment filtrer les Activity Logs sur Microsoft.Authorization pour identifier quel assignment a bloqué quelle ressource, et pourquoi."
draft: false
tags:
  - azure
  - policy
  - governance
  - monitoring
cover:
  image: cover.jpg
  alt: "Azure policy Custom message"
  relative: true
---

Quand une Azure Policy bloque un déploiement, le message d'erreur dans Terraform ou dans le portail indique souvent l'assignment responsable, mais pas toujours le détail complet de la règle violée. Les Activity Logs Azure gardent la trace complète de chaque refus.

## Où chercher dans le portail

Les Activity Logs sont accessibles depuis plusieurs endroits : au niveau subscription, resource group, ou directement depuis une ressource. Pour diagnostiquer un Deny policy, allez dans **Activity Log**.

![activity_log](screenshoot-1.png)

## Filtrer les événements Policy

Les événements liés aux Azure Policies ont une opération de type `Microsoft.Authorization/*`.

Pour isoler les Deny :

1. Dans l'Activity Log, cliquer sur **Add filter**
2. Filtre **Operation** : rechercher `Microsoft.Authorization/policies`
3. Filtre **Status** : `Failed` (les Deny apparaissent comme des opérations échouées)
4. Ajuster la plage de temps selon vos besoins

![activity_log_filters](screenshoot-2.jpg)

## Lire le détail d'un événement Deny

Cliquer sur un événement pour voir le détail. Les informations clés sont dans l'onglet **JSON** de l'événement :

Le champ `policies` contient :

- `policyAssignmentName` : le nom de l'assignment qui a déclenché le Deny
- `policyDefinitionName` : la policy précise dans l'initiative
- `policySetDefinitionName` : l'initiative parente (si applicable)

![activity_log_error_details](screenshoot-3.jpg)

## Via PowerShell

Pour filtrer en ligne de commande sur les dernières 24h :

```powershell
$startTime = (Get-Date).AddHours(-24)

Get-AzActivityLog `
  -ResourceGroupName "mon-rg" `
  -StartTime $startTime `
  -Status "Failed" |
  Select-Object EventTimestamp, Caller, ResourceId, Status
```

Pour obtenir le détail JSON complet d'un événement spécifique :

```powershell
$startTime = Get-Date "2024-12-10T08:00:00Z"

Get-AzActivityLog `
  -ResourceGroupName "mon-rg" `
  -StartTime $startTime `
  -Status "Failed" |
  Select-Object -First 1 |
  Select-Object -ExpandProperty Properties |
  ConvertTo-Json -Depth 10
```

## Via KQL dans Log Analytics

Si les Activity Logs sont envoyés vers un workspace Log Analytics (recommandé en production) :

```kql
AzureActivity
| where OperationNameValue startswith "microsoft.authorization/policies"
| where ActivityStatusValue == "Failed"
| extend policyInfo = parse_json(Properties).policies
| project
    TimeGenerated,
    Caller,
    ResourceId,
    OperationName,
    policyInfo
| order by TimeGenerated desc
```

Voici une façon rapide de diagnostiquer pourquoi un déploiement a échoué.

> 💡 Dans un autre article, j'avais présenté un système permettant de personnaliser les messages de `Deny` d'une policy. La mise en place de ces messages permettra de gagner du temps sur vos futures investigations dans l'Activity Log.
