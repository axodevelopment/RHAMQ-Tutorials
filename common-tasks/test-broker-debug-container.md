# on your machine (term1)

oc get route

oc get secret amqp-acceptor-ptls -n amq-broker -o jsonpath='{.data.tls\.crt}' | base64 -d > server.crt

# on another terminal (term2)

oc debug -n amq-broker --image=registry.access.redhat.com/ubi8/ubi-minimal

microdnf install -y nmap-ncat bind-utils java-17-openjdk wget tar gzip

mkdir -p /tmp/amq-test/ssl

wget https://archive.apache.org/dist/activemq/activemq-artemis/2.31.2/apache-artemis-2.31.2-bin.tar.gz

tar xzf apache-artemis-2.31.2-bin.tar.gz

# on your machine (term1)

oc get pods | grep debug

oc cp server.crt amq-broker/<debug-pod-name>:/tmp/amq-test/ssl/

oc cp server.crt amq-broker/image-debug-gjr5w:/tmp/amq-test/ssl/

# on another terminal (term2)

keytool -import -v -alias ca -file /tmp/amq-test/ssl/server.pem \
  -keystore /tmp/amq-test/ssl/truststore.p12 \
  -storetype PKCS12 \
  -storepass securepass \
  -noprompt

/apache-artemis-2.31.2/bin/artemis producer \
  --protocol amqp \
  --url "amqps://broker-amqp-acceptor-0-svc.amq-broker.svc:5672?transport.trustStoreType=PKCS12&transport.trustStoreLocation=/tmp/amq-test/ssl/truststore.p12&transport.trustStorePassword=securepass&transport.verifyHost=false&sslEnabled=true" \
  --message "Test SSL Message via Service" \
  --destination myQueue

NOTE:
need route if testing passthrough

  /apache-artemis-2.31.2/bin/artemis producer \
  --protocol amqp \
  --url "amqps://<route>:443?transport.trustStoreType=PKCS12&transport.trustStoreLocation=/tmp/amq-test/ssl/truststore.p12&transport.trustStorePassword=securepass&transport.verifyHost=false&sslEnabled=true" \
  --message "Test SSL Message via Route" \
  --destination myQueue