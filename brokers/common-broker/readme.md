# Leader-Follower AMQ Broker Example

Hopefully after following the next steps you'll be able to setup and understand a Leader-Follower broker configuration.

## Table of Contents

   [Prerequisites](#prerequisites)
   [Summary](#summary)
   [Diagrams](#diagrams)

Steps to deploy peer brokers:

1. [Create Certs](#create-certs--build-truststore)  
2. [Deploy Broker](#deploy-broker)  
3. [Test internal TLS](#test-internal-tls)   

---

## Prerequisites

- OpenShift CLI (oc)  
- Access to an OpenShift cluster  
- Basic familiarity with OpenShift projects, ServiceAccounts, and Secrets  
- Oracle database image or a container image that can be used in OpenShift  
- (Optional) Existing TLS certificates. Otherwise, you can generate or fetch sample certificates.  

We will need a truststore dir for this one so make a directory for your truststore.

```bash
oc new-project common-broker
```

While this command should default your poject to common-broker if you are working outside of this project please ensure the -n commands are used appropriately.
---


## Summary

what we want to achieve
create certs or use your own
create secret from cert
deploy broker
test external passthrough
test internal tls through debug pod

---

# Create Certs + Build truststore

Create a directory for your tls cert work

```bash
wget -O server-keystore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/server-keystore.jks

wget -O server-ca-truststore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/server-ca-truststore.jks

wget -O client-ca-truststore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/client-ca-truststore.jks
```

Note:
You should have 3 files
-client-ca-truststore.jks
-server-ca-truststore.jks
-server-keystore.jks

Note that these are infact JKS ks types
keytool -list -v -keystore server-keystore.jks -storepass securepass

note in brocker.yaml
```bash
    acceptors
      - name: amqp-acceptor
        sslEnabled: true
        sslSecret: amqp-acceptor-secret  <--
```

Lets create the secret called "amqp-acceptor-secret"
Note:
If you are doing one-way TLS you may be asking what do I use for my client.ts.  In this case, OCP secrets require both a server ks and a client ts, you can just copy the server ks to create a ts, since this value won't be used.

```bash
oc create secret generic amqp-acceptor-secret \
--from-file=broker.ks=server-keystore.jks \
--from-file=client.ts=client-ca-truststore.jks \
--from-literal=keyStorePassword=securepass \
--from-literal=trustStorePassword=securepass
```

For a local truststore that we will use to test with

Just for clarity we have the keystore already but im doing this step for clarity...
```bash
oc get secret amqp-acceptor-secret -n common-broker -o jsonpath='{.data.client\.ts}' | base64 -d > cs-client-truststore.jks

oc get secret amqp-acceptor-secret -n common-broker -o jsonpath='{.data.broker\.ks}' | base64 -d > cs-server-keystore.jks
```

Note:
You should have 5 files
-client-ca-truststore.jks
-server-ca-truststore.jks
-server-keystore.jks
-cs-server-keystore.jks
-cs-client-truststore.jks

Now we can create the new combined-truststore.jks
Note:
Based upon your JDK, the deststoretype will default to different values, here to maintain consistency we will use JKS so the deststoretype flag of JKS is used, but if you are using PKCS the steps are similar.

Add the server ks
```bash
keytool -importkeystore \
  -srckeystore cs-server-keystore.jks \
  -destkeystore combined-truststore.jks \
  -srcstorepass securepass \
  -deststorepass securepass \
  -deststoretype JKS
```

Add the client t
```bash
keytool -importkeystore \
  -srckeystore cs-client-truststore.jks \
  -destkeystore combined-truststore.jks \
  -srcstorepass securepass \
  -deststorepass securepass \
  -deststoretype JKS
```

check if everything is correct
```bash
keytool -list -v -keystore combined-truststore.jks -storepass securepass
```

Note:
If you are following this step by step the combined-truststore.jks should have 3 entries.  If you are doing a self signed cert or otherwise another set of certs you'll need to inspect the combined-truststore equivalent of yours to ensure they entries are present.  Again, the client details in my case are simply there for brevity, AMQ will present the server.ks crt files for authentication.

---

# Deploy Broker

oc apply -f broker.yaml

Wait until the broker finishes rolling out and is running.

```bash
oc get pods -w
```

note in brocker.yaml
```bash
    acceptors
      - name: amqp-acceptor <--
        sslEnabled: true
        sslSecret: amqp-acceptor-secret  
```

Lets get the route we are using...
```bash
oc get route | grep amqp-acceptor
```

broker-amqp-acceptor-0-svc-rte-common-broker.apps.cluster.domain

In my case:
broker-amqp-acceptor-0-svc-rte-common-broker.apps.axolab.axodevelopment.dev

<broker.metadata.name>-<acceptor.name>-<ordinal>-svc-rte-<broker.metadata.namespace>...

Ok so we now have a broker lets test it.



You should see the cert you added to the broker being presented in your browser if you navigate to the route in 
oc get route

if you used the broker.yaml
broker-amqp-acceptor-0-svc-rte-common-broker.apps...

---

# Test internal TLS

I have moved this over to another location in the repo...

Please refer to test-tool.md

https://github.com/axodevelopment/RHAMQ-Tutorials/blob/main/common-tasks/test-tool.md
