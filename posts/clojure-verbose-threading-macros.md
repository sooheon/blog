---
title: Verbose Clojure Threading Macros
date: 2020-04-12
tags: 
 - Clojure
---

I recently read [this](https://findka.com/blog/rec-sys-in-30-lines/) blog post on a collaborative filtering algorithm in 30 lines of Clojure. The author's way of annotating a threading macro's intermediate evaluations with inline comments was nice, and something I've done repeatedly for myself in debugging. I've never quite been able to get this behavior out of the Cider and Cursive debuggers (maybe because they deal with the already macroexpanded forms?). To evaluate such forms interactively, I usually delete or comment out the trailing forms, adding back one at a time.

```clojure
(->> [1 2 3]
  (map inc)
  ;(apply *)
  ;(repeat 5)
  ;(reduce /)
  )
  ;; uncomment one line at a time
```

As an exercise in macros, which I have almost no experience with, I took a stab at creating verbose versions of the threading macros. First, checking the source code for some `clojure.core` macros...

```clojure
(defmacro ->>
  "Threads the expr through the forms. Inserts x as the
  last item in the first form, making a list of it if it is not a
  list already. If there are more forms, inserts the first form as the
  last item in second form, etc."
  {:added "1.1"}
  [x & forms]
  (loop [x x, forms forms]
    (if forms
      (let [form (first forms)
            threaded (if (seq? form)
              (with-meta `(~(first form) ~@(next form)  ~x) (meta form))
              (list form x))]
        (recur threaded (next forms)))
      x)))
```

Immediately, the different types of quoting and unquoting scare me off. Let's try something easier first.

```clojure
(defmacro when
  "Evaluates test. If logical true, evaluates body in an implicit do."
  {:added "1.0"}
  [test & body]
  (list 'if test (cons 'do body)))

(macroexpand '(when true 
                (println "true!")
                :true))
;; => (if true (do (println "true!") :true))
```

When you use the `when` macro in your code, it takes a boolean and a variable
number of forms as a `body`, constructs a list that looks just like an `if` form
(because it is one). At eval time, it's this macroexpanded `if` sexp that is
actually run.

Coming back to the `->>` macro, you can see there's a `loop...recur` in there,
where the the construction of the nested lists takes place. The terminating
condition is when `forms` is empty and `x` consists of the fully nested form. I
wanted print statements at every loop, so my first try looked like this:

```clojure
(defmacro verbose->>
  [x & forms]
  (loop [x x forms forms]
    (if forms
      (let [form (first forms)
            threaded (if (seq? form)
                       (with-meta `(~(first form) ~@(next form) ~x) (meta form))
                       (list form x))]
        (do 
          (println "x:" x)
          (println "threaded:" threaded)
          (println "=>" (eval threaded)))
        (recur threaded (next forms)))
      x)))

(verbose->> [1 2 3]
  (map inc)
  (apply *)
  (repeat 4)
  (reduce /))

;; x: [1 2 3]
;; threaded: (map inc [1 2 3])
;; => (2 3 4)
;; x: (map inc [1 2 3])
;; threaded: (apply * (map inc [1 2 3]))
;; => 24
;; x: (apply * (map inc [1 2 3]))
;; threaded: (repeat 4 (apply * (map inc [1 2 3])))
;; => (24 24 24 24)
;; x: (repeat 4 (apply * (map inc [1 2 3])))
;; threaded: (reduce / (repeat 4 (apply * (map inc [1 2 3]))))
;; => 1/576
```

It works! However, there's an obvious bug here, which you'd see right away if
you're familiar with how macros work: my logging code evaluates at compile time,
during macroexpansion, not at runtime. Running my code in the REPL is a poor
test, because it doesn't distinguish macroexpansion and run time evaluation.
Separating compile time and run time, I can see that my logging code tries to
run even when I'm just defining a function, and the argument symbol is being
passed to to `verbose->>`. Of course, that symbol has no value at compile time,
and it returns an error.

```clojure
(defn do-stuff [nums]
  (verbose->> nums
    (map inc)
    (apply *)
    (repeat 4)
    (reduce /)))

;; x: nums
;; threaded: (map inc nums)
;; Unexpected error (NoSuchMethodException) macroexpanding verbose->> at (src/collaborative_filtering.clj:2:3).
;; collaborative_filtering$do_stuff$eval1601__1602.<init>()
```

Time to change the approach. I don't want any `print`s to be run at compile
time, so lets construct an intermediate collection of printing fns, and return a
final form that will `doseq` over the collection and finally return the original
nested form.

```clojure
(defmacro verbose->>
  [x & forms]
  (loop [x x
         forms forms
         intermediate []]
    (if forms
      (let [form (first forms)
            threaded (if (seq? form)
                       (with-meta `(~(first form) ~@(next form) ~x) (meta form))
                       (list form x))]
        (recur
          threaded
          (next forms)
          (conj intermediate `(do
                                (println "x: ")
                                (pp/pprint ~x)
                                (println "\nthreaded: ")
                                (pp/pprint '~threaded)
                                (println "\n=> ")
                                (pp/pprint ~threaded)
                                (println "\n-------")))))
      (list 'do
        `(doseq [i# ~intermediate]
           i#)
        x))))
```

First, the backtick is the same as the `quote` function, except that it allows
those unquotes to be used inside. `~x` unquotes x, allowing it to be evaluated
and returning the value bound to the symbol x at compile time. `'~threaded` will
return whatever is bound to `threaded`, without evaluating it, and `~threaded`
will return the evaluated value. I added printing because intermediate results
can get pretty long.

Same deal with `~intermediate` as I want the actual value of the collection to
iterate over, and finally, `i#`, which ensures that when Clojure gives an
internal name to the `i` binding (like `i__1467__auto__`, which you can see when
macroexpanding), it remains consistent.

```clojure
(let [alice {:a :like
	         :b :like
	         :c :dislike
	         :d :like
	         :e :like}
      bob {:a :like
           :b :dislike
           :c :dislike
           :d :dislike
           :e :like}
      carol {:a :like
	         :b :dislike}
      users [alice bob carol]
      explore (distinct (mapcat keys users))
      cooc (cooccurrence users)
      score (fn [{:keys [like dislike]
      			  :or {like 0 dislike 0}}]
      		 (/ (+ like 1) (+ like dislike 2)))]
  (recommend
    {:n 2
     :cooc cooc
     :user-ratings carol
     :score score
     :explore explore}))

;; x:
;; {:a :like, :b :dislike}
;; 
;; threaded:
;; (map cooc user-ratings)
;; 
;; =>
;; ({:b {:like 1, :dislike 2},
;;   :c {:dislike 2},
;;   :d {:like 1, :dislike 1},
;;   :e {:like 2}}
;;  {:a {:like 2}, :c {:dislike 1}, :d {:dislike 1}, :e {:like 1}})
;; 
;; -------
;; x:
;; ({:b {:like 1, :dislike 2},
;;   :c {:dislike 2},
;;   :d {:like 1, :dislike 1},
;;   :e {:like 2}}
;;  {:a {:like 2}, :c {:dislike 1}, :d {:dislike 1}, :e {:like 1}})
;; 
;; threaded:
;; (apply
;;   merge-with
;;   (fn* [p1__1801# p2__1802#] (merge-with + p1__1801# p2__1802#))
;;   (map cooc user-ratings))
;; 
;; =>
;; {:b {:like 1, :dislike 2},
;;  :c {:dislike 3},
;;  :d {:like 1, :dislike 2},
;;  :e {:like 3},
;;  :a {:like 2}}
;; 
;; -------
;;
;; [...]
```

A little verbose, but it does what I wanted it to! Obviously it does a lot of
unnecessary work, computing the result from scratch at every iteration, rather
than saving the intermediate results. I'll leave that improvement for another
day, but I quite enjoyed demystifying the different macro unquoting styles while
making a simple helper fn.
