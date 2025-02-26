# Zero to TLS - Deploying a broker from no SSL to using SSL


## Table of Contents

   [Prerequisites](#prerequisites)
   [Summary](#summary)
   [Diagrams](#diagrams)

Steps to deploy federated brokers:

1. [Create OpenShift Projects](#1-create-openshift-projects-and-broker-deploy)  
2. [Without SSL](#2-testing-the-endpoint-from-within-the-ocp-cluster)  
3. [Test With SSL](#3-database-setup)  
4. [Deploy the Oracle Database](#4-deploy-the-oracle-database)   
5. [Prepare TLS Certificates](#5-certs)  
6. [Deploy the Peer Brokers](#6-deploying-the-peer-brokers)    

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



## 3. Testing (With SSL) the endpoint from within the OCP cluster 















Now back to ### TERM1

### Get the debug pod name
```bash
oc get pods | grep debug
```

Should look something like:
image-debug-<uniqu>

You'll need this for later