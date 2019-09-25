---
title: Parallelizing sequence fns with core.async
tags:
  - Clojure
  - Datomic
  - AWS
layout: layouts/post.njk
---

Recently I built a system that periodically hit a lot of web APIs,
transformed the data, and then mass upserted to a database (sound
familiar, anyone?). Both the data ingest and DB communications (to
Datomic Cloud in this case) were I/O bound, and I did a lot of waiting
at the REPL developing the pipeline.

This was bearable, but the processing time became a problem in
production, when I set up the pipeline as a lambda (with Datomic
[Ions]) to be triggered with AWS CloudWatch events. Whatever ran in a
lambda ion could not take too long to complete.

```
[scheduled CW event] -> [task lambda ion] -> [a ton of I/O bound processes]
                                ^                           V
                                 \--timeout before return--/
```

To make the lambda return immediately and process asynchronously, I
used the [pipeline] method recommended by Cognitect, but thought it
was a bit clunky to be generalized outside of this specific context.

Soon after, [this] post by `joinr` on reddit showed me a more succint
example of core.async usage. His `pmap!` function used the same
mechanism of `core.async/pipeline-blocking` to preserve collection
order, but simplified the plumbing with `core.async/to-chan` and took
arbitrary `f` and `xs` as inputs just like `map`. I realized that it
was just one more abstraction away from taking any function that
returns a transducer in place of `map`.

```lisp
(defn parallelize
  "Given a `fnc` capable of creating a transducer when called with 1-arity
   such as `map`, `filter`, or `mapcat`, returns a fn which runs given `fnc` in
   parallel."
  [fnc]
  (fn parallel-fn
    ([concurrent f xs]
     (let [output-chan (a/chan)]
       (a/pipeline-blocking concurrent
                            output-chan
                            (fnc f)
                            (a/to-chan xs))
       (a/<!! (a/into [] output-chan))))
    ([f xs] (parallel-fn (.availableProcessors (Runtime/getRuntime)) f xs))))

(def pmap! (parallelize map))
(def pfilter! (parallelize filter))
(def pmapcat! (parallelize mapcat))
```

These three functions ended up saving me a ton of time building and
testing I/O heavy pipelines, and inspired me to start curating a
proper personal utils lib. Now, I'm enjoying reading through
[sbelak/huri], [plumatic/plumbing], [jacobobryant/trident] and the
like. Please don't hesitate to email me with any other nice utils
libraries.

[pipeline]: https://docs.datomic.com/cloud/best.html#pipeline-transactions
[this]: https://old.reddit.com/r/Clojure/comments/cwgvvi/clojure_vs_blub_lang_parallelism/eybc21c/
[Ions]: https://docs.datomic.com/cloud/ions/ions-tutorial.html#add-item-lambda
[plumatic/plumbing]: https://github.com/plumatic/plumbing
[sbelak/huri]: https://github.com/sbelak/huri
[jacobobryant/trident]: https://github.com/jacobobryant/trident/blob/master/src/trident/util/datomic.cljc