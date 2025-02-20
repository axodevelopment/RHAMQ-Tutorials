what we want to achieve

create certs or use your own

create secret from cert

deploy broker

test external passthrough

test internal tls through debug pod


# Pre req
oc new-project common-broker

# Create Certs + Build truststore

Create a directory for your tls cert work

wget -O server-keystore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/server-keystore.jks

wget -O server-ca-truststore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/server-ca-truststore.jks

wget -O client-ca-truststore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/client-ca-truststore.jks

Note that these are infact JKS ks types
keytool -list -v -keystore server-keystore.jks -storepass securepass

note in brocker.yaml

    acceptors
      - name: amqp-acceptor
        sslEnabled: true
        sslSecret: amqp-acceptor-secret  <--

oc create secret generic amqp-acceptor-secret \
--from-file=broker.ks=server-keystore.jks \
--from-file=client.ts=client-ca-truststore.jks \
--from-literal=keyStorePassword=securepass \
--from-literal=trustStorePassword=securepass

For a local truststore that we will use to test with

oc get secret amqp-acceptor-secret -n common-broker -o jsonpath='{.data.client\.ts}' | base64 -d > cs-client-truststore.jks

oc get secret amqp-acceptor-secret -n common-broker -o jsonpath='{.data.broker\.ks}' | base64 -d > cs-server-keystore.jks

Now we can create the new combined-truststore.jks

keytool -importkeystore \
  -srckeystore cs-server-keystore.jks \
  -destkeystore combined-truststore.jks \
  -srcstorepass securepass \
  -deststorepass securepass \
  -deststoretype JKS

keytool -importkeystore \
  -srckeystore cs-client-truststore.jks \
  -destkeystore combined-truststore.jks \
  -srcstorepass securepass \
  -deststorepass securepass \
  -deststoretype JKS

check if everything is correct
keytool -list -v -keystore combined-truststore.jks -storepass securepass

# Deploy Broker

oc apply -f broker.yaml

You should see the cert you added to the broker being presented in your browser if you navigate to the route in 
oc get route

if you used the broker.yaml
broker-amqp-acceptor-0-svc-rte-common-broker.apps...