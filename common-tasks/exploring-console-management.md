# Zero to TLS - Exploring Console


## Table of Contents

   [Prerequisites](#prerequisites)
   [Summary](#tutorial)

---

## Prerequisites

- OpenShift CLI (oc)  
- Access to an OpenShift cluster  
- Basic familiarity with OpenShift projects, ServiceAccounts, and Secrets  
- (Optional) Existing TLS certificates. Otherwise, you can generate or fetch sample certificates.  

---

## Tutorial

To create a `queue` you need to first have some logical representation of which is represented by an `address`.

http://localhost:8161/console/jolokia/exec/org.apache.activemq.artemis:broker=!%22amq-broker!%22,component=addresses,address=!%22flifo.actuals!%22,subcomponent=queues,routing-type=!%22anycast!%22,queue=!%22flifo.actuals!%22/disable()

http://localhost:8161/console/jolokia/exec/org.apache.activemq.artemis:broker=!%22amq-broker!%22,component=addresses,address=!%22flifo.actuals!%22,subcomponent=queues,routing-type=!%22anycast!%22,queue=!%22flifo.actuals!%22/countMessages()

---