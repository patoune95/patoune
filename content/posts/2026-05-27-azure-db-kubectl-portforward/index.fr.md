---
title: "Se connecter à une base Azure sans accès direct via socat et kubectl port-forward"
date: 2026-05-27T10:00:00+02:00
draft: false
tags:
  - azure
  - kubernetes
  - postgresql
  - networking
  - devops
  - sécurité
cover:
  image: cover.jpeg
  alt: "Tunnel kubectl port-forward vers Azure PostgreSQL"
---

En environnement professionnel, les bases de données Azure (PostgreSQL, MySQL, SQL Server…) sont souvent exposées uniquement via un **Private Endpoint** : elles ne sont accessibles qu'au sein du réseau privé Azure, sans aucune IP publique. Résultat : depuis son poste de développement, il est impossible de s'y connecter directement avec un client comme DBeaver ou `psql`.

Pourtant, le cluster AKS (Azure Kubernetes Service) qui tourne dans le même VNet, lui, y a accès. Ce guide explique comment exploiter ce fait pour créer un tunnel sécurisé jusqu'à la base, sans modifier les règles réseau ni ouvrir le moindre port public.

## Architecture de la solution

![Architecture de la solution](aks-postgres-architecture.svg)

Le flux est le suivant :

1. Un pod temporaire fait tourner `socat`, qui écoute sur un port et relaie les connexions vers la base Azure.
2. `kubectl port-forward` crée un tunnel entre ton port local et ce pod.
3. Ton client SQL se connecte sur `localhost:5432` comme si la base était en local.

## Prérequis

- `kubectl` configuré et pointant sur le bon cluster AKS
- Accès au namespace cible (droits de créer un Pod et d'utiliser `port-forward`)
- Le nom d'hôte de ta base Azure (ex. : `patoune-server.postgres.database.azure.com`)

## Étape 1 - Déployer le pod socat

Crée un manifeste `socat-pod.yaml` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: socat-db
  namespace: default
  labels:
    app: socat-db
spec:
  containers:
    - name: socat
      image: alpine/socat:latest
      command:
        - socat
        - TCP-LISTEN:5432,fork,reuseaddr
        - TCP:patoune-server.postgres.database.azure.com:5432
      ports:
        - containerPort: 5432
  restartPolicy: Never
```

> Remplace `patoune-server.postgres.database.azure.com` par le FQDN de ta base, visible dans le portail Azure → ton serveur PostgreSQL → **Vue d'ensemble**.

Applique le manifeste :

```bash
kubectl apply -f socat-pod.yaml
```

Attends que le pod soit `Running` :

```bash
kubectl get pod socat-db -w
```

## Étape 2 - Ouvrir le tunnel avec kubectl port-forward

```bash
kubectl port-forward pod/socat-db 5432:5432
```

Le terminal reste occupé tant que le tunnel est actif. Garde-le ouvert et travaille dans un autre terminal.

Tu devrais voir :

```
Forwarding from 127.0.0.1:5432 -> 5432
Forwarding from [::1]:5432 -> 5432
```

## Étape 3 - Se connecter avec ton client SQL

### Avec psql

```bash
psql -h localhost -p 5432 -U monutilisateur -d mabase
```

### Avec DBeaver

Dans la configuration de la connexion :

| Champ | Valeur |
|---|---|
| Hôte | `localhost` |
| Port | `5432` |
| Base de données | `mabase` |
| Utilisateur | `monutilisateur` |
| Mot de passe | _(ton mot de passe Azure)_ |
| SSL | Requis (Azure l'impose) |

> Pour PostgreSQL sur Azure, le mode SSL est obligatoire. Dans DBeaver, active **SSL → Require** et désactive la vérification du certificat si tu n'as pas le CA Azure en local (ou ajoute-le pour un setup propre).

## Gestion du port déjà utilisé

Si le port `5432` est déjà pris sur ta machine (PostgreSQL local installé par exemple), utilise un port alternatif :

```bash
kubectl port-forward pod/socat-db 15432:5432
```

Et connecte-toi sur `localhost:15432`.

## Nettoyage

Une fois la session terminée, coupe le tunnel (`Ctrl+C`) et supprime le pod :

```bash
kubectl delete pod socat-db
```

Ne laisse pas ce pod tourner indéfiniment : il ouvre une porte réseau vers ta base depuis le cluster.

## 📝  Considérations de sécurité

Cette technique est pratique mais doit rester **réservée aux usages de développement ou de débogage ponctuels** :

- Le pod `socat` n'a aucune authentification propre — toute personne pouvant exécuter `kubectl port-forward` sur ce pod peut atteindre la base.
- Limite les droits RBAC : seul le namespace concerné devrait autoriser la création de pods et l'usage de `port-forward`.
- Pense à supprimer le pod dès que la session est terminée.
- Pour un accès récurrent, préfère une solution comme **Azure Bastion** ou un **VPN point-to-site** vers le VNet Azure.


## ⚠️  Protéger son cluster contre ce contournement

Cette technique illustre un vecteur d'attaque réel si utilisé à des fins malveillantes, même si dans la plupart des cas, c'est surtout une aide pour le debug. Dans les faits, tout utilisateur disposant des droits de créer un pod et d'utiliser `kubectl port-forward` peut atteindre n'importe quelle ressource réseau accessible depuis le cluster, y compris des bases de données de production. Voici comment durcir un cluster AKS pour réduire cette surface d'exposition.

### 1. Restreindre `port-forward` et `exec` via le RBAC

Les verbes Kubernetes `pods/portforward` et `pods/exec` sont souvent accordés par défaut dans des rôles trop larges. Un rôle de développeur en production ne devrait pas en avoir besoin :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-readonly
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  # pods/portforward et pods/exec sont volontairement absents
```

Audite les rôles existants avec :

```bash
kubectl get rolebindings,clusterrolebindings -A -o json \
  | jq '.items[] | select(.rules[]?.resources[]? | contains("portforward"))'
```

### 2. Appliquer des NetworkPolicies d'egress sur les namespaces sensibles

Par défaut, Kubernetes n'impose aucune restriction sur le trafic sortant des pods. Une politique de refus par défaut force à expliciter chaque flux autorisé :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
```

### 3. Bloquer les images non approuvées avec un contrôleur d'admission

Un pod `socat` basé sur `alpine/socat:latest` ne devrait pas pouvoir démarrer en production. Un outil comme **Kyverno** permet de n'autoriser que les images issues d'un registre privé :

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-registries
      match:
        resources:
          kinds: [Pod]
      validate:
        message: "Seules les images depuis myregistry.azurecr.io sont autorisées."
        pattern:
          spec:
            containers:
              - image: "myregistry.azurecr.io/*"
```

### 4. Activer les audit logs et Microsoft Defender for Containers

- **Audit logs API Server** : active-les dans AKS (Portail → Diagnostics settings) pour tracer chaque appel `port-forward` ou `exec`.
- **Microsoft Defender for Containers** : détecte les comportements suspects comme le lancement d'un processus réseau inhabituel dans un pod ou l'installation de binaires à l'exécution.

### 5. Segmenter par namespace avec le principe du moindre privilège

- Un namespace par environnement (`dev` / `staging` / `production`) avec des `RoleBinding` distincts.
- Seul le namespace de développement devrait autoriser la création de pods ad hoc et l'usage de `port-forward`.
- Des `ResourceQuota` et `LimitRange` limitent par ailleurs la création de ressources imprévues.

### 6. L'usage de la commande `kubectl exec`

L'usage de cette commande est très puissant et permet de faciliter certaines opérations pour le debug, cependant, celle-ci est aussi une porte ouverte à l'exécution d'autres commandes qui pourraient mettre à mal la sécurité de vos clusters. 

> C'est pour cette raison que je recommande aussi un blocage de cette commande lorsque nécessaire.

---

## Conclusion

`socat` + `kubectl port-forward` forment un duo redoutablement efficace pour accéder ponctuellement à une ressource réseau privée Azure depuis son poste local. La mise en place prend moins de cinq minutes et ne nécessite aucune modification des règles réseau ou des NSG Azure. C'est l'outil idéal pour déboguer en urgence ou accéder à une base en préprod sans compromettre la posture de sécurité du projet. Il faut cependant faire attention à ne laisser de possibilité d'opérer ce genre de manipulation qu'en environnement de développement car cette technique permet aussi de laisser une porte ouverte de sécurité importante.

---

## Références

### Kubernetes / CNCF

- [Use Port Forwarding to Access Applications in a Cluster](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) — documentation officielle `kubectl port-forward`
- [kubectl port-forward — référence CLI](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/)
- [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) — configurer et restreindre les droits Kubernetes
- [Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/) — bonnes pratiques RBAC
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) — contrôle du trafic inter-pods et egress
- [Kubernetes API Server Bypass Risks](https://kubernetes.io/docs/concepts/security/api-server-bypass-risks/) — risques liés aux accès directs au nœud

### Azure / AKS

- [Secure Pod Traffic with Network Policies in AKS](https://learn.microsoft.com/en-us/azure/aks/use-network-policies) — activer et configurer les NetworkPolicies sur AKS
- [Best Practices for Network Policies in AKS](https://learn.microsoft.com/en-us/azure/aks/network-policy-best-practices) — recommandations Microsoft pour la segmentation réseau
- [Best Practices for Network Resources in AKS](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-network) — bonnes pratiques réseau pour les opérateurs AKS
- [Networking Concepts in AKS](https://learn.microsoft.com/en-us/azure/aks/concepts-network) — vue d'ensemble de l'architecture réseau AKS
