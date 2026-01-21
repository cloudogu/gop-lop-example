# gop-lop-example

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
VERSION='0.14.0'

# Start k3d 
bash <(curl -s \
  https://raw.githubusercontent.com/cloudogu/gitops-playground/$VERSION/scripts/init-cluster.sh)

# For velero
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v6.2.1/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v6.2.1/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v6.2.1/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml

k create ns $NAMESPACE
k create ns argocd

kubectl create secret generic component-operator-helm-registry \
  --from-literal=config.json='{"auths": {"'registry.cloudogu.com'": {"auth": "'$(echo -n "${DOGU_REGISTRY_USERNAME}:${DOGU_REGISTRY_PASSWORD}" | base64)'"}}}' \
  --namespace="${NAMESPACE}"

kubectl create configmap component-operator-helm-repository \
  --from-literal=endpoint="registry.cloudogu.com" \
  --from-literal=schema="oci" \
  --from-literal=plainHttp="false" \
  --from-literal=insecureTls="false"  \
  --namespace="${NAMESPACE}"
  
kubectl create secret docker-registry ces-container-registries \
  --docker-server="registry.cloudogu.com" \
  --docker-username="${DOGU_REGISTRY_USERNAME}" \
  --docker-password="${DOGU_REGISTRY_PASSWORD}" \
  --docker-email="${DOGU_REGISTRY_USERNAME}" \
  --namespace="${NAMESPACE}"
  
kubectl create secret generic k8s-dogu-operator-dogu-registry \
  --from-literal=endpoint="https://dogu.cloudogu.com/api/v2/dogus" \
  --from-literal=urlschema="default" \
  --from-literal=username="${DOGU_REGISTRY_USERNAME}" \
  --from-literal=password="${DOGU_REGISTRY_PASSWORD}" \
  --namespace="${NAMESPACE}"

cat << EOF | kubectl apply -f -
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
helm upgrade gop -i oci://ghcr.io/cloudogu/gop-helm --version 0.4.0 -n gop  --create-namespace --kube-context k3d-gitops-playground  --values - <<EOF
image:
  tag: ${VERSION}
config:
  features:
    argocd:
      active: true
  content:
    repos:
      - url: https://github.com/cloudogu/gop-lop-example
        path: repos
        templating: true
        type: FOLDER_BASED
        overwriteMode: UPGRADE
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

(login with `admin`/`admin`)

Another 5-10 Minutes later, you can access LOP like so

```bash
xdg-open https://$(kubectl get svc ces-loadbalancer  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')/scm
```

You can log in with user `admin` and this password:
```bash
kubectl get secret ldap-config -o go-template='{{index .data "config.yaml" | base64decode}}'
```

## See also
* https://github.com/cloudogu/ecosystem-core/blob/develop/docs/operations/argoCD_en.md