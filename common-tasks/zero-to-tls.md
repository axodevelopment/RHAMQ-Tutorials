# Zero to TLS - Deploying a broker from no SSL to using SSL


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

## 1. Create OpenShift Projects

Create two new projects (namespaces) on your OpenShift cluster for the Oracle database and the peer brokers:

```bash
oc new-project ssl-test-broker
```

---




