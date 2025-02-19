# Federation AMQ Broker Example

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
oc new-project amq-east
oc new-project amq-west
```

---


note in v2
in west broker => 'AMQPConnections.target.uri=tcp://broker-east-all-0-svc.amq-east.svc.cluster.local:61616'
in east broker => ''AMQPConnections.target.uri=tcp://broker-west-all-0-svc.amq-west.svc.cluster.local:61616'

ie west broker federates (connects to west) and vise versa

you should see in the logs
*******************************************************************************************************************************
Connected on Server AMQP Connection target on broker-west-all-0-svc.amq-west.svc.cluster.local:61616 after 9 retries
*******************************************************************************************************************************


We can see what addresses have been created

oc exec -i broker-west-ss-0 -- /home/jboss/amq-broker/bin/artemis address show --acceptor all --user admin --password admin

Connection brokerURL = tcp://broker-west-ss-0.broker-west-hdls-svc.amq-west.svc.cluster.local:61616
$sys.mqtt.sessions
activemq.management.c1f7ef85-d1b1-45e0-97ed-ad5b17fe978e
activemq.notifications
DLQ
ExpiryQueue
-> federation-control-link:abc:39c7e57d-ee84-11ef-8677-0a580a820157:94a2d55e-76b9-48eb-b44e-36e0667b4986
-> federation-events-receiver:abc:33c0784c-ee84-11ef-8631-0a580a820156:71b2b603-cb3e-4a4a-b7ac-bac0ff12a2de
-> tracking


oc exec -i broker-east-ss-0 -- /home/jboss/amq-broker/bin/artemis address show --acceptor all --user admin --password admin

Connection brokerURL = tcp://broker-east-ss-0.broker-east-hdls-svc.amq-east.svc.cluster.local:61616
$sys.mqtt.sessions
activemq.management.34c7924a-5e3e-49b7-9f1c-3654f0d7f8e3
activemq.notifications
DLQ
ExpiryQueue
federation-control-link:abc:33c0784c-ee84-11ef-8631-0a580a820156:d781013a-8e54-4583-86f1-2583e506a52b
federation-events-receiver:abc:39c7e57d-ee84-11ef-8677-0a580a820157:8add9fed-c03d-4b8b-8a5a-dbcae67dcc27
tracking


Since we see both federation receivers over the federation abc we can try the tracking queue.

oc exec -i broker-east-ss-0 -n amq-east -- /home/jboss/amq-broker/bin/artemis address show --acceptor all --user admin --password admin

oc exec -i broker-west-ss-0 -n amq-west -- /home/jboss/amq-broker/bin/artemis address show --acceptor all --user admin --password admin


produce on east -> consume on west


oc exec -i broker-east-ss-0 -n amq-east -- /home/jboss/amq-broker/bin/artemis producer --acceptor all --destination queue://tracking --user admin --password admin --message-count 1 --message 1

oc exec -i broker-west-ss-0 -n amq-west  -- /home/jboss/amq-broker/bin/artemis consumer --acceptor all --destination queue://tracking --user admin --password admin --message-count 1