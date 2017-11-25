---
layout: default
title: About Cambium - Structured logging for Clojure
---
# [Home](/) | About | [Documentation](/documentation.html) | [Discuss](/discuss.html)

## Rationale

Log messages are quite useful for human consumption but they are not easily machine parseable. It is machine parsing
of log event data that makes them many times more valuable, especially in the age of information overload. Consider a
log message and the same information as a structured log event:

```clojure
"Order placed vide AXO/41294 with AcmeCorp for part number F-39942, quantity 5 pairs"

;; versus

{:event        "order.placed"
 :order-id     "AXO/41294"
 :customer     "AcmeCorp"
 :part-number  "F-39942"
 :quantity     5
 :denomination "pair"}
```

Data is a first class concept in Clojure - our logs should be the same. Structured logging means logs are first class
data.


### What is Cambium?

Cambium is a collection of libraries to help you log events as structured data. Cambium primarily wraps
[SLF4J](https://www.slf4j.org/) and extends [clojure/tools.logging](https://github.com/clojure/tools.logging)
to provide a data oriented logging API. SLF4J needs a backing implementation for its API, so Cambium also provides
implementation hooks for the data orientation.

You can control the degree of flexibility you need with Cambium's modularity. It builds just a little over the API
exposed by _clojure/tools.logging_, so it should be familiar enough for most people to adopt easily. Combined with
the ubiquity of SLF4J on the JVM, you can tap into all the Java library eco-system without any worry. You may choose
any SLF4J backing implementation as long as it is adapted to use Cambium's data orientation.


### How it works

At minimum, you need a Cambium codec and the _cambium.core_ module to start logging. You also need an SLF4j backend
configured to use the chosen Cambium codec for the logs to appear at some destination. Cambium makes use of SLF4j
[Mapped Diagnostic Context (MDC)](https://www.slf4j.org/api/org/slf4j/MDC.html) to communicate the data attributes to
the logging backend. Hence, the logging backend must support SLF4j MDC and should be configured to use the codec.
The Cambium codec is also used to overcome the lack of builtin support in SLF4j MDC for nested data.

Cambium comes with [Logback](https://logback.qos.ch/) backend modules for SLF4j, named `cambium/cambium.logback.*`.
Cambium does not provide any support for configuring or initializing Logback, though it includes all dependencies
required to do so.


<a href='https://github.com/cambium-clojure'><img style='position: absolute; top: 0; right: 0; border: 0;' src='https://camo.githubusercontent.com/652c5b9acfaddf3a9c326fa6bde407b87f7be0f4/68747470733a2f2f73332e616d617a6f6e6177732e636f6d2f6769746875622f726962626f6e732f666f726b6d655f72696768745f6f72616e67655f6666373630302e706e67' alt='Fork me on GitHub' data-canonical-src='https://s3.amazonaws.com/github/ribbons/forkme_right_orange_ff7600.png'></a>
