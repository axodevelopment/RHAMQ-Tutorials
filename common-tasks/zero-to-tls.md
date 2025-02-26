# Zero to TLS - Deploying a broker from no SSL to using SSL


## Table of Contents

   [Prerequisites](#prerequisites)
   [Summary](#summary)
   [Diagrams](#diagrams)

Steps to deploy federated brokers:

1. [Create OpenShift Projects](#1-create-openshift-projects-and-broker-deploy)  
2. [Without SSL](#2-testing-the-endpoint-from-within-the-ocp-cluster)  
3. [Test With SSL](#3-testing-with-ssl---self-signed-single-the-endpoint-from-within-the-ocp-cluster)  
4. [Test With SSL - Chain](#4-testing-with-ssl---full-chain-the-endpoint-from-within-the-ocp-cluster)   
5. [Cert commands](#5-certs)  

---

## Prerequisites

- OpenShift CLI (oc)  
- Access to an OpenShift cluster  
- Basic familiarity with OpenShift projects, ServiceAccounts, and Secrets  
- (Optional) Existing TLS certificates. Otherwise, you can generate or fetch sample certificates.  

---


## Summary

Here we will be deploying a broker without SSL testing that we can send messages to it, and then editing it to have ssl and ensuring we can send messages.

# Zero To TLS - AMQ Broker (OCP Deploy)

TLS Supported Protocols:
TLSv1,TLSv1.1,TLSv1.2 (defaults to JVM)

Default Truststore: JKS (Red Hat Recommends PKCS)

Cipher Suite:
Red Hat recommends for connectors to not specify a list of cipher suites

---

## Diagrams



---

## 1. Create OpenShift Projects and Broker deploy

Create the new projects (namespaces) on your OpenShift cluster:

```bash
oc new-project ssl-test-broker
```

note in brocker.yaml
```bash
  acceptors:
    - name: amqp-acceptor
      port: 5672
      protocols: all
      connectionsAllowed: 5
```
Since there is no mention of SSL in here there is no SSL required.  This will be a sanity check for connectivity...

Create the broker (double check namespaces or use your own if you would like):

```bash
oc apply -f ./refs/broker.v1.yaml
```

Wait for initialiation

```bash
oc get pods -w
```

Lets check the logs to make sure we have what we need:

```bash
oc logs broker-ss-0 --follow
```

You should see something like:

```bash
AMQ221007: Server is now active
```

Also check there are no errors displaying...

---

## 2. Testing (Without SSL) the endpoint from within the OCP cluster 

This is covered in more detail in test-tool.md  This will be the short route for now.

Ideally you'll open another terminal noted from here on out as term2

This will create a debug pod with that --image.  Note you will need to either build the docker image with your changes or pull it from my repo and push it to a repo your OCP environment has access to.

### TERM2
```bash
oc debug -n ssl-test-broker --image=axodevelopment/artemistools:v1.0.0.3
```

### TERM1
Now back to TERM1

### Get the service for the acceptor

By default the struture should be:

```bash
<broker.metadata.name>-<broker.spec.acceptors[].name>-<ord-cluster>-svc
```

If you are using the defaults

```bash
oc get svc
```

part-url = broker-amqp-acceptor-0-svc

This will help us build a url for the deub pod:

```bash
<protocol>://<part-url>.<namespace>.svc:<port>?transport.verifyHost=false&sslEnabled=false
```

--url "amqps://broker-amqp-acceptor-0-svc.ssl-test-broker.svc:5672?transport.verifyHost=false&sslEnabled=false"

### TERM2
Now back to TERM2

```bash
bin/artemis producer \
  --protocol amqp \
  --url "amqp://broker-amqp-acceptor-0-svc.ssl-test-broker.svc:5672?transport.verifyHost=false&sslEnabled=false" \
  --message "Test Message via Service" \
  --destination myQueue
```

In the debug pod you should be able to execute this command and send messages to the queue...

At this point after hitting enter 1000 messages will be produced to queue 'myQueue' and in the debug pod you should see something like...

```bash
Connection brokerURL = amqp://broker-amqp-acceptor-0-svc.ssl-test-broker.svc:5672?transport.verifyHost=false&sslEnabled=false
Producer myQueue, thread=0 Started to calculate elapsed time ...

Producer myQueue, thread=0 Produced: 1000 messages
Producer myQueue, thread=0 Elapsed time in second : 3 s
Producer myQueue, thread=0 Elapsed time in milli second : 3986 milli seconds
```



## 3. Testing (With SSL - Self Signed Single) the endpoint from within the OCP cluster 

While you don't need to be on the project "ssl-test-broker".  For sake of tutorial lets make sure you are back on the core project.

Set project to ssl-test-broker:

```bash
oc project ssl-test-broker
```

note in brocker.yaml

```bash
  acceptors:
    - name: amqp-acceptor
      port: 5672
      protocols: all
      sslEnabled: true <--
      sslSecret: amqp-acceptor-secret <--
      connectionsAllowed: 5
```

### Lets build some certs
I have noted the properties that will change with this to enable SSL.  Before we apply this broker however we want to create the certs.  To do that we will follow the steps in the Red Hat documentation for simplicity sake.

That documentation is located here:
https://docs.redhat.com/en/documentation/red_hat_amq_broker/7.12/html-single/deploying_amq_broker_on_openshift/index#proc-br-configuring-one-way-tls_broker-ocp

NOTE: My environment defaults to PKCS12 and not JKS

Generate a self-signed certificate for the broker key store.

```bash
keytool -genkey -alias broker -keyalg RSA -keystore broker.ks
```

Export the certificate from the broker key store, so that it can be shared with clients. Export the certificate in the Base64-encoded .pem format. For example:

```bash
keytool -export -alias broker -keystore broker.ks -file broker_cert.pem
```

On the client, create a client trust store that imports the broker certificate.

```bash
keytool -import -alias broker -keystore client.ts -file broker_cert.pem
```

Lets make sure these are PKCS format, most environments default that way

```bash
keytool -list -v -keystore broker.ks --storepass securepass
```


Now with these details we can create the secret in ocp:

```bash
oc create secret generic amqp-acceptor-secret \
--from-file=broker.ks=broker.ks \
--from-file=client.ts=client.ts \
--from-literal=keyStorePassword=securepass \
--from-literal=trustStorePassword=securepass
```

### Now we can deploy the broker changes

Create the broker (double check namespaces or use your own if you would like):

```bash
oc apply -f ./refs/broker.v2.yaml
```

Note: because we have modified the acceptor this does require the pod to be rebuilt / redeployed.

Wait for initialiation

```bash
oc get pods -w
```

Lets check the logs to make sure we have what we need:

```bash
oc logs broker-ss-0 --follow
```

You should see something like:

```bash
AMQ221007: Server is now active
```

Also check there are no errors displaying...

Now back to ### TERM1

If you didn't close the debug pod skip ahead to the url building part its time to send messages...

### Get the debug pod name
```bash
oc get pods | grep debug
```

Should look something like:
image-debug-<uniqu>


By default the struture should be:

```bash
<broker.metadata.name>-<broker.spec.acceptors[].name>-<ord-cluster>-svc
```

If you are using the defaults

```bash
oc get svc
```

part-url = broker-amqp-acceptor-0-svc

This will help us build a url for the deub pod:

```bash
<protocol>://<part-url>.<namespace>.svc:<port>?transport.verifyHost=false&sslEnabled=false"
"amqps://broker-amqp-acceptor-0-svc.ssl-test-broker.svc:5672?transport.trustStoreType=PKCS12&transport.trustStoreLocation=/tmp/amq-test/ssl/truststore.p12&transport.trustStorePassword=securepass&transport.verifyHost=false&sslEnabled=true
```

--url "amqps://broker-amqp-acceptor-0-svc.ssl-test-broker.svc:5672?transport.trustStoreType=PKCS12&transport.trustStoreLocation=/tmp/amq-test/ssl/truststore.p12&transport.trustStorePassword=securepass&transport.verifyHost=false&sslEnabled=true"


One more thing we want to copy the client.ts into debug pod
```bash
oc cp client.ts ssl-test-broker/image-debug-wwlvz:/tmp/amq-test/ssl/truststore.p12
```

### TERM2
Now back to TERM2

```bash
ls /tmp/amq-test/ssl
```

openssl s_client -connect broker-amqp-acceptor-0-svc.ssl-test-broker.svc:5672 -showcerts

```bash
bin/artemis producer \
  --protocol amqp \
  --url "amqps://broker-amqp-acceptor-0-svc.ssl-test-broker.svc:5672?transport.trustStoreType=PKCS12&transport.trustStoreLocation=/tmp/amq-test/ssl/truststore.p12&transport.trustStorePassword=securepass&transport.verifyHost=false&sslEnabled=true" \
  --message "Test Message via Service" \
  --destination myQueue
```


## 4. Testing (With SSL - Full Chain) the endpoint from within the OCP cluster 

First we should discover all certs we will need to make this work.  There will be a slight difference between the Client and Broker.  In this case, I am referring to 3 different certs which will make a full chain.

There is a Server.pem Server.key.

Note:
While I am using .pem it can be .cer just cat to see if it is in pem format or not.

Lets start with:

```bash
openssl x509 -in server.pem -text -noout

openssl x509 -in server.pem -text -noout | grep -A 1 "Issuer:" && openssl x509 -in server.pem -text -noout | grep -A 1 "Subject:"
```

Since these (in this case) will return as we might discover we will have 3 certs and one of them with a key.


S: server.cer (Server certificate)
Skey: server.key (Private key)
I: Intermediate.cer (Intermediate CA)
R: ROOTCA.cer (Root CA)

I'll use the alias here so I don't have to type as much (S, Skey, I, R).

### Prereq

First step will be to create the ca-chain.pem, order matters.  You can of course simplify and optimize these steps but I want to be clear here as to what goes where:

```bash
cat I R > ca-chain.pem
cat S I R > server-full-chain.pem
```

### Truststore

Then we can create the truststore from that

```bash
keytool -import -file ca-chain.pem -alias ca-chain -trustcacerts -keystore truststore.pkcs12 -storepass securepass -storetype PKCS12
```

### Broker keystore

Now lets build the keystore for the broker:

```bash
openssl pkcs12 -export -inkey Skey -in server-full-chain.pem -out server-keystore.pkcs12 -name broker -password pass:securepass
```


### Lets test our stores first

We should see the appropriate values here...

The keystore should show "PrivateKeyEntry" and a chain length of 3, while the truststore should show "trustedCertEntry".

```bash
keytool -list -v -keystore truststore.pkcs12 -storepass securepass -storetype PKCS12

keytool -list -v -keystore server-keystore.pkcs12 -storepass securepass -storetype PKCS12
```

### Deploment time

Lets make a secret:

Lets add a new acceptor
```bash
  acceptors:
    - name: amqp-acceptor
      port: 5672
      protocols: all
      sslEnabled: true
      sslSecret: amqp-acceptor-secret
      connectionsAllowed: 5
    - name: amqp-acceptor-fc <--
      port: 5672
      protocols: all
      sslEnabled: true
      sslSecret: amqp-acceptor-secret-fc
      connectionsAllowed: 5
```

Note we are creating a new secret with a new name

```bash
oc create secret generic amqp-acceptor-secret-fc \
  --from-file=broker.ks=server-keystore.pkcs12 \
  --from-file=client.ts=truststore.pkcs12 \
  --from-literal=keyStorePassword=securepass \
  --from-literal=trustStorePassword=securepass
```

Lets deploy the new broker.v3

```bash
oc apply -f ./refs/broker.v3.yaml
```

Wait for initialiation

```bash
oc get pods -w
```

Lets check the logs to make sure we have what we need:

```bash
oc logs broker-ss-0 --follow
```

You should see something like:

```bash
AMQ221007: Server is now active
```

Also check there are no errors displaying...

## With Evertyhing running Test the svc