Connection/Protocol Comparison:

# Table 1: Protocol Modules (Log Entries)

| Timestamp               | Module                    | Protocol Support |
|-------------------------|---------------------------|------------------|
| 2025-02-17 18:17:50,818 | artemis-server            | CORE             |
| 2025-02-17 18:17:50,818 | artemis-amqp-protocol     | AMQP             |
| 2025-02-17 18:17:50,819 | artemis-hornetq-protocol  | HORNETQ          |
| 2025-02-17 18:17:50,819 | artemis-mqtt-protocol     | MQTT             |
| 2025-02-17 18:17:50,819 | artemis-openwire-protocol | OPENWIRE         |
| 2025-02-17 18:17:50,820 | artemis-stomp-protocol    | STOMP            |

# Table 2: Connection/Protocol Comparison

| RabbitMQ             | AMQ                      | ArtemisNotes                                 |
|----------------------|--------------------------|----------------------------------------------|
| AMQP 0-9-1 (primary) | AMQP 1.0, CORE (primary) | AMQ CORE protocol can have better performance|
| MQTT                 | MQTT                     | *Similar*                                    |
| STOMP                | STOMP                    | *                                            |
| HTTP-AMQ             | NA                       | no HTTP protocol                             |
| Stream               | NA                       | none                                         |



