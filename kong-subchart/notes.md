oc create ns kong
oc create secret generic kong-enterprise-license --from-file=license=./license.json -n kong
openssl req -new -x509 -nodes -newkey ec:<(openssl ecparam -name secp384r1) \
  -keyout ./cluster.key -out ./cluster.crt \
  -days 1095 -subj "/CN=kong_clustering"
oc create secret tls kong-cluster-cert --cert=./cluster.crt --key=./cluster.key -n kong
oc adm policy add-scc-to-group anyuid system:serviceaccounts:kong-cp-kong
oc new-app -n kong --template=postgresql-ephemeral --param=POSTGRESQL_USER=kong --param=POSTGRESQL_PASSWORD=kong123 --param=POSTGRESQL_DATABASE=kong
helm install kong -n kong kong/kong -f cp-values.yaml

oc get secret -n openshift-gitops redhat-kong-gitops-cluster -ojsonpath='{.data.admin\.password}' | base64 -d


---

## There are issues with migration job but overall resources are getting initiated correctly.

oc apply -f -<<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kong-cp
  namespace: openshift-gitops
spec:
  destination:
    namespace: kong
    server: https://kubernetes.default.svc
  project: default
  source:
    path: kong-subchart
    repoURL: https://github.com/mpaulgreen/kong-patterns.git
    targetRevision: main
EOF

oc apply -f -<<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gitops-kong-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
- kind: ServiceAccount
  name: redhat-kong-gitops-argocd-application-controller
  namespace: openshift-gitops
EOF

oc apply -f -<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kong-cp-kong
  namespace: kong
EOF





- Final Result
//// Kustomize with helm subcharts

do not have values.yaml

helm dependency build && helm template --release-name redhat-kong . > all.yaml && kustomize build

/// but in our case it will be too much of a patches needs to be applied. values.yaml with cp and dp configuration is a better and quicker option