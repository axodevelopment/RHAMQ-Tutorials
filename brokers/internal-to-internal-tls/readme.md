# Example broker using reencrypt-route

This will be a general implementation of a Federated AMQ Broker setup.



############## WARNING ############
This is auto managed certs do not use with aceptors that are exposed, or external client usage WARNING@#OIJF

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
oc new-project amq-broker
```

---

