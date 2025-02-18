### Basic Auth 
Now using basic auth

on the broker kind: ActiveMQArtemis

deploymentPlan.requireLogin: true

spec.adminPassword: admin
spec.adminUser: admin

spec.acceptors[n].securityEnabled: true
spec.acceptors[n].saslMechanisms: PLAIN

oc apply -f basic-broker-v2.yaml

oc get pods -w

oc port-forward service/basic-broker-amqp-0-svc 5672:5672

=== on the client === 
quarkus.qpid-jms.url=amqp://localhost:5672
quarkus.qpid-jms.username=admin
quarkus.qpid-jms.password=admin

quarkus.log.category."org.apache.qpid.jms".level=DEBUG

oc apply -f basic-broker-security.yaml

=== on the client === 
quarkus.qpid-jms.username=testuser
quarkus.qpid-jms.password=testpassword

oc port-forward service/basic-broker-amqp-0-svc 5672:5672

quarkus.qpid-jms.username=admin
quarkus.qpid-jms.password=admin

Error sending message: AMQ229032: User: admin does not have permission='SEND' on address myQueue [condition = amqp:unauthorized-access]


next: ActiveMQArtemisSecurity CRD is deprecated

-- next 
  brokerProperties:
    - "securityEnabled=true"
    - "security.role.amq.send = *"
    - "security.role.amq.consume = *"
    - "security.role.amq.createDurableQueue = *"
    - "security.role.amq.createNonDurableQueue = *"
    - "security.role.amq.createAddress = *"
    - "jaas.config.name=PropertiesLogin"
    - "jaas.config.properties.user=file:/opt/security/artemis-users.properties"
    - "jaas.config.properties.role=file:/opt/security/artemis-roles.properties"


apiVersion: v1
kind: ConfigMap
metadata:
  name: artemis-security-config
  namespace: basic-broker
data:
  artemis-users.properties: |
    testuser=testpassword
    testuser.role=amq
  artemis-roles.properties: |
    amq=send,receive,createAddress,createDurableQueue,createNonDurableQueue



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