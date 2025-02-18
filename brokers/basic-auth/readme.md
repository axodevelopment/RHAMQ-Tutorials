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



