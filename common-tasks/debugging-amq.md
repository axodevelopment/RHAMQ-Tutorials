apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
spec:
  ...
  config:
    env:
    - name: ARGS
      value: "--zap-log-level=debug --lease-duration=18 --renew-deadline=12 --retry-period=3"

In broker yaml I add
```bash
spec:
  env:
    - name: JAVA_ARGS_APPEND
      value: "-Dwebconfig.bindings.artemis.uri=http://0.0.0.0:8161 -Dartemis.acceptors.amqp.host=0.0.0.0"
    - name: AMQ_EXTRA_ARGS
      value: "--relax-jolokia"
```

This allows me to bind to the interface with port-forwarding to my terminal.

Get the broker xml
oc exec broker-ss-0 -- cat amq-broker/etc/broker.xml