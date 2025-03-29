# Exploring Console


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

Since the console uses JMX as a component here I would not normally recommend exposing the console via a route but use something like port-forwarding where you can reduce the surface area of who has access and from where.

First you will want to use the hdls svc and then port forward that

This service should already exist if you enable the console in the broker but here it is for reference:

```bash
  ports:
    - name: jgroups
      protocol: TCP
      appProtocol: tcp
      port: 7800
      targetPort: 7800
    - name: console-jolokia
      protocol: TCP
      appProtocol: http
      port: 8161
      targetPort: 8161
    - name: all
      protocol: TCP
      port: 61616
      targetPort: 61616
```

The port-forward call from this service for the console-jolokia target at 8161

```bash
oc port-forward svc/broker-hdls-svc 8161:8161
```

By default the username and password is `admin` + `admin` you can configure this in your broker properties:

```bash
  adminUser: admin
  adminPassword: admin
```

---

So first thing to note, this fuse based UI that makes up console, is really 4 major components, we won't go over all of them but I want to give you an idea of here is the main screen.

![ConsoleView](https://github.com/axodevelopment/RHAMQ-Tutorials/blob/main/images/console-overview.jpg)


---

Lets start with Artemis and JMX

First with Artemis I can look at a variety of settings for a particular address or queue.

![ConsoleView](https://github.com/axodevelopment/RHAMQ-Tutorials/blob/main/images/console-artemis.jpg)


Then we can look at JMX.  This page allows me to run multiple command get see some high level data on what is happening side my queues.

![JmxView](https://github.com/axodevelopment/RHAMQ-Tutorials/blob/main/images/console-jmx.jpg)


---

Jolokia API is a powerful api that you can also use to do commands directly against the broker

How you could disable a queue:

```bash
http://localhost:8161/console/jolokia/exec/org.apache.activemq.artemis:broker=!%22amq-broker!%22,component=addresses,address=!%22flifo.actuals!%22,subcomponent=queues,routing-type=!%22anycast!%22,queue=!%22flifo.actuals!%22/disable()
```

An example of counting messages in a queue:

```bash
http://localhost:8161/console/jolokia/exec/org.apache.activemq.artemis:broker=!%22amq-broker!%22,component=addresses,address=!%22flifo.actuals!%22,subcomponent=queues,routing-type=!%22anycast!%22,queue=!%22flifo.actuals!%22/countMessages()
```

![JolokiaView](https://github.com/axodevelopment/RHAMQ-Tutorials/blob/main/images/console-jolokia.jpg)

To get the jolokia api calls like above you can click on the screen where it says copy jolokia api call and it will give you the link to the call you are making here in the operations tab.

---