# Example broker using reencrypt-route

This will be a general implementation of a Federated AMQ Broker setup.

## Table of Contents

   [Prerequisites](#prerequisites)
   [Summary](#summary)
   [Diagrams](#diagrams)

Steps to deploy federated brokers:

1. [Create OpenShift Projects](#1-create-openshift-projects)  
2. [Set Up Permissions](#2-setup-oracle-permissions)  
3. [Database Account Setup](#3-database-setup)  
4. [Deploy the Oracle Database](#4-deploy-the-oracle-database)   
5. [Prepare TLS Certificates](#5-certs)  
6. [Deploy the Peer Brokers](#6-deploying-the-peer-brokers)    

---

## Prerequisites

- OpenShift CLI (oc)  
- Access to an OpenShift cluster  
- Basic familiarity with OpenShift projects, ServiceAccounts, and Secrets  
- Oracle database image or a container image that can be used in OpenShift  
- (Optional) Existing TLS certificates. Otherwise, you can generate or fetch sample certificates.  

---


## Summary

We are setting up a Federated architecture for ActiveMQ Artemis brokers that...

---

## Diagrams



---

## 1. Create OpenShift Projects

Create two new projects (namespaces) on your OpenShift cluster for the Oracle database and the peer brokers:

```bash
oc new-project amq-ss-broker
```

---

keytool -genkey -alias broker -keyalg RSA -keystore broker.ks -storepass password123

keytool -export -alias broker -keystore broker.ks -file broker_cert.pem

keytool -export -alias broker -keystore broker.ks -storepass password123 -rfc -file broker_cert.pem

Confirm you have a pem

Converting an Existing DER File into PEM

openssl x509 -inform DER -in broker_cert.der -out broker_cert.pem

verify 
openssl x509 -in broker_cert.pem -noout -text


When generating a secret, OpenShift requires you to specify both a key store and a trust store. The trust store key is generically named client.ts. For one-way TLS between the broker and a client, a trust store is not actually required. However, to successfully generate the secret, you need to specify some valid store file as a value for client.ts. The preceding step provides a "dummy" value for client.ts by reusing the previously-generated broker key store file. This is sufficient to generate a secret with all of the credentials required for one-way TLS.

cp broker.ks client.ks

oc create secret generic amqp-acceptor-secret \
  --from-file=broker.ks=./broker.ks \
  --from-file=client.ts=./client.ks \
  --from-literal=keyStorePassword=password123 \
  --from-literal=trustStorePassword=password123 \
  -n amq-ss-broker

of apply -f w broker.yaml

oc get pods -w

If you go to the route (oc get route) you should see your ingress / wildcard cert

we need that for our external app to connect

we need to obtain the rout root ca
oc get secret router-certs-default \
  -n openshift-ingress \
  -o jsonpath='{.data.tls\.crt}' \
  | base64 -d > router.crt


Testing because im not sure TLS Streams are supported for reencrypt scenarios by OCP Route

oc get pod broker-ss-0 -o wide

IP = 10.130.1.145


pod/e4-43-4b-db-d8-30-debug-7jb7h

tcpdump -i any host 10.130.1.145 -w /tmp/pod_traffic.pcap

tcpdump -i any -n -s 0 host 192.168.30.195 and port 443 -c 20 -w /tmp/ingress_traffic.pcap



# trying to prove amqp doesn't work in ocp reencrypt since it is a TLS Stream

oc get secret amqp-acceptor-secret -o jsonpath="{.data.broker\.ks}" | base64 -d > broker.ks

-rfc to make pem

keytool -exportcert -rfc -alias broker -keystore broker.ks -storepass password123 -file broker.pem

