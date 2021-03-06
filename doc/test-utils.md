# Trapperkeeper Test Utils

Trapperkeeper provides some [utility code](../test/puppetlabs/trapperkeeper/testutils)
for use in tests. The code is available in a separate "test" jar that you may depend
on by using a classifier in your project dependencies.

```clojure
  (defproject yourproject "1.0.0"
    ...
    :profiles {:dev {:dependencies [[puppetlabs/trapperkeeper "x.y.z" :classifier "test"]]}})
```

## Logging

Some utilities to ensure that the correct logging messages are generated by your application code are located in
[logging.clj](../test/puppetlabs/trapperkeeper/testutils/logging.clj)

### with-test-logging

All logging messages generated by code executed using the `with-test-logging` macro are available for later inspection.
A simple example would be:

```clojure
(with-test-logging
  (log/info "hello log")
  (is (logged? #"^hello log$"))
  (is (logged? #"^hello log$" :info)))
```

In this code `(log/info "hello log")` generates the message `hello log` at the `INFO` level. The existence of this
message is then later tested by the `logged?` function.

### logged?

The `logged?` function always takes a regex as its first parameter, declared with Clojure's `#"pattern"` notation. If
this regex matches any line that has been logged within the body of a `with-test-logging` then `true` will be returned.

The optional second parameter is a keyword describing a log level to specifically search in. It currently can be one of
`:trace`, `:debug`, `:info`, `:warn`, `:error` or `:fatal`.

## Testing Services

For the most part, your Trapperkeeper service definitions should be written as
very thin wrappers around regular clojure functions.  Thus, the vast majority
of your tests can be written as normal clojure unit tests that operate on those
functions directly.

However, it can be useful to have a few tests that actually boot up a Trapperkeeper
application instance; this allows you to, for example, verify that the services
that you have a dependency on get injected correctly.

To this end, the testutils library code includes some helper functions and macros
for creating a Trapperkeeper application.  The macros should be preferred in
most cases; they generally start with the prefix `with-app-`, and allow you to
create a temporary Trapperkeeper app given a list of services.  They will take
care of some important mechanics for you:

* Making sure that no JVM shutdown hooks are registered during tests, as they
  would be during a normal Trapperkeeper application boot sequence
* Making sure that the app is shut down properly after the test completes.

Here are some of the most useful ones:

### with-app-with-config

This macro allows you to specify your services directly, and to pass in a map
of configuration data that the app should use:

```clj
(ns services.test-service-1)

(defprotocol TestService1
   (test-fn [this]))

(defservice test-service1
  TestService1
  []
  (test-fn [this] "foo"))
```
```clj
(ns services.test-service2)

(defservice test-service2
  ;;...
  )
```
```clj
(ns test.services-test
   (:require services.test-service-1 :as t1))

(with-app-with-config app
   [test-service1 test-service2]
   {:myconfig {:foo "foo"
               :bar "bar"}}
   (let [test-svc  (get-service app :TestService1)]
      (is (= "baz" (t1/test-fn test-svc))))
```

### with-app-with-cli-data

This variant is very similar, but instead of passing a map of config data, you
pass a map of parsed cli args, such as the path to a config file on disk that
should be processed to build the actual application configuration:

```clj
(with-app-with-cli-data app
   [test-service1 test-service2]
   {:config "./test-resources/config.ini"}
   (let [test-svc  (get-service app :TestService1)]
      (is (= "baz" (t1/test-fn test-svc))))
```

### with-app-with-cli-args

This version accepts a vector of command line args:

```clj
(with-app-with-cli-args app
   [test-service1 test-service2]
   ["--config" "./test-resources/config.ini" "--debug"]
   (let [test-svc  (get-service app :TestService1)]
      (is (= "baz" (t1/test-fn test-svc))))
```

### with-app-with-empty-config

This version is useful when you don't need to pass in any configuration data
at all to the services

```clj
(with-app-with-empty-config app
   [test-service1 test-service2]
   (let [test-svc  (get-service app :TestService1)]
      (is (= "baz" (t1/test-fn test-svc))))
```

For each of the above macros, there is generally a `bootstrap-services-with-*`
function that will behave similarly; however, the `bootstrap-*` functions don't
handle the cleanup / shutdown behaviors for you, so should only be used in rare
cases.
