---
title: "Connecting to an Azure database without direct access using socat and kubectl port-forward"
date: 2026-05-27T10:00:00+02:00
draft: false
tags:
  - azure
  - kubernetes
  - postgresql
  - networking
  - devops
  - security
cover:
  image: cover.jpeg
  alt: "kubectl port-forward tunnel to Azure PostgreSQL"
---

In professional environments, Azure databases (PostgreSQL, MySQL, SQL Server…) are often exposed exclusively via a **Private Endpoint**: they are only reachable within the Azure private network, with no public IP. The result: from your development workstation, it is impossible to connect directly using a client like DBeaver or `psql`.

However, the AKS cluster (Azure Kubernetes Service) running in the same VNet does have access. This guide explains how to leverage that fact to create a secure tunnel to the database, without modifying any network rules or opening a single public port.

## Solution architecture

![Solution architecture](aks-postgres-architecture.svg)

The flow is as follows:

1. A temporary pod runs `socat`, which listens on a port and forwards connections to the Azure database.
2. `kubectl port-forward` creates a tunnel between your local port and that pod.
3. Your SQL client connects to `localhost:5432` as if the database were local.

## Prerequisites

- `kubectl` configured and pointing to the correct AKS cluster
- Access to the target namespace (rights to create a Pod and use `port-forward`)
- The hostname of your Azure database (e.g. `patoune-server.postgres.database.azure.com`)

## Step 1 - Deploy the socat pod

Create a `socat-pod.yaml` manifest:

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

> Replace `patoune-server.postgres.database.azure.com` with your database's FQDN, visible in the Azure portal → your PostgreSQL server → **Overview**.

Apply the manifest:

```bash
kubectl apply -f socat-pod.yaml
```

Wait for the pod to be `Running`:

```bash
kubectl get pod socat-db -w
```

## Step 2 - Open the tunnel with kubectl port-forward

```bash
kubectl port-forward pod/socat-db 5432:5432
```

The terminal stays occupied as long as the tunnel is active. Keep it open and work in another terminal.

You should see:

```
Forwarding from 127.0.0.1:5432 -> 5432
Forwarding from [::1]:5432 -> 5432
```

## Step 3 - Connect with your SQL client

### With psql

```bash
psql -h localhost -p 5432 -U myuser -d mydb
```

### With DBeaver

In the connection configuration:

| Field | Value |
|---|---|
| Host | `localhost` |
| Port | `5432` |
| Database | `mydb` |
| Username | `myuser` |
| Password | _(your Azure password)_ |
| SSL | Required (enforced by Azure) |

> For PostgreSQL on Azure, SSL mode is mandatory. In DBeaver, enable **SSL → Require** and disable certificate verification if you don't have the Azure CA locally (or add it for a clean setup).

## Handling a port already in use

If port `5432` is already taken on your machine (e.g. a local PostgreSQL installation), use an alternative port:

```bash
kubectl port-forward pod/socat-db 15432:5432
```

Then connect on `localhost:15432`.

## Cleanup

Once the session is over, stop the tunnel (`Ctrl+C`) and delete the pod:

```bash
kubectl delete pod socat-db
```

Do not leave this pod running indefinitely: it opens a network gateway to your database from within the cluster.

## 📝 Security considerations

This technique is handy but must remain **reserved for development or one-off debugging use cases**:

- The `socat` pod has no authentication of its own — anyone who can run `kubectl port-forward` on that pod can reach the database.
- Restrict RBAC rights: only the relevant namespace should allow pod creation and `port-forward` usage.
- Remember to delete the pod as soon as the session is over.
- For recurring access, prefer a solution like **Azure Bastion** or a **point-to-site VPN** to the Azure VNet.

## ⚠️ Protecting your cluster against this bypass

This technique illustrates a real attack vector if used with malicious intent, even though in most cases it is simply a debugging aid. In practice, any user with rights to create a pod and use `kubectl port-forward` can reach any network resource accessible from the cluster, including production databases. Here is how to harden an AKS cluster to reduce this exposure surface.

### 1. Restrict `port-forward` and `exec` via RBAC

The Kubernetes verbs `pods/portforward` and `pods/exec` are often granted by default in overly broad roles. A developer role in production should not need them:

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
  # pods/portforward and pods/exec are intentionally absent
```

Audit existing roles with:

```bash
kubectl get rolebindings,clusterrolebindings -A -o json \
  | jq '.items[] | select(.rules[]?.resources[]? | contains("portforward"))'
```

### 2. Apply egress NetworkPolicies on sensitive namespaces

By default, Kubernetes imposes no restriction on outbound traffic from pods. A default-deny policy forces you to explicitly allow each permitted flow:

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

### 3. Block unapproved images with an admission controller

A `socat` pod based on `alpine/socat:latest` should not be allowed to start in production. A tool like **Kyverno** can restrict images to those from a private registry:

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
        message: "Only images from myregistry.azurecr.io are allowed."
        pattern:
          spec:
            containers:
              - image: "myregistry.azurecr.io/*"
```

### 4. Enable audit logs and Microsoft Defender for Containers

- **API Server audit logs**: enable them in AKS (Portal → Diagnostic settings) to trace every `port-forward` or `exec` call.
- **Microsoft Defender for Containers**: detects suspicious behaviour such as launching an unusual network process inside a pod or installing binaries at runtime.

### 5. Segment by namespace with the principle of least privilege

- One namespace per environment (`dev` / `staging` / `production`) with separate `RoleBinding` definitions.
- Only the development namespace should allow ad hoc pod creation and `port-forward` usage.
- `ResourceQuota` and `LimitRange` objects further limit the creation of unexpected resources.

### 6. The `kubectl exec` command

The `kubectl exec` command is very powerful and can make certain debugging operations easier; however, it is also a gateway to executing other commands that could undermine the security of your clusters.

> For this reason, I also recommend blocking this command when necessary.

---

## Conclusion

`socat` + `kubectl port-forward` form a remarkably effective duo for temporarily accessing a private Azure network resource from your local machine. Setup takes less than five minutes and requires no changes to network rules or Azure NSGs. It is the ideal tool for urgent debugging or accessing a pre-production database without compromising the project's security posture. That said, you should ensure that this kind of technique is only possible in development environments, as it can also leave a significant security door open.

---

## References

### Kubernetes / CNCF

- [Use Port Forwarding to Access Applications in a Cluster](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) — official `kubectl port-forward` documentation
- [kubectl port-forward — CLI reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/)
- [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) — configuring and restricting Kubernetes permissions
- [Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/) — RBAC best practices
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) — controlling inter-pod and egress traffic
- [Kubernetes API Server Bypass Risks](https://kubernetes.io/docs/concepts/security/api-server-bypass-risks/) — risks related to direct node access

### Azure / AKS

- [Secure Pod Traffic with Network Policies in AKS](https://learn.microsoft.com/en-us/azure/aks/use-network-policies) — enabling and configuring NetworkPolicies on AKS
- [Best Practices for Network Policies in AKS](https://learn.microsoft.com/en-us/azure/aks/network-policy-best-practices) — Microsoft recommendations for network segmentation
- [Best Practices for Network Resources in AKS](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-network) — networking best practices for AKS operators
- [Networking Concepts in AKS](https://learn.microsoft.com/en-us/azure/aks/concepts-network) — overview of the AKS network architecture
