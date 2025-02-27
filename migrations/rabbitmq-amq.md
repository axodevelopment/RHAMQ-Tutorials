Connection/Protocol Comparison:

# Table 1: Protocol Modules (Log Entries)

| Module                    | Protocol Support |
|---------------------------|------------------|
| artemis-server            | CORE             |
| artemis-amqp-protocol     | AMQP             |
| artemis-hornetq-protocol  | HORNETQ          |
| artemis-mqtt-protocol     | MQTT             |
| artemis-openwire-protocol | OPENWIRE         |
| artemis-stomp-protocol    | STOMP            |

# Table 2: Connection/Protocol Comparison

| RabbitMQ             | AMQ                      | ArtemisNotes                                 |
|----------------------|--------------------------|----------------------------------------------|
| AMQP 0-9-1 (primary) | AMQP 1.0, CORE (primary) | AMQ CORE protocol can have better performance|
| MQTT                 | MQTT                     | *Similar*                                    |
| STOMP                | STOMP                    | *                                            |
| HTTP-AMQ             | NA                       | no HTTP protocol                             |
| Stream               | NA                       | none                                         |



