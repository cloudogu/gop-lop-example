# gop-deploy-lop-example

Deploy a GitOps-managed LowOps Platform (LOP) on a plain Kubernetes cluster.

This is an example that combines the usage of [GOP](https://github.com/cloudogu/gitops-playground) with [LOP](https://github.com/cloudogu/k8s-ecosystem/).  
GOP is used to boostrap LOP and manage it via GitOps via Argo CD.  
This example lays the foundation to centrally manage multiple LOP instances/tenants 🚀.

## Running locally

Note: For now this only runs on Linux and maybe also on MacOS.

Reason: In this setup using k3d LOP is only accessible via the container IP address of the k3d container.
With Docker Desktop (on Mac and Windows) these are not accessible from the host.
It should work on Mac when using Orbstack, though.


If you are running Ubuntu you might have to do the following to avoid crashes of the LDAP pod:  
`sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.slapd`

```bash
  DOGU_REGISTRY_USERNAME='xzy' # or robot$...
  DOGU_REGISTRY_PASSWORD=''
NAMESPACE=ecosystem
GOP_VERSION='c076ee11' # 0.15.0 preview
GOP_CHART_VERSION='0.4.0'
CONTEXT=k3d-gitops-playground

# Start k3d 
bash <(curl -s \
  https://raw.githubusercontent.com/cloudogu/gitops-playground/$GOP_VERSION/scripts/init-cluster.sh) --bind-ports=443:443

# For velero
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v6.2.1/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml --context "$CONTEXT" 
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v6.2.1/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml --context "$CONTEXT" 
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v6.2.1/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml --context "$CONTEXT" 

k create ns $NAMESPACE --context "$CONTEXT" 
k create ns argocd --context "$CONTEXT" 

kubectl create secret generic component-operator-helm-registry --context "$CONTEXT" \
  --from-literal=config.json='{"auths": {"'registry.cloudogu.com'": {"auth": "'$(echo -n "${DOGU_REGISTRY_USERNAME}:${DOGU_REGISTRY_PASSWORD}" | base64)'"}}}' \
  --namespace="${NAMESPACE}"

kubectl create configmap component-operator-helm-repository --context "$CONTEXT" \
  --from-literal=endpoint="registry.cloudogu.com" \
  --from-literal=schema="oci" \
  --from-literal=plainHttp="false" \
  --from-literal=insecureTls="false"  \
  --namespace="${NAMESPACE}"
  
kubectl create secret docker-registry ces-container-registries --context "$CONTEXT" \
  --docker-server="registry.cloudogu.com" \
  --docker-username="${DOGU_REGISTRY_USERNAME}" \
  --docker-password="${DOGU_REGISTRY_PASSWORD}" \
  --docker-email="${DOGU_REGISTRY_USERNAME}" \
  --namespace="${NAMESPACE}"
  
kubectl create secret generic k8s-dogu-operator-dogu-registry --context "$CONTEXT"  \
  --from-literal=endpoint="https://dogu.cloudogu.com/api/v2/dogus" \
  --from-literal=urlschema="default" \
  --from-literal=username="${DOGU_REGISTRY_USERNAME}" \
  --from-literal=password="${DOGU_REGISTRY_PASSWORD}" \
  --namespace="${NAMESPACE}"

cat << EOF | kubectl apply --context "$CONTEXT" -f -
apiVersion: v1
kind: Secret
metadata:
  name: cloudogu-oci-registry-k8s
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: "helm"
  name: "Cloudogu-Registry"
  url: "registry.cloudogu.com/k8s"
  enableOCI: "true"
  username: "$DOGU_REGISTRY_USERNAME"
  password: "$DOGU_REGISTRY_PASSWORD"
EOF

# Deploy GOP and make it deploy LOP
# Don't deploy GOP ingress controller for now, so it leaves IP to be taken by LOP ingress controller
helm upgrade gop -i oci://ghcr.io/cloudogu/gop-helm --version $GOP_CHART_VERSION -n gop  --create-namespace --kube-context "$CONTEXT"  --values - <<EOF
image:
  tag: ${GOP_VERSION}
config:
  application:
    baseUrl: https://localhost
  scm:
    scmManager:
      helm:
        version: "3.11.3" # TODO remove once updated in GOP
        values:
          ingress:
            ingressClassName: k8s-ecosystem-ces-service
  features:
    argocd:
      active: true
      values:
        argo-cd:
          server:
            ingress:
              ingressClassName: "k8s-ecosystem-ces-service"
  content:
    repos:
      - url: https://github.com/cloudogu/gop-deploy-lop-example
        path: repos
        templating: true
        type: FOLDER_BASED
        overwriteMode: UPGRADE
#    variables:
#      fqdn: lop.example.com
EOF
```

Follow the instructions of the helm chart to wait for the installation to finish.

After deployment is finished (1-2 Mins), you can access the management cluster via

```bash
k port-forward -n scm-manager svc/scmm 8080:80
k port-forward -n argocd svc/argocd-server 8081:80
```
* SCM-Manager (git server) UI at http://localhost:8080
* the Argo CD UI at http://localhost:8081

(login with `admin`/`admin`) - you **should** [change it](#Change-default-passwords)!

Another 5-10 Minutes later, you can access LOP like so

```bash
xdg-open https://$(kubectl get svc ces-loadbalancer  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')/scm
```

You can log in with user `admin` and this password:
```bash
kubectl get secret ldap-config -o go-template='{{index .data "config.yaml" | base64decode}}'
```


## Change default passwords

Argo CD:
```shell
NEW_PASSWORD="YourNewPassword123!"

BCRYPT_HASH=$(htpasswd -nB -b admin "$NEW_PASSWORD" | cut -d ":" -f 2)

kubectl -n argocd patch secret argocd-secret \
  -p "{\"stringData\": {\"admin.password\": \"$BCRYPT_HASH\", \"admin.passwordMtime\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}}"
  
kubectl patch secret  argocd-repo-creds-scm -n argocd -p "{\"data\":{\"password\":\"$(echo -n "$NEW_PASSWORD" | base64)\"}}"

# SCM-Manager 
curl --fail -u 'admin:admin' 'http://localhost:8080/scm/api/v2/me/password' \
  -X PUT \
  -H 'Content-Type: application/vnd.scmm-passwordChange+json;v=2' \
  --data-raw '{"oldPassword":"admin","newPassword":"'"$NEW_PASSWORD"'"}'
```

## Upgrading GOP using GOP

Remove `overwriteMode: UPGRADE` or blueprint will be overwritten

Problem:
* `UPGRADE` - would overwrite blueprint
* But we need `repos/argocd/cluster-resources/apps/argocd/projects/cluster-resources.yaml` -> `sourceRepos: *` or additional source repo `registry.cloudogu.com/k8s`

http://localhost:8080/scm/repo/argocd/cluster-resources/code/sources/main/apps/argocd/applications/gop.yaml/

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gop
  namespace: argocd
spec:
  project: argocd
  sources:
    - repoURL: ghcr.io/cloudogu
      chart: gop-helm
      targetRevision: 0.4.0
      helm:
        valuesObject:
          # GOP config used to install gop
          # kubectl get cm -n gop -l  app.kubernetes.io/instance=gop  -o jsonpath='{.items[0].data.config\.yaml}'
          #image:
          #  tag: ...
  destination:
    server: https://kubernetes.default.svc
    namespace: gop
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

## See also
* https://github.com/cloudogu/ecosystem-core/blob/develop/docs/operations/argoCD_en.md