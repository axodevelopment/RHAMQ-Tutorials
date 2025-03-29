# Zero to TLS - Managing Queues


## Table of Contents

- [Prerequisites](#prerequisites)
- [Summary](#tutorial)

---

## Prerequisites

- OpenShift CLI (oc)  
- Access to an OpenShift cluster  
- Basic familiarity with OpenShift projects, ServiceAccounts, and Secrets  
- (Optional) Existing TLS certificates. Otherwise, you can generate or fetch sample certificates.  

---

## Tutorial

To create a `queue` you need to first have some logical representation of which is represented by an `address`.

So lets take a look at what is happening in the following example yaml:
```bash
   brokerProperties:
    - "# Enable auto-creation of queues"
    - addressSettings."#".autoCreateQueues=true
    - addressSettings."#".autoCreateAddresses=true
    - "# Configure default address settings"
    - addressSettings."#".defaultQueueRoutingType=ANYCAST
    - addressSettings."#".defaultAddressRoutingType=ANYCAST
```
Lets expand:

```bash
   brokerProperties:
    - "# Enable auto-creation of queues" # This is just a comment, to help me organize
    - addressSettings."#".autoCreateQueues=true # first item to note is the `#` this is a wildcard token to match all addresses
    # the property here is for any address given, I am allowing auto creation of queues.
    - addressSettings."#".autoCreateAddresses=true
    # tied with the above for any address given, if an address isn't already created go ahead and create one
    - "# Configure default address settings"
    - addressSettings."#".defaultQueueRoutingType=ANYCAST
    - addressSettings."#".defaultAddressRoutingType=ANYCAST
    # these settings say that for any address, make the default queue and address types `ANYCAST` (basid / default queue type)
```

So what is happening here practically

`addressSettings.` Is saying we want to set a value on the addressSettings.  Normally what would come nexst is the name of the address like so...

`addressSettings."someaddress".` then I can specify various properties of an address that I want to have highlighting the above properties as an example (Auto Create queues / addresses when a specified queue or address doesn't exist).

Note on match statements `"someaddress"` is a type of matching statement.  There are others like the above:

`"somestring"` is an exact match in property.
`"#"` catch all match, technically a multi token match 
`"*"` will match one token (i'll go over this more in a bit)

So given the following table
| Value             | Tokens         |
|--------------------|--------------------------------|
| `a`       | a | 
| `a.b`    | a, b |
| `a.b.c`   | a, b, c | 
| `a.b.c.d`        | a, b, c, d| 
| `a.b.c.e`        | a, b, c, e|
| `a.c`        | a, c|
| `b.a`        | b.a |   

A couple examples for us to understand what gets returned
| Match             | Values         |
|--------------------|--------------------------------|
| `"a"`       | `a` | 
| `"a.*"`    | `a.b`, `a.c` |
| `"a.#"`   | `a.b`, `a.b.c`, `a.b.c.d`, `a.b.c.e`, `a.c` | 
| `b.#`        | `b.a` |
| `a.b.c.*` |  `a.b.c.d`, `a.b.c.e` |


Hopefully this helps.

---

So lets start by making a queue for ourselves and see the Artemis Console.

Lets make a custom address that we will have manage our new queue with the following setup.

- A new address called `flifo`
    - routingType of `ANYCAST`
    - queues
        - "flifo.actuals" of type `ANYCAST`
        - `flifo.schedules` of type `MULTICAST`
- Lets have the DLQ and Expiry set
    - deadLetterAddress=aDLQ
    - expiryAddress=aExpiryQueue

for the broker properties this maps to:

```bash
  brokerProperties:
    - addressConfigurations."flifo".routingTypes=ANYCAST
    - addressConfigurations."flifo".queueConfigs."flifo.actuals".routingType=ANYCAST
    - addressConfigurations."flifo".queueConfigs."flifo.schedules".routingType=MULTICAST
    - addressSettings."flifo.*".deadLetterAddress=aDLQ
    - addressSettings."flifo.*".expiryAddress=aExpiryQueue
    - addressSettings."flifo.*".maxDeliveryAttempts=3
```


When we look into console we see:

![ConsoleView](https://github.com/axodevelopment/RHAMQ-Tutorials/blob/main/images/address-status.jpg)

You might notice that while the main queues look right, that the `aDLQ` and `aExpiryQueue` are missing.  By default `DLQ` and `ExpiryQueue` are created but, if you want to rename them like we did above, you'll want to approach it one of two ways.  Either use autoCreateQeueu / Addresses or manually specify them.

autocreateQueues exampe:

```bash
  brokerProperties:
    - "# Enable auto-creation of queues"
    - addressSettings."#".autoCreateQueues=true
    - addressSettings."#".autoCreateAddresses=true

```

Or you can manually create the address:

```bash

    - addressConfigurations."aDLQ".routingTypes=ANYCAST
    - addressConfigurations."aDLQ".queueConfigs."aDLQ".routingType=ANYCAST
    - addressConfigurations."aExpiryQueue".routingTypes=ANYCAST
    - addressConfigurations."aExpiryQueue".queueConfigs."aExpiryQueue".routingType=ANYCAST

```

That being said, unless needed, I would recommend using the default DLQ and ExpiryQueue. With that you would be just left with the configuration of:

```bash
    - "# Enable auto-creation of queues"
    - addressSettings."#".autoCreateQueues=true
    - addressSettings."#".autoCreateAddresses=true
    - addressConfigurations."flifo".routingTypes=ANYCAST
    - addressConfigurations."flifo".queueConfigs."flifo.actuals".routingType=ANYCAST
    - addressConfigurations."flifo".queueConfigs."flifo.schedules".routingType=MULTICAST
```

---



## Reference Links

Complete list of properties available.

https://docs.redhat.com/en/documentation/red_hat_amq_broker/7.11/html/configuring_amq_broker/ref-br-broker-properties_configuring#addresssettings

