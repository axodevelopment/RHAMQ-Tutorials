# Debug Broker Example Steps

As Admons you may want to easily test if the broker is working if you don't have access to various apps, these steps allow you to debug from a pod so you don't have to build a custom app or expose other tooling which you may not want to do.

I have attached the Dockerfile in this repo it is a peer to this document.  For quick reference here is the breakdown of that file.

```bash
FROM registry.access.redhat.com/ubi8/ubi-minimal

USER root

ENV ARTEMIS_VERSION=2.31.2 \
    ARTEMIS_HOME=/opt/apache-artemis \
    JAVA_HOME=/usr/lib/jvm/jre-17

RUN microdnf update -y && \
    microdnf install -y \
    nmap-ncat \
    bind-utils \
    java-17-openjdk \
    wget \
    tar \
    gzip \
    openssl \
    && microdnf clean all \
    && rm -rf /var/cache/yum

RUN mkdir -p /opt \
    /tmp/amq-test/ssl

RUN wget https://archive.apache.org/dist/activemq/activemq-artemis/${ARTEMIS_VERSION}/apache-artemis-${ARTEMIS_VERSION}-bin.tar.gz -O /tmp/artemis.tar.gz && \
    tar -xzf /tmp/artemis.tar.gz -C /opt && \
    mv /opt/apache-artemis-${ARTEMIS_VERSION} ${ARTEMIS_HOME} && \
    rm -f /tmp/artemis.tar.gz

ENV PATH="${ARTEMIS_HOME}/bin:${PATH}"

RUN chown -R 1001:0 ${ARTEMIS_HOME} /tmp/amq-test && \
    chmod -R g=u ${ARTEMIS_HOME} /tmp/amq-test

USER 1001

WORKDIR ${ARTEMIS_HOME}

CMD ["sleep", "infinity"]
```

In short, I am downloading amq-artemis, installing it and making it available to call.

The goal here being that when I run this as a debug pod, I have access to artemis and items like keytool so that I can run certain utilities with artemis and ultimate push some test messages to our AMQ Broker.

At a high level, we will need 2 terminals.

On terminal (term1) we will have access to any .ks we have that.
On another terminal (term2) we will be running the oc debug command which gives us a bash entry to the debug pod.

# (term2)
# run the tool image

This will create a debug pod with that --image.  Note you will need to either build the docker image with your changes or pull it from my repo and push it to a repo your OCP environment has access to.
```bash
oc debug -n common-broker --image=axodevelopment/artemistools:v1.0.0.3
```

# (term1)
# get the jks
I created a directory for this.

note in brocker.yaml
```bash
    acceptors
      - name: amqp-acceptor
        sslEnabled: true
        sslSecret: amqp-acceptor-secret  <--
```
Get the ks we will use for this pod...
```bash
oc get secret amqp-acceptor-secret -n common-broker -o jsonpath='{.data.broker\.ks}' | base64 -d > broker.ks
```

These steps apply to a JKS type but ultimately if you have a PKCS ultimate again we just need an unencrypted PEM file

Get the alias for the pem you want to pull

# get pem crt out of jks
```bash
keytool -exportcert -rfc \
  -keystore broker.ks \
  -alias server \
  -file server.pem \
  -storepass securepass
```

if you cat server.pem, you should see an unencrypted pem
```bash
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

# get the debug pod name
```bash
oc get pods | grep debug
```

# copy the pem over
I am using an example of image-debug-qdsnh, note you should use the value you got from the step above
```bash
oc cp server.pem common-broker/image-debug-qdsnh:/tmp/amq-test/ssl/
```

# in the debug pod (term2)
#
# confirm we have the pem
Double check on the server.pem
```bash
cat /tmp/amq-test/ssl/server.pem
```

# create th pkcs12 store
```bash
keytool -import -v -alias ca -file /tmp/amq-test/ssl/server.pem \
  -keystore /tmp/amq-test/ssl/truststore.p12 \
  -storetype PKCS12 \
  -storepass securepass \
  -noprompt
```

```bash
bin/artemis producer \
  --protocol amqp \
  --url "amqps://broker-amqp-acceptor-0-svc.common-broker.svc:5672?transport.trustStoreType=PKCS12&transport.trustStoreLocation=/tmp/amq-test/ssl/truststore.p12&transport.trustStorePassword=securepass&transport.verifyHost=false&sslEnabled=true" \
  --message "Test SSL Message via Service" \
  --destination myQueue
```

We should see something along the following:

```bash
Connection brokerURL = amqps://broker-amqp-acceptor-0-svc.common-broker.svc:5672?transport.trustStoreType=PKCS12&transport.trustStoreLocation=/tmp/amq-test/ssl/truststore.p12&transport.trustStorePassword=securepass&transport.verifyHost=false&sslEnabled=true
Producer myQueue, thread=0 Started to calculate elapsed time ...

Producer myQueue, thread=0 Produced: 1000 messages
Producer myQueue, thread=0 Elapsed time in second : 3 s
Producer myQueue, thread=0 Elapsed time in milli second : 3986 milli seconds
```