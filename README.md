# spec-forms
A Clojure(Script) library for creating human-readable error messages from [specs](https://clojure.org/guides/spec), with some helper functions for using those messages with [Reforms](https://github.com/bilus/reforms).

The current version depends on [spec-tools](https://github.com/metosin/spec-tools), but when spec-alpha2 is released, this dependency will be dropped and spec-forms will create Specs directly.

## Install

Add to deps.edn:
```
code204/spec-forms {:mvn/version "0.1.0-alpha2"}
reforms {:mvn/version "0.4.3"}
```

Or add to project.clj:
```
[code204/spec-forms "0.1.0-alpha2"]
[reforms "0.4.3"]
```

## Usage
### Spec Creation

First, you need some specs. You can define error messages inline with the `spec-forms.alpha/validator` macro, or you can use normal specs in conjunction with the [phrase](https://github.com/alexanderkiel/phrase) library. We'll start with the `validator` macro. It takes two arguments: a predicate function to be called on the value, and an error message for when the predicate returns false, e.g.,
```clojure
(require '[spec-forms.alpha :use validator])

(validator
  #(>= 16 (count %))
  "Must be 16 characters or less.")
```
The `validator` simply adds the message to the data returned by `clojure.spec.alpha/explain-data`, under the `:reason` key. If you happen to have existing code using `spec-tools` that returns a :reason, then you can use those specs with no changes.

An example `.cljc` file with a full spec definition:
```clojure
(ns sf-test
  (:require [clojure.spec.alpha :as s]
            [clojure.string :as str]
            [spec-forms.alpha :as sf])
  #?(:cljs
     (:require-macros [clojure.spec.alpha])))

(defn max-length [n]
  (sf/validator
    #(>= n (count %))
    (str "Must be " n " characters or less.")))

(defn min-length [n]
  (sf/validator
    #(<= n (count %))
    (str "Must be at least " n " characters long.")))

(def non-blank
  (sf/validator
    #(and % (not (str/blank? %)))
    "Must not be blank"))
    
(s/def ::login (s/and (min-length 2) (max-length 4) non-blank))
(s/def ::password (s/and (min-length 2) (max-length 4) non-blank))

(s/def ::login-form (s/keys :req [::login ::password]))
```

### Validation with clojure.spec

To validate on your own (rather than using reforms), just call `spec.alpha/explain` or `spec.alpha/explain-data` on any of the specs that you created earlier. `explain` prints the failure reason, while `explain-data` returns a vector of problems which each have a :reason key with the failure message.

```clojure
(in-ns 'sf-test)

(s/explain ::login "")
;; "" - failed: Must be at least 2 characters long. spec: :sf-test/login
;= nil

(s/explain-data ::login "")
;=#:clojure.spec.alpha{:problems
                       ({:path [],
                         :pred
                         (clojure.core/fn
                          [%]
                          (clojure.core/<= n (clojure.core/count %))),
                         :val "",
                         :via [:sf-test/login],
                         :in [],
                         :reason
                         "Must be at least 2 characters long."}),
                       :spec :sf-test/login,
                       :value ""}
```

### Form Validation

We use the [Reforms validation helpers](https://github.com/bilus/reforms#validation), but we use `spec-forms.alpha.validate!` instead of `reforms.core/validate!` The spec-forms `validate!` function takes four arguments: the data and ui-state atoms, the spec name, and an optional configuration map:
```clojure
(spec-forms.alpha/validate! data ui-state :sf-test/login-form
   {:default-error-message "Oops!"})
```
The configuration map supports three options:
* `:default-error-message` A message to return if no other message is found. Default: "Failed validation"
* `:message-fn` A 2-argument function to be called with the problem as returned by `clojure.spec.alpha/explain-data` and the configuration map. It should return a String. It's this function's responsibility to parse the `:reason` of the problem and return the `:default-error-messsage`.
* `:required-key-message` This is used when a key is not present in the form but is required, as when `clojure.spec.alpha/keys` is used. The validator for the key will not be called due to the design of Clojure Spec. It can be a function that will receive the missing key (if you'd like different messages for different keys), or a String used for all keys. Default: `"Required"`.

Reforms implements form data as a map, so we will need a map definition in our spec, such as `(s/def ::login-form (s/keys :req [::login ::password]))` in the Usage example earlier. `::login-form` is the spec name which we give as the third argument of `validate!`. `data` is an atom containing the form data map. `ui-state` is an atom (initially nil) that Reforms uses to keep track of validation errors. See the [Reforms docs](https://github.com/bilus/reforms#validation) for more information.

An example implementation using Rum (you'll need to add [rum-reforms](https://clojars.org/rum-reforms) as a dependency to run this):
```clojure
(ns sf-test.client
  (:require [clojure.spec.alpha :as s]
            [reforms.rum :include-macros true :as f]
            [reforms.validation :include-macros true :as v]
            [rum.core :as rum :refer (defcs)]
            [spec-forms.alpha :as sf]
            [sf-test]))

(defn login!
  [data ui-state]
  (when (sf/validate! data ui-state :sf-test/login-form)
    (js/alert "Logged in!")))

(defcs login-form
  < (rum/local nil ::ui-state)
  [state data]
  (let [comp (:rum/react-component state)
        ui-state (::ui-state state)]
    (v/form
      ui-state
      (v/text "Login" data [:sf-test/login])
      (v/password "Password" data [:sf-test/password])
      (f/form-buttons
        (f/button-primary "Log In"
          #(do (login! data ui-state)
               (rum/request-render comp)))))))

(defn mount-form []
  (rum/mount
   (login-form (atom nil))
   (js/document.getElementById "app")))

(defn init []
  (js/console.log "init")
  (mount-form))
```

### Using with phrase

It's very simple to use [phrase](https://github.com/alexanderkiel/phrase) with spec-forms. We just need to create a custom message-fn and pass it as an option to `validate!`:

```clojure
(require 'phrase.alpha 'spec-forms.alpha)

(defn spec-problem->message [{:keys [reason val via]} opts]
  (or reason
    (phrase.alpha/phrase-first {} (last via) val)
    (:default-failure-message opts "Failed validation")))
```

Wherever you call `validate!`:
```clojure
(spec-forms.alpha/validate! data ui-state :your-spec-ns/your-spec
  {:message-fn spec-problem->message})
```
