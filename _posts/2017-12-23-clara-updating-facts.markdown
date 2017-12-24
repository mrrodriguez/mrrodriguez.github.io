---
layout: post
title: "Updating facts in Clara Rules"
author: Mike Rodriguez
date: 2017-12-23
tags: [Rules, Rules engines, Rule engine, Rete, Clara, Clojure, ClojureScript, Java, development, best practices, programming]
---


## A common question

I've seen the question come up several times concerning how to "update" or "modify" facts within working memory. The external system (or person) that is feeding facts into the rule session's working memory may need to "make changes" to facts that have previously been inserted into the working memory.

Clara rules does not have any direct constructs for expressing this. However, there are several ways to deal with this issue. These will be described in this post.

### First, why it is difficult?

By default, fact insertion in Clara is done by what is typically called a "logical" insert. The rules engine has what is called a Truth Maintenance System, the TMS. When a fact is inserted logically, it means that the fact is tracked by the TMS. For a logically inserted fact, the TMS has memory of the facts that made the conditions of the rule satisfied that caused the insertion. If any of these conditions later become unsatisfied by changes to working memory state, the logically inserted fact is automatically retracted. I have [explained this in more depth in an earlier post](http://www.metasimple.org/2017/02/28/clarifying-rules-engines.html) for those interested in more details.

The TMS and logical inserts are useful because they allow the rules to be expressed in a declarative way, i.e. free from depending on the order that rules are evaluated and fired. This is useful for several reasons.

Here are just a few of these reasons:
* Maintaining the rules as the system grows
* Avoiding an imperative style thought process to reason about rules
* Allowing the rules engine to optimize evaluation in an efficient order


Using logical inserts results in the rules engine managing fact retraction automatically. It is rare that there is a need to explicitly retract facts in rule logic. It is often even difficult to do so correctly, due to how it may interfere with the TMS. This directly affects the problem described in this post.

Having said this, consider this attempt to "update" a fact with a basic rule:

```clj

(ns mikerod.clara.updating.facts
  (:require [clara.rules :as r]))

(defrecord UpdatingFact [id timestamp value])

(r/defrule updating-fact-bad-idea
  [?old <- UpdatingFact (= ?id id) (= ?ts timestamp)]
  [?new <- UpdatingFact (= ?id id) (< ?ts timestamp)]
  =>
  (r/retract! ?old)
  (r/insert! ?new))

(r/defquery find-fact []
  [?fact <- UpdatingFact])

```

This doesn't logically make sense. The fact updated bound to `?new` is logically inserted. It relies on the fact bound to `?old` to exist in order for the rule `updating-fact-bad-idea` to be satisfied. The `r/retract!` of `?old` invalidates this logical insertion. There are several existing issues [found here](https://github.com/cerner/clara-rules/issues/229) and [also here](https://github.com/cerner/clara-rules/issues/321) surrounding the TMS when it comes to performing `r/retract!`'s that immediately invalidate the `r/insert!` done in the same rule evaluation cycle.

Even if the `r/retract!` issues are resolved, the problem remains that this doesn't logically make sense to manage the fact update this way.

It is desirable to find an approach that can work in harmony with the TMS. That is what will be explored below.

### Approach 1: Append only updates

In some situations, it may be appropriate to not be concerned with ever needing to remove facts from working memory. This means that an update to an existing fact can be modeled as a new fact entirely. Newly inserted facts can have some way relate to "older versions" of facts that are conceptually related, e.g. via an unique identifier. The rules that rely on this type of fact, can chose to explicitly depend on the "most recent" if that is the desired semantics.

This approach is similar to the idea of an "append only" database. Facts are never removed, instead only new facts are added. A new fact could even be made to express the removal of an existing fact. 

The working memory of a rules session is likely being held within the memory of a single process. This means that if the process is long-lived and the session is continually updated with these new facts, then memory limitations may make this append only approach infeasible. However, there may be times when the session's working memory state has a finite lifetime making this approach is a good fit.

This pattern can be seen in the following example:

```clj

(ns mikerod.clara.updating.facts
  (:require [clara.rules :as r]
            [clara.rules.accumulators :as acc]))

(defrecord UpdatingFactSnapshot [id timestamp value])
(defrecord UpdatingFact [id timestamp value])

(defrecord OtherFact [id timestamp value])
(defrecord Result [value])

(r/defrule find-newest-updating-fact-snapshot
  [?newest <- (acc/max :timestamp :returns-fact true) :from [UpdatingFactSnapshot]]
  =>
  (r/insert! (map->UpdatingFact ?newest)))

(r/defrule uses-updating-fact-type
  [OtherFact (= ?v value)]
  [UpdatingFact (= ?v value)]
  =>
  (r/insert! (->Result ?v)))

(r/defquery find-results
  []
  [?r <- Result])

(def base-session (r/mk-session 'mikerod.clara.updating.facts))

(defn run-rules [session facts]
  (let [s (-> session
              (r/insert-all facts)
              (r/fire-rules))

        res (mapv :?r (r/query s find-results))]
    {:session s
     :results res}))

(def run1
  (run-rules base-session
             [(->OtherFact 1 1 1)
              (->OtherFact 2 1 100)

              (->UpdatingFactSnapshot 3 1 1)]))

(def run2
  (run-rules (:session run1)
             [(->UpdatingFactSnapshot 3 2 100)]))

(:results run1)
;;= [#mikerod.clara.updating.facts.Result{:value 1}]

(:results run2)
;;= [#mikerod.clara.updating.facts.Result{:value 100}]

```

All facts that need to be updated can be inserted as a "snapshot". A rule can then process the "history" of these facts over time and produce the newest, oldest, etc.

### Approach 2: Externally managed updates

When "Approach 1" does not work due to something like memory limitations, updating facts can be managed by an external process rather than to try to implement it within the rules logic.

This pattern can be seen in the following example:

```clj

(ns mikerod.clara.updating.facts
  (:require [clara.rules :as r]
            [clara.rules.accumulators :as acc]))

(defrecord UpdatingFact [id timestamp value])

(defrecord OtherFact [id timestamp value])
(defrecord Result [value])

(r/defrule uses-updating-fact-type
  [OtherFact (= ?v value)]
  [UpdatingFact (= ?v value)]
  =>
  (r/insert! (->Result ?v)))

(r/defquery find-updating-fact [:?id]
  [?fact <- UpdatingFact (= ?id id)])

(r/defquery find-results
  []
  [?r <- Result])

(def base-session (r/mk-session 'mikerod.clara.updating.facts))

(defn run-rules [session facts]
  (let [s (-> session
              (r/insert-all facts)
              (r/fire-rules))

        res (mapv :?r (r/query s find-results))]
    {:session s
     :results res}))

(defn query-id-matches [session updates]
  (into []
        (comp (mapcat #(-> session
                           (r/query find-updating-fact :?id (:id %))))
              (map :?fact))
        updates))

(defn update-facts [session updates]
  (let [facts (query-id-matches session updates)
        s (-> (apply r/retract session facts) ; `r/retract-all` would be good to add to the API
              (r/insert-all updates)
              r/fire-rules)
        res (mapv :?r (r/query s find-results))]
    {:session s
     :results res}))

(def run1
  (run-rules base-session
             [(->OtherFact 1 1 1)
              (->OtherFact 2 1 100)

              (->UpdatingFact 3 1 1)]))

(def run2
  (-> run1
      :session
      (update-facts [(->UpdatingFact 3 2 100)])))

(:results run1)
;;= [#mikerod.clara.updating.facts.Result{:value 1}]

(:results run2)
;;= [#mikerod.clara.updating.facts.Result{:value 100}]

```

The external process in this case would interact with a rules session using `update-facts`. This only handles fact updates and not fact retractions. However, this could easily be added by giving another argument to `update-facts`, such as `retracts` where the `retracts` matched facts to be retracted only with no corresponding `r/insert-all`.

The advantages of "Approach 2" is that a session can be long-lived in a single process memory space. It also allows the rule logic to be simplified to not manage changes to facts. If the rules are written to take full advantage of the TMS via logical fact insertions, the rules network will (efficiently) update the state of all rules affected by updates to the session coming from this external process.

Another potential benefit to this approach is that the external process could chose to store the history of states the rule session working memory was in from one update to the next. In the example above, notice that both `run1` and `run2` can be examined independently. This allows for the external process to have visibility to changes over time. This reification of the history of the working memory state may be useful for "replay" or "undo" sort of operations.

This is an advantage that comes from Clara's persistent immutable working memory.

## The future

There may still be compelling reasons for Clara rules to have more direct support for expressing fact updates. Some further thought needs to be put into this. Perhaps it will be possible later to have an explicit `r/update!` and/or `r/update` function provided by Clara that allows for facts to be explicitly updated in a way that is correctly coordinated with the TMS.

Along with added expressivity, another compelling reason to pursue this may be to allow the rules engine to have a way to optimize this operation that isn't possible when it is not explicitly expressed.

That's a subject for a later post.
