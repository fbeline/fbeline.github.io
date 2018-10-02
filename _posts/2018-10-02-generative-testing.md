---
title: Generative testing
description: Generative testing with Clojure.
date: 2018-10-02 00:44:00
---

The generative testing idea was initially implemented in a Haskell library called QuickCheck built by John Hughes and Koen Claessen at 1999. You can find the original paper [here](http://www.cs.tufts.edu/~nr/cs257/archive/john-hughes/quick.pdf ).

---

In the generative testing approach, the programmer provides specifications that a function should satisfy; then the test runner randomly generates test cases that attempts to falsify the provided specification. Once such a test is found,  the input data is simplified to a minimal failing subset. (The process of reducing the failing input to a minimal subset is also known as shrinking).

Why does it matter?

When writing tests for a function you usually create no more than 3 to 5 different tests for it, but with generative testing, you can quickly generate thousands, and that exploratory behavior tends to find unpredicted scenarios. Also, is more comfortable to auto-generate tests than write them, just imagine how many input variations some functions could assume: negative, zero, empty, nil, one, many, etc.

The following examples will use the Clojure library [test.check](https://github.com/clojure/test.check), see their website for more information.

## Caesar cipher

To illustrate the usage of generative testing and how powerful it is, we will implement the [Caesar cipher](https://en.wikipedia.org/wiki/Caesar_cipher) encryption.

```clojure
(defn caesar-cypher [op text shift]
  (->> text
       (mapv #(-> % int (op shift) (mod 123) char))
       (apply str)))

(def encrypt (partial caesar-cypher +))
(def decrypt (partial caesar-cypher -))
```

Now let's define the properties that our functions should hold. In that case, the decryption of an encrypted message should match the original text.

```clojure
(def caesar-cipher-property
  (prop/for-all [v gen/string-alphanumeric
                 n gen/int]
    (= v (-> v (encrypt n) (decrypt n)))))
```

As you can see we have two generators, one to the message (alphanumeric string) and one to the shift (int).

```clojure
(tc/quick-check 1000 caesar-cipher-property)
=> {:result true, :pass? true, :num-tests 1000, :time-elapsed-ms 93, :seed 1538443015722}
```

The result shows to us that 1000 random tests were made and all of them passed. But if we change our text generator to be a string and not be limited to alphanumeric values, we should see an error as the implementation is not ready for special characters.

```clojure
(def caesar-cipher-property
  (prop/for-all [v gen/string
                 n gen/int]
    (= v (-> v (encrypt n) (decrypt n)))))

(tc/quick-check 1000 caesar-cipher-property)
=>
{:shrunk {:total-nodes-visited 44,
          :depth 6,
          :pass? false,
          :result false,
          :result-data nil,
          :time-shrinking-ms 2,
          :smallest ["" 0]},
 :failed-after-ms 0,
 :num-tests 5,
 :seed 1538443511706,
 :fail ["Ç" 3],
 :result false,
 :result-data nil,
 :failing-size 4,
 :pass? false}
```

After 5 test cases the property specification was invalidated with the input `["Ç" 3]`. The minimal failing subset is demonstrated inside `:shrunk`.

So let's change it to accept all the ASCII table values and test it again.

```clojure
(defn caesar-cypher [op text shift]
  (->> text
       (mapv #(-> % int (op shift) (mod 256) char))
       (apply str)))

(def encrypt (partial caesar-cypher +))
(def decrypt (partial caesar-cypher -))

(def caesar-cipher-property
     (prop/for-all [v gen/string
                    n gen/int]
                  (= v (-> v (encrypt n) (decrypt n)))))


(tc/quick-check 1000 caesar-cipher-property)
=> {:result true, :pass? true, :num-tests 1000, :time-elapsed-ms 63, :seed 1538444040938}
```

Now the implementation is working for any valid character.

## Flatten

In order to give another example, we will test the `clojure.core/flatten` function. To do so, we specify a generator that creates nested lists of Integers.

```clojure
(def nested-lists-gen (gen/list (gen/list gen/int)))

;; generating 3 random samples 
(gen/sample nested-lists-gen 3)
=> (() ((-1)) (()))
```

With the generator in hands, we define the property spec, asserting that after test.check apply the generated value to `flatten` it returns a list of Integers.

```clojure
(def flatten-property
  (prop/for-all [v nested-lists-gen]
    (let [r (flatten v)]
      (every? integer? r))))

(tc/quick-check 100 flatten-property)
=> {:result true, :pass? true, :num-tests 100, :time-elapsed-ms 109, :seed 1538445795665}
```

As expected the `flatten` function property holds the specification.

## Going further
The test.check has much more complex properties than those you've seen here, feel free to extend the examples and try it by yourself.

Some sources for further reading:

- [design-use-quickcheck](https://begriffs.com/posts/2017-01-14-design-use-quickcheck.html)
- [Testing the Hard Stuff and Staying Sane](https://www.youtube.com/watch?v=zi0rHwfiX1Q)

