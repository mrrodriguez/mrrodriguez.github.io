---
layout: post
title: "Clojure Serialization With Fressian"
author: Mike Rodriguez
date: 2018-02-19
tags: [Fressian, Datomic, Clojure, serialization, deserialization, performance, SerDe, development, programming, Java]
---


## Background

A while back I worked on a feature to add a performant serialization/durability implementation for use in [Clara rules](https://github.com/cerner/clara-rules). In this work, I ended up using the [Fressian serialization format](https://github.com/Datomic/fressian/wiki). I found this to be a fast, binary format that offers an idiomatic Clojure style of extensibility. This is not surprising since the Fressian format, and several of its implementations, are provided by the Clojure core team.

The purpose of this post is to describe my experience with using Fressian as a Clojure serialization format. The "durability" feature in Clara rules is [documented here](http://www.clara-rules.org/docs/durability/) and the initial [pull request is here](https://github.com/cerner/clara-rules/pull/219), but that is not directly relevant to the rest of this post.

## Why use Fressian?

I was on the hunt for a serialization format that I could use to efficiently serialize and deserialize Clojure data across processes. This search led me to several options. Among these (not necessarily all inclusive) are [Kryo](https://github.com/EsotericSoftware/kryo), [Nippy](https://github.com/ptaoussanis/nippy), the built-in [Java serialization](https://docs.oracle.com/javase/tutorial/jndi/objects/serial.html), [Apache Avro](https://avro.apache.org/), and [Fressian](https://github.com/Datomic/fressian).

I will use the term "Schema evolution support" to mean the ability to serialize data at some point in time and later be able to deserialize it in an environment that may have new expectations for how that data may look.

An example is if you serialized a Clojure record `A` with definition:
```clj
(defrecord A [x])
```

And wished to deserialize it later with an updated definition:
```clj
(defrecord A [x y])
```

Cases can get more complicated than this of course. I will emphasis that my usage of Fressian did not require any strong support for this schema evolution feature. However, due to the extensibility of Fressian (that will be seen in examples below), this could be achieved. The examples I give will not attempt to support schema evolution for the sake of brevity.

As a very brief summary of the serialization formats I mentioned below:

* Kryo claims to be an efficient, binary format for serializing Java data. It has some support for schema evolution.
* Nippy claims (boldly) that it is the "The fastest serialization library for Clojure". I haven't tried it out recently enough to have an opinion. At the time that I worked on this, Nippy was in an earlier stage of development. It looks to have more use now. It does seem that it may be a good contender for use in Clojure serialization. It's extension mechanism looks similar to Fressian, so I'd expect it to have comparable characteristics in terms of features and schema evolution.
* Java serialization is not known for its speed. It also is typically thought of as a bloated format. It has some support for schema evolution. Clojure has many classes marked as Java Serializable, however, this is a brittle, field and type dependent sort of serialization. I don't believe it is intended to be a robust form of serialization beyond some simple data transport cases.
* Avro is a popular serialization format used frequently within the "big data" Hadoop ecosystem. I have had quite a bit of experience using it in the past. It is an efficient, binary format that also has facilities for schema evolution.
* Fressian claims to be an efficient, binary format with emphasis on extensibility and working with data *values*. There are implementations provided for Java and Clojure among others. The Clojure implementation will be the emphasis of this post. Schema evolution could be accomplished via the extension mechanism as well.

I ended up choosing Fressian as a default durability implementation in Clara. Now, I think that Nippy would be the closest second place contender due to it having similar benefits in terms of speed and extensibility mechanism.

I don't want to spend a lot of time discussing the tradeoffs of these various serialization formats in this post. Instead, I will focus on how I was able to utilize Fressian for Clojure data serialization across processes.

## Clojure serialization

When I say "Clojure serialization" I mean serializing Clojure data from some process to another process that deserializes the data back into the semantically equivalent Clojure data structures. In particular, this is Clojure to Clojure serialization. This is accomplished by extending Fressian appropriately on both the read (deserialization) and write (serialization) side.
The ideas here are not limited to only being applicable for Clojure to Clojure serialization. For example, the reader extensions could be done in an alternative language into semantically similar data structures of that language.

There are many reasons why this style of efficient, binary Clojure serialization maybe useful. One reason is to efficiently send Clojure data across processes, perhaps when distributing data across a distributed process space. Another is to persist Clojure data in and out of memory, such as retrieving it on a per-request basis in a web service.

## How Fressian works

I do not want to go too deep into all the inner workings of Fressian. I encourage looking at [the Fressian format documentation](https://github.com/Datomic/fressian/wiki) to understand it better.

I will only say here that Fressian serialization works by writing readers and writers to handle serialization and deserialization of data. We call these extension points the read and write handlers. This extension mechanism is very flexible. The extension is done (only by default) using object type-base dispatch that respects type hierarchies. Most of this can be replaced when appropriate.
The standard extension mechanism works fine for what I will describe below.

## The technical details

There are open source Clojure Fressian data reader and writer implementations available in the [org.clojure/data.fressian](https://github.com/clojure/data.fressian) repository. This is convenient and may make it seem like there is nothing else to discuss in this post because the implementation is already done.
Unfortunately, this implementation is incomplete in the data structures covered and lacking a few features that were critical to meet the needs that I had. Several of the Clojure data structures are deserialized (by default) back as Java (mutable) data structures, e.g. a Clojure persistent hash set deserializes as a java.util.HashSet. Even if they aren't subsequently mutated, this may cause confusion, e.g. for predicates like `set?`.

I think that these same missing handlers may apply to others' usage of Fressian as well. There may be good reasons why `org.clojure/data.fressian` doesn't provide some of these features out-of-the-box. I do not know more details on that.

For a "real world" example, an implementation of Fressian read and write handlers [1] can be seen [in the Clara repo here](https://github.com/cerner/clara-rules/blob/0.16.1/src/main/clojure/clara/rules/durability/fressian.clj). This is actively used as part of Clara's durability feature.

In this post, the specific Fressian handler implementations I will provide here are for:

* [Clojure metadata](https://clojure.org/reference/metadata)
* Persistent set
* Persistent sorted (tree) set
* Persistent sorted (tree) map
* Persistent vector
* Persistent list
* Persistent seq (as a fallback)
* java.lang.Class type

All of the code used below implementing the Fressian handlers can be [viewed in this gist](https://gist.github.com/mrrodriguez/5b681b191e96ceb137f768b0942415de).

#### A note on serializing functions

Serializing functions is a complex subject. It will not be covered here. A function isn't generally thought of as a "data structure" anyways, so isn't necessarily relevant to the current discussion. There may be a later post that discusses ways of serializing Clojure functions though, since it has came up for me, and I've seen it come up "in the wild", on a few occasions.

#### Default handlers 

First, let's look at using the default behavior from `org.clojure/data.fressian`. Pay attention the the example outputs:

```clj

(require '[clojure.data.fressian :as fres])

(defn serde-obj
  "Serializes and deserializes (aka 'serde') the object `obj`."
  [obj]
  ;; Serialize
  (with-open [baos (java.io.ByteArrayOutputStream.)
              wtr (fres/create-writer baos)]
    (fres/write-object wtr obj)

    ;; Deserialize
    (with-open [bais (java.io.ByteArrayInputStream. (.toByteArray baos))
                rdr (fres/create-reader bais)]
      (fres/read-object rdr))))

;;;; A few examples:

;;;; Persistent set

(serde-obj #{1 2 3})
;;= #{1 2 3}

(type (serde-obj #{1 2 3}))
;;= java.util.HashSet

(set? (serde-obj #{1 2 3}))

;;;; Persistent sorted (tree) set

(serde-obj (sorted-set 1 2 3))
;;= #{1 2 3}

(type (serde-obj (sorted-set 1 2 3)))
;;= java.util.HashSet

(set? (serde-obj (sorted-set 1 2 3)))
;;= false

(sorted? (serde-obj (sorted-set 1 2 3)))
;;= false

;;;; Lazy seq

(serde-obj (lazy-seq '(1)))
;;= []

(type (serde-obj (lazy-seq '(1))))
;;= java.util.Arrays$ArrayList

(seq? (serde-obj (lazy-seq '(1))))
;;= false

;;;; Metadata

(meta (serde-obj (with-meta #{1} {:hello :world})))
;;= nil

;;;; Java Class

(serde-obj String)
;; IllegalArgumentException Cannot write class java.lang.String as tag null  <...etc...>

```

Hopefully those examples motivate the reason why some may wish to have better support for "native" Clojure datastructures when deserializing in Clojure via Fressian.

#### The necessary custom handlers

In order to understand the implementations of Fressian handler, the general structure of a handler should be understood already. I will defer that explanation to [the wiki from `org.clojure/data.fressian`](https://github.com/clojure/data.fressian/wiki/Creating-custom-handlers).

To implement the necessary Fressian handlers to address the above concerns, first I'll start with some general helper functions.

In all examples, assume the following imports:

```clj
;;; NOTE: Avoid reflection by using type hints for performance.
;;; Serialization often is an area where performance is important.
(import '[org.fressian Writer Reader StreamingWriter])
```

First, a few functions to deal with metadata preservation:

```clj

(defn write-with-meta
  "Writes the object to the writer under the given tag.  If the record has metadata, the metadata
   will also be written. The optional `write-fn` will be used to write the object if given. If 
   not given, the default is Writer.writeList().
   `read-with-meta` (below) will associated this metadata back with the object when reading."
  ([w tag o]
   (write-with-meta w tag o (fn [^Writer w o] (.writeList w o))))
  ([^Writer w tag o write-fn]
   (let [m (meta o)]
     (do
       (.writeTag w tag 2)
       (write-fn w o)
       (if m
         (.writeObject w m)
         (.writeNull w))))))

(defn- read-meta [^Reader rdr]
  (some->> rdr
           .readObject
           (into {})))

(defn read-with-meta
  "Reads an object from the reader that was written via `write-with-meta` (above).  If the object
   was written with metadata the metadata will be associated on the object returned. The `build-fn`
   is called on the read object and is used to do any additional construnction necessary for 
   data structure."
  [^Reader rdr build-fn]
  (let [o (build-fn (.readObject rdr))
        m (read-meta rdr)]
    (cond-> o
      m (with-meta m))))

```

Second, a general map writer that is useful for maps and record types:

```clj

(defn write-map
  "Writes a map as Fressian with the tag 'map' and all keys cached."
  [^Writer w m]
  (.writeTag w "map" 1)
  (.beginClosedList ^StreamingWriter w)
  (reduce-kv
   (fn [^Writer w k v]
     (.writeObject w k true)
     (.writeObject w v))
   w
   m)
  (.endList ^StreamingWriter w))

```

Third, there are a few functions needed for dealing with Clojure record types, i.e. from `defrecord`.

The idea is to write a Clojure record as a map, while preserving its metadata and record type. The built-in Clojure "factory function" name of the form `map->MyRecord` is serialized along with the record. This is the function that is defined along with record definitions by the `defrecord` macro. When deserializing the record data, this name is used to resolve the factory function at runtime. It is then called on the deserialized map to return the correct record type.

Here are helpers to preserve the record factory function name:

```clj

(require '[clojure.main :as cm])

;;; Use this map to cache the symbol for the map->RecordNameHere factory function created for
;;; every Clojure record to improve serialization performance.
;;; See https://github.com/cerner/clara-rules/issues/245 for more extensive discussion.
(def ^:private ^java.util.Map class->factory-fn-sym
  (java.util.Collections/synchronizedMap
   (java.util.WeakHashMap.)))

(defn record-map-constructor-name
  "Return the 'map->' prefix, factory constructor function for a Clojure record."
  [rec]
  (let [klass (class rec)]
    (if-let [cached-sym (.get class->factory-fn-sym klass)]
      cached-sym
      (let [class-name (.getName ^Class klass)
            idx (.lastIndexOf class-name (int \.))
            ns-nom (.substring class-name 0 idx)
            nom (.substring class-name (inc idx))
            factory-fn-sym (symbol (str (cm/demunge ns-nom)
                                        "/map->"
                                        (cm/demunge nom)))]
        (.put class->factory-fn-sym klass factory-fn-sym)
        factory-fn-sym))))

```

NOTE: The read handler implementation for records expect the original record defining namespace(s) to be pre-loaded (e.g. via `require`) before deserializing the data. This assumption can be removed if the record factory function symbol is used to first ensure the namespace is required. I don't show that here just to keep it simpler.

```clj

(defn write-record
  "Same as `write-with-meta`, but with Clojure record support.  The type of the record will
   be preserved."
  [^Writer w tag rec]
  (let [m (meta rec)]
    (.writeTag w tag 3)
    (.writeObject w (record-map-constructor-name rec) true)
    (write-map w rec)
    (if m
      (.writeObject w m)
      (.writeNull w))))

(defn read-record
  "Same as `read-with-meta`, but with Clojure record support.  The type of the record will
   be preserved."
  [^Reader rdr]
  (let [builder (-> (.readObject rdr) resolve deref)
        build-map (.readObject rdr)
        m (read-meta rdr)]
    (cond-> (builder build-map)
      m (with-meta m))))

```

Forth, sorted (tree) sets and maps have one tricky part. When using `sorted-set` or `sorted-map`, the collection is sorted by a default comparator on the keys. This is sometimes called the "natural order". However, when a sorted collection type uses a custom comparator via `sorted-set-by` or `sorted-map-by`, the comparator is an function embedded on the object via some implementation detail. The Fressian handlers do not deal with serializing functions here. I mentioned above about them being a complex subject of their own.

Recall from above that the Clojure record type is preserved by serializing the name of the factory function that builds that record type. A similar approach can be done here. The only caveat is that a collection built via `sorted-set-by` or `sorted-map-by` may not have a named function associated with them. The proposed Fressian handlers here expect that these collections are given a named function in order to work around this limitation. For the sake of this discussion, the function that names the comparator used in the sorted collection can be placed in the metadata associated with the key `:fressian.custom/comparator-name`.

Then add the following functions to handle finding the comparator from a sorted collection type:

```clj

(defn sorted-comparator-name
  "Sorted collections are not easily serializable since they have an opaque function object instance
   associated with them.  To deal with that, the sorted collection can provide a 
   :fressian.custom/comparator-name in the metadata that indicates a symbolic name for the function 
   used as the comparator.  With this name the function can be looked up and associated to the
   sorted collection again during deserialization time.
   * If the sorted collection metadata has a :fressian.custom/comparator-name, then the symbol value is returned.
   * If the sorted collection has the clojure.lang.RT/DEFAULT_COMPARATOR, returns nil.
   * If neither of the above are true, an exception is thrown indicating that there is no way to provide a useful
     name for this sorted collection, so it won't be able to be serialized."
  [^clojure.lang.Sorted s]
  (let [cname (-> s meta :fressian.custom/comparator-name)]

    ;; Fail if reliable serialization of this sorted coll isn't possible.
    (when (and (not cname)
               (not= (.comparator s) clojure.lang.RT/DEFAULT_COMPARATOR))
      (throw (ex-info (str "Cannot serialize sorted collection with non-default"
                           " comparator because no :fressian.custom/comparator-name provided in metadata.")
                      {:sorted-coll s
                       :comparator (.comparator s)})))

    cname))

(defn seq->sorted-set
  "Helper to create a sorted set from a seq given an optional comparator."
  [s ^java.util.Comparator c]
  (if c
    (clojure.lang.PersistentTreeSet/create c (seq s))
    (clojure.lang.PersistentTreeSet/create (seq s))))

(defn seq->sorted-map
  "Helper to create a sorted map from a seq given an optional comparator."
  [s ^java.util.Comparator c]
  (if c
    (clojure.lang.PersistentTreeMap/create c ^clojure.lang.ISeq (sequence cat s))
    (clojure.lang.PersistentTreeMap/create ^clojure.lang.ISeq (sequence cat s))))

```

NOTE: The sorted collection functions have to rely on some Clojure implementation details, such as `clojure.lang.Sorted`, `clojure.lang.PersistentTreeSet/create`,  and `clojure.lang.RT/DEFAULT_COMPARATOR`. However, these do not often change and there is no direct API exposed. Also, this implementation has the same caveat as deserializing a record type. The namespace that the symbol is associated with needs to be pre-loaded before deserialization tries to resolve the symbol to find the function to call. The same way as in the record case to workaround this applies here as well.

With the the above functions in place, here are the Fressian write handlers:

```clj

(import '[org.fressian.handlers WriteHandler])

(def write-handlers
  {;; Persistent set
   clojure.lang.APersistentSet
   {"clj/set"
    (reify WriteHandler
      (write [_ w o]
        (write-with-meta w "clj/set" o)))}

   ;; Persistent sorted (tree) set
   {clojure.lang.PersistentTreeSet
    {"clj/treeset" (reify WriteHandler
                     (write [_ w o]
                       (let [cname (sorted-comparator-name o)]
                         (.writeTag w "clj/treeset" 3)
                         (if cname
                           (.writeObject w cname true)
                           (.writeNull w))
                         ;; Preserve metadata.
                         (if-let [m (meta o)]
                           (.writeObject w m)
                           (.writeNull w))
                         (.writeList w o))))}}

   ;; Persistent sorted (tree) map
   {clojure.lang.PersistentTreeMap
    {"clj/treemap" (reify WriteHandler
                     (write [_ w o]
                       (let [cname (sorted-comparator-name o)]
                         (.writeTag w "clj/treemap" 3)
                         (if cname
                           (.writeObject w cname true)
                           (.writeNull w))
                         ;; Preserve metadata.
                         (if-let [m (meta o)]
                           (.writeObject w m)
                           (.writeNull w))
                         (write-map w o))))}}

   ;; Persistent vector
   {clojure.lang.APersistentVector
    {"clj/vector" (reify WriteHandler
                    (write [_ w o]
                      (write-with-meta w "clj/vector" o)))}}

   ;; Persistent list
   ;; NOTE: The empty list is a different data type in Clojure and has to be handled separately.
   {clojure.lang.PersistentList
    {"clj/list" (reify WriteHandler
                  (write [_ w o]
                    (write-with-meta w "clj/list" o)))}}

   {clojure.lang.PersistentList$EmptyList
    {"clj/emptylist" (reify WriteHandler
                       (write [_ w o]
                         (let [m (meta o)]
                           (do
                             (.writeTag w "clj/emptylist" 1)
                             (if m
                               (.writeObject w m)
                               (.writeNull w))))))}}

   ;; Persistent seq & lazy seqs
   {clojure.lang.ASeq
    {"clj/aseq" (reify WriteHandler
                  (write [_ w o]
                    (write-with-meta w "clj/aseq" o)))}}

   {clojure.lang.LazySeq
    {"clj/lazyseq" (reify WriteHandler
                     (write [_ w o]
                       (write-with-meta w "clj/lazyseq" o)))}}


   ;; java.lang.Class type
   {Class
    {"java/class" (reify WriteHandler
                    (write [_ w c]
                      (.writeTag w "java/class" 1)
                      (.writeObject w (symbol (.getName ^Class c)) true)))}}})

```

Now, here are the Fressian read handlers:

```clj
(import '[org.fressian.handlers ReadHandler])

(def read-handlers
  {;; Persistent set
   "clj/set" (reify ReadHandler
               (read [_ rdr tag component-count]
                 (read-with-meta rdr set)))

   ;; Persistent sorted (tree) set
   "clj/treeset" (reify ReadHandler
                   (read [_ rdr tag component-count]
                     (let [c (some-> rdr .readObject resolve deref)
                           m (.readObject rdr)
                           s (-> (.readObject rdr)
                                 (seq->sorted-set c))]
                       (if m
                         (with-meta s m)
                         s))))
   
   
   ;; Persistent sorted (tree) map
   "clj/treemap" (reify ReadHandler
                   (read [_ rdr tag component-count]
                     (let [c (some-> rdr .readObject resolve deref)
                           m (.readObject rdr)
                           s (seq->sorted-map (.readObject rdr) c)]
                       (if m
                         (with-meta s m)
                         s))))
   
   ;; Persistent vector
   "clj/vector" (reify ReadHandler
                  (read [_ rdr tag component-count]
                    (read-with-meta rdr vec)))

   ;; Persistent list
   ;; NOTE: The empty list is a different data type in Clojure and has to be handled separately.
   "clj/list" (reify ReadHandler
                (read [_ rdr tag component-count]
                  (read-with-meta rdr #(apply list %))))

   "clj/emptylist" (reify ReadHandler
                     (read [_ rdr tag component-count]
                       (let [m (read-meta rdr)]
                         (cond-> '()
                           m (with-meta m)))))

   ;; Persistent seq & lazy seqs

   "clj/aseq" (reify ReadHandler
                (read [_ rdr tag component-count]
                  (read-with-meta rdr sequence)))

   "clj/lazyseq" (reify ReadHandler
                   (read [_ rdr tag component-count]
                     (read-with-meta rdr sequence)))
   
   ;; java.lang.Class type
   "java/class" (reify ReadHandler
                  (read [_ rdr tag component-count]
                    (resolve (.readObject rdr))))})

```

#### Results

Now let's use these read and write handlers to compare against the "Default handlers" demonstrated earlier:

```clj

(require '[clojure.data.fressian :as fres])

(def write-handler-lookup
  (-> write-handlers
      fres/associative-lookup
      fres/inheritance-lookup))

(def read-handler-lookup
  (fres/associative-lookup read-handlers))

(defn serde-obj
  "Serializes and deserializes (aka 'serde') the object `obj`."
  [obj]
  ;; Serialize
  (with-open [baos (java.io.ByteArrayOutputStream.)
              wtr (fres/create-writer baos :handlers write-handler-lookup)]
    (fres/write-object wtr obj)

    ;; Deserialize
    (with-open [bais (java.io.ByteArrayInputStream. (.toByteArray baos))
                rdr (fres/create-reader bais :handlers read-handler-lookup)]
      (fres/read-object rdr))))

;;;; A few examples:

;;;; Persistent set

(serde-obj #{1 2 3})
;;= #{1 2 3}

(type (serde-obj #{1 2 3}))
;;= clojure.lang.PersistentHashSet

(set? (serde-obj #{1 2 3}))
;;= true

;;;; Persistent sorted (tree) set

(serde-obj (sorted-set 1 2 3))
;;= #{1 2 3}

(type (serde-obj (sorted-set 1 2 3)))
;;= clojure.lang.PersistentTreeSet

(set? (serde-obj (sorted-set 1 2 3)))
;;= true

(sorted? (serde-obj (sorted-set 1 2 3)))
;;= true

;;;; Lazy seq

(serde-obj (lazy-seq ()))
;;= []

(type (serde-obj (lazy-seq '(1))))
;;= clojure.lang.LazySeq

(seq? (serde-obj (lazy-seq '(1))))
;;= true

;;;; Metadata

(meta (serde-obj (with-meta #{1} {:hello :world})))
;;= {:hello :world}

;;;; Java Class

(serde-obj String)
;;= java.lang.String

(type (serde-obj String))
;;= java.lang.Class

(class? (serde-obj String))
;;= true

```

## Conclusion

I hope that this may assist anyone who is interested in extending Fressian for similar purposes. Some of these ideas may apply to other serialization libraries as well if they have similar extension mechanisms.

### Notes

[1] A few helper functions were used here to reduce duplication in the reader/writer implementations.
