1. Install Community Grafana Operator via UI
2. Install RedHat OpenShift GitOps Operator
 * Patch OpenShift GitOps argocd.yaml 
       ```
        oc patch argocd openshift-gitops \
        -n openshift-gitops \
        --type=merge \
        -p '{"spec": {"kustomizeBuildOptions": "--load-restrictor LoadRestrictionsNone --enable-helm"}}'
        ```
 * Restart all pods `for i in $(oc get pods -n openshift-gitops | awk '{print $1}' | grep -v NAME);do oc delete pod $i -n openshift-gitops;done` or use `oc rollout deployment etc..`
3. Install Red Hat OpenShift Pipelines
4. Install cert-manager Operator
5. Quick Install opr-paas operator
    - https://github.com/belastingdienst/opr-paas
    - kubectl apply -f https://github.com/belastingdienst/opr-paas/releases/latest/download/install.yaml
    - kubectl apply -f https://raw.githubusercontent.com/belastingdienst/opr-paas/refs/heads/main/examples/resources/_v1alpha2_paasconfig.yaml