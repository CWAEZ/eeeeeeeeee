# incognito

<p align="center">
<a href="https://clojurians.slack.com/archives/CB7GJAN0L"><img src="https://img.shields.io/badge/clojurians%20slack-join%20channel-blueviolet"/></a>
<a href="https://clojars.org/io.replikativ/incognito"> <img src="https://img.shields.io/clojars/v/io.replikativ/incognito.svg" /></a>
<a href="https://circleci.com/gh/replikativ/incognito"><img src="https://circleci.com/gh/replikativ/incognito.svg?style=shield"/></a>
<a href="https://github.com/replikativ/incognito/tree/development"><img src="https://img.shields.io/github/last-commit/replikativ/incognito/development"/></a>
<a href="https://versions.deps.co/replikativ/incognito" title="Dependencies Status"><img src="https://versions.deps.co/replikativ/incognito/status.svg" /></a>
</p>

Different Clojure(Script) serialization protocols like `edn`, `fressian` or
`transit` offer different ways to serialize custom types. In general they fall
back to maps for unknown Clojure record types, which is a reasonable default in
many situations. But when you build a distributed data management system parts
of your system might not care about the record types while others do. This
library safely wraps unknown record types and therefore allows to unwrap them
later. It also unifies record serialization between `fressian` and `transit` as
long as you can express your serialization format in Clojure's default
datastructures.

The general idea is that most custom Clojure datatypes (in particular records)
can be expressed in Clojure datastructures if you do not need a custom binary
format, e.g. for efficiency or performance. With incognito you do not need
custom handlers for every serialization format. But you can still provide them
of course, once you hit efficiency problems. Incognito is at the moment not
supposed to provide serialization directly to storage, so you have to be able to
serialize your custom types in memory.

## Normalization

All types are normalized between Clojure and ClojureScript, where for
ClojureScript the namespace separator `/` is replaced by `.` and `-` by `_`.
This means you need to define your handlers with `.` and `_` on all runtimes, an example is shown below.


## Usage

Add this to your project dependencies:
[![Clojars Project](http://clojars.org/io.replikativ/incognito/latest-version.svg)](http://clojars.org/io.replikativ/incognito)

Include all serialization libraries you need:
```clojure
[org.clojure/data.fressian com.cognitect/transit-clj "0.8.297"]
[io.replikativ/incognito "0.2.7"]
```

In general you can control serialization by `write-handlers` and
`read-handlers`,

```clojure
(defrecord Bar [a b])

(def write-handlers (atom {'my_user.ns.Bar (fn [bar] bar)}))
(def read-handlers (atom {'my_user.ns.Bar map->Bar}))
```

To handle custom non-record types you have to transform it into a readable
data-structure. You can test the roundtrip directly with incognito-reader and
writer:

```clojure
(require '[clj-time.core :as t])
(require '[clj-time.format :as tf])

(incognito-writer 
  ;; read-handlers
  {'org.joda.time.DateTime (fn [r] (str r))}

  (t/now))

(incognito-reader 
  ;; write-handlers
  {'org.joda.time.DateTime (fn [r] (t/date-time r))}

  {:tag 'org.joda.time.DateTime, :value "2017-04-17T13:11:29.977Z"})

```
*NOTE*: The syntax quote for the read handler is necessary such that you can 
deserialize unknown classes.

A write-handler has to return an associative datastructure which is
internally stored as an untyped map together with the tag information.


## Plugging incognito into your serializer

You need to wrap the base map handler in the different serializers so incognito
can wrap the serialization for its own handlers. 

(Extracted from the tests):

### edn strings

```clojure
(require '[incognito.edn :refer [read-string-safe]])

(let [bar (map->Bar {:a [1 2 3] :b {:c "Fooos"}})]
  (= bar (->> bar
              pr-str
              (read-string-safe {})
              pr-str
              (read-string-safe read-handlers))))
```


### transit
```clojure
(require '[incognito.transit :refer [incognito-write-handler incognito-read-handler]]
         '[cognitect.transit :as transit])

(let [bar (map->Bar {:a [1 2 3] :b {:c "Fooos"}})]
  (= (assoc bar :c "banana")
     (with-open [baos (ByteArrayOutputStream.)]
       (let [writer (transit/writer baos :json
                                    {:handlers {java.util.Map
                                                (incognito-write-handler
                                                 write-handlers)}})]
         (transit/write writer bar)
         (let [bais (ByteArrayInputStream. (.toByteArray baos))
               reader (transit/reader bais :json
                                      {:handlers {"incognito"
                                                  (incognito-read-handler read-handlers)}})]
           (transit/read reader))))))
```

### fressian

```clojure
(require '[clojure.data.fressian :as fress]
         '[incognito.fressian :refer [incognito-read-handlers
                                      incognito-write-handlers]])

(let [bar (map->Bar {:a [1 2 3] :b {:c "Fooos"}})]
  (= (assoc bar :c "banana")
     (with-open [baos (ByteArrayOutputStream.)]
       (let [w (fress/create-writer baos
                                    :handlers
                                    (-> (merge fress/clojure-write-handlers
                                               (incognito-write-handlers write-handlers))
                                        fress/associative-lookup
                                        fress/inheritance-lookup))] ;
         (fress/write-object w bar)
         (let [bais (ByteArrayInputStream. (.toByteArray baos))]
           (fress/read bais
                       :handlers
                       (-> (merge fress/clojure-read-handlers
                                  (incognito-read-handlers read-handlers))
                           fress/associative-lookup)))))))
```


## TODO

- fix transit support for https://github.com/cognitect/transit-java/issues/31
- move serialization dependencies into dev profile


## Changelog

### 0.2.6
- Fix namespace normalization between Clojure and ClojureScript. Bump transit
  dependencies and deactivate tests because of
  https://github.com/cognitect/transit-java/issues/31.

### 0.2.5
- ClojureScript fressian support (for konserve filestore on node.js). Thanks to
  Ferdinand K??hne!


## License

Copyright ?? 2015-2021 Christian Weilbach, Ferdinand K??hne

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
