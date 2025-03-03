# Connection/Protocol Client details:

## Table 1: Client To Acceptor url

It is important to distinguish how your client protocol will behave with an acceptor.  If for example you set `protocols: all` your url format will need to be mapped accordingly.  In this repo, most protocols are assumed to be `amqp`.

| Client Protocol | Details | AMQ Broker Acceptor Protocol | Protocol URL Format| Typical Port |
|-----------------|---------|------------------------------|--------------------|--------------|
| OpenWire (Non-SSL) | Used by OpenWire JMS clients  | OPENWIRE                            | tcp://host:61616                    | 61616        |
| OpenWire (SSL)     | Secure OpenWire connection    | OPENWIRE with sslEnabled: true      | ssl://host:61617                    | 61617        |
| Core (Non-SSL)     | Used by ActiveMQ Artemis Core clients | CORE                       | tcp://host:61616                    | 61616        |
| Core (SSL)         | Secure Core connection       | CORE with sslEnabled: true          | tcp://host:61617?sslEnabled=true     | 61617        |
| AMQP (Non-SSL)     | Used by AMQP 1.0 clients (e.g., Qpid JMS) | AMQP                      | amqp://host:5672                     | 5672         |
| AMQP (SSL)         | Secure AMQP connection      | AMQP with sslEnabled: true          | amqps://host:5671                    | 5671         |
| MQTT (Non-SSL)     | Used by MQTT clients        | MQTT                               | mqtt://host:1883                     | 1883         |
| MQTT (SSL)         | Secure MQTT connection      | MQTT with sslEnabled:true          | mqtts://host:8883                    | 8883         |
| STOMP (Non-SSL)    | Used by STOMP clients       | STOMP                              | stomp://host:61613                    | 61613        |
| STOMP (SSL)        | Secure STOMP connection     | STOMP with sslEnabled: true         | stomps://host:61614                   | 61614        |
