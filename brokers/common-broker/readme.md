what we want to achieve

create certs or use your own

create secret from cert

deploy broker

test external passthrough

test internal tls through debug pod

---

# Pre req
oc new-project common-broker

---

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

---

# Deploy Broker

oc apply -f broker.yaml

You should see the cert you added to the broker being presented in your browser if you navigate to the route in 
oc get route

if you used the broker.yaml
broker-amqp-acceptor-0-svc-rte-common-broker.apps...

---

# Test internal TLS

Ideally you will need to have two terminals now known as term1 and term2

@@@ WARNING / NOTE @@@
If you have an RHACS product that is enforcing a baseline this pod may get killed early since we are going to be dyn building a pod...

@@@ WARNING / NOTE @@@
Depending on firewall setup from OCP to internet this may also not work but it gives you an idea of how to build a container for debugging

# on your machine (term1)

oc get route

Since we already have cs-server-keystore.jks lets get the crt file from that.

keytool -exportcert \
  -keystore cs-server-keystore.jks \
  -alias server \
  -storepass securepass \
  -file server.crt \
  -rfc

# on another terminal (term2)

oc debug -n common-broker --image=registry.access.redhat.com/ubi8/ubi-minimal

microdnf install -y nmap-ncat bind-utils java-17-openjdk wget tar gzip

mkdir -p /tmp/amq-test/ssl

wget https://archive.apache.org/dist/activemq/activemq-artemis/2.31.2/apache-artemis-2.31.2-bin.tar.gz

tar xzf apache-artemis-2.31.2-bin.tar.gz

# on your machine (term1)

oc get pods | grep debug

oc cp server.crt common-broker/<debug-pod-name>:/tmp/amq-test/ssl/

oc cp server.crt common-broker/image-debug-2ls9n:/tmp/amq-test/ssl/

# on another terminal (term2)

keytool -import -v -alias ca -file /tmp/amq-test/ssl/server.crt \
  -keystore /tmp/amq-test/ssl/truststore.p12 \
  -storetype PKCS12 \
  -storepass securepass \
  -noprompt

/apache-artemis-2.31.2/bin/artemis producer \
  --protocol amqp \
  --url "amqps://broker-amqp-acceptor-0-svc.common-broker.svc:5672?transport.trustStoreType=PKCS12&transport.trustStoreLocation=/tmp/amq-test/ssl/truststore.p12&transport.trustStorePassword=securepass&transport.verifyHost=false&sslEnabled=true" \
  --message "Test SSL Message via Service" \
  --destination myQueue

NOTE:
need route if testing passthrough

  /apache-artemis-2.31.2/bin/artemis producer \
  --protocol amqp \
  --url "amqps://<route>:443?transport.trustStoreType=PKCS12&transport.trustStoreLocation=/tmp/amq-test/ssl/truststore.p12&transport.trustStorePassword=securepass&transport.verifyHost=false&sslEnabled=true" \
  --message "Test SSL Message via Route" \
  --destination myQueue
