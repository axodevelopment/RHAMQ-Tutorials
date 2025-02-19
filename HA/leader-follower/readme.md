# Leader-Follower AMQ Broker Example

Hopefully after following the next steps you'll be able to setup and understand a Leader-Follower broker configuration.

## Table of Contents

1. [Prerequisites](#prerequisites)  
2. [Create OpenShift Projects](#1-create-openshift-projects)  
3. [Set Up Permissions](#2-setup-oracle-permissions)  
4. [Database Account Setup](#3-database-setup)  
5. [Deploy the Oracle Database](#4-deploy-the-oracle-database)   
6. [Prepare TLS Certificates](#5-certs)  
7. [Deploy the Peer Brokers](#6-deploying-the-peer-brokers)  
8. [Additional Notes](#additional-notes)  

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

2.) Grant permissions to run the Oracle container:
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

---

## 3. Database setup

create a secret to store the Oracle db password:
```bash
oc create secret generic oracle-db-pass \
  --from-literal=password=secret \
  -n oracle-db-amq
```

---

## 4. Deploy the Oracle database

First we will deploy the statefulset yaml for oracle.
```bash
oc apply -f db-statefulset.yaml
oc get pods -w -n oracle-db-amq
```

However it is not actually done not only do we need to wait for the pods to finish, we also want to wait for the db to finish inializing.  This includes creating internal user accounts, setting up its local storage etc.

```bash
oc logs oracle-db-0 --follow -n oracle-db-amq
```

You will be looking for the following message (in Oracle's case):
```bash
#########################
DATABASE IS READY TO USE!
#########################
```

WARNING
Only after this message appears should you continue with the next steps.  If you processed forward beforte it fully initializes you may have issues with the jdbc connector we are using with the broker configuration ahead.

---

## 5. Certs

Time for certs, its the same steps as in the basic-tls so...

Option A: Copy existing secret from the previous step in the basic broker namespace...
If you are using a mac here is an example copy step:
```bash
oc get secret ext-amqp-acceptor-ssl-secret -n basic-broker -o yaml | \
  sed 's/namespace: basic-broker/namespace: peer-broker/' | \
  sed 's/creationTimestamp:.*$/creationTimestamp: null/' | \
  sed '/resourceVersion:/d' | \
  sed '/uid:/d' | \
  oc create -f -
```

If you want you can just use the first line to get the yaml and do a manual clean or equivalent tool to strip out the annotations and data not used / needed for this project.

```bash
oc get secret ext-amqp-acceptor-ssl-secret -n basic-broker -o yaml >> ext-amqp-acceptor-ssl-secret.yaml

vim secret.yaml

and delete these entries
  creationTimestamp: "2025-02-18T06:36:53Z"
  resourceVersion: "34625546"
  uid: 0cddef2f-cc39-4f5c-b270-d8467832bcd6

afterwards you can

oc apply -f ext-amqp-acceptor-ssl-secret.yaml
```

Option B: Download the sample scripts
Create a trustore location:

1.) Download .jks files: (these are public test ones provided by Apache / Artemis)
```bash
wget -O server-keystore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/server-keystore.jks
wget -O server-ca-truststore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/server-ca-truststore.jks
wget -O client-ca-truststore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/client-ca-truststore.jks
```

2.) Create a secret from the files:
```bash
oc create secret generic ext-amqp-acceptor-ssl-secret \
--from-file=broker.ks=server-keystore.jks \
--from-file=client.ts=client-ca-truststore.jks \
--from-literal=keyStorePassword=securepass \
--from-literal=trustStorePassword=securepass
```

3.) (Optional) Collect the client truststore and server keystore from the secret (This is more for clients).
```bash
oc get secret ext-amqp-acceptor-ssl-secret -n peer-broker -o jsonpath='{.data.client\.ts}' | base64 -d > cs-client-truststore.jks

oc get secret ext-amqp-acceptor-ssl-secret -n peer-broker -o jsonpath='{.data.broker\.ks}' | base64 -d > cs-server-keystore.jks
```

4.) (Optional) Import both into a combined truststore (This is more for clients).
```bash
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
```

5.) (Optional) Verify combined contents (This is more for clients).
```bash
keytool -list -v \
  -keystore combined-truststore.jks \
  -storepass securepass
```

Option C: Use your own

...

---

## 6. Deploying the Peer Brokers

Now we can deploy the brokers.  
```bash
oc apply -f peer-broker-a-v1.yaml
oc apply -f peer-broker-b-v1.yaml
```

What are we changing in the broker yaml files?  We want to ensure that we have persistence Enabled turned off, as we are writing to Oracle, and that clustered is off since our HAPolicy will be shared via db.
```bash
spec:
  deploymentPlan:
    size: 1
    clustered: false
    persistenceEnabled: false
```

Note we override the livenessProbe to avoid spinning pod.
```bash
    livenessProbe:
      exec:
        command:
        - test
        - -f
        - /home/jboss/amq-broker/lock/cli.lock
```

To use the Oracle DB for persistence in this way we need to run an init container to create the ARTEMIS_EXTRA_LIBS directory and download the jars so the broker can load it.
```bash
resourceTemplates:
    - selector:
        kind: StatefulSet
      patch:
        kind: StatefulSet
        spec:
          template:
            spec:
              initContainers:
                - name: oracle-database-jdbc-driver-init
                  image: registry.redhat.io/amq7/amq-broker-rhel8:7.12
                  volumeMounts:
                    - name: amq-cfg-dir
                      mountPath: /amq/init/config
                  command:
                    - "bash"
                    - "-c"
                    - "mkdir -p /amq/init/config/extra-libs && curl -Lo /amq/init/config/extra-libs/ojdbc11.jar https://download.oracle.com/otn-pub/otn_software/jdbc/233/ojdbc11.jar"
```

If we have done everything correctly only broker should be running in the peer-broker namespace
```bash
oc get pods -n peer-broker
peer-broker-a-ss-0   0/1     Running   0          58m
peer-broker-b-ss-0   1/1     Running   0          58m
```

In the logs of the first pod (broker A), you may see:

```bash
AMQ221034: Waiting indefinitely to obtain primary lock
```
This indicates the broker is in standby, waiting for the lock.


In the logs of the second pod (broker B), you may see:

```bash

AMQ221007: Server is now active
```
This indicates the broker that obtained the lock is now the leader (active)




# notes You can ignore this section, it will be removed when I finalize this page.

# TODO 
1.) Add diagram of leader-follower relationship for oracle
2.) Add some clarity around the livenessProbe change
3.) Add more details around ARTEMIS_EXTRA_LIBS

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


