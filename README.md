# Overview of RH AMQ Broker Tutorials

There are several independant tutorials in this repo

## Table of Contents

Tutorial listing

1. [Basic Broker Configus](#basic-brokers)  
2. [HA Broker Configs](#ha-configurations)  

---

## Prerequisites

- OpenShift CLI (oc)  
- Access to an OpenShift cluster  
- Basic familiarity with OpenShift projects, ServiceAccounts, and Secrets
- (Optional) Existing TLS certificates. Otherwise, you can generate or fetch sample certificates.  

---


## Summary

We are setting up a Federated architecture for ActiveMQ Artemis brokers that...

---


# RHAMQ-Tutorials
A tutorial covering Artemis / RH-AMQ


To start please refer to /brokers/basic-broker

This will cover the basics of creating an AMQ Broker without authentication.

The next section would be /brokers/basic-auth

Aftewards for TLS refer to /brokers/basic-tls.


# reference documents
https://access.redhat.com/articles/2791941

---

# Basic Brokers

Very high level broker with a single quque...

/brokers/basic-broker

-

Continuing from above but adding basic auth

/brokers/basic-auth

-

Lastly SSL connection

/brokers/basic-tls

---

# HA Configurations

/brokers/HA
