
# working dir
/Users/michaelwilson/truststore/full-chain-t1

# Root CA pk
openssl genrsa -out rootca.key.pem 4096

# Root CA cert pem
openssl req -x509 -new -nodes -key rootca.key.pem -sha256 -days 3650 \
  -subj "/CN=RootCA" \
  -out rootca.cert.pem

# Int CA pk
openssl genrsa -out intermediate.key.pem 4096

# csr int
openssl req -new -key intermediate.key.pem -sha256 -subj "/CN=IntermediateCA" \
  -out intermediate.csr.pem

# create the int pem
openssl x509 -req -in intermediate.csr.pem -CA rootca.cert.pem -CAkey rootca.key.pem \
  -CAcreateserial -out intermediate.cert.pem -days 1825 -sha256

# create server pk
openssl genrsa -out server.key.pem 2048

# csr for srv
openssl req -new -key server.key.pem -sha256 -subj "/CN=server.test.redhat.com" \
  -out server.csr.pem

# srver pem
openssl x509 -req -in server.csr.pem -CA intermediate.cert.pem -CAkey intermediate.key.pem \
  -CAcreateserial -out server.test.redhat.com.cert.pem -days 825 -sha256

#######
BEGIN TESTING
#######


We want to only use the server pem / key to create a bad broker keystore

cp server.test.redhat.com.cert.pem server.cert.pem

openssl pkcs12 -export -inkey server.key.pem -in server.cert.pem -out server-only.pkcs12 -name server -passout pass:securepass

we have the pkcs we want the jks
keytool -importkeystore -srckeystore server-only.pkcs12 -srcstoretype pkcs12 \
  -destkeystore keystore.jks -deststoretype JKS \
  -deststorepass securepass -srcstorepass securepass

cp keystore.jks client.jks

oc create secret generic amqp-acceptor-secret \
--from-file=broker.ks=keystore.jks \
--from-file=client.ts=client.jks \
--from-literal=keyStorePassword=securepass \
--from-literal=trustStorePassword=securepass




