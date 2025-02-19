# Leader-Follower AMQ Broker Example

Hopefully after following the next steps you'll be able to setup and understand a Leader-Follower broker configuration.

## Table of Contents

1. [Prerequisites](#prerequisites)  
2. [Create OpenShift Projects](#1-create-openshift-projects)  
3. [Set Up Permissions](#2-setup-oracle-permissions)  
4. [Database Account Setup](#database-account-setup)  
5. [Deploy the Oracle Database](#deploy-the-oracle-database)  
6. [Wait for DB Initialization](#wait-for-db-initialization)  
7. [Prepare TLS Certificates](#prepare-tls-certificates)  
8. [Deploy the Peer Brokers](#deploy-the-peer-brokers)  
9. [Additional Notes](#additional-notes)  

---

## Prerequisites

- OpenShift CLI (oc)  
- Access to an OpenShift cluster  
- Basic familiarity with OpenShift projects, ServiceAccounts, and Secrets  
- Oracle database image or a container image that can be used in OpenShift  
- (Optional) Existing TLS certificates. Otherwise, you can generate or fetch sample certificates.  

---

## 1. Create OpenShift Projects

Create two new projects (namespaces) on your OpenShift cluster for the Oracle database and the peer brokers:

```bash
oc new-project oracle-db-amq
oc new-project peer-broker
```

---

## 2. Setup Oracle Permissions

An Oracle DB like most others needs elevated permissions.

1.) Create a service account for Oracle (oracle-sa):

```bash
oc create serviceaccount oracle-sa -n oracle-db-amq
```

2.) Grant permissions to run the Oracle container
```bash
oc adm policy add-scc-to-user anyuid -z oracle-sa -n oracle-db-amq
oc adm policy add-scc-to-user anyuid -z oracle-sa
```

3.) (Optional) Confirm the service account has permission to use the anyuid SCC:
```bash
oc adm policy who-can use scc anyuid
...
###
system:serviceaccount:oracle-db-amq:oracle-sa
```








# leader-follower

Broker's competing for lock on backend db

# create project
oc new-project oracle-db-amq
oc new-project peer-broker

# permissions

oracle needs permissions

oc create serviceaccount oracle-sa -n oracle-db-amq
oc adm policy add-scc-to-user anyuid -z oracle-sa -n oracle-db-amq
oc adm policy add-scc-to-user anyuid -z oracle-sa

confirm oracle has permissions
oc adm policy who-can use scc anyuid

# db account setup
oc create secret generic oracle-db-pass \
  --from-literal=password=secret \
  -n oracle-db-amq

# deploy db

oc apply -f db-statefulset.yaml

oc get pods -w

DO NOT PROCECEED UNTIL DB IS FULLY INIT
db still needs to be init (this takes time, in oracle case, up to several minutes)
oc logs oracle-db-0 --follow

you should see
```
#########################
DATABASE IS READY TO USE!
#########################
```

# prep certs

get certs


If you have the certs from the basic-broker tls still installed the same cert here is used 

oc get secret ext-amqp-acceptor-ssl-secret -n basic-broker -o yaml | \
sed 's/namespace: basic-broker/namespace: peer-broker/' | \
sed 's/creationTimestamp:.*$/creationTimestamp: null/' | \
sed '/resourceVersion:/d' | \
sed '/uid:/d' | \
oc create -f -


-- or --

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

oc get secret ext-amqp-acceptor-ssl-secret -n peer-broker -o jsonpath='{.data.client\.ts}' | base64 -d > cs-client-truststore.jks

oc get secret ext-amqp-acceptor-ssl-secret -n peer-broker -o jsonpath='{.data.broker\.ks}' | base64 -d > cs-server-keystore.jks

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

# peer brokers



oc apply -f peer-broker-a-v1.yaml
oc apply -f peer-broker-b-v1.yaml


in logs of POD 0/1 => "AMQ221034: Waiting indefinitely to obtain primary lock"
in logs of POD 1/1 => "AMQ221007: Server is now active"


# add a dr


