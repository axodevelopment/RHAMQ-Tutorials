# pre-req

oc new-project basic-broker

### basic-broker v1

oc apply -f basic-broker-v1.yaml

oc get svc

How the svc name gets defined
name = <broker.name>-<acceptor.name>-0-svc

# how do we test
## port forward to localhost
Since we are testing amqp we will use the amqp svc

```
oc port-forward service/basic-broker-amqp-0-svc 5672:5672
```

configure your app to localhost to point to this svc

```
quarkus.qpid-jms.url=amqp://localhost:5672
quarkus.log.category."org.apache.qpid.jms".level=DEBUG
```

## check console

find the 'wconsj' svc
basic-broker-wconsj-0-svc

oc apply -f console-route.yaml

view console to see your messages going through

oc get route
should see something like
broker-console                  broker-console-basic-broker.apps.axolab.axodevelopment.dev                         basic-broker-wconsj-0-svc   8161       edge/Redirect   None

How the url name is framed.
broker-console-basic-broker.apps.axolab.axodevelopment.dev
<route-name>-<route-ns>.apps.<cluster-name>.<domain>

login with admin / admin

Artemis => amq-broker -> addresses -> myQueue -> queues -> anycast -> myQueue
Operations
countMessages()
Execute

if you have sent messages into the queue you should see messages there.

## check pvc

you could also mount the pvc directly and explore the internals

oc get pvc -n basic-broker
basic-broker-basic-broker-ss-0

while ti should be in /opt/basic-broker/data you can check by
oc exec -it basic-broker-ss-0 -- df -h
or check the volumeMounts on the broker yaml itself:
oc get pod basic-broker-ss-0 -o yaml | grep -A 5 volumeMounts

in my case you can see the directory of data at
oc exec -it basic-broker-ss-0 -- ls -la /opt/basic-broker/data
or get a bash
oc exec -it basic-broker-ss-0 -- /bin/bash

under journal you can see activemq-data-1.amq you can cat that file to see your data in there.

### Basic Auth 
Now using basic auth

on the broker kind: ActiveMQArtemis

spec.adminPassword: admin
spec.adminUser: admin


=== on the client === 
quarkus.qpid-jms.url=amqp://localhost:5672
quarkus.qpid-jms.username=admin
quarkus.qpid-jms.password=admin

quarkus.log.category."org.apache.qpid.jms".level=DEBUG


### TLS Auth

onto basic-broker-3.yaml

need to create a combined ts so that the broker can auth the client and the client can auth the broker

get the apache test ks (or use yours)...

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

oc get secret ext-amqp-acceptor-ssl-secret -n basic-broker -o jsonpath='{.data.client\.ts}' | base64 -d > client-truststore.jks

oc get secret ext-amqp-acceptor-ssl-secret -n basic-broker -o jsonpath='{.data.broker\.ks}' | base64 -d > server-keystore.jks

create the combined-trustore and put both the client and server ks and trusstore.

keytool -importkeystore \
  -srckeystore server-keystore.jks \
  -destkeystore combined-truststore.jks \
  -srcstorepass securepass \
  -deststorepass securepass

keytool -importkeystore \
  -srckeystore client-truststore.jks \
  -destkeystore combined-truststore.jks \
  -srcstorepass securepass \
  -deststorepass securepass

Now we can apply the broker
oc apply -f basic-broker-3.yaml

oc apply -f service.yaml

by default the route should be:
basic-broker-amqp-0-svc-rte
<broker.name>-<acceptor.name>-0-svc-rte

fqdn
<broker.name>-<acceptor.name>-0-svc-rte-<broker.namespace>.apps.<cluster.name>.<domainname>

but since service.yaml has a route specified we will use that route.

the route should be.
rn=<route.name>-<route.namespace>.apps.<cluster.name>.<domainname>

here is my jms url:
quarkus.qpid-jms.url=amqps://amqp-acceptor-svc-rte-oracle-jdbc-shared-store.apps.axolab.axodevelopment.dev:443?transport.trustStoreLocation=/Users/michaelwilson/truststore/amq-oracle-jdbc-shared-store/combined-truststore.jks&transport.trustStorePassword=securepass&transport.verifyHost=false

ampqs://<rn>:443?transport.trustStoreLocation=/<trusts-storepath>/combined-truststore.jks&transport.trustStorePassword=securepass&transport.verifyHost=false

### can confirm data in the pvc
through the debug pod or the console you can see data flow through