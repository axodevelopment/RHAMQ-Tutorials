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