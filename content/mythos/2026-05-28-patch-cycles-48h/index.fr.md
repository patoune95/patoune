---
title: "Raccourcir les cycles de patch : pourquoi 48h n'est plus une option pour les CVEs critiques"
date: 2026-05-28T10:00:00+02:00
description: "Les vulnérabilités les plus critiques sont exploitées en quelques heures. Voici comment restructurer ses cycles de patch pour répondre en moins de 48h."
draft: false
author: Thomas L.
tags:
  - security
  - patch-management
  - vulnerability
  - devops
  - ci-cd
cover:
  image: /mythos/cover.jpg
  alt: "Pipeline de réponse aux CVEs critiques, Mythos"
  relative: false
---

## Le contexte : des vulnérabilités vieilles de décennies, toujours exploitables

Le rapport Mythos Preview (avril 2026) a mis en lumière une réalité inconfortable : parmi les 10 000+ vulnérabilités identifiées, les plus critiques ne sont pas nécessairement les plus récentes. Le bug SACK TCP dans OpenBSD traîne depuis 27 ans. Le buffer overflow FFmpeg H.264 depuis 16 ans. Le stack overflow NFS dans FreeBSD depuis 17 ans.

Ce qui a changé, c'est la vitesse à laquelle ces failles peuvent désormais être exploitées. Avec l'émergence d'outils d'exploitation assistés par IA, la fenêtre entre publication d'un patch et exploitation active se mesure maintenant en heures, pas en semaines.

Les cycles de patch mensuels ou hebdomadaires ne sont plus acceptables pour les CVEs critiques.

## Le problème : la fenêtre d'exploitation rétrécit

La grande majorité des organisations fonctionnent encore selon des cycles de patch définis par la logique opérationnelle : une fenêtre de maintenance hebdomadaire, un cycle mensuel Patch Tuesday, ou pire, un processus de change management qui ajoute plusieurs semaines de délai entre la publication d'un patch et son déploiement.

Ce modèle avait du sens dans un monde où l'exploitation d'une vulnérabilité nécessitait des semaines de recherche et d'analyse. Ce n'est plus le monde dans lequel on opère.

Les trois risques principaux :

- **Exploitation automatisée** : des modèles IA peuvent analyser un patch publié, déduire la vulnérabilité sous-jacente par diff, et générer un exploit en quelques heures
- **Asymétrie des moyens** : l'attaquant n'a besoin de réussir qu'une fois, le défenseur doit bloquer toutes les tentatives
- **Visibilité publique** : chaque CVE publié avec un score CVSS ≥ 9.0 génère immédiatement une attention massive dans la communauté offensive

> ⚠️ Le FreeBSD NFS stack overflow (CVE-2026-4747) autorise un RCE non-authentifié via 6 paquets séquentiels. Sur une infrastructure NFS exposée, un attaquant peut compromettre une machine en quelques secondes une fois l'exploit disponible.

## La solution : un pipeline de patch structuré par criticité

### Définir des SLOs par niveau de sévérité

La première étape est de formaliser des SLOs (Service Level Objectives) de patching qui créent une obligation contractuelle interne :

| Sévérité CVSS | Délai maximum | Mode de déploiement |
|---|---|---|
| Critical (≥ 9.0) | 48h | Déploiement d'urgence, bypass change management standard |
| High (7.0 – 8.9) | 7 jours | Déploiement accéléré avec revue light |
| Medium (4.0 – 6.9) | 30 jours | Cycle normal |
| Low (< 4.0) | 90 jours | Prochaine release planifiée |

Ces SLOs doivent être outillés, pas juste documentés. Une CVE critique connue et non patchée au-delà de 48h doit déclencher une alerte automatique.

### Automatiser la détection et la qualification

Le pipeline commence par la surveillance continue des sources de CVEs :

```bash
# Exemple de souscription aux flux NVD via API
curl -s "https://services.nvd.nist.gov/rest/json/cves/2.0?cvssV3Severity=CRITICAL&resultsPerPage=10" \
  | jq '.vulnerabilities[].cve | {id: .id, published: .published, description: .descriptions[0].value}'
```

En pratique, on n'interroge pas l'API NVD à la main. On intègre un outil de scanning dans le pipeline CI/CD :

- **Trivy** pour les images container et les dépendances applicatives
- **Grype** comme alternative à Trivy pour le scanning de CVEs
- **Syft** pour générer des SBOMs (Software Bill of Materials) auditables
- **Dependabot** pour les dépendances de code source

L'idée est de recevoir une alerte au moment où la CVE est publiée, pas lors du prochain scan planifié.

### Structurer le pipeline de réponse

Un pipeline de patch d'urgence efficace ressemble à ça :

![Workflow de securité](security_pipeline.svg)

- 🔴 = Urgence (CVE, alertes, deployment)
- 🟠 = Phase de qualification
- 🟣 = Point de décision
- 🟢 = Succès (monitoring, closed)
- 🟡 = Branche "Exposed + Patch"

### La qualification rapide : éviter les faux positifs

Pas toutes les CVEs critiques nécessitent une intervention immédiate. La qualification doit répondre à trois questions :

1. **Le composant vulnérable est-il présent dans notre stack ?** Un buffer overflow dans OpenBSD n'est pas urgent si l'infra tourne sur Ubuntu.
2. **La surface d'attaque est-elle exposée ?** Un RCE dans NFS est critique uniquement si NFS est accessible depuis un réseau non-maîtrisé.
3. **Un patch est-il disponible ?** Si non, quelles mitigations temporaires sont applicables immédiatement ?

> 💡 Une matrice d'inventaire des composants (SBOM, Software Bill of Materials) est un pré-requis pour répondre à la première question en moins de 30 minutes. Sans SBOM, la qualification devient une recherche manuelle qui consomme précisément le temps qu'on essaie de gagner.

## Mise en pratique : intégrer Trivy dans un pipeline CI/CD

Voici un exemple d'intégration dans GitHub Actions qui bloque le déploiement si une CVE critique est détectée :

```yaml
name: Security Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 */6 * * *'  # scan toutes les 6h

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Scan image pour CVEs critiques
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'my-app:latest'
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL'
          ignore-unfixed: true
```

L'option `exit-code: '1'` bloque le pipeline si une CVE critique non-patchée est détectée sur le composant scanné. `ignore-unfixed: true` évite de bloquer sur des vulnérabilités pour lesquelles aucun patch n'est encore disponible ; ces cas peuvent être documentés séparément.

Pour les dépendances applicatives, Dependabot peut ouvrir automatiquement des PRs de mise à jour :

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "daily"
    open-pull-requests-limit: 10
    groups:
      security-patches:
        applies-to: security-updates
        update-types:
          - "patch"
          - "minor"
```

## Trivy, un outil performant mais qui peut poser question

⚠️ **Trivy a été compromis deux fois en 2026.**

Une première brèche le 28 février via un workflow GitHub Actions mal configuré, puis une attaque majeure le 19 mars par le groupe TeamPCP : 76 tags de release rétroactivement empoisonnés, des binaires malveillants distribués sur GitHub Releases, Docker Hub et Amazon ECR.

Les versions infectées volaient des secrets CI/CD et déployaient un mécanisme de persistance systemd. Les versions sûres sont `v0.69.3+` pour le scanner, `v0.35.0+` pour `trivy-action` et `v0.2.6+` pour `setup-trivy`. Avant d'intégrer Trivy dans votre pipeline, vérifiez la date de l'advisory officiel Aqua Security et envisagez des alternatives selon votre contexte. (sources en bas avec les liens)

## Grype et Syft : une alternative SBOM-first

Étant donné la compromission de Trivy, **Grype** et **Syft** (tous deux maintenus par Anchore) constituent une alternative solide et cohérente.

La philosophie est différente de Trivy : au lieu de scanner directement l'image ou le dépôt, on sépare les deux étapes.

1. **Syft** génère un SBOM, soit un inventaire exhaustif de tous les composants présents dans l'artefact
2. **Grype** scanne ce SBOM pour détecter les CVEs connues

Cette séparation a un avantage concret : le SBOM peut être stocké, versionné, et réutilisé pour des scans ultérieurs sans reconstruire l'image.

### Générer un SBOM avec Syft

```bash
# Scanner une image container
syft my-app:latest -o spdx-json > sbom.spdx.json

# Scanner un répertoire (code source, dépendances)
syft dir:. -o spdx-json > sbom.spdx.json

# CycloneDX : format plus compact, bien supporté par les outils downstream
syft my-app:latest -o cyclonedx-json > sbom.cdx.json
```

Le format SPDX est le standard recommandé pour l'interopérabilité. CycloneDX est plus compact et mieux supporté par certains outils d'analyse.

### Scanner le SBOM avec Grype

```bash
# Scanner directement une image
grype my-app:latest

# Scanner un SBOM existant
grype sbom:./sbom.spdx.json

# Bloquer uniquement sur les CVEs critiques avec un patch disponible
grype sbom:./sbom.spdx.json --only-fixed --fail-on critical
```

L'option `--fail-on critical` fait échouer le processus avec un code de retour non-zéro si une CVE critique est trouvée, directement intégrable dans un pipeline CI/CD.

### Intégration dans GitHub Actions

```yaml
name: SBOM + Vulnerability Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 */6 * * *'

jobs:
  sbom-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Générer le SBOM
        uses: anchore/sbom-action@v0
        with:
          image: my-app:latest
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Scanner les CVEs
        uses: anchore/scan-action@v3
        with:
          sbom: sbom.spdx.json
          fail-build: true
          severity-cutoff: critical
          only-fixed: true

      - name: Stocker le SBOM comme artefact
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.spdx.json
```

Stocker le SBOM comme artefact de build permet de l'interroger lors d'une CVE publiée ultérieurement, sans avoir à reconstruire l'image.

### Interroger le SBOM pour qualification rapide

Quand une CVE est publiée, la première question à répondre est : "est-ce qu'on utilise ce composant ?" Avec un SBOM disponible :

```bash
# Vérifier la présence d'un composant spécifique
cat sbom.spdx.json | jq '.packages[] | select(.name == "openssl") | {name, versionInfo}'

# Vue tabulaire rapide
syft my-app:latest -o table | grep -i ffmpeg
```

Sans SBOM, cette vérification devient une fouille manuelle des Dockerfiles, lock files, et dépendances transitives, soit exactement ce qui fait dépasser le délai de qualification de 2 heures.

## Bonnes pratiques

### Séparer le circuit de patch d'urgence du change management standard

Le change management existe pour une bonne raison : éviter les régressions en production. Mais un processus de validation de 2 semaines est incompatible avec un SLO de 48h pour les CVEs critiques.

La solution n'est pas de supprimer le change management, mais de définir une **voie rapide** pour les patches de sécurité critiques :

- Revue par deux personnes au lieu de passage en CAB
- Tests automatisés obligatoires comme condition de merge
- Déploiement en canary (5-10% du trafic) pendant 2h avant rollout complet
- Rollback automatique si les métriques d'erreur augmentent

### Automatiser les tests de régression pour aller vite en sécurité

On ne peut pas aller vite si on a peur de casser quelque chose. La contrepartie d'un déploiement d'urgence, c'est une suite de tests automatisés solide :

- **Smoke tests** : vérification que l'application démarre et répond
- **Tests d'intégration** sur les chemins critiques (authentication, paiement, etc.)
- **Tests de contrat** pour les APIs exposées

Sans ces tests, chaque patch de sécurité est un risque opérationnel qui pousse les équipes à repousser les déploiements. C'est exactement l'inverse de ce qu'on cherche.

### Maintenir un inventaire de composants à jour (SBOM)

Un SBOM généré à chaque build et stocké avec l'artefact permet de répondre en quelques secondes à "est-ce qu'on utilise ce composant vulnérable ?". La section [Grype et Syft](#grype-et-syft--une-alternative-sbom-first) détaille comment mettre cette pratique en place concrètement.

## Conclusion

La pression exercée par les outils d'exploitation automatisés oblige à repenser fondamentalement les cycles de patch. 48h pour les CVEs critiques n'est pas un objectif ambitieux, c'est le minimum requis pour maintenir une posture de sécurité acceptable.

Ce qui rend cet objectif atteignable en pratique :

- Des SLOs formalisés qui créent une obligation, pas juste une recommandation
- Un pipeline de scanning continu qui détecte les CVEs en temps réel
- Une voie rapide de déploiement découplée du change management standard
- Des tests automatisés suffisants pour déployer vite sans prendre de risque inconsidéré
- Un SBOM à jour pour qualifier rapidement l'exposition

Le point le plus souvent négligé est le SBOM. Sans connaissance précise de ce qui tourne en production, toute la mécanique amont est ralentie par une phase de qualification manuelle qui consomme précisément le temps qu'on ne peut pas se permettre de perdre.

## Liens

- [NVD : National Vulnerability Database](https://nvd.nist.gov/)
- [Trivy : Scanner de vulnérabilités open source](https://github.com/aquasecurity/trivy)
- [Grype : Scanner de CVEs basé sur SBOM](https://github.com/anchore/grype)
- [Syft : Générateur de SBOM](https://github.com/anchore/syft)
- [Dependabot : GitHub docs](https://docs.github.com/en/code-security/dependabot)
- [SPDX : Standard SBOM](https://spdx.dev/)
- [OpenSSF Scorecard : Santé sécurité des projets open source](https://securityscorecards.dev/)

## Sources concernant la compromission Trivy

- [GitHub Advisory GHSA-69fq-xp46-6x23](https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23)
- [Aqua Security : investigation complète](https://www.aquasec.com/blog/trivy-supply-chain-attack-what-you-need-to-know/)
- [StepSecurity : 2e compromission](https://www.stepsecurity.io/blog/trivy-compromised-a-second-time---malicious-v0-69-4-release)
- [CrowdStrike : analyse technique](https://www.crowdstrike.com/en-us/blog/from-scanner-to-stealer-inside-the-trivy-action-supply-chain-compromise/)

## Acronymes

| Acronyme | Signification |
|---|---|
| API | Application Programming Interface |
| CAB | Change Advisory Board, comité d'approbation des changements |
| CI/CD | Continuous Integration / Continuous Delivery, intégration et déploiement continus |
| CVE | Common Vulnerabilities and Exposures, identifiant standardisé d'une vulnérabilité |
| CVSS | Common Vulnerability Scoring System, système de score de sévérité des CVEs |
| ECR | Elastic Container Registry, registre d'images container Amazon |
| IA | Intelligence Artificielle |
| NFS | Network File System, protocole de partage de fichiers réseau |
| NIST | National Institute of Standards and Technology, agence fédérale américaine des standards |
| NVD | National Vulnerability Database, base de données officielle des CVEs maintenue par le NIST |
| PR | Pull Request, demande de fusion de code dans un repository git |
| RCE | Remote Code Execution, exécution de code arbitraire à distance |
| SACK | Selective ACKnowledgement, mécanisme TCP d'accusé de réception sélectif |
| SBOM | Software Bill of Materials, inventaire des composants logiciels d'un artefact |
| SLO | Service Level Objective, objectif de niveau de service et engagement interne mesurable |
