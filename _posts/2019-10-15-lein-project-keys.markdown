---
layout: post
title: "Leiningen Task Argument Splicing"
author: Mike Rodriguez
date: 2019-10-15
tags: [Leiningen, lein, Clojure, ClojureScript, clj, cljs, task, alias, aliases, splice, splicing, project, development, programming, Java]
---


## Background

As mentioned in the [Leiningen sample project.clj](https://github.com/technomancy/leiningen/blob/2.9.1/sample.project.clj#L231-L233) (version 2.9.1 shown here, but it has been this way for quite some time), there is explicit support for "splicing in" task argument values from the project map. This can be a useful and dynamic feature in some cases.

As an example, consider specifying a common task template that is generic across projects:

```clj
:aliases {"some-task" ["do-something" :project/name :project/version]}
```

Depending on what project directory this task `lein some-task` is ran from, `lein` will automatically fill in the name and version from the `(defproject org.something/project "1.0.0" ...)` declaration in the `project.clj`.

This can be made more useful in using something like [`lein-parent`](https://github.com/achin/lein-parent):

In the "child" project:

```clj
:plugins [[lein-parent "0.3.7"]]
:parent-project {:coords [org.something/my-parent "1.0.0"]
                 :inherit [:aliases]}
```

Where now you could call `lein some-task` from the child project, and the `:project/name` and `:project/version` would be set to the child project information automatically.

## Going deeper

The examples above mentioned the somewhat "special" project keys, `:project/name` and `:project/version`. These are automatically added by `lein` when it initializes the project map. However, you can add arbitrary keys of your own to use for the same purpose:

```clj
(defproject org.something/project "1.0.0"
  :some-value "hello-world"
  :aliases {"some-task" ["do-something" :project/some-value]}
  ...)
```
With this, it would then be possible to run a task on this project, such as `lein some-task`, and the `:project/some-value` will be automatically replaced with it's value `"hello-world"`.

## And deeper

Recently, I discovered a "feature" that's been in [Leiningen for a while now](https://github.com/technomancy/leiningen/commit/c2b40834e0b3c60105d7204e81232c270e9a589e) (it's been modified from this original commit over time, but the basic idea remains). I want to mention it here only because I cannot find much information on it via internet searches and perhaps someone will find it valuable.

It seems that any project map key that has a name that matches the regex `#"-path$"` (singular) or `#"-paths$"` (plural) will automatically be considered to be a string value representing a file path. It will be automatically made absolute by resolving it relative to the current project. This can be seen in this fn:

```clj
(defn- absolutize-path [{:keys [root] :as project} key]
  (cond (re-find #"-path$" (name key))
        (update project key (partial absolutize root))

        (re-find #"-paths$" (name key))
        (update project key #(with-meta* (map (partial absolutize root) %)
                               (meta %)))

        :else project))
```

Now consider having a parent project:
```clj
(defproject org.something/parent-project "1.0.0"
  :some-path "stuff"
  :aliases {"some-task" ["do-something" :project/some-path]}
  ...)
```

Then a child:
```clj
(defproject org.something/child-project "1.0.0"
  :plugins [[lein-parent "0.3.7"]]
  :parent-project {:coords [org.something/my-parent "1.0.0"]
                   :inherit [:some-path :aliases]}
  ...)
```

When doing `lein some-task` in the child, the value of `:project/some-path` will automatically become the value `"/my/full/path/to/child/stuff"`, rather than just `"stuff"`. This can be useful in some situations.


## Conclusion

I plan to write a bit more on Leiningen things I've found over time that I do not often find documented anywhere. Stay tuned.
