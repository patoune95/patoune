---
title: "Présentation de Microsoft Retina"
date: 2024-03-25T23:03:19+01:00
description: "Découverte de Microsoft Retina, outil open-source de surveillance réseau pour AKS : modes de capture via CLI ou CRD, métriques Basic et Advanced pour Prometheus."
tags:
  - azure
  - retina
  - aks
  - monitoring
cover:
  image: cover.jpg
  alt: "Retina"
  relative: true
---

Durant ma participation à la KubeCon, j'ai eu l'occasion de pouvoir participer à la *Azure day with Kubernetes*.
Un nouvel outil en cours de développement a été présenté : `retina`

## Qu'est-ce que Retina ?

Retina est un outil permettant de faire une surveillance du réseau et des services AKS. L'objectif de Retina est de récolter de la télémétrie personnalisable et de la pousser sur différents services (Prometheus, via un PV etc...).

## Fonctionnement

Retina propose plusieurs modes de récolte d'information:

### *Via la CLI* :

Dans ce mode de capture, le lancement d'une commande via la cli Retina permet la création d'un job Kubernetes qui va effectuer une capture en se basant sur un node selector. La durée par défaut est d'une minute mais il est possible de le configurer via la commande de la CLI via `--no-wait=true` pour que ça tourn en continue. D'autres options sont possibles comme capper la taille du fichier de sortie pour la capture, de faire du filtrage par **\[ip\]:\[port\]** ou encore via une DNS query **udp port 53**.

*Exemple de capture avec une durée de 10 minutes et une taille de fichier de sortie max à 100MB*

```powershell
kubectl retina capture create --host-path /mnt/capture \ 
--namespace capture --node-selectors "kubernetes.io/os=linux" \ 
--duration=10m --max-size=100
```

*Schema de fonctionnement via la CLI:*
![retina-cli-schema](https://retina.sh/assets/images/capture-architecture-without-operator-8b30bdccc806411433443aee233e699a.png)

### *Via le déploiement d'une CRD (Custom Resource Definition)*:

Dans ce mode de fonctionnement, le déploiement d'une ressource kube *Capture* va permettre de pouvoir avoir un service qui va faire une récolte continue.

*Exemple de définition:*

```yaml
apiVersion: retina.sh/v1alpha1
kind: Capture
metadata:
  name: example-capture
spec:
  captureConfiguration:
    captureOption:
      duration: "30s"
      maxCaptureSize: 100
      packetSize: 1500
    captureTarget:
      namespaceSelector:
        matchLabels:
          app: target-app
  outputConfiguration:
    hostPath: /captures
    blobUpload: blob-sas-url
```

*Schema de fonctionnement via le déploiement d'une CRD:*
![retina-crd-schema](https://retina.sh/assets/images/capture-architecture-with-operator-2fd7bdbb30046b54d4075aa48de7bf8a.png)

## Que capture Retina ?

Retina propose différents modes de captures:

- Basic : Rapatriement des mesures agrégées par Node (cluster / instance) et leurs métriques  (Nombre / taille de paquets transmis, Nombre / taille de paquets perdus, informations que retournerait la commande *netstats*)

- Advanced : Métrique *Basic* avec en plus des données de contexte (source/destination ip, source/destination namespace, source/destination pod, source/destination workload) et des métriques supplémentaires lié au DNS. Les métriques concernant les paquets sont quant à elles plus détaillées.

### Des métriques paramétrables et extensibles

Toutes ces metriques sont basées sur un système de plugins qu'il faut définir à l'installation de Retina. J'ignore pour le moment s'il est possible de pouvoir en ajouter ou en supprimer post-installation mais j'imagine que cette fonctionnalité sera disponible par la suite.

## Et après ?

De mon point de vue, je trouve que cet outil permet de combler un défaut que l'on rencontrait sur AKS, à savoir : Le manque de métrique. L'outil est actuellement en version *0.0.3* à l'heure où j'écris cet article mais il y a une forte communauté autour de ce projet et j'imagine qu'il va rapidement prendre de l'ampleur. La possibilité d'enrichir les métriques avec du contenu personnalisable est déjà prévue et répondra pour beaucoup à un besoin d'y ajouter ses propres informations. Pour ma part, je ne l'utiliserais pas encore en production car le projet est encore trop immature mais je pense qu'il faut y garder un oeil (ou une rétine 😁).Ce projet semble en tout cas très prometteur.

## Sources

- [Site officiel de Retina](https://retina.sh/)
- [Repository GitHub](https://github.com/microsoft/retina)
