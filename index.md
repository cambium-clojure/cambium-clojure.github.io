---
layout: default
title: Cambium Home - Structured logging for Clojure
---
# Home | [About](/about.html) | [Documentation](/documentation.html) | [Discuss](/discuss.html)


_**Cambium requires Clojure 1.6 or higher, Java 6 or higher.**_


## Latest release

### API

|-----------------------------------------------------------------------------------------|--------------------------------------|----------------------------------------------|-----------------------------------------------------------------|
| Module                                                                                  | Description                          | Clojars artifact                             | Dependencies                                                    |
|-----------------------------------------------------------------------------------------|--------------------------------------|----------------------------------------------|-----------------------------------------------------------------|
| [cambium.core](https://github.com/cambium-clojure/cambium.core)                         | SLF4j based core API implementation  | `[cambium/cambium.core             "1.1.0"]` | [SLF4j](https://www.slf4j.org/) |
| [cambium.codec-simple](https://github.com/cambium-clojure/cambium.codec-simple)         | Simple, non-nested codec             | `[cambium/cambium.codec-simple     "1.0.0"]` | |
| [cambium.codec-cheshire](https://github.com/cambium-clojure/cambium.codec-cheshire)     | Cheshire based nesting-capable codec | `[cambium/cambium.codec-cheshire   "1.0.0"]` | [Cheshire](https://github.com/dakrone/cheshire)                 |


### Backend: Logback

|-----------------------------------------------------------------------------------------|--------------------------------------|----------------------------------------------|-----------------------------------------------------------------|
| Module                                                                                  | Description                          | Clojars artifact                             | Dependencies                                                    |
|-----------------------------------------------------------------------------------------|--------------------------------------|----------------------------------------------|-----------------------------------------------------------------|
| [cambium.logback.core](https://github.com/cambium-clojure/cambium.logback.core)         | Core Logback backend                 | `[cambium/cambium.logback.core     "0.4.4"]` | [Logback](https://logback.qos.ch/)                              |
| [cambium.logback.json](https://github.com/cambium-clojure/cambium.logback.json)         | JSON Logback backend                 | `[cambium/cambium.logback.json     "0.4.4"]` | [Jackson](https://github.com/FasterXML/jackson)                 |
| [cambium.logback.rabbitmq](https://github.com/cambium-clojure/cambium.logback.rabbitmq) | RabbitMQ appender for Logback        | `[cambium/cambium.logback.rabbitmq "0.4.4"]` | [RabbitMQ client](https://www.rabbitmq.com/java-client.html)    |


## Quickstart

### Configure application for only Text logs

Include the following dependencies in your project:

```clojure
[cambium/cambium.core         "1.1.0"]
[cambium/cambium.codec-simple "1.0.0"]
[cambium/cambium.logback.core "0.4.4"]
```

Then, create a file `resources/logback.xml` with the following content:

```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <!-- encoders are assigned the type
         ch.qos.logback.classic.encoder.PatternLayoutEncoder by default -->
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg { %mdc }%n</pattern>
    </encoder>
  </appender>
  <root level="debug">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

In your application, log events as data:

```clojure
(ns myapp.main
  (:require
    [cambium.core  :as log]))

(defn -main
  [& args]
  (log/info "Application started")
  (log/info {:args (vec args) :argc (count args)} "Arguments received")
  ;; load rest of the application
  )
```

#### Text log output

The log output comes to the console when we run the app:

```
$ lein run foo 10
18:56:42.054 [main] INFO  myapp.main - Application started { ns=myapp.main, line=8, column=3 }
18:56:42.060 [main] INFO  myapp.main - Arguments received { args=["foo" "10"], argc=2, ns=myapp.main, line=9, column=3 }
```

When you run the same command without creating the `resources/logback.xml` file:

```
$ lein run foo 10
18:57:48.836 [main] INFO myapp.main - Application started
18:57:48.842 [main] INFO myapp.main - Arguments received
```

### Configure application for JSON (and Text) logs

Include the following dependencies in your project:

```clojure
[cambium/cambium.core           "1.1.0"]
[cambium/cambium.codec-cheshire "1.0.0"]
[cambium/cambium.logback.json   "0.4.4"]
```

Create `resources/logback.xml` file in your project with the following content:

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
          <layout class="cambium.logback.json.FlatJsonLayout">
            <jsonFormatter class="ch.qos.logback.contrib.jackson.JacksonJsonFormatter">
              <!-- prettyPrint is probably ok in dev, but usually not ideal in production: -->
              <prettyPrint>true</prettyPrint>
            </jsonFormatter>
            <!-- <context>api</context> -->
            <timestampFormat>yyyy-MM-dd'T'HH:mm:ss.SSS'Z'</timestampFormat>
            <timestampFormatTimezoneId>UTC</timestampFormatTimezoneId>
            <appendLineSeparator>true</appendLineSeparator>
          </layout>
        </encoder>
    </appender>
    <root level="debug">
      <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

Now before your application logs any event, configure the SLF4j backend to use the codec:

```clojure
(ns myapp.main
  (:require
    [cambium.codec :as codec]
    [cambium.core  :as log]
    [cambium.logback.json.flat-layout :as flat]))

(defn -main
  [& args]
  (flat/set-decoder! codec/destringify-val)  ; configure backend with the codec
  (log/info "Application started")
  (log/info {:args (vec args) :argc (count args)} "Arguments received")
  ;; start rest of the app
  )
```

#### JSON log output

The log output comes to the console:

```json
$ lein run foo 10
{
  "timestamp" : "2017-12-01T13:31:42.265Z",
  "level" : "INFO",
  "thread" : "main",
  "ns" : "myapp.main",
  "line" : 12,
  "column" : 3,
  "logger" : "myapp.main",
  "message" : "Application started",
  "context" : "default"
}
{
  "timestamp" : "2017-12-01T13:31:42.323Z",
  "level" : "INFO",
  "thread" : "main",
  "args" : [ "foo", "10" ],
  "argc" : 2,
  "ns" : "myapp.main",
  "line" : 13,
  "column" : 3,
  "logger" : "myapp.main",
  "message" : "Arguments received",
  "context" : "default"
}
```


## License

Copyright © 2017-2020 [Shantanu Kumar](https://github.com/kumarshantanu)

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.

<a href='https://github.com/cambium-clojure'><img style='position: absolute; top: 0; right: 0; border: 0;' src='https://camo.githubusercontent.com/652c5b9acfaddf3a9c326fa6bde407b87f7be0f4/68747470733a2f2f73332e616d617a6f6e6177732e636f6d2f6769746875622f726962626f6e732f666f726b6d655f72696768745f6f72616e67655f6666373630302e706e67' alt='Fork me on GitHub' data-canonical-src='https://s3.amazonaws.com/github/ribbons/forkme_right_orange_ff7600.png'></a>
