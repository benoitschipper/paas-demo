# Live Demo — PaaS CRD Examples

These three examples are designed for a **live booth / stand demo**.  
Walk visitors through them in order — each one builds on the previous and introduces new concepts.

---

## The core idea

> **One YAML file → full team environment on OpenShift.**

A DevOps team writes a single `Paas` custom resource.  
The `opr-paas` operator reads it and automatically provisions:

| What | How |
|---|---|
| Namespaces | One per declared environment stage |
| RBAC & Groups | Bound to AD/LDAP queries or explicit users |
| Resource Quotas | Enforced cluster-wide |
| ArgoCD instance | Scoped to the team, bootstrapped from their Git repo |
| Grafana instance | Team-scoped observability |
| SSO (Keycloak) | Application authentication out of the box |

---

## Demo progression

### 🟢 [01 — Simple](01-simple/simple-paas.yaml)

**Audience hook:** *"This is the absolute minimum. One file, one command — and you have a namespace, a group, quotas, and a private ArgoCD."*

| Field | Value |
|---|---|
| Namespaces | `dev` |
| Groups | `admins` (1 user) |
| Capabilities | ArgoCD |
| Quota | 8 Gi memory · 2 CPU · 10 Gi storage |

**Talk track:**
- Point at `spec.quota` — the team declares what they need, the platform enforces it.
- Point at `spec.capabilities.argocd.custom_fields.git_url` — the operator bootstraps ArgoCD straight from the team's own repo.
- Point at `spec.groups` and `spec.namespaces` — one group, one namespace, done.

---

### 🟡 [02 — Intermediate](02-intermediate/intermediate-paas.yaml)

**Audience hook:** *"Real teams have multiple environments and multiple roles. Still just one file."*

| Field | Value |
|---|---|
| Namespaces | `dev`, `tst`, `prd`, `tekton` |
| Groups | `admins` · `developers` · `viewers` |
| Capabilities | ArgoCD + **Grafana** |
| Quota | 16 Gi memory · 4 CPU · 25 Gi storage |

**Talk track:**
- Show how `spec.namespaces` maps groups to environments — developers can't accidentally touch `prd` unless you put them there.
- Show `grafana: {}` — two characters and the team gets a full Grafana instance with cluster metrics pre-wired.
- Show the `viewers` group — product owners get read-only access without any cluster admin involvement.

---

### 🔴 [03 — Advanced](03-advanced/advanced-paas.yaml)

**Audience hook:** *"Enterprise scale: LDAP groups, SSO, acceptance gates, audit access — still one file."*

| Field | Value |
|---|---|
| Namespaces | `dev`, `tst`, `acc`, `prd`, `tekton`, `monitoring` |
| Groups | `platform-admins` · `developers` · `release-managers` · `auditors` |
| Capabilities | ArgoCD + Grafana + **SSO** |
| Quota | 64 Gi memory · 16 CPU · 200 Gi storage |

**Talk track:**
- Point at `spec.groups[platform-admins].query` — this is an LDAP/AD CN query. No user lists to maintain; group membership is managed in your corporate directory.
- Show `release-managers` only appearing in `acc` and `prd` namespaces — fine-grained promotion gates without any extra tooling.
- Show `auditors` in `prd` and `monitoring` with `view` role — external compliance access, zero cluster admin effort.
- Show `sso: {}` — Keycloak deployed and integrated, teams get SSO for their apps immediately.

---

## How to apply any of these

```bash
# Apply directly to the cluster
kubectl apply -f paas-live-demo-files/01-simple/simple-paas.yaml

# Or let ArgoCD sync it — add an Application pointing at the folder
```

The operator picks it up within seconds and starts provisioning.

---

## What visitors might ask

**"Who manages the operator?"**  
The Platform Team — once. DevOps teams never touch it.

**"What if a team needs more quota?"**  
They update their `Paas` CR and open a PR. The Platform Team reviews and merges. GitOps all the way.

**"Can teams add their own namespaces?"**  
Yes — they add a line under `spec.namespaces` in their own file. No ticket, no waiting.

**"What about secrets?"**  
Sealed Secrets is part of the platform stack. Teams encrypt secrets client-side and commit the sealed version to Git.

**"Is this OpenShift-specific?"**  
The operator targets OpenShift (it uses OpenShift Groups and Quotas), but the concept is portable.
