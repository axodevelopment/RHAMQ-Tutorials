# pre-req

oc new-project basic-broker

### basic-broker v1

oc apply -f basic-broker-v1.yaml

oc get svc

How the svc name gets defined
name = <broker.name>-<acceptor.name>-0-svc

# how do we test
## port forward to localhost
Since we are testing amqp we will use the amqp svc

```
oc port-forward service/basic-broker-amqp-0-svc 5672:5672
```

configure your app to localhost to point to this svc

```
quarkus.qpid-jms.url=amqp://localhost:5672
quarkus.log.category."org.apache.qpid.jms".level=DEBUG
```

## check console

find the 'wconsj' svc
basic-broker-wconsj-0-svc

oc apply -f console-route.yaml

view console to see your messages going through

oc get route
should see something like
broker-console                  broker-console-basic-broker.apps.axolab.axodevelopment.dev                         basic-broker-wconsj-0-svc   8161       edge/Redirect   None

How the url name is framed.
broker-console-basic-broker.apps.axolab.axodevelopment.dev
<route-name>-<route-ns>.apps.<cluster-name>.<domain>

login with admin / admin

Artemis => amq-broker -> addresses -> myQueue -> queues -> anycast -> myQueue
Operations
countMessages()
Execute

if you have sent messages into the queue you should see messages there.

## check pvc

you could also mount the pvc directly and explore the internals

oc get pvc -n basic-broker
basic-broker-basic-broker-ss-0

while ti should be in /opt/basic-broker/data you can check by
oc exec -it basic-broker-ss-0 -- df -h
or check the volumeMounts on the broker yaml itself:
oc get pod basic-broker-ss-0 -o yaml | grep -A 5 volumeMounts

in my case you can see the directory of data at
oc exec -it basic-broker-ss-0 -- ls -la /opt/basic-broker/data
or get a bash
oc exec -it basic-broker-ss-0 -- /bin/bash

under journal you can see activemq-data-1.amq you can cat that file to see your data in there.

