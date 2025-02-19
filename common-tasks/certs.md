# the certs task

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