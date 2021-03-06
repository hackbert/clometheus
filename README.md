# clometheus

__clometheus__ is a zero dependency instrumentation library for [Prometheus](https://prometheus.io/).

[![Clojars Project](https://img.shields.io/clojars/v/online.duevel/clometheus.svg)](https://clojars.org/online.duevel/clometheus)

[![Build Status](https://travis-ci.com/carlduevel/clometheus.svg?branch=master)](https://travis-ci.com/carlduevel/clometheus)

[![Dependencies Status](https://jarkeeper.com/carlduevel/clometheus/status.svg)](https://jarkeeper.com/carlduevel/clometheus)

## Why?

There are a lot of clojure wrappers for the [Java Prometheus client](https://github.com/prometheus/client_java) out there.
They all force you to register metrics before using them, because that is
how the java client works. This is a good idea in Java, but not so much in Clojure
as it leads to code like this:

```clojure
(defonce my-registry (prometheus/create-registry)) ; a registry object to pass around and take care off.
(defonce __my-counter (register-counter :my-dear-counter registry)) ; a ref that is not interesting because it will not be referenced later
....

(prometheus/inc! :my-dear-counter) ; This is error prone: Get the label wrong and after all: What type is it of? What labels does it have?

```
Instead to instrument with clometheus, you just do it on the spot.
No registry to take care of (if you do not want to) and no registration necessary:

```clojure
(require '[clometheus.core :as c])
(c/inc! (c/counter "my_counter"))
```
Clometheus can do this because it does not just wrap the prometheus library but the instrumentation
code is implemented in Clojure itself (with the summary implementation being the sole exception as it is the same
as in the Java client).

## Usage

### Counters
A simple counter:
```clojure
(def request-counter (c/counter "requests_total"))
(c/inc! request-counter)

```
Or with more bells and whistles:

```clojure

(def request-counter-with-labels (c/counter "requests_total" :description "Counter for http requests." :labels ["rc" "method"]))

(c/inc! request-counter-with-labels :by 2 :labels {"rc" 200 "method" "GET"})

```
Remember: Counters can only go up!

### Gauges
A simple gauge:
```clojure
(def sessions-gauge (c/gauge "sessions"))
(c/set! sessions-gauge 12)
(c/inc! sessions-gauge)
(c/dec! sessions-gauge)
```
Or with description and labels:
```clojure
(def db-conn-gauge (c/gauge "database_connections" :description "Database connection count" :lables ["db" "host"]))
(c/inc! db-conn-gauge :lables {"db" "my-db" "host" "somehost"})

 ```
 Sometimes a callback gauge is convenient:

 ```clojure
 (c/gauge  "connection_pool_size" :description "Size of connection pool" :callback-fn #(42))
 ```
### Histograms
A histogram with labels:
```clojure
(def http-histogram (c/histogram "http_duration_in_s" :description "duration for processing an http request" :labels ["method" "path" "rc"]))
(c/observe! http-histogram 0.01 :labels {"method" "get" "path" "/" "rc" "200"})
```
This will use the default bucket sizes of the prometheus Java client.
But you can also define your own:

```clojure
(c/histogram "special_buckets" :buckets [0.1 1 10])
```

### Summaries
You can have a summary without specifying quantiles:
```clojure
(def summary (c/summary "look_ma_no_quantiles"))
(c/observe summary 1.0)
```
But more likely you want some quantiles:
```clojure
(c/summary "kafka_lag_in_seconds" :labels ["consumer_group_id" "topic"] :quantiles [(c/quantile 0.5  0.05)(c/quantile 0.9  0.01) (c/quantile 0.99 0.001)])
```

### Exposing metrics
Create a compojure route to export your metrics:
```clojure
(require '[clometheus.txt-format :as txt]
         '[compojure.core :refer (GET)])
(GET "/metrics" [] (txt/metrics-response))
```
It is possible to use the prometheus java client alongside clometheus.
Exporting metrics can then be done like this:

```clojure
(require '[clometheus.txt-format :as txt]
         '[clometheus.core :as c])
(import io.prometheus.client.exporter.common.TextFormat
        io.prometheus.client.CollectorRegistry
        clometheus.core.ICollectorRegistry)


(defn- text-format
  ([] (text-format  CollectorRegistry/defaultRegistry c/default-registry))
  ([^CollectorRegistry java-client-registry ^ICollectorRegistry clometheus-registry]
   (with-open [out (java.io.StringWriter.)]
     (TextFormat/write004 out (.metricFamilySamples java-client-registry))
     (doseq [sample (c/collect clometheus-registry)]
       (txt/write out sample))
     (str out))))

(defn metrics-response
  ([] (metrics-response CollectorRegistry/defaultRegistry c/default-registry))
  ([^CollectorRegistry java-client-registry  ^ICollectorRegistry clometheus-registry]
   {:headers {"Content-Type" "text/plain; version=0.0.4; charset=utf-8"}
    :status  200
    :body    (text-format java-client-registry clometheus-registry)}))
```
### Misc
In order to [avoid missing
metrics](https://www.robustperception.io/existential-issues-with-metrics)
you should define metrics before using them.
So for an error counter do not:
```clojure
(c/inc! (c/counter "my_error_counter"))
```
But define the error counter beforehand so it does not show up out of nowhere:
```clojure
(def my-error-counter (c/counter "my_error_counter"))
[...]
(c/inc! my-error-counter)
```

## License

This project is licensed under the Apache License 2.0.
