---
name: backend-patterns-clojure
description: Backend architecture patterns and best practices for Clojure using reitit, ring, and muuntaja. Immutable data structures, functional composition, and spec validation.
---

# Clojure Backend Patterns

Backend architecture patterns for Clojure applications using reitit (routing), ring (HTTP), and muuntaja (JSON).

## API Design with Reitit

### RESTful Route Structure

```clojure
(require '[reitit.ring :as ring]
         '[reitit.coercion.spec]
         '[muuntaja.core :as m])

;; ✅ GOOD: Resource-based routing
(def app
  (ring/ring-handler
    (ring/router
      [["/api"
        ["/markets" {:get    {:handler get-markets}
                     :post   {:handler create-market}}]
        ["/markets/:id" {:get    {:handler get-market}
                         :put    {:handler update-market}
                         :delete {:handler delete-market}}]
        ["/markets/:id/positions" {:get  {:handler get-positions}
                                   :post {:handler create-position}}]]]
      {:data {:coercion   reitit.coercion.spec/coercion
              :muuntaja   m/instance
              :middleware [middleware/wrap-format]}})))

;; Query parameters for filtering
;; GET /api/markets?status=active&sort=volume&limit=20
```

### Handler Pattern with Ring

```clojure
(ns myapp.handlers.markets
  (:require [clojure.spec.alpha :as s]
            [ring.util.response :as response]))

;; ✅ GOOD: Pure handler functions
(defn get-markets
  [{:keys [query-params]}]
  (let [status (:status query-params)
        limit  (parse-long (:limit query-params "10"))
        markets (db/find-markets {:status status :limit limit})]
    (response/response {:success true
                        :data markets})))

(defn get-market
  [{{:keys [id]} :path-params}]
  (if-let [market (db/find-market-by-id id)]
    (response/response {:success true
                        :data market})
    (-> (response/response {:success false
                            :error "Market not found"})
        (response/status 404))))

;; ❌ BAD: Stateful handlers
(def *cache* (atom {}))

(defn get-market-bad [request]
  (swap! *cache* assoc :last-request request)  ; Side effect
  ;; Handler logic
  )
```

## Data Access Patterns

### Repository Pattern with Protocols

```clojure
(ns myapp.repository.market)

;; ✅ GOOD: Protocol for abstraction
(defprotocol MarketRepository
  (find-all [this filters])
  (find-by-id [this id])
  (create [this market-data])
  (update-market [this id market-data])
  (delete-market [this id]))

;; Implementation for PostgreSQL
(defrecord PostgresMarketRepository [db]
  MarketRepository
  (find-all [_ filters]
    (let [status (:status filters)
          limit  (:limit filters 10)]
      (jdbc/query db
        (cond-> ["SELECT * FROM markets"]
          status (concat ["WHERE status = ?" status])
          limit  (concat ["LIMIT ?" limit])))))

  (find-by-id [_ id]
    (jdbc/get-by-id db :markets id))

  (create [_ market-data]
    (jdbc/insert! db :markets market-data))

  (update-market [_ id market-data]
    (jdbc/update! db :markets market-data ["id = ?" id]))

  (delete-market [_ id]
    (jdbc/delete! db :markets ["id = ?" id])))

;; Usage with dependency injection
(def market-repo (->PostgresMarketRepository db-spec))
```

### Query Builder Pattern

```clojure
(ns myapp.db.query)

;; ✅ GOOD: Composable query builders
(defn base-query [table]
  {:select [:*]
   :from   [table]})

(defn with-status [query status]
  (assoc query :where [:= :status status]))

(defn with-limit [query limit]
  (assoc query :limit limit))

(defn with-offset [query offset]
  (assoc query :offset offset))

(defn with-order [query column direction]
  (assoc query :order-by [[column direction]]))

;; Usage with threading macro
(defn find-active-markets [limit offset]
  (-> (base-query :markets)
      (with-status "active")
      (with-order :volume :desc)
      (with-limit limit)
      (with-offset offset)
      sql/format
      (jdbc/query db)))

;; ❌ BAD: String concatenation
(defn find-markets-bad [status]
  (jdbc/query db
    [(str "SELECT * FROM markets WHERE status = '" status "'")]))  ; SQL injection!
```

## Middleware Patterns

### Authentication Middleware

```clojure
(ns myapp.middleware.auth
  (:require [buddy.sign.jwt :as jwt]
            [ring.util.response :as response]))

;; ✅ GOOD: Pure middleware composition
(defn wrap-auth [handler secret]
  (fn [request]
    (if-let [token (-> request
                       (get-in [:headers "authorization"])
                       (clojure.string/replace #"^Bearer " ""))]
      (try
        (let [claims (jwt/unsign token secret)
              request' (assoc request :user claims)]
          (handler request'))
        (catch Exception _
          (-> (response/response {:error "Invalid token"})
              (response/status 401))))
      (-> (response/response {:error "Missing authorization"})
          (response/status 401)))))

;; Usage
(def app
  (-> handler
      (wrap-auth jwt-secret)
      wrap-params
      wrap-json-body))
```

### Logging Middleware

```clojure
(ns myapp.middleware.logging
  (:require [clojure.tools.logging :as log]))

;; ✅ GOOD: Structured logging
(defn wrap-logging [handler]
  (fn [request]
    (let [start (System/currentTimeMillis)
          {:keys [request-method uri]} request
          response (handler request)
          duration (- (System/currentTimeMillis) start)]
      (log/info {:event "request"
                 :method request-method
                 :uri uri
                 :status (:status response)
                 :duration-ms duration})
      response)))

;; ❌ BAD: Unstructured logging
(defn wrap-logging-bad [handler]
  (fn [request]
    (println "Request:" (:uri request))  ; Not structured, not production-ready
    (handler request)))
```

## Service Layer with Spec Validation

### Domain Service Pattern

```clojure
(ns myapp.service.market
  (:require [clojure.spec.alpha :as s]
            [clojure.spec.gen.alpha :as gen]))

;; Define specs
(s/def ::id string?)
(s/def ::name (s/and string? #(< 0 (count %) 200)))
(s/def ::status #{:active :resolved :closed})
(s/def ::volume number?)
(s/def ::created-at inst?)

(s/def ::market
  (s/keys :req-un [::id ::name ::status ::volume ::created-at]))

(s/def ::create-market-request
  (s/keys :req-un [::name]
          :opt-un [::status]))

;; ✅ GOOD: Service with validation
(defn create-market [repo market-data]
  (if (s/valid? ::create-market-request market-data)
    (let [market (merge {:id (random-uuid)
                         :status :active
                         :volume 0
                         :created-at (java.time.Instant/now)}
                        market-data)]
      (repository/create repo market)
      {:success true :data market})
    {:success false
     :error "Invalid market data"
     :details (s/explain-data ::create-market-request market-data)}))

;; ✅ GOOD: Composition with threading
(defn search-markets [repo embedding-service query limit]
  (->> query
       (embedding-service/generate-embedding embedding-service)
       (vector-search limit)
       (map :id)
       (repository/find-by-ids repo)
       (sort-by-similarity query)))
```

## Error Handling Patterns

### Result Type Pattern

```clojure
(ns myapp.util.result)

;; ✅ GOOD: Explicit success/failure
(defn success [data]
  {:success true :data data})

(defn failure [error]
  {:success false :error error})

(defn success? [result]
  (:success result))

(defn failure? [result]
  (not (:success result)))

;; Usage
(defn process-order [order]
  (cond
    (nil? order)
    (failure "Order not found")

    (empty? (:items order))
    (failure "Order has no items")

    (not (has-inventory? order))
    (failure "Insufficient inventory")

    :else
    (success (complete-order order))))

;; Handle result
(let [result (process-order order)]
  (if (success? result)
    (println "Success:" (:data result))
    (println "Error:" (:error result))))
```

### Exception Handling with Try/Catch

```clojure
(ns myapp.handler.api
  (:require [ring.util.response :as response]))

;; ✅ GOOD: Comprehensive error handling
(defn fetch-market-data [id]
  (try
    (let [market (db/find-by-id id)]
      (if market
        (response/response {:success true :data market})
        (-> (response/response {:error "Not found"})
            (response/status 404))))
    (catch java.sql.SQLException e
      (log/error e "Database error")
      (-> (response/response {:error "Database error"})
          (response/status 500)))
    (catch Exception e
      (log/error e "Unexpected error")
      (-> (response/response {:error "Internal error"})
          (response/status 500)))))

;; ❌ BAD: No error handling
(defn fetch-market-data-bad [id]
  (response/response {:data (db/find-by-id id)}))
```

## Immutability & State Management

### Atoms for Application State

```clojure
;; ✅ GOOD: Atom for mutable state
(def app-state
  (atom {:cache {}
         :connections #{}}))

(defn cache-market [id market]
  (swap! app-state assoc-in [:cache id] market))

(defn get-cached-market [id]
  (get-in @app-state [:cache id]))

;; ✅ GOOD: Pure functions with immutable data
(defn update-market-volume [market volume]
  (assoc market :volume volume))

(defn add-position [market position]
  (update market :positions conj position))

;; ❌ BAD: Mutable Java objects
(def bad-cache (java.util.HashMap.))  ; Avoid mutable Java collections
```

### Data Transformation with Threading Macros

```clojure
;; ✅ GOOD: Thread-first for data transformations
(defn process-markets [markets]
  (->> markets
       (filter #(= (:status %) :active))
       (map #(assoc % :volume-usd (* (:volume %) (:price %))))
       (sort-by :volume-usd >)
       (take 10)))

;; ✅ GOOD: Thread-last for nested access
(defn get-user-market-count [user-id]
  (-> (db/find-user user-id)
      :markets
      count))

;; ❌ BAD: Nested function calls
(defn process-markets-bad [markets]
  (take 10
    (sort-by :volume-usd >
      (map #(assoc % :volume-usd (* (:volume %) (:price %)))
        (filter #(= (:status %) :active) markets)))))
```

## Caching with Core.cache

```clojure
(ns myapp.cache
  (:require [clojure.core.cache.wrapped :as cache]))

;; ✅ GOOD: TTL cache
(def market-cache
  (cache/ttl-cache-factory {} :ttl 300000))  ; 5 minutes

(defn get-market-cached [id]
  (cache/lookup-or-miss market-cache
                        id
                        (fn [_] (db/find-market-by-id id))))

(defn invalidate-market [id]
  (swap! market-cache cache/evict id))

;; ✅ GOOD: LRU cache for limited memory
(def user-cache
  (cache/lru-cache-factory {} :threshold 1000))

(defn get-user-cached [id]
  (cache/lookup-or-miss user-cache
                        id
                        (fn [_] (db/find-user-by-id id))))
```

## Async with Core.async

```clojure
(ns myapp.async
  (:require [clojure.core.async :as async :refer [<! >! go chan]]))

;; ✅ GOOD: Channel-based async processing
(defn process-events [event-ch]
  (go
    (loop []
      (when-let [event (<! event-ch)]
        (try
          (handle-event event)
          (catch Exception e
            (log/error e "Event processing failed")))
        (recur)))))

;; ✅ GOOD: Pipeline for parallel processing
(defn index-markets-async [markets]
  (let [in (async/to-chan! markets)
        out (chan 100)]
    (async/pipeline-async 4
      out
      (fn [market ch]
        (go
          (let [result (<! (embedding-service/embed market))]
            (>! ch result)
            (async/close! ch))))
      in)
    out))

;; Usage
(let [result-ch (index-markets-async markets)]
  (go
    (loop []
      (when-let [result (<! result-ch)]
        (println "Indexed:" result)
        (recur)))))
```

## Database Transactions

```clojure
(ns myapp.db.transaction
  (:require [next.jdbc :as jdbc]))

;; ✅ GOOD: Transaction with jdbc/with-transaction
(defn create-market-with-position [db market-data position-data]
  (jdbc/with-transaction [tx db]
    (let [market (jdbc/execute-one!
                   tx
                   ["INSERT INTO markets (name, status) VALUES (?, ?)
                     RETURNING *"
                    (:name market-data)
                    (:status market-data)])
          position (jdbc/execute-one!
                     tx
                     ["INSERT INTO positions (market_id, user_id, shares)
                       VALUES (?, ?, ?)"
                      (:id market)
                      (:user-id position-data)
                      (:shares position-data)])]
      {:market market :position position})))

;; ❌ BAD: No transaction (race condition possible)
(defn create-market-with-position-bad [db market-data position-data]
  (let [market (jdbc/execute-one! db ["INSERT INTO markets ..."])
        position (jdbc/execute-one! db ["INSERT INTO positions ..."])]
    ;; If second insert fails, first is committed anyway
    {:market market :position position}))
```

## Rate Limiting

```clojure
(ns myapp.middleware.rate-limit
  (:require [clojure.core.cache.wrapped :as cache]))

;; ✅ GOOD: Token bucket rate limiter
(def rate-limits
  (atom {}))

(defn check-rate-limit [identifier max-requests window-ms]
  (let [now (System/currentTimeMillis)
        requests (get @rate-limits identifier [])
        recent (filter #(> (+ % window-ms) now) requests)]
    (if (>= (count recent) max-requests)
      false
      (do
        (swap! rate-limits assoc identifier (conj recent now))
        true))))

(defn wrap-rate-limit [handler max-requests window-ms]
  (fn [request]
    (let [ip (or (get-in request [:headers "x-forwarded-for"]) "unknown")]
      (if (check-rate-limit ip max-requests window-ms)
        (handler request)
        (-> (response/response {:error "Rate limit exceeded"})
            (response/status 429))))))

;; Usage
(def app
  (-> handler
      (wrap-rate-limit 100 60000)))  ; 100 requests per minute
```

## Testing Patterns

### Unit Testing with clojure.test

```clojure
(ns myapp.service.market-test
  (:require [clojure.test :refer :all]
            [myapp.service.market :as market]))

;; ✅ GOOD: Pure function testing
(deftest test-calculate-similarity
  (testing "identical vectors return 1.0"
    (is (= 1.0 (market/cosine-similarity [1 0 0] [1 0 0]))))

  (testing "orthogonal vectors return 0.0"
    (is (= 0.0 (market/cosine-similarity [1 0 0] [0 1 0])))))

;; ✅ GOOD: Mock repository with reify
(deftest test-create-market-service
  (let [mock-repo (reify MarketRepository
                    (create [_ market]
                      market)
                    (find-by-id [_ id]
                      {:id id :name "Test Market"}))
        result (market/create-market mock-repo {:name "New Market"})]
    (is (:success result))
    (is (= "New Market" (-> result :data :name)))))

;; ✅ GOOD: Integration testing with test fixtures
(use-fixtures :each
  (fn [f]
    (jdbc/execute! test-db ["DELETE FROM markets"])
    (f)))

(deftest test-market-repository
  (let [repo (->PostgresMarketRepository test-db)
        market {:name "Test" :status "active"}
        created (repository/create repo market)]
    (is (uuid? (:id created)))
    (is (= "Test" (:name created)))))
```

## Configuration Management

```clojure
(ns myapp.config
  (:require [environ.core :refer [env]]))

;; ✅ GOOD: Environment-based configuration
(def config
  {:db {:host     (env :db-host "localhost")
        :port     (Integer/parseInt (env :db-port "5432"))
        :database (env :db-name "myapp")
        :user     (env :db-user)
        :password (env :db-password)}
   :api {:port (Integer/parseInt (env :port "3000"))
         :jwt-secret (env :jwt-secret)}
   :cache {:ttl-ms (Integer/parseInt (env :cache-ttl "300000"))}})

;; ❌ BAD: Hardcoded secrets
(def bad-config
  {:db {:password "secret123"}})  ; Never commit secrets!
```

**Remember**: Embrace immutability, composition, and pure functions. Clojure's strengths are simplicity and data-oriented design.
