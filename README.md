# Overview of RH AMQ Broker Tutorials

There are several independant tutorials in this repo

## Table of Contents

Tutorial listing

1. [Basic Broker Configus](#basic-brokers)  
2. [HA Broker Configs](#ha-configurations)  
3. [Common Tasks](#common-tasks) 
4. [Migrations](#migrations) 

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

High level examples of various types of brokers with different auth and settings are located in the "brokers" folder.

| Name               | Description                    | Status           |
|--------------------|--------------------------------|------------------|
| basic-auth         | How to configure basic auth    | Draft *          |
| basic-broker       | Limited impl with no auth      | Draft *          |
| basic-tls          | Self Signed Cert broker        | Draft *          |
| common-broker      | Some bells and whistles        | In Progress      |
| external-to-int    | Attempt to do reencrypt        | Early            |
| int-to-int         | Leveraging auto-ssl in OCP     | In Progress      |

---

# HA Configurations

High level examples of HA implementations with different setups are located in the "HA" folder.


| Name               | Description                    | Status           |
|--------------------|--------------------------------|------------------|
| dr                 | Sample dr broker setup         | Draft *          |
| federation         | Same federation setup          | Draft *          |
| leader-follower ex | How to leader-follower (DB)    | Draft *          |


# Common Tasks

High level tutorials or walkthroughs on various types of tasks are located in the "common-tasks" folder.

| Name               | Description                    | Status           |
|--------------------|--------------------------------|------------------|
| zero-to-tls        | From working with no ssl to ssl| Draft *          |
| full cert chain    | Tutorials may use full cert so this is my setup.          | Draft *          |
| debugging-amq      | How to debug amq (DB)          | Early            |
| test tool          | How to use my test tool        | Draft *          |


# Migrations

High level thoughts or details on migrating from one system to AMQ are located in the "migrations" folder.

| Name               | Description                    | Status           |
|--------------------|--------------------------------|------------------|
| rabbitmq           | Strategies for migration from RMQ | Early         |