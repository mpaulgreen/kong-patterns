oc create secret generic kong-enterprise-license --from-file=license=./license.json -n kong
openssl req -new -x509 -nodes -newkey ec:<(openssl ecparam -name secp384r1) \
  -keyout ./cluster.key -out ./cluster.crt \
  -days 1095 -subj "/CN=kong_clustering"
oc create secret tls kong-cluster-cert --cert=./cluster.crt --key=./cluster.key -n kong
oc new-app -n kong --template=postgresql-ephemeral --param=POSTGRESQL_USER=kong --param=POSTGRESQL_PASSWORD=kong123 --param=POSTGRESQL_DATABASE=kong
helm install kong -n kong kong/kong -f cp-values.yaml