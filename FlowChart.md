# PaaS Operator — How It Works

This document explains how the **Platform Team** and **DevOps Teams** interact with the [opr-paas](https://github.com/belastingdienst/opr-paas) operator, and how a DevOps team goes from a single YAML file to a fully running application on OpenShift.

> 📂 **Further reading**
> - [`demo/`](demo/README.md) — Platform Team repo: operator install, PaasConfig, platform ArgoCD bootstrap, capability Helm charts
> - [`paas-demo-app/`](paas-demo-app/README.md) — DevOps Team repo: application code, ArgoCD bootstrap, deploy manifests, Grafana dashboard

---

## Responsibilities at a glance

| | Platform Team | DevOps Team |
|---|---|---|
| **Repo** | `demo/` | `demo/apps/example-paas/` + `paas-demo-app/` |
| **What they own** | Cluster bootstrap, operators, PaasConfig, platform ArgoCD | Their `Paas` CR, their application code, their ArgoCD apps |
| **Done once?** | Yes — bootstrap is a one-time setup | No — teams iterate on their app continuously |

---

## Flow

```mermaid
flowchart TD
    %% ACTORS
    PT(["🏗️ Platform Team"])
    DT(["👨‍💻 DevOps Team"])

    %% ① PLATFORM BOOTSTRAP
    subgraph PLATFORM ["① Platform Team — Bootstrap (done once)  [repo: demo]"]
        direction TB
        P1["Install operators on cluster<br/>(opr-paas, ArgoCD, Kyverno,<br/>Sealed Secrets, Grafana Operator)"]
        P2["Apply PaasConfig<br/>demo/apps/paas-operator/paas-config.yaml<br/>Defines capabilities: argocd, grafana, sso"]
        P3["Bootstrap platform ArgoCD<br/>demo/bootstrap/<br/>Applications for operator + capabilities"]
        P4["Platform ArgoCD<br/>(openshift-gitops)<br/>Syncs all platform apps continuously"]
        P5["ApplicationSets<br/>demo/apps/paas-capabilities/<br/>appset_paas-argocd / grafana / sso"]

        P1 --> P2 --> P3 --> P4 --> P5
    end

    %% ② DEVOPS PAAS CR
    subgraph DEVOPS_CONFIG ["② DevOps Team — Request Environment  [repo: demo/apps/example-paas]"]
        direction TB
        D1["DevOps Team writes Paas CR<br/>demo/apps/example-paas/example-paas.yaml<br/>• quotas  • namespaces  • groups/RBAC<br/>• capabilities: argocd → git_url: paas-demo-app"]
        D2["Platform Team syncs this path<br/>via ArgoCD Application:<br/>demo/bootstrap/app_example-paas.yaml"]
        D1 --> D2
    end

    %% ③ PAAS OPERATOR
    subgraph OPERATOR ["③ opr-paas Operator — Provisions Team Environment"]
        direction TB
        O1["Reads Paas CR"]
        O2["Creates Namespaces<br/>example-paas-dev/tst/acc/prd/tekton"]
        O3["Applies RBAC & Groups<br/>admins / developers / viewers"]
        O4["Enforces Resource Quotas<br/>32Gi memory · 8 CPU · 50Gi storage"]
        O5["Triggers Kyverno<br/>Auto-generates NetworkPolicies<br/>per namespace"]
        O6["Provisions ArgoCD Capability<br/>(via ApplicationSet)<br/>Deploys team-scoped ArgoCD instance<br/>in example-paas-argocd namespace"]
        O7["Provisions Grafana Capability<br/>Deploys team Grafana instance"]
        O8["Provisions SSO Capability<br/>Keycloak for team auth"]

        O1 --> O2
        O1 --> O3
        O1 --> O4
        O1 --> O5
        O1 --> O6
        O1 --> O7
        O1 --> O8
    end

    %% ④ TEAM ARGOCD
    subgraph TEAM_ARGOCD ["④ Team ArgoCD — Deployed by PaaS Operator  [repo: paas-demo-app]"]
        direction TB
        A1["Team ArgoCD instance<br/>example-paas-argocd<br/>Bootstrapped from:<br/>git_url: paas-demo-app<br/>git_path: argocd/bootstrap"]
        A2["ArgoCD App: paas-demo-build<br/>→ argocd/build/<br/>Tekton Pipeline resources"]
        A3["ArgoCD App: paas-demo-deploy-dev<br/>→ argocd/deploy/dev/"]
        A4["ArgoCD App: paas-demo-deploy-tst<br/>→ argocd/deploy/tst/"]
        A5["ArgoCD App: paas-demo-deploy-acc<br/>→ argocd/deploy/acc/"]
        A6["ArgoCD App: paas-demo-deploy-prd<br/>→ argocd/deploy/prd/"]
        A7["ArgoCD App: paas-demo-observe<br/>→ argocd/deploy/observe/<br/>GrafanaDashboard CR"]

        A1 --> A2
        A1 --> A3
        A1 --> A4
        A1 --> A5
        A1 --> A6
        A1 --> A7
    end

    %% ⑤ APP DELIVERY
    subgraph APP ["⑤ DevOps Team — App Delivery  [repo: paas-demo-app]"]
        direction TB
        G1["👨‍💻 Code change pushed to Git"]
        G2["GitHub Actions or Tekton CI<br/>Builds & pushes container image<br/>to ghcr.io"]
        G3["Team ArgoCD detects Git change<br/>syncs manifests to cluster"]
        G4["🖥️ App running in<br/>example-paas-dev/tst/acc/prd"]
        G5["📊 Grafana Dashboard<br/>in example-paas-grafana"]

        G1 --> G2 --> G3 --> G4
        G3 --> G5
    end

    %% CONNECTIONS
    PT --> PLATFORM
    DT --> DEVOPS_CONFIG
    PLATFORM -->|"Platform ArgoCD applies Paas CR to cluster"| OPERATOR
    DEVOPS_CONFIG -->|"Paas CR applied to cluster"| OPERATOR
    OPERATOR -->|"Bootstraps team ArgoCD<br/>pointing at paas-demo-app"| TEAM_ARGOCD
    DT -->|"Owns & maintains"| APP
    TEAM_ARGOCD -->|"Syncs app manifests"| APP

    %% STYLES
    style PLATFORM      fill:#1e3a5f,color:#fff,stroke:#4a90d9
    style DEVOPS_CONFIG fill:#2d4a1e,color:#fff,stroke:#6abf40
    style OPERATOR      fill:#4a1e3a,color:#fff,stroke:#d940a0
    style TEAM_ARGOCD   fill:#3a2a1e,color:#fff,stroke:#d98040
    style APP           fill:#1e3a3a,color:#fff,stroke:#40d9d9
```

---

## Step-by-step explanation

### ① Platform Team — Bootstrap *(done once)*
The Platform Team sets up the cluster foundation using the [`demo/`](demo/) repo:
- Installs all required operators (opr-paas, ArgoCD, Kyverno, Sealed Secrets, Grafana Operator)
- Applies a `PaasConfig` that defines which capabilities are available (ArgoCD, Grafana, SSO) and how they are templated
- Bootstraps the platform-level ArgoCD (`openshift-gitops`) which continuously syncs everything in `demo/`
- In this case, apply the bootstrap applications by hand (for demo purposes)

### ② DevOps Team — Request an Environment
The DevOps Team's only interaction with the `demo` repo is a **single YAML file**:
[`demo/apps/example-paas/example-paas.yaml`](demo/apps/example-paas/example-paas.yaml)

This `Paas` CR declares:
- Which namespaces they need (`dev`, `tst`, `acc`, `prd`, `tekton`)
- Who gets access and with what role (admins, developers, viewers)
- Which capabilities to enable, and for ArgoCD: which Git repo to bootstrap from

The Platform Team's ArgoCD syncs this file to the cluster via [`demo/bootstrap/app_example-paas.yaml`](demo/bootstrap/app_example-paas.yaml).

### ③ opr-paas Operator — Automatic Provisioning
Once the `Paas` CR lands on the cluster, the operator takes over and automatically provisions:
- All declared namespaces
- RBAC and group bindings
- Resource quotas
- Kyverno-generated NetworkPolicies
- Capability instances: a team-scoped ArgoCD, Grafana, and SSO

### ④ Team ArgoCD — Bootstrapped by PaaS
The ArgoCD instance the operator created reads the `git_url` and `git_path` from the `Paas` CR and bootstraps itself from the DevOps team's own repo ([`paas-demo-app/argocd/bootstrap/`](paas-demo-app/argocd/bootstrap/)). From this point on, the team manages their own ArgoCD applications entirely independently.

### ⑤ DevOps Team — Ship Their Application
The team works entirely in [`paas-demo-app/`](paas-demo-app/):
- Push code → GitHub Actions or Tekton builds and pushes the container image
- Their ArgoCD detects the Git change and syncs manifests into the namespaces PaaS created
- Grafana dashboards are deployed automatically via the `observe` ArgoCD application

> **One YAML file. One team. Full environment — namespaces, RBAC, ArgoCD, Grafana, SSO — all automatic.**
