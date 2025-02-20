# Outside of pod
# Get both certs from the chain
oc get secret amqp-acceptor-ptls -n amq-broker -o jsonpath='{.data.tls\.crt}' | base64 -d > full-chain.crt

oc get secret amqp-acceptor-ptls -n amq-broker -o jsonpath='{.data.tls\.crt}' | base64 -d > server.crt

do steps of starting debug pod
oc get pods
oc cp server.crt amq-broker/image-debug-q22hb:/tmp/amq-test/ssl/

# creating debug pod

oc debug -n amq-broker --image=registry.access.redhat.com/ubi8/ubi-minimal

microdnf install -y nmap-ncat bind-utils java-17-openjdk wget tar gzip

nslookup broker-amqp-acceptor-0-svc-rte-amq-broker.apps.axolab.axodevelopment.dev
nslookup broker-amqp-acceptor-0-svc.amq-broker.svc

nc -zv broker-amqp-acceptor-0-svc-rte-amq-broker.apps.axolab.axodevelopment.dev 443
nc -zv broker-amqp-acceptor-0-svc.amq-broker.svc 5672

wget https://archive.apache.org/dist/activemq/activemq-artemis/2.31.2/apache-artemis-2.31.2-bin.tar.gz
tar xzf apache-artemis-2.31.2-bin.tar.gz

@@@ Make sure you are following the flow, likely need two terminals but you are the secret we need is on your machine, we need to copy that to this pod THEN we can add it to the keystore

keytool -import -alias broker -file /tmp/amq-test/ssl/server.crt -keystore /tmp/amq-test/ssl/truststore.jks -storepass securepass -noprompt

/tmp/amq-test/apache-artemis-2.31.2/bin/artemis producer \
  --url "amqps://broker-amqp-acceptor-0-svc-rte-amq-broker.apps.axolab.axodevelopment.dev:443?sslEnabled=true&trustStorePath=/tmp/amq-test/ssl/truststore.jks&trustStorePassword=securepass&verifyHost=false" \
  --message "Test SSL Message via Route" \
  --destination myQueue