# (term1)
# run the tool image

oc debug -n common-broker --image=axodevelopment/artemistools:v1.0.0.0

# get the jks
oc get secret amqp-acceptor-secret -n common-broker -o jsonpath='{.data.broker\.ks}' | base64 -d > broker.ks

# get pem crt out of jks
keytool -exportcert -rfc \
  -keystore broker.ks \
  -alias server \
  -file server.pem \
  -storepass securepass

# get the debug pod name
oc get pods | grep debug

# copy the pem over
oc cp server.pem common-broker/image-debug-qdsnh:/tmp/amq-test/ssl/

# in the debug pod (term2)
#
# confirm we have the pem
cat /tmp/amq-test/ssl/server.pem

# create th pkcs12 store
keytool -import -v -alias ca -file /tmp/amq-test/ssl/server.pem \
  -keystore /tmp/amq-test/ssl/truststore.p12 \
  -storetype PKCS12 \
  -storepass securepass \
  -noprompt

bin/artemis producer \
  --protocol amqp \
  --url "amqps://broker-amqp-acceptor-0-svc.common-broker.svc:5672?transport.trustStoreType=PKCS12&transport.trustStoreLocation=/tmp/amq-test/ssl/truststore.p12&transport.trustStorePassword=securepass&transport.verifyHost=false&sslEnabled=true" \
  --message "Test SSL Message via Service" \
  --destination myQueue
