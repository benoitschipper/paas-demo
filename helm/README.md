# paas-capabilities

Capabilities die binnen paas-operator op het cluster beschikbaar zijn, zoals ArgoCD en Grafana. Zie https://belastingdienst.github.io/opr-paas/latest/administrators-guide/capabilities/ voor meer info over capabilities.

In het cluster worden capabilities beschikbaar gemaakt via een ApplicationSet in [openshift-gitops/apps/paas-capabilities]

## Building and packaging helm capabilities

Login to GHCR with a username and a valid Personal Access Token (classic).

```bash
$ helm registry login ghcr.io -u [username]
Password:
Login Succeeded
```

Package the helm chart.

```bash
$ cd helm
$ helm package capability-argocd/
Successfully packaged chart and saved it to: helm/capability-argocd-0.1.0.tgz
```

Push the package to the registry:

```bash
$ helm push capability-argocd-0.1.0.tgz oci://ghcr.io/benoitschipper/paas-demo
Pushed: ghcr.io/benoitschipper/paas-demo/capability-argocd:0.1.0
Digest: sha256:9dbaa24d4fc1a18c2acc1acea1cac9c5fe6827aa09f155da37a2d34109176f5c
```
