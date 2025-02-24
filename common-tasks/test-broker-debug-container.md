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

On terminal (term1) we will have access to any 
On one terminal we will be running the oc debug command which gives us a bash entry to the debug pod.


# on your machine (term1)

oc get route

oc get secret amqp-acceptor-ptls -n amq-broker -o jsonpath='{.data.tls\.crt}' | base64 -d > server.crt

# on another terminal (term2)

oc debug -n amq-broker --image=registry.access.redhat.com/ubi8/ubi-minimal

microdnf install -y nmap-ncat bind-utils java-17-openjdk wget tar gzip

mkdir -p /tmp/amq-test/ssl

wget https://archive.apache.org/dist/activemq/activemq-artemis/2.31.2/apache-artemis-2.31.2-bin.tar.gz

tar xzf apache-artemis-2.31.2-bin.tar.gz

# on your machine (term1)

oc get pods | grep debug

oc cp server.crt amq-broker/<debug-pod-name>:/tmp/amq-test/ssl/

oc cp server.crt amq-broker/image-debug-gjr5w:/tmp/amq-test/ssl/

# on another terminal (term2)

keytool -import -v -alias ca -file /tmp/amq-test/ssl/server.pem \
  -keystore /tmp/amq-test/ssl/truststore.p12 \
  -storetype PKCS12 \
  -storepass securepass \
  -noprompt

/apache-artemis-2.31.2/bin/artemis producer \
  --protocol amqp \
  --url "amqps://broker-amqp-acceptor-0-svc.amq-broker.svc:5672?transport.trustStoreType=PKCS12&transport.trustStoreLocation=/tmp/amq-test/ssl/truststore.p12&transport.trustStorePassword=securepass&transport.verifyHost=false&sslEnabled=true" \
  --message "Test SSL Message via Service" \
  --destination myQueue

NOTE:
need route if testing passthrough

  /apache-artemis-2.31.2/bin/artemis producer \
  --protocol amqp \
  --url "amqps://<route>:443?transport.trustStoreType=PKCS12&transport.trustStoreLocation=/tmp/amq-test/ssl/truststore.p12&transport.trustStorePassword=securepass&transport.verifyHost=false&sslEnabled=true" \
  --message "Test SSL Message via Route" \
  --destination myQueue