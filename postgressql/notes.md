```
oc apply -f -<<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: postgressql
  namespace: openshift-gitops
spec:
  destination:
    namespace: kong
    server: https://kubernetes.default.svc
  project: default
  source:
    path: postgressql
    repoURL: https://github.com/mpaulgreen/kong-patterns.git
    targetRevision: main
EOF
```