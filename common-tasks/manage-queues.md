# Zero to TLS - Managing Queues


## Table of Contents

   [Prerequisites](#prerequisites)
   [Summary](#summary)

Steps to deploy federated brokers:

1. [Create OpenShift Projects](#1-create-openshift-projects-and-broker-deploy)  
2. [Without SSL](#2-testing-the-endpoint-from-within-the-ocp-cluster)  
3. [Test With SSL](#3-testing-with-ssl---self-signed-single-the-endpoint-from-within-the-ocp-cluster)  
4. [Test With SSL - Chain](#4-testing-with-ssl---full-chain-the-endpoint-from-within-the-ocp-cluster)   
5. [Cert commands](#5-certs)  

---

## Prerequisites

- OpenShift CLI (oc)  
- Access to an OpenShift cluster  
- Basic familiarity with OpenShift projects, ServiceAccounts, and Secrets  
- (Optional) Existing TLS certificates. Otherwise, you can generate or fetch sample certificates.  

---

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




---

## Reference Links

Complete list of properties available.

https://docs.redhat.com/en/documentation/red_hat_amq_broker/7.11/html/configuring_amq_broker/ref-br-broker-properties_configuring#addresssettings

