# Kubernetes Security (Stage 5)

## Topics Covered
26. Service Accounts
27. RBAC in Kubernetes
28. Pod Security (Pod Security Admission)
29. Secrets Management Best Practices
30. Network Policies (Deep Dive)
31. AKS Workload Identity

---

## Security Model Overview

Kubernetes security is multi-layered. No single control is sufficient.

```mermaid
flowchart TD
    ID[Identity: ServiceAccounts / Workload Identity] --> AUTHN[Authentication]
    AUTHN --> AUTHZ[Authorization: RBAC]
    AUTHZ --> ADMISSION[Admission: Pod Security / Policy]
    ADMISSION --> NET[Network: NetworkPolicy]
    NET --> DATA[Data: Secrets + encryption + external stores]
    DATA --> AUDIT[Audit/observability]
```

Defense-in-depth means combining all layers consistently.

---

## 26) Service Accounts

### What a ServiceAccount is
A ServiceAccount (SA) is an identity for pods inside the cluster. Pods use SA tokens to authenticate to the Kubernetes API and in-cluster controllers.

By default, each namespace has a `default` ServiceAccount. Using it in production is a common anti-pattern.

---

### ServiceAccount token projection
Modern Kubernetes uses **bound projected tokens** instead of long-lived auto-generated secrets.

Benefits:
- short-lived tokens
- audience scoping
- rotation support
- reduced token leakage risk

```mermaid
sequenceDiagram
    participant Pod
    participant Kubelet
    participant API as kube-apiserver

    Pod->>Kubelet: Start with ServiceAccount reference
    Kubelet->>API: Request bound token for pod/SA/audience
    API-->>Kubelet: Short-lived JWT
    Kubelet-->>Pod: Mount token at projected volume path
    Pod->>API: Call API with Bearer token
```

---

### ServiceAccount example

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: app
automountServiceAccountToken: false
```

Set `automountServiceAccountToken: false` by default and enable only where needed.

Pod using SA with projected token:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api
  namespace: app
spec:
  serviceAccountName: app-sa
  automountServiceAccountToken: false
  containers:
    - name: api
      image: nginx:1.25
      volumeMounts:
        - name: k8s-token
          mountPath: /var/run/secrets/tokens
          readOnly: true
  volumes:
    - name: k8s-token
      projected:
        sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
              audience: "https://kubernetes.default.svc"
```

---

### Expert Section — Service Accounts

- Never rely on `default` SA for application workloads.
- Disable auto-mount globally and opt-in only for pods needing API access.
- Scope token audience strictly; avoid generic tokens valid for unintended consumers.
- For controllers/operators, separate SA per component to minimize blast radius.
- Monitor token file access patterns in hardened environments.

---

## 27) RBAC in Kubernetes

### Core RBAC objects

| Object | Scope | Purpose |
|---|---|---|
| `Role` | Namespace | set of allowed verbs on namespaced resources |
| `ClusterRole` | Cluster | rules for cluster-scoped resources or reusable namespaced rules |
| `RoleBinding` | Namespace | binds Role/ClusterRole to subject in namespace |
| `ClusterRoleBinding` | Cluster | binds ClusterRole across cluster |

Subjects can be `User`, `Group`, or `ServiceAccount`.

---

### Authorization flow

```mermaid
flowchart LR
    REQ[API request] --> AUTHN[Authenticate caller]
    AUTHN --> AUTHZ[RBAC authorizer evaluates rules]
    AUTHZ -->|match verb+resource+scope| ALLOW[Allow]
    AUTHZ -->|no matching rule| DENY[Deny]
```

RBAC is additive allow; no explicit deny in native RBAC rules.

---

### Least-privilege example

Role to allow read-only ConfigMaps in one namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader
  namespace: app
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
```

RoleBinding for `app-sa`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: config-reader-binding
  namespace: app
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: app
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: config-reader
```

---

### ClusterRole + ClusterRoleBinding example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
  - kind: ServiceAccount
    name: metrics-agent
    namespace: observability
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: node-reader
```

---

### RBAC verification commands

```bash
kubectl auth can-i get pods --as=system:serviceaccount:app:app-sa -n app
kubectl auth can-i --list --as=system:serviceaccount:app:app-sa -n app
```

---

### Expert Section — RBAC

- Prefer namespace-scoped `Role` + `RoleBinding` first; escalate only when required.
- Avoid wildcard rules (`*` verbs/resources/apiGroups`) in production.
- Separate human admin RBAC from workload RBAC; do not share identities.
- Periodically review stale bindings for deleted service accounts.
- Use audit logs and policy checks to detect privilege creep.

---

## 28) Pod Security (Pod Security Admission)

### What PSA is
Pod Security Admission (PSA) enforces pod-level security standards via namespace labels.

Profiles:
- `privileged`
- `baseline`
- `restricted`

Modes:
- `enforce`
- `audit`
- `warn`

---

### PSA workflow

```mermaid
flowchart TD
    POD[Pod create request] --> API[kube-apiserver]
    API --> PSA[Pod Security Admission]
    PSA --> NSLBL[Read namespace security labels]
    NSLBL --> CHECK[Evaluate pod spec against profile]
    CHECK -->|passes| ALLOW[Admit pod]
    CHECK -->|fails enforce| DENY[Reject pod]
    CHECK -->|fails audit/warn| LOG[Allow + warn/audit]
```

---

### Enforcing restricted profile

```bash
kubectl label ns app \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted --overwrite
```

---

### Expert Section — PSA

- Roll out with `warn` and `audit` first.
- Use namespace segmentation by trust level.
- Pair PSA with policy engines (Kyverno/Gatekeeper).
- Validate third-party charts for PSA compatibility.
- Track and expire exceptions.

---

## 29) Secrets Management Best Practices

### Secret handling workflow

```mermaid
flowchart LR
    KV[External secret store: Key Vault] --> ESO[External Secrets / CSI Driver]
    ESO --> K8S[Projected secret in pod or synced k8s Secret]
    K8S --> APP[Application runtime]
    APP --> AUDIT[Access logs + alerts]
```

### Expert Section — Secrets Management

- Treat secret access as privileged with audit controls.
- Align secret TTL with workload reload capabilities.
- Implement break-glass access with time bounds.
- Block plaintext secret patterns via admission.
- Prefer identity-based auth over stored credentials.

---

## 30) Network Policies (Deep Dive)

### Deep-dive policy workflow

```mermaid
flowchart TD
    TRAFFIC[Pod A -> Pod B traffic] --> SELECT{Is Pod B selected by ingress policy?}
    SELECT -->|No| ALLOW1[Allowed by default behavior]
    SELECT -->|Yes| CHECKING{Any ingress rule matches source+port?}
    CHECKING -->|Yes| ALLOW2[Ingress allowed]
    CHECKING -->|No| DENY1[Ingress denied]

    ALLOW2 --> EGRESSSEL{Is Pod A selected by egress policy?}
    EGRESSSEL -->|No| ALLOW3[Egress allowed]
    EGRESSSEL -->|Yes| EGRESSCHK{Any egress rule matches dest+port?}
    EGRESSCHK -->|Yes| ALLOW4[Egress allowed]
    EGRESSCHK -->|No| DENY2[Egress denied]
```

### Expert Section — NetworkPolicy Deep Dive

- Policy behavior is additive.
- Model both ingress and egress.
- Standardize labels early.
- Validate with runtime tests.
- Confirm CNI supports required features.

---

## 31) AKS Workload Identity

### Trust model workflow

```mermaid
sequenceDiagram
    participant Pod
    participant SA as K8s ServiceAccount Token
    participant AAD as Microsoft Entra ID
    participant KV as Azure Key Vault

    Pod->>SA: Read projected token
    Pod->>AAD: Token exchange (federated)
    AAD->>AAD: Validate issuer/subject/audience
    AAD-->>Pod: Azure access token
    Pod->>KV: Call Key Vault with bearer token
    KV-->>Pod: Secret response
```

### Expert Section — AKS Workload Identity

- Bind one SA per workload boundary.
- Federated subject must match exactly.
- Keep Azure RBAC scope narrow.
- Test negative paths.
- Prefer workload identity over SP secrets.

---

## End-to-End Security Workflow

```mermaid
flowchart TD
    SA[ServiceAccount identity] --> RBAC[RBAC authorization]
    RBAC --> PSA[Pod Security Admission checks]
    PSA --> NP[NetworkPolicy enforcement]
    NP --> SEC[Secrets delivery model]
    SEC --> WI[Workload identity for external cloud access]
    WI --> AZ[Azure resources with least-privilege RBAC]
```
