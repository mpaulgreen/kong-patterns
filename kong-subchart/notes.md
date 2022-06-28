# Scratch pad

```
oc create ns kong
oc create secret generic kong-enterprise-license --from-file=license=./license.json -n kong
openssl req -new -x509 -nodes -newkey ec:<(openssl ecparam -name secp384r1) \
  -keyout ./cluster.key -out ./cluster.crt \
  -days 1095 -subj "/CN=kong_clustering"
oc create secret tls kong-cluster-cert --cert=./cluster.crt --key=./cluster.key -n kong
oc adm policy add-scc-to-group anyuid system:serviceaccounts:kong



oc adm policy add-scc-to-group anyuid system:serviceaccounts:kong
oc new-app -n kong --template=postgresql-ephemeral --param=POSTGRESQL_USER=kong --param=POSTGRESQL_PASSWORD=kong123 --param=POSTGRESQL_DATABASE=kong


oc new-app \
    -e POSTGRESQL_USER=kong \
    -e POSTGRESQL_PASSWORD=kong123 \
    -e POSTGRESQL_DATABASE=kong \
    postgresql:9.5
helm install kong -n kong kong/kong -f cp-values.yaml

oc get secret -n openshift-gitops redhat-kong-gitops-cluster -ojsonpath='{.data.admin\.password}' | base64 -d

lhtX13BnGU6wpQgZxskjqvmaIC05bOdJ
```

## There are issues with migration job but overall resources are getting initiated correctly.
```
oc apply -f -<<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kong
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
```
```
oc apply -f -<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kong
  namespace: kong
EOF
```


- expose svc admin

```bash
oc expose svc kong-kong-admin -n kong
oc expose svc kong-kong-manager -n kong
```

- create secure routes

```bash
oc create route passthrough kong-kong-manager-tls --port=kong-manager-tls --service=kong-kong-manager -n kong
oc create route passthrough kong-kong-admin-tls --port=kong-admin-tls --service=kong-kong-admin -n kong
```

- update the ADMIN URI in the deployment

```bash
oc patch deploy -n kong kong-kong -p "{\"spec\": { \"template\" : { \"spec\" : {\"containers\":[{\"name\":\"proxy\",\"env\": [{ \"name\" : \"KONG_ADMIN_API_URI\", \"value\": \"$(oc get route -n kong kong-kong-admin -ojsonpath='{.spec.host}')\" }]}]}}}}"
```

- save the cluster/clustertelemetry endpoints for later

```bash
export CLUSTER_URL=$(oc -n kong get svc kong-kong-cluster -ojson | jq -r '.status.loadBalancer.ingress[].hostname')
export CLUSTER_TELEMETRY_URL=$(oc -n kong get svc kong-kong-clustertelemetry -ojson | jq -r '.status.loadBalancer.ingress[].hostname')
```

- check the management UI is working at:

```bash
oc get route -n kong kong-kong-manager --template='{{.spec.host}}'
```

- check the admin endpoint is available

```bash
http `oc get route -n kong kong-kong-admin --template='{{.spec.host}}'` | jq .version
"2.8.1.1-enterprise-edition"
```




## Final Inference
//// Kustomize with helm subcharts

do not have values.yaml

helm dependency build && helm template --release-name redhat-kong . > all.yaml && kustomize build

/// but in our case it will be too much patches needed to be applied. values.yaml with cp and dp configuration is a better and quicker option


curl -k -H "Content-Type: application/yaml" -X PUT --data-binary @ns.yaml http://127.0.0.1:8001/api/v1/namespaces/openshift-gitops/finalize