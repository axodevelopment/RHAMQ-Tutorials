# leader-follower

Broker's competing for lock on backend db

# create project
oc new-project oracle-db-amq

# permissions

oracle needs permissions

oc create serviceaccount oracle-sa -n oracle-db-amq
oc adm policy add-scc-to-user anyuid -z oracle-sa -n oracle-db-amq
oc adm policy add-scc-to-user anyuid -z oracle-sa

confirm oracle has permissions
oc adm policy who-can use scc anyuid

# db account setup
oc create secret generic oracle-db-pass \
  --from-literal=password=secret \
  -n oracle-db-amq

# deploy db

oc apply -f db-statefulset.yaml

oc get pods -w

DO NOT PROCECEED UNTIL DB IS FULLY INIT
db still needs to be init (this takes time, in oracle case, up to several minutes)
oc logs oracle-db-0 --follow


wait for logs