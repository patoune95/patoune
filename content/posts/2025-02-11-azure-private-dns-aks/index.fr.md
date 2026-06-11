---
title: "Azure Private DNS et AKS : résoudre les Private Endpoints depuis le cluster"
date: 2025-02-11T10:00:00+02:00
description: "Zones DNS privées, Virtual Network Links, split-horizon DNS, et le cas concret de PostgreSQL Flexible Server accessible depuis AKS."
draft: false
tags:
  - azure
  - kubernetes
  - aks
  - réseau
  - dns
  - private-endpoint
cover:
  image: cover.jpeg
  alt: "Azure Private DNS and AKS"
  relative: true
---

Les Private Endpoints Azure attachent un service managé (PostgreSQL, Key Vault, Storage...) au réseau privé via une IP interne. Le problème : pour que les pods AKS puissent résoudre le FQDN de cette ressource vers son IP privée plutôt que vers son IP publique, il faut configurer correctement les zones DNS privées et les Virtual Network Links.

## Architecture du problème

Quand Azure crée un Private Endpoint pour, par exemple, un PostgreSQL Flexible Server, il crée automatiquement une zone DNS privée du type `privatelink.postgres.database.azure.com`. Cette zone contient un enregistrement A qui mappe le FQDN du serveur vers l'IP privée du Private Endpoint.

Mais pour que les VMs (et les pods AKS) dans un VNet puissent utiliser cette zone DNS privée, il faut créer un **Virtual Network Link** entre la zone privée et le VNet du cluster AKS.

Sans ce lien, les pods AKS résoudront le FQDN via les DNS publics d'Azure et obtiendront... l'IP privée (Azure DNS retourne quand même l'IP privée pour les endpoints privés dans le même tenant), mais si le cluster est dans un VNet différent sans peering ou sans le bon VNet Link, la résolution échouera.

![recordset](screenshoot-1.jpg)

## Les zones DNS privées par service

Chaque type de ressource Azure a sa propre zone DNS privée pour les Private Endpoints :

| Service                    | Zone DNS privée                           |
| -------------------------- | ----------------------------------------- |
| PostgreSQL Flexible Server | `privatelink.postgres.database.azure.com` |
| Azure SQL Database         | `privatelink.database.windows.net`        |
| Key Vault                  | `privatelink.vaultcore.azure.net`         |
| Storage Account (blob)     | `privatelink.blob.core.windows.net`       |
| Azure Container Registry   | `privatelink.azurecr.io`                  |

## Configurer le Virtual Network Link

### Ce qu'est un Virtual Network Link

Un Virtual Network Link n'est pas une règle réseau ni un tunnel, c'est uniquement une **autorisation DNS**.

> 💡 Il dit à Azure : "ce VNet a le droit de consulter cette zone DNS privée".

Sans le link, quand un pod fait une requête DNS, le résolveur Azure[^azure-dns-ip] ignore la zone privée et ne trouve pas la réponse :

```
Pod AKS → requête DNS → résolveur Azure 168.63.129.16
                              ↓
                    VNet linké à la zone ? ❌
                              ↓
                    zone ignorée → résolution échoue
```

Avec le link :

```
Pod AKS → requête DNS → résolveur Azure 168.63.129.16
                              ↓
                    VNet linké à la zone ? ✅
                              ↓
                    retourne l'IP privée du Private Endpoint
```

C'est tout. Pas de configuration réseau supplémentaire, juste ce lien entre le VNet et la zone.

### En Terraform :

```hcl
# Zone DNS privée (déclarée explicitement ; en mode portail, l'intégration DNS du Private Endpoint peut la créer pour vous)
resource "azurerm_private_dns_zone" "postgres" {
  name                = "privatelink.postgres.database.azure.com"
  resource_group_name = azurerm_resource_group.main.name
}

# Lien entre la zone privée et le VNet AKS
resource "azurerm_private_dns_zone_virtual_network_link" "postgres_aks" {
  name                  = "link-postgres-aks-vnet"
  resource_group_name   = azurerm_resource_group.main.name
  private_dns_zone_name = azurerm_private_dns_zone.postgres.name
  virtual_network_id    = azurerm_virtual_network.aks.id
  registration_enabled  = false   # Pas d'enregistrement auto des VMs
}
```

![virtual-network-link](screenshoot-2.jpg)

## Le cas PostgreSQL Flexible Server

PostgreSQL Flexible Server est un cas légèrement plus complexe car il utilise une zone DNS privée différente selon le mode de déploiement. En mode Private Access (VNet Integration) sans Private Endpoint, il s'appuie sur une zone de la forme `<nom-serveur>.private.postgres.database.azure.com`. En mode Private Endpoint, il utilise `privatelink.postgres.database.azure.com`.

### Vérifier la résolution depuis un pod

```bash
# Lancer un pod de debug temporaire
kubectl run dns-test --image=nicolaka/netshoot --rm -it --restart=Never -- bash

# Dans le pod, tester la résolution
nslookup mon-serveur.postgres.database.azure.com
nslookup mon-serveur.privatelink.postgres.database.azure.com

# Tester la connectivité TCP
nc -zv mon-serveur.postgres.database.azure.com 5432
```

Si `nslookup` retourne une IP en `10.x.x.x` (IP privée du Private Endpoint), la résolution fonctionne. Si ça retourne une IP publique ou une erreur NXDOMAIN, le Virtual Network Link est manquant ou mal configuré.

## Split-horizon DNS

Le split-horizon (ou split-brain DNS) est la situation où le même FQDN retourne une IP différente selon l'origine de la requête :

- Depuis le VNet Azure (avec le VNet Link) → IP privée du Private Endpoint
- Depuis internet → IP publique (ou NXDOMAIN si pas d'accès public)

C'est le comportement attendu et voulu avec les Private Endpoints. Les pods AKS, qui sont dans le VNet, doivent toujours recevoir l'IP privée.

## Pièges courants

**VNet Link manquant sur le bon VNet.** Si AKS est dans le VNet A mais que le Private Endpoint est dans le VNet B (relié par peering), le VNet Link doit être créé sur le **VNet A** (le VNet du cluster AKS), pas sur le VNet B.

**Multiple VNets dans un hub-and-spoke.** Dans une architecture hub-and-spoke, la zone DNS privée est souvent liée au VNet hub. Pour que les pods dans un VNet spoke puissent résoudre, il faut soit lier la zone à chaque spoke, soit s'assurer que le DNS du spoke transmet les requêtes au DNS du hub (Azure DNS Resolver).

**Kube-dns coreDNS.** CoreDNS dans AKS transmet les requêtes non-Kubernetes à Azure DNS (168.63.129.16). Si Azure DNS peut résoudre via la zone privée (grâce au VNet Link), tout fonctionne. Si non, la requête va vers les DNS publics et retourne l'IP publique.

## Vérification finale

```bash
# Depuis un pod dans le cluster
kubectl run dns-check --image=alpine --rm -it --restart=Never -- sh -c "
  apk add --no-cache bind-tools postgresql-client &&
  nslookup mon-serveur.postgres.database.azure.com &&
  psql -h mon-serveur.postgres.database.azure.com -U adminuser -c 'SELECT version();'
"
```

## Ressources

- [What is IP address 168.63.129.16?](https://learn.microsoft.com/fr-fr/azure/virtual-network/what-is-ip-address-168-63-129-16)
- [Azure Private DNS](https://learn.microsoft.com/fr-fr/azure/dns/private-dns-overview)
- [Virtual network links](https://learn.microsoft.com/fr-fr/azure/dns/private-dns-virtual-network-links)

[^azure-dns-ip]: `168.63.129.16` est l'IP virtuelle du résolveur DNS d'Azure, identique dans tous les VNets. Elle est réservée par Microsoft et uniquement accessible depuis l'intérieur d'un VNet. [Documentation Microsoft](https://learn.microsoft.com/fr-fr/azure/virtual-network/what-is-ip-address-168-63-129-16)
