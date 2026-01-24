---
name: coding-standards-clojure
description: Pure Clojure/ClojureScript language coding standards focusing on functional programming, immutability, and idiomatic patterns.
---

# Clojure/ClojureScript Coding Standards

Pure language standards for Clojure and ClojureScript development.

## Part 1: Core Language Idioms

### Naming Conventions

```clojure
;; ✅ GOOD: Kebab-case, predicates end with ?, side effects with !, conversions with ->
(defn calculate-total-price [items] (reduce + (map :item/price items)))
(defn active? [user] (:user/active user))
(defn save-to-db! [data] ...)
(defn user->summary [user] (select-keys user [:user/id :user/name]))
(def ^:dynamic *database* nil)

;; ❌ BAD: camelCase, Java-style predicates
(defn calculateTotalPrice [items] ...)
(defn is-palindrome [s] ...)
```

**Keyword Namespacing**:
- Internal domain data MUST use explicit namespaced keywords (`:user/id`, `:order/total`)
- Auto-resolved keywords (`::id`) are ONLY used in spec definitions
- Generic keywords (`:id`, `:name`) are only acceptable for external data at boundaries

### Namespaced Keywords

```clojure
;; ✅ GOOD: Explicit namespaced keywords for domain data
(def user {:user/id "123" :user/name "Alice" :user/email "alice@example.com"})

;; ✅ GOOD: Convert at boundaries
(defn api-response->user [json-data]
  {:user/id (get json-data "id")
   :user/name (get json-data "name")
   :user/email (get json-data "email")})

;; ❌ BAD: Generic keywords for internal data
{:id "123" :name "Alice"}  ; Which domain's ID? Collision-prone
```

### Data Structures & Destructuring

```clojure
;; Vectors for sequences, maps for key-value, sets for unique
(def users ["Alice" "Bob"])

;; Destructuring with namespaced keywords
(defn display-user [{:user/keys [name email]}] (str name " <" email ">"))

;; Destructuring at boundaries (external data with generic keywords)
(defn greet [external-data]
  (let [{:keys [name age]} external-data]  ; Boundary: external format
    (str "Hello " name ", age " age)))
```

### Threading Macros

```clojure
;; ✅ GOOD: -> for first position, ->> for last position (collections)
(-> user (assoc :user/name "Alice") (update :user/age inc))
(->> items (filter :item/active) (map :item/price) (reduce +))
(some-> user :user/address :address/postal-code (subs 0 3))

;; ✅ GOOD: cond-> for conditional transformations
(defn apply-user-rules [user tier]
  (cond-> user
    (= tier :tier/premium) (assoc :user/discount 0.1)
    (some? (:user/email user)) (assoc :user/email-verified true)
    (> (:user/age user 0) 18) (assoc :user/adult true)))

;; ❌ BAD: Wrong threading macro
(-> items (filter :item/active) (map :item/price))  ; filter/map expect coll as last arg
```

### Conditionals & Predicates

```clojure
;; Complete predicate calls
(when-not (empty? coll) (process coll))
(if-let [result (fetch-data id)] (process result) (handle-error))

;; cond for multiple conditions, case for value matching
(cond
  (empty? items) "No items"
  (< (count items) 10) "Few items"
  :else "Many items")

(case (:event/type event)
  :event/user-created (handle-created event)
  :event/user-updated (handle-updated event)
  (log-unknown-event event))
```

### nil, Empty Collections, and Existence Checks

**Critical distinction**: `contains?` checks for KEY existence, NOT value truthiness.

```clojure
;; ✅ GOOD: Use some? to check if a value is non-nil
(when (some? (:user/email user))
  (send-email (:user/email user)))

;; ✅ GOOD: Use contains? ONLY to check if a key exists
(when (contains? user :user/email)
  (println "Email key exists (value may be nil)"))

;; ⚠️ COMMON MISTAKE: contains? does NOT check values
(contains? {:user/email nil} :user/email)  ; => true (key exists!)

;; Maps check KEY, sets check element, vectors check INDEX
(contains? [10 20 30] 1)  ; => true (index 1 valid)
(contains? [10 20 30] 10)  ; => false (10 is not an index!)

;; ✅ GOOD: Policy
(when (seq items) (process-items items))  ; seq for truthiness
(or (not-empty filtered-items) default-items)  ; not-empty returns coll or nil
(if (some? result) (handle-result result) (handle-missing))  ; explicit non-nil
```

### Idiomatic Patterns

```clojure
;; Numeric predicates, comparisons
(inc x) (dec x) (pos? x) (neg? x) (zero? x) (< 5 x 10)

;; Sets and keywords as functions
(filter #{:role/admin :role/moderator} user-roles)
(map :user/id users)

;; Avoid useless wrappers
(filter even? coll)           ; ✅ GOOD
(filter #(even? %) coll)      ; ❌ BAD
```

## Part 2: Functional Architecture

### Boundaries

**Definition**: Boundaries are interfaces where your application interacts with the external world.

**Boundaries include**: HTTP handlers, database access, message queues, file I/O, browser storage, JavaScript interop, external API calls.

**Rules**:
- External formats (JSON with string keys, unqualified keywords) are ONLY permitted at boundaries
- Boundaries MUST transform external data to internal domain format (namespaced keywords)
- Boundary functions SHOULD end with `!` to indicate side effects
- All domain logic operates on internal format with namespaced keywords

```clojure
;; ✅ GOOD: Boundary transforms external data to internal format
(require '[cljs.core.async :refer [go <!]]
         '[cljs-http.client :as http]
         '[cheshire.core :as json])

(defn fetch-user-from-api! [user-id]
  (go
    (let [resp (<! (http/get (str "/api/users/" user-id) {:with-credentials? false}))
          body (json/parse-string (:body resp))]
      {:user/id (get body "id") :user/name (get body "name") :user/email (get body "email")})))

;; ✅ GOOD: Pure domain logic uses only namespaced keywords
(defn calculate-user-tier [user]
  (cond
    (:user/premium user) :tier/premium
    (> (:user/purchase-count user 0) 10) :tier/gold
    :else :tier/standard))
```

### Pure Functions

```clojure
;; ✅ GOOD: Pure logic separated from I/O
(defn calculate-discount [price tier]
  (case tier :tier/premium (* price 0.9) :tier/standard (* price 0.95) price))

(defn apply-discount-logic [order tier]
  (update order :order/price #(calculate-discount % tier)))

;; ❌ BAD: Mixing I/O with logic
(defn apply-discount [order-id]
  (let [order (fetch-order order-id)
        user (fetch-user (:order/user-id order))]
    (save-order (update order :order/price * 0.9))))
```

### Immutability

```clojure
;; ✅ GOOD: Create new values
(defn add-item [cart item] (update cart :cart/items conj item))
(-> user (assoc :user/name "Alice") (update :user/login-count inc))

;; ❌ BAD: Unnecessary mutation
(defn calculate-total [items]
  (let [total (atom 0)]
    (doseq [item items] (swap! total + (:item/price item)))
    @total))

;; ✅ GOOD: Use reduce
(defn calculate-total [items] (reduce + (map :item/price items)))
```

### Function Composition

```clojure
;; Transducers for performance
(defn calculate-total [items]
  (transduce (map :item/price) + 0 items))

;; comp for composition
(def normalize-name (comp str/trim str/lower-case))
```

### Protocols & Multimethods

```clojure
;; Protocols for polymorphism
(defprotocol Drawable
  (draw [this] "Draw the shape"))

(defrecord Circle [radius]
  Drawable
  (draw [this] (str "Circle with radius " (:radius this))))

;; Multimethods for open extension
(defmulti process-event :event/type)
(defmethod process-event :event/user-created [event] (register-user (:event/user event)))
(defmethod process-event :default [event] (log-unknown-event event))
```

### State Management

```clojure
;; Atoms for independent state
(swap! app-state update :app/users conj new-user)

;; Refs for coordinated transactions
(dosync (alter account-a - 100) (alter account-b + 100))

;; Agents for async changes
(send log-agent conj {:log/message "Event"})
```

## Part 3: Error Handling

### Result Objects Pattern

```clojure
;; {:ok value} | {:error {:type :domain/code :message "user" :log-message "internal" :data {...} :log-context {...}}}
(defn divide [a b]
  (if (zero? b)
    {:error {:type :math/division-by-zero
             :message "Cannot divide by zero"
             :data {:dividend a :divisor b}}}
    {:ok (/ a b)}))

(defn process-payment [payment-info]
  (cond
    (invalid-card? payment-info)
    {:error {:type :payment/invalid-card :message "Card number is invalid"
             :log-message "Invalid card validation failed"
             :data {:card-last-4 (get-last-4 (:payment/card payment-info))}
             :log-context {:user-id (:payment/user-id payment-info)}}}
    (insufficient-funds? payment-info)
    {:error {:type :payment/insufficient-funds :message "Insufficient funds"
             :log-message "Insufficient balance for transaction"
             :data {:required (:payment/amount payment-info)}
             :log-context {:user-id (:payment/user-id payment-info)}}}
    :else
    {:ok {:transaction/id (generate-id) :transaction/status :success}}))

;; Helpers
(defn ok? [result]
  (and (map? result) (contains? result :ok)))

(defn error? [result]
  (and (map? result) (contains? result :error)))

(defn unwrap
  "Extracts value from result or throws. Use for programmer errors only."
  [result]
  (cond
    (ok? result) (:ok result)
    (error? result)
    (let [err (:error result)
          msg (or (:log-message err) (:message err))]
      (throw (ex-info msg
                      (merge {:type (:type err)}
                             (:data err)
                             (:log-context err)))))
    :else (throw (ex-info "Invalid result format" {:result result}))))

(defn unwrap-or
  "Extracts value from result or returns default. Use for expected failures."
  [result default]
  (if (ok? result) (:ok result) default))

;; ❌ BAD: Inconsistent formats - no structure or non-namespaced types
{:error "Something wrong"}  {:error {:type :generic}}
```

### Exceptions

```clojure
;; Use ex-info with namespaced types and structured data
(throw (ex-info "Invalid age" {:type :validation/age-invalid :field :age :value age}))

;; Catch specific exceptions, wrap at boundaries
(try (Integer/parseInt user-input)
  (catch NumberFormatException e
    {:error {:type :input/invalid-number :message "Must be a number" :data {:input user-input}}}))

;; Don't leak low-level errors
(try (query-database {:id id})
  (catch Exception e
    (throw (ex-info "Failed to fetch user" {:type :database/query-failed :user-id id} e))))
```

### Preconditions & Validation

```clojure
;; :pre/:post for INTERNAL invariants (programmer errors)
(defn calculate-discount [price rate]
  {:pre [(number? price) (>= price 0) (<= 0 rate 1)]
   :post [(number? %) (>= % 0) (<= % price)]}
  (* price (- 1 rate)))

;; Explicit validation for USER input
(defn validate-input [price]
  (cond
    (not (number? price))
    {:error {:type :validation/type-mismatch :message "Price must be a number"}}
    (< price 0)
    {:error {:type :validation/out-of-range :message "Price must be non-negative"}}
    :else {:ok price}))
```

### Spec Validation

```clojure
(require '[clojure.spec.alpha :as s])

;; Define specs with explicit namespaced keywords
(s/def :user/id string?)
(s/def :user/name (s/and string? #(> (count %) 0)))
(s/def :user/email (s/and string? #(re-matches #".+@.+\..+" %)))
(s/def :user/user (s/keys :req [:user/id :user/name :user/email]))

;; Validate at boundaries
(defn create-user [user-data]
  (if (s/valid? :user/user user-data)
    {:ok (save-user user-data)}
    {:error {:type :validation/spec-failed
             :message "Invalid user data"
             :data (s/explain-data :user/user user-data)}}))

;; Use for generative testing
(s/fdef calculate-discount
  :args (s/cat :price number? :rate (s/double-in :min 0.0 :max 1.0))
  :ret number?)
```

### Expected vs Unexpected Errors

```clojure
;; Return data for expected failures
(defn process-payment [order]
  (cond
    (insufficient-funds? order)
    {:error {:type :payment/insufficient-funds :data {...}}}
    (invalid-card? order)
    {:error {:type :payment/invalid-card :data {...}}}
    :else
    {:ok {:transaction/id (generate-id) :transaction/status :success}}))

;; Throw for unexpected failures
(when-not (string? id)
  (throw (ex-info "Invalid ID type" {:type :programmer-error :expected "string"})))
```

## Part 4: ClojureScript Differences

### JavaScript Interop

```clojure
;; Property access, method calls
(.-length js-array) (.toUpperCase js-string) (.log js/console "Hello")

;; ✅ GOOD: Convert at boundaries ONLY (js->clj, clj->js)
(clj->js {:name "Alice" :age 30})
(js->clj js-object :keywordize-keys true)
(defn handle-api-response! [js-response]
  (let [user-data (js->clj js-response :keywordize-keys true)]
    {:user/id (:id user-data) :user/name (:name user-data)}))
```

### Async & Browser APIs

```clojure
;; ✅ GOOD: Async timeout pattern with Result Object
(defn fetch-with-timeout [url ms]
  (let [result-ch (promise-chan)
        timeout-ch (timeout ms)]
    (go
      (try
        ;; cljs-http returns a channel with response map
        (let [response (<! (http/get url {:with-credentials? false}))]
          (put! result-ch {:ok response}))
        (catch js/Error e
          (put! result-ch {:error {:type :http/error :message "Request failed"
                                   :log-message (.-message e) :data {:url url}}}))))
    (go
      (let [[value ch] (alts! [result-ch timeout-ch])]
        (if (= ch timeout-ch)
          {:error {:type :http/timeout :message "Request timed out"
                   :data {:url url :timeout-ms ms}}}
          value)))))

;; Promises
(-> (js/fetch "/api/users")
    (.then #(.json %))
    (.then #(js->clj % :keywordize-keys true))
    (.catch #(js/console.error "Error:" %)))

;; Local storage
(defn save-to-storage! [key value]
  (.setItem js/localStorage (name key) (pr-str value)))

(defn load-from-storage [key]
  (when-let [value (.getItem js/localStorage (name key))]
    ;; ⚠️ SECURITY: Use read-string ONLY for data you control (self-written data)
    ;; For external/user input, use safe parsers like JSON
    (cljs.reader/read-string value)))
```

### Compilation

```clojure
;; Type hints for performance
(defn sum-numbers [^js/Array arr]
  (areduce arr idx ret 0 (+ ret (aget arr idx))))
```

## Testing

```clojure
;; Clojure
(ns myapp.core-test
  (:require [clojure.test :refer [deftest is testing]]
            [myapp.core :as sut]))

(deftest calculate-total-test
  (is (= 30 (sut/calculate-total [{:item/price 10} {:item/price 20}]))))

;; ClojureScript: use refer-macros
(ns myapp.core-test
  (:require [cljs.test :refer-macros [deftest is]]))
```

## Common Anti-Patterns

```clojure
;; ❌ Don't define vars inside functions
(defn foo [] (def x 5) (* x 2))  ; Creates global var

;; ❌ Don't shadow core names
(defn process [map filter] ...)  ; Shadows clojure.core/map and filter

;; ✅ Rename parameters
(defn process [coll pred] (filter pred coll))

;; ❌ Deeply nested conditionals
(if user (if (:user/admin user) (if market (if (:market/active market) (do-thing)))))

;; ✅ Use and for multiple conditions
(when (and user (:user/admin user) market (:market/active market))
  (do-thing))
```

## Documentation

```clojure
;; Docstrings for public APIs
(defn search-markets
  "Searches markets using semantic similarity. Returns vector sorted by similarity."
  ([query] (search-markets query 10))
  ([query limit] (perform-search query {:limit limit})))

;; Comments explain WHY, not WHAT
;; Exponential backoff prevents overwhelming API during outages
(def retry-delay (min 30000 (* 1000 (Math/pow 2 retry-count))))
```
