Connection/Protocol Comparison:

2025-02-17 18:17:50,818 INFO  [org.apache.activemq.artemis.core.server] AMQ221043: Protocol module found: [artemis-server]. Adding protocol support for: CORE
2025-02-17 18:17:50,818 INFO  [org.apache.activemq.artemis.core.server] AMQ221043: Protocol module found: [artemis-amqp-protocol]. Adding protocol support for: AMQP
2025-02-17 18:17:50,819 INFO  [org.apache.activemq.artemis.core.server] AMQ221043: Protocol module found: [artemis-hornetq-protocol]. Adding protocol support for: HORNETQ
2025-02-17 18:17:50,819 INFO  [org.apache.activemq.artemis.core.server] AMQ221043: Protocol module found: [artemis-mqtt-protocol]. Adding protocol support for: MQTT
2025-02-17 18:17:50,819 INFO  [org.apache.activemq.artemis.core.server] AMQ221043: Protocol module found: [artemis-openwire-protocol]. Adding protocol support for: OPENWIRE
2025-02-17 18:17:50,820 INFO  [org.apache.activemq.artemis.core.server] AMQ221043: Protocol module found: [artemis-stomp-protocol]. Adding protocol support for: STOMP


RabbitMQ                AMQ                              ArtemisNotes
AMQP 0-9-1 (primary)    AMQP 1.0, CORE (primary)         AMQ CORE protocol can have better performance
MQTT                    MQTT                             *Similar*
STOMP                   STOMP                            *
HTTP-AMQ                NA                               no HTTP protocol
Stream                  NA                               none