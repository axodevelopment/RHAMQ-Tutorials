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

you should see
```
#########################
DATABASE IS READY TO USE!
#########################
```

# prep certs

get certs


If you have the certs from the basic-broker tls still installed the same cert here is used 

-- or --

wget -O server-keystore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/server-keystore.jks
wget -O server-ca-truststore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/server-ca-truststore.jks
wget -O client-ca-truststore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/client-ca-truststore.jks

create a secret for them

oc create secret generic ext-amqp-acceptor-ssl-secret \
--from-file=broker.ks=server-keystore.jks \
--from-file=client.ts=client-ca-truststore.jks \
--from-literal=keyStorePassword=securepass \
--from-literal=trustStorePassword=securepass

pull the client-truststore and if you need to repull the server-keystore

oc get secret ext-amqp-acceptor-ssl-secret -n basic-broker -o jsonpath='{.data.client\.ts}' | base64 -d > cs-client-truststore.jks

oc get secret ext-amqp-acceptor-ssl-secret -n basic-broker -o jsonpath='{.data.broker\.ks}' | base64 -d > cs-server-keystore.jks

create the combined-trustore and put both the client and server ks and trusstore.

keytool -importkeystore \
  -srckeystore cs-server-keystore.jks \
  -destkeystore combined-truststore.jks \
  -srcstorepass securepass \
  -deststorepass securepass

keytool -importkeystore \
  -srckeystore cs-client-truststore.jks \
  -destkeystore combined-truststore.jks \
  -srcstorepass securepass \
  -deststorepass securepass

check if everything is correct
keytool -list -v -keystore combined-truststore.jks -storepass securepass

# peer brokers

oc new-project peer-broker-a
oc new-project peer-broker-b

oc apply -f peer-broker-a-v1.yaml
oc apply -f peer-broker-b-v1.yaml


in logs of POD 0/1 => "AMQ221034: Waiting indefinitely to obtain primary lock"
in logs of POD 1/1 => "AMQ221007: Server is now active"


# add a dr

oc new-project peer-broker-dr
