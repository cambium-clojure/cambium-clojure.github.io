---
layout: default
title: Cambium Documentation - Structured logging for Clojure
---
# [Home](/) | [About](/about.html) | Documentation | [Discuss](/discuss.html)

## Cambium core

### Requiring namespaces

```clojure
(require '[cambium.core :as log])
(require '[cambium.mdc  :as mlog])
```

### Namespace based loggers

A namespace based logger uses the namespace where the log event is originated as the logger name. Like
`clojure.tools.logging/<log-level>`, Cambium defines namespace loggers for various levels:

```clojure
(log/info "this is a log message")                                          ; simple message logging
(log/info {:latency-ms 331 :module "registration"} "App registered")        ; context and message
(log/debug {:module "order-processing"} "sequence-id verified")
(log/error {:module "user-feedback"} exception "Email notification failed") ; context, exception, msg
```

Available log levels: `trace`, `debug`, `info`, `warn`, `error`, `fatal`.

### Custom loggers

You can define custom loggers (with predefined logger name) that you can use from any namespace as follows:

```clojure
(log/deflogger metrics "METRICS")
(log/deflogger txn-log "TXN-LOG" :info :fatal)

(metrics {:latency-ms 331 :module "registration"} "app.registration.success") ; context and message
(txn-log {:module "order-processing"} exception "Stock unavailable")          ; context, exception, msg
(txn-log "Order processed")                                                   ; simple message logging
```

### Context propagation

Value based context can be propagated as follows:

```clojure
;; Propagate specified context in current thread
(log/with-logging-context {:user-id "X1234"}
  ...
  (log/info {:job-id 89} "User was assigned a new job")
  ...)

;; wrap an existing fn with specified context
(log/wrap-logging-context {:user-id "X1234"} user-assign-job)  ; creates a wrapped fn
```

#### MDC propagation

Unlike value based propagation MDC propagation happens wholesale, i.e. the entire current MDC map is replaced with a
new map. Also, no conversion is applied to MDC; they are required to have string keys and values. See example below:

```clojure
;; Propagate specified context in current thread
(mlog/with-raw-mdc {"userid" "X1234"}
  ...
  (log/info {:job-id 89} "User was assigned a new job")
  ...)

;; wrap an existing fn with specified context
(mlog/wrap-raw-mdc user-assign-job)  ; creates a wrapped fn that inherits current context
(mlog/wrap-raw-mdc {"userid" "X1234"} user-assign-job)  ; creates wrapped fn inheriting specified context
```

The most common use of MDC propagation is to pass the logging context to child threads in a concurrent scenario.


## Cambium codec

A Cambium codec governs how the log attributes are encoded and decoded before they are effectively sent to a log
layout. A codec constitutes the following:

| Var in `cambium.codec` ns     | Type               | Description                                 |
|-------------------------------|--------------------|---------------------------------------------|
| cambium.codec/nested-nav?     | Boolean            | Should context read/write be nesting-aware? |
| cambium.codec/stringify-key   | (fn [k]) -> String | Encodes log attribute key as string         |
| cambium.codec/stringify-val   | (fn [v]) -> String | Encodes log attribute value as string       |
| cambium.codec/destringify-val | (fn [s]) -> value  | Decodes log attribute from string form      |

Note: You need only one codec implementation in a project.

Cambium comes with two codec modules, `cambium/cambium.codec-simple` and `cambium/cambium.codec-cheshire`. While
`codec-simple` is a very simple codec that converts everything to string, codec-cheshire is a nesting-aware codec
that preserves types as long as they are valid [JSON](https://en.wikipedia.org/wiki/JSON).

### Nested context

Sometimes context values may be nested and need manipulation. Cambium requires nesting-aware codec for dealing with
nested context. Almost all nesting-aware backends need to be configured to use the codec before logging events:

```clojure
(require '[cambium.codec :as codec])    ; assuming we have codec-cheshire
(require '[cambium.logback.json.flat-layout :as flat])

(flat/set-decoder! codec/destringify-val)
```

See nesting-navigation example below:

```clojure
(log/with-logging-context {:order {:client "XYZ Corp"
                                   :item-count 10}}
  ;; ..other processing..
  (log/with-logging-context {[:order :id] "F-123456"}
    ;; here the context will be {"order" {"client" "XYZ Corp" "item-count" 10 "id" "F-123456"}}
    (log/info "Order processed successfully")))
;; the logging API processes nested MDC correctly when the codec is nesting-capable
(log/info {:order {:event-id "foo"}} "Foo happened")
```

## Logback backend

Cambium modules for Logback have add-on features described in the sections below.

### Overriding log level at runtime

Logback-bundle provides support for overriding log levels at runtime using a strategy based TurboFilter.

#### One time setup

Add the following Logback configuration (or equivalent) to logback.xml:

```xml
<turboFilter class="logback_bundle.core.StrategyTurboFilter">
  <name>mdcStrategy</name>
</turboFilter>
<turboFilter class="logback_bundle.core.StrategyTurboFilter">
  <name>multiStrategy</name>
</turboFilter>
```

#### Configure log level overrides

Configure strategy:

```clojure
(require '[cambium.logback.core.strategy :as strategy])

;; set new log level to be determined by MDC attribute `forcelevel`
(strategy/set-mdc-strategy! "mdcStrategy" "forcelevel")

;; set levels for logger names for next 15 seconds, root logger being set to ERROR
(strategy/set-multi-strategy! "multiStrategy" "error" (strategy/multi-millis-validator 15000))
(strategy/set-log-level! "multiStrategy" ["com.foo" "com.bar"] "debug")
```

#### Generate log events

```clojure
(log/debug "If this code is in com.foo.* or com.bar.* namespace this event will be logged.")

(import '[org.slf4j MDC])

(MDC/put "forcelevel" "debug")  ; force new level to be considered DEBUG
(log/debug "This event will be logged overriding any existing INFO/WARN/ERROR level")
```

#### Remove log level overrides

You can remove log level overrides at any time:

```clojure
(strategy/remove-strategy! "mdcStrategy")
(strategy/remove-strategy! "multiStrategy")
```

### Configuring logback_bundle.json.FlatJsonLayout

The `cambium.logback.json.FlatJsonLayout` class is designed to accommodate customizations to preserve types/nesting
of MDC attributes and wholesale MDC transformation.

```clojure
(require '[cambium.logback.json.flat-layout :as flat])
```

#### Setting MDC value decoder

Setting a value decoder is useful when you encode MDC values (for preserving data types/nesting etc.) and want them
to be decoded before they are sent to the appender.

```clojure
;; assume EDN-encoded value, so parse using EDN reader
(flat/set-decoder! clojure.edn/read-string)
```

#### Setting MDC map transformer

In a large number of cases this may not be necessary. However, applying arbitrary transformation to the MDC map is
useful at times to backfill custom attributes, or to normalize certain attributes.

```clojure
(import 'java.util.Map)
;; insert an extra attribute to the MDC
(flat/set-transformer! (fn [^Map m] (.put m "extra-attr" "some value") m))
```


<a href='https://github.com/cambium-clojure'><img style='position: absolute; top: 0; right: 0; border: 0;' src='https://camo.githubusercontent.com/652c5b9acfaddf3a9c326fa6bde407b87f7be0f4/68747470733a2f2f73332e616d617a6f6e6177732e636f6d2f6769746875622f726962626f6e732f666f726b6d655f72696768745f6f72616e67655f6666373630302e706e67' alt='Fork me on GitHub' data-canonical-src='https://s3.amazonaws.com/github/ribbons/forkme_right_orange_ff7600.png'></a>
