---
name: frontend-patterns-reagent
description: Reagent and Re-frame frontend patterns for ClojureScript applications. Component design, state management, and performance optimization.
---

# Reagent Frontend Patterns

Modern patterns for Reagent (ClojureScript React) and Re-frame applications. Language-level Clojure standards in coding-standards-clojure.md.

## Component Forms

### Form-1: Simple Function Component

Stateless components that receive props and return hiccup.

```clojure
;; ✅ GOOD: Pure function component
(defn button
  [{:keys [on-click disabled? variant label]
    :or {disabled? false
         variant :primary}}]
  [:button
   {:on-click on-click
    :disabled disabled?
    :class (str "btn btn-" (name variant))}
   label])

;; Usage
[button {:label "Click me"
         :on-click #(js/alert "Clicked!")
         :variant :secondary}]

;; ✅ GOOD: Destructuring for clarity
(defn market-card
  [{:keys [id name description volume status]}]
  [:div.market-card
   [:h3 name]
   [:p description]
   [:div.stats
    [:span.volume volume]
    [:span.status status]]])
```

### Form-2: Component with Local State

Returns a render function that closes over local state (r/atom).

```clojure
(ns myapp.components
  (:require [reagent.core :as r]))

;; ✅ GOOD: Local state with atom
(defn counter []
  (let [count (r/atom 0)]
    (fn []
      [:div
       [:p "Count: " @count]
       [:button {:on-click #(swap! count inc)} "+"]
       [:button {:on-click #(swap! count dec)} "-"]
       [:button {:on-click #(reset! count 0)} "Reset"]])))

;; ✅ GOOD: Search input with debounce
(defn search-input
  [{:keys [on-search delay]
    :or {delay 500}}]
  (let [query (r/atom "")
        timeout-id (r/atom nil)]
    (fn [{:keys [on-search]}]
      [:input
       {:type "text"
        :value @query
        :placeholder "Search markets..."
        :on-change (fn [e]
                     (let [value (-> e .-target .-value)]
                       (reset! query value)
                       ;; Clear previous timeout
                       (when @timeout-id
                         (js/clearTimeout @timeout-id))
                       ;; Set new timeout
                       (reset! timeout-id
                         (js/setTimeout
                           #(on-search value)
                           delay))))}])))

;; ✅ GOOD: Toggle component
(defn expandable-section
  [{:keys [title children]}]
  (let [expanded? (r/atom false)]
    (fn [{:keys [title children]}]
      [:div.expandable
       [:div.header
        {:on-click #(swap! expanded? not)}
        [:h3 title]
        [:span (if @expanded? "▼" "▶")]]
       (when @expanded?
         [:div.content children])])))
```

### Form-3: Component with Lifecycle

Uses `r/create-class` for lifecycle methods.

```clojure
;; ✅ GOOD: Component with lifecycle hooks
(defn chart [data]
  (let [chart-instance (r/atom nil)
        chart-node (r/atom nil)]
    (r/create-class
     {:display-name "chart"

      :component-did-mount
      (fn [this]
        (let [node (r/dom-node this)]
          (reset! chart-node node)
          (reset! chart-instance
            (create-chart node (r/props this)))))

      :component-did-update
      (fn [this [_ old-props]]
        (let [new-props (r/props this)]
          (when (not= (:data old-props) (:data new-props))
            (update-chart @chart-instance (:data new-props)))))

      :component-will-unmount
      (fn [_]
        (when @chart-instance
          (destroy-chart @chart-instance)))

      :reagent-render
      (fn [data]
        [:div.chart-container])})))

;; ✅ GOOD: Auto-focus input
(defn auto-focus-input []
  (r/create-class
   {:component-did-mount
    (fn [this]
      (-> (r/dom-node this)
          (.focus)))

    :reagent-render
    (fn [{:keys [value on-change placeholder]}]
      [:input
       {:type "text"
        :value value
        :on-change on-change
        :placeholder placeholder}])}))

;; ✅ GOOD: Window event listener
(defn window-size-tracker []
  (let [size (r/atom {:width js/window.innerWidth
                      :height js/window.innerHeight})]
    (r/create-class
     {:component-did-mount
      (fn [_]
        (let [handler (fn []
                        (reset! size {:width js/window.innerWidth
                                      :height js/window.innerHeight}))]
          (js/window.addEventListener "resize" handler)
          ;; Store handler for cleanup
          (set! (.-resize-handler js/window) handler)))

      :component-will-unmount
      (fn [_]
        (js/window.removeEventListener "resize" (.-resize-handler js/window)))

      :reagent-render
      (fn []
        [:div
         [:p "Width: " (:width @size)]
         [:p "Height: " (:height @size)]])})))
```

## Re-frame Patterns

### Events

```clojure
(ns myapp.events
  (:require [re-frame.core :as rf]))

;; ✅ GOOD: Simple event
(rf/reg-event-db
  ::set-loading
  (fn [db [_ loading?]]
    (assoc db :loading loading?)))

;; ✅ GOOD: Event with validation
(rf/reg-event-db
  ::set-search-query
  (fn [db [_ query]]
    (if (string? query)
      (assoc db :search-query query)
      db)))

;; ✅ GOOD: Event with effects
(rf/reg-event-fx
  ::fetch-markets
  (fn [{:keys [db]} _]
    {:db (assoc db :loading true)
     :http-xhrio {:method :get
                  :uri "/api/markets"
                  :response-format (ajax/json-response-format {:keywords? true})
                  :on-success [::fetch-markets-success]
                  :on-failure [::fetch-markets-failure]}}))

(rf/reg-event-db
  ::fetch-markets-success
  (fn [db [_ markets]]
    (-> db
        (assoc :markets markets)
        (assoc :loading false)
        (dissoc :error))))

(rf/reg-event-db
  ::fetch-markets-failure
  (fn [db [_ error]]
    (-> db
        (assoc :error (str "Failed to fetch markets: " error))
        (assoc :loading false))))

;; ✅ GOOD: Interceptors for common logic
(rf/reg-event-db
  ::create-market
  [(rf/inject-cofx :now)]
  (fn [db [_ market]]
    (let [new-market (assoc market
                       :id (random-uuid)
                       :created-at (:now cofx))]
      (update db :markets conj new-market))))

;; ✅ GOOD: Chaining events
(rf/reg-event-fx
  ::login-success
  (fn [{:keys [db]} [_ user]]
    {:db (assoc db :user user)
     :dispatch [::fetch-user-data (:id user)]
     :navigate [:home]}))
```

### Subscriptions

```clojure
(ns myapp.subs
  (:require [re-frame.core :as rf]))

;; ✅ GOOD: Simple subscription
(rf/reg-sub
  ::markets
  (fn [db _]
    (:markets db)))

(rf/reg-sub
  ::loading?
  (fn [db _]
    (:loading db)))

;; ✅ GOOD: Derived subscription
(rf/reg-sub
  ::active-markets
  :<- [::markets]
  (fn [markets _]
    (filter :active markets)))

;; ✅ GOOD: Computed subscription with parameters
(rf/reg-sub
  ::market-by-id
  :<- [::markets]
  (fn [markets [_ id]]
    (first (filter #(= (:id %) id) markets))))

;; ✅ GOOD: Multiple input subscriptions
(rf/reg-sub
  ::filtered-markets
  :<- [::markets]
  :<- [::search-query]
  :<- [::selected-status]
  (fn [[markets query status] _]
    (cond->> markets
      (seq query)
      (filter #(str/includes? (str/lower-case (:name %))
                              (str/lower-case query)))

      status
      (filter #(= (:status %) status)))))

;; ✅ GOOD: Memoized expensive computation
(rf/reg-sub
  ::market-stats
  :<- [::markets]
  (fn [markets _]
    {:total (count markets)
     :active (count (filter :active markets))
     :total-volume (reduce + (map :volume markets))}))

;; Usage in components
(defn market-list []
  (let [markets @(rf/subscribe [::active-markets])
        loading? @(rf/subscribe [::loading?])]
    (if loading?
      [:div "Loading..."]
      [:div
       (for [market markets]
         ^{:key (:id market)}
         [market-card market])])))
```

### Effects and Coeffects

```clojure
;; ✅ GOOD: Custom effect
(rf/reg-fx
  :save-to-local-storage
  (fn [{:keys [key value]}]
    (.setItem js/localStorage (name key) (pr-str value))))

;; Usage
(rf/reg-event-fx
  ::save-preferences
  (fn [{:keys [db]} [_ preferences]]
    {:db (assoc db :preferences preferences)
     :save-to-local-storage {:key :user-preferences
                             :value preferences}}))

;; ✅ GOOD: Custom coeffect
(rf/reg-cofx
  :local-storage
  (fn [coeffects key]
    (assoc coeffects :local-storage
      (when-let [value (.getItem js/localStorage (name key))]
        (cljs.reader/read-string value)))))

;; Usage
(rf/reg-event-fx
  ::load-preferences
  [(rf/inject-cofx :local-storage :user-preferences)]
  (fn [{:keys [db local-storage]} _]
    {:db (assoc db :preferences local-storage)}))

;; ✅ GOOD: Navigate effect
(rf/reg-fx
  :navigate
  (fn [route]
    (reitit/push-state route)))

(rf/reg-event-fx
  ::go-to-market
  (fn [_ [_ market-id]]
    {:navigate [:market {:id market-id}]}))
```

## State Management Patterns

### Reactive Atoms (ratom)

```clojure
;; ✅ GOOD: Simple reactive atom
(def app-state (r/atom {:count 0
                        :name ""}))

;; Components automatically re-render when dereferenced values change
(defn counter-display []
  [:div "Count: " (:count @app-state)])

;; ✅ GOOD: Reactions for derived state
(def count (r/atom 0))
(def doubled (r/reaction (* 2 @count)))

(defn display []
  [:div
   [:p "Count: " @count]
   [:p "Doubled: " @doubled]])

;; ✅ GOOD: Track for side effects
(def query (r/atom ""))

(r/track!
  (fn []
    (when (seq @query)
      (js/console.log "Searching for:" @query)
      (search-markets @query))))
```

### Cursors for Nested State

```clojure
;; ✅ GOOD: Cursor into nested state
(def app-state (r/atom {:user {:name "Alice"
                                :email "alice@example.com"}
                        :settings {:theme "dark"
                                   :notifications true}}))

(def user-cursor (r/cursor app-state [:user]))
(def theme-cursor (r/cursor app-state [:settings :theme]))

;; Components can use cursors as if they were atoms
(defn user-form []
  (let [user @user-cursor]
    [:div
     [:input {:value (:name user)
              :on-change #(swap! user-cursor assoc :name (-> % .-target .-value))}]
     [:input {:value (:email user)
              :on-change #(swap! user-cursor assoc :email (-> % .-target .-value))}]]))
```

## Performance Optimization

### Avoid Unnecessary Re-renders

```clojure
;; ✅ GOOD: Deref only what you need
(defn market-name-display [market-id]
  (let [market @(rf/subscribe [::market-by-id market-id])]
    [:div (:name market)]))  ; Only re-renders when name changes

;; ❌ BAD: Dereferencing entire atom
(defn market-display []
  (let [state @app-state]  ; Re-renders on ANY state change
    [:div (:name (:selected-market state))]))

;; ✅ GOOD: Use reactions for derived data
(defn expensive-computation-component []
  (let [markets @(rf/subscribe [::markets])
        sorted (r/reaction (sort-by :volume markets))]  ; Memoized
    [:div
     (for [market @sorted]
       ^{:key (:id market)}
       [market-card market])]))
```

### Keys for Dynamic Lists

```clojure
;; ✅ GOOD: Stable unique keys
(defn market-list []
  (let [markets @(rf/subscribe [::markets])]
    [:div
     (for [market markets]
       ^{:key (:id market)}  ; Unique, stable key
       [market-card market])]))

;; ❌ BAD: Index as key (unstable)
(defn market-list []
  (let [markets @(rf/subscribe [::markets])]
    [:div
     (for [[idx market] (map-indexed vector markets)]
       ^{:key idx}  ; Bad: unstable when list reorders
       [market-card market])]))
```

### Lazy Sequences and Virtualization

```clojure
;; ✅ GOOD: Only render visible items
(defn virtualized-list
  [{:keys [items item-height container-height]}]
  (let [scroll-top (r/atom 0)
        visible-start (r/reaction (Math/floor (/ @scroll-top item-height)))
        visible-end (r/reaction (+ @visible-start (Math/ceil (/ container-height item-height))))
        visible-items (r/reaction (subvec items @visible-start (min (count items) @visible-end)))]
    (fn [{:keys [items item-height container-height]}]
      [:div.virtual-list
       {:style {:height container-height
                :overflow-y "auto"}
        :on-scroll #(reset! scroll-top (-> % .-target .-scrollTop))}
       [:div {:style {:height (* (count items) item-height)
                      :position "relative"}}
        (for [[idx item] (map-indexed vector @visible-items)]
          ^{:key (:id item)}
          [:div {:style {:position "absolute"
                         :top (* (+ @visible-start idx) item-height)
                         :height item-height}}
           [item-component item]])]])))
```

## Form Handling

### Controlled Inputs

```clojure
;; ✅ GOOD: Controlled form with validation
(defn create-market-form []
  (let [form-data (r/atom {:name ""
                           :description ""
                           :end-date ""})
        errors (r/atom {})
        submitting? (r/atom false)]

    (letfn [(validate []
              (let [data @form-data
                    errs (cond-> {}
                           (str/blank? (:name data))
                           (assoc :name "Name is required")

                           (> (count (:name data)) 200)
                           (assoc :name "Name must be under 200 characters")

                           (str/blank? (:description data))
                           (assoc :description "Description is required")

                           (str/blank? (:end-date data))
                           (assoc :end-date "End date is required"))]
                (reset! errors errs)
                (empty? errs)))

            (handle-change [field]
              (fn [e]
                (let [value (-> e .-target .-value)]
                  (swap! form-data assoc field value)
                  ;; Clear error when user types
                  (swap! errors dissoc field))))

            (handle-submit [e]
              (.preventDefault e)
              (when (validate)
                (reset! submitting? true)
                (-> (create-market @form-data)
                    (.then (fn [_]
                             (reset! form-data {:name "" :description "" :end-date ""})
                             (reset! submitting? false)))
                    (.catch (fn [error]
                              (swap! errors assoc :submit (str error))
                              (reset! submitting? false))))))]

      (fn []
        [:form {:on-submit handle-submit}
         [:div
          [:input
           {:type "text"
            :value (:name @form-data)
            :on-change (handle-change :name)
            :placeholder "Market name"}]
          (when-let [error (:name @errors)]
            [:span.error error])]

         [:div
          [:textarea
           {:value (:description @form-data)
            :on-change (handle-change :description)
            :placeholder "Description"}]
          (when-let [error (:description @errors)]
            [:span.error error])]

         [:div
          [:input
           {:type "date"
            :value (:end-date @form-data)
            :on-change (handle-change :end-date)}]
          (when-let [error (:end-date @errors)]
            [:span.error error])]

         [:button
          {:type "submit"
           :disabled @submitting?}
          (if @submitting? "Creating..." "Create Market")]]))))
```

## Error Handling

### Error Boundaries (via JS Interop)

```clojure
;; ✅ GOOD: Error boundary component
(defn error-boundary
  [_]
  (let [error (r/atom nil)]
    (r/create-class
     {:display-name "error-boundary"

      :component-did-catch
      (fn [_ err info]
        (js/console.error "Error caught:" err info)
        (reset! error {:message (.-message err)
                       :stack (.-stack err)}))

      :reagent-render
      (fn [{:keys [children fallback]}]
        (if @error
          (if fallback
            [fallback @error #(reset! error nil)]
            [:div.error-fallback
             [:h2 "Something went wrong"]
             [:p (:message @error)]
             [:button {:on-click #(reset! error nil)} "Try again"]])
          children))})))

;; Usage
[error-boundary
 {:fallback (fn [error reset]
              [:div
               [:h2 "Error: " (:message error)]
               [:button {:on-click reset} "Retry"]])}
 [app]]
```

### Async Error Handling

```clojure
;; ✅ GOOD: Error handling in effects
(rf/reg-event-fx
  ::fetch-markets
  (fn [{:keys [db]} _]
    {:db (assoc db :loading true :error nil)
     :http-xhrio {:method :get
                  :uri "/api/markets"
                  :response-format (ajax/json-response-format {:keywords? true})
                  :on-success [::fetch-success]
                  :on-failure [::fetch-failure]}}))

(rf/reg-event-db
  ::fetch-failure
  (fn [db [_ {:keys [status status-text]}]]
    (-> db
        (assoc :loading false)
        (assoc :error {:type :fetch-error
                       :status status
                       :message (str "Failed to fetch: " status-text)}))))
```

## JavaScript Interop Patterns

### Calling JS Libraries

```clojure
;; ✅ GOOD: Call JS methods
(defn use-chart-library [data]
  (r/create-class
   {:component-did-mount
    (fn [this]
      (let [node (r/dom-node this)
            chart (js/Chart. node
                    (clj->js {:type "bar"
                              :data data
                              :options {:responsive true}}))]
        (set! (.-chart this) chart)))

    :component-will-unmount
    (fn [this]
      (when-let [chart (.-chart this)]
        (.destroy chart)))

    :reagent-render
    (fn [_]
      [:canvas])}))

;; ✅ GOOD: Promise handling
(defn fetch-data []
  (-> (js/fetch "/api/data")
      (.then #(.json %))
      (.then #(js->clj % :keywordize-keys true))
      (.then #(rf/dispatch [::data-loaded %]))
      (.catch #(rf/dispatch [::data-error %]))))

;; ✅ GOOD: Event listeners
(defn click-outside-detector [on-click-outside]
  (r/create-class
   {:component-did-mount
    (fn [this]
      (let [node (r/dom-node this)
            handler (fn [e]
                      (when-not (.contains node (.-target e))
                        (on-click-outside)))]
        (js/document.addEventListener "click" handler)
        (set! (.-click-handler this) handler)))

    :component-will-unmount
    (fn [this]
      (js/document.removeEventListener "click" (.-click-handler this)))

    :reagent-render
    (fn [{:keys [children]}]
      [:div children])}))
```

---

**Remember**: Reagent embraces React's component model with Clojure's functional style. Use Form-1 for simple components, Form-2 for local state, and Form-3 only when you need lifecycle methods.
