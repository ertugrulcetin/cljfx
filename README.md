# Cljfx

Cljfx is a declarative, dynamic and extensible wrapper for JavaFX
applications.

## Rationale

I wanted to have an elegant, declarative and composable UI
library for JVM and couldn't find one. Cljfx is inspired by react,
reagent, re-frame and fn-fx.

Like react, it allows to specify only desired layout, and handles
all actual changes underneath. Unlike react (and web in general) it does
not impose xml-like structure of everything possibly having multiple
children, thus it uses maps instead of hiccup for describing layout.

Like reagent, it allows to specify component descriptions using simple
constructs such as data and functions. Unlike reagent, it rejects using
multiple stateful reactive atoms for state and instead prefers composing
ui in more pure manner.

Like re-frame, it provides an approach to building large applications
using subscriptions and events to separate view from logic. Unlike
re-frame, it has no hard-coded global state, and subscriptions work on
referentially transparent values instead of ever-changing atoms.

Like fn-fx, it wraps underlying JavaFX library so developer can describe
everything with clojure data. Unlike fn-fx, it is more dynamic, allowing
users to use maps and functions instead of macros and deftypes, and has
more explicit and extensible lifecycle for components.

## Introduction

### Hello world

Components in cljfx are described by maps with `:fx/type` key. By
default, fx-type can be either a keyword corresponding to some JavaFX
class, or a function, which receives this map as argument and returns
another description. Minimal example:
```clj
(ns example
  (:require [cljfx.api :as fx]))

(fx/on-fx-thread
  (fx/create-component
    {:fx/type :stage
     :showing true
     :title "Cljfx example"
     :width 300
     :height 100
     :scene {:fx/type :scene
             :root {:fx/type :v-box
                    :alignment :center
                    :children [{:fx/type :label
                                :text "Hello world"}]}}}))
```
Evaluating this code will create and show this window:

![](doc/hello-world.png)

### App

To be truly useful, there should be some state and changes over time,
for this matter there is an `app` abstraction, which is a function that
you may call whenever you want with new description, and
cljfx will advance all the mutable state underneath to match this
description. Example:
```clj
(def app
  (fx/create-app))

(defn root [{:keys [showing]}]
  {:fx/type :stage
   :showing showing
   :scene {:fx/type :scene
           :root {:fx/type :v-box
                  :padding 50
                  :children [{:fx/type :button
                              :text "close"
                              :on-action (fn [_]
                                           (app {:fx/type root
                                                 :showing false}))}]}}})

(app {:fx/type root
      :showing true})
```
Evaluating this code will show this:

![](doc/app-example.png)

Clicking `close` button will hide this window.

App batches descriptions and re-renders views on fx thread only with last
received description, so it is safe to call many times at once. Calls to app
function return derefable that will contain component value with most recent
description.

### Atoms

Example above works, but it's not very convenient: what we'd really like
is to have a single global state as a value in an atom, derive our
description of JavaFX state from this value, and change this atom's
contents instead. Here is how it's done:
```clj
;; Define application state

(def *state
  (atom {:title "App title"}))

;; Define render functions

(defn title-input [{:keys [title]}]
  {:fx/type :text-field
   :on-text-changed #(swap! *state assoc :title %)
   :text title})

(defn root [{:keys [title]}]
  {:fx/type :stage
   :showing true
   :title title
   :scene {:fx/type :scene
           :root {:fx/type :v-box
                  :children [{:fx/type :label
                              :text "Window title input"}
                             {:fx/type title-input
                              :title title}]}}})

;; Create app with middleware that maps incoming data - description -
;; to component description that can be used to render JavaFX state.
;; Here description is just passed as an argument to function component.

(def app
  (fx/create-app
    :middleware (fx/wrap-map-desc assoc :fx/type root)))

;; Convenient way to add watch to an atom + immediately render app

(fx/mount-app *state app)
```
Evaluating code above pops up this window:

![](doc/state-example.png)

Editing input then immediately updates displayed app title.

### Map events

Consider this example:

```clj
(defn todo-view [{:keys [text id done]}]
  {:fx/type :h-box
   :children [{:fx/type :check-box
               :selected done
               :on-selected-changed #(swap! *state assoc-in [:by-id id :done] %)
              {:fx/type :label
               :style {:-fx-text-fill (if done :grey :black)}
               :text text}]})
```

There are problems with using functions as event handlers:
1. Performing mutation from these handlers requires coupling with that
state, thus making `todo-view` dependent on mutable `*state`
2. Updating state from listeners complects logic with view, making
application messier over time
3. There are unnecessary reassignments to `on-selected-changed`:
functions have no equality semantics other than their identity, so on
every change to this view (for example, when changing it's text),
`on-selected-changed` will be replaced with another function with same
behavior.

To mitigate these problems, cljfx allows to define event handlers as
arbitrary maps, and provide a function to an app that performs actual
handling of these map-events (with additional `:fx/event` key containing
dispatched event):

```clj

;; Define view as just data

(defn todo-view [{:keys [text id done]}]
  {:fx/type :h-box
   :spacing 5
   :padding 5
   :children [{:fx/type :check-box
               :selected done
               :on-selected-changed {:event/type ::set-done :id id}}
              {:fx/type :label
               :style {:-fx-text-fill (if done :grey :black)}
               :text text}]})

;; Define single map-event-handler that does mutation

(defn map-event-handler [event]
  (case (:event/type event)
    ::set-done (swap! *state assoc-in [:by-id (:id event) :done] (:fx/event event))))

;; Provide map-event-handler to app as an option

(fx/mount-app
  *state
  (fx/create-app
    :middleware (fx/wrap-map-desc assoc :fx/type root)
    :opts {:fx.opt/map-event-handler map-event-handler}))

```

You can see full example at [examples/e09_todo_app.clj](examples/e09_todo_app.clj).

### Interactive development

Another useful aspect of app function that should be used during development is
refresh functionality: you can call app function with zero args and it will
recreate all the components with current description.

See walk-through in [examples/e12_interactive_development.clj](examples/e12_interactive_development.clj)
as an example of how to iterate on cljfx app in REPL.

### Special keys

Sometimes components accept specially treated keys. Main uses are:

1. Reordering of nodes (instead of re-creating them) in parents that may
   have many children. Descriptions that have `:fx/key` during
   advancing get reordered instead of recreated if their position in
   child list is changed. Consider this example:
   ```clj
   (let [component-1 (fx/create-component
                       {:fx/type :v-box
                        :children [{:fx/type :label
                                    :fx/key 1
                                    :text "- buy milk"}
                                   {:fx/type :label
                                    :fx/key 2
                                    :text "- buy socks"}]})
         [milk-1 socks-1] (vec (.getChildren (fx/instance component-1)))
         component-2 (fx/advance-component
                       component-1
                       {:fx/type :v-box
                        :children [{:fx/type :label
                                    :fx/key 2
                                    :text "- buy socks"}
                                   {:fx/type :label
                                    :fx/key 1
                                    :text "- buy milk"}]})
         [socks-2 milk-2] (vec (.getChildren (fx/instance component-2)))]
     (and (identical? milk-1 milk-2)
          (identical? socks-1 socks-2)))
   => true
   ```
   With `:fx/key`-s specified, advancing of this component reordered
   children of VBox, and didn't change text of any labels, because their
   descriptions stayed the same.
2. Providing extra props available in certain contexts. If node is
   placed inside a pane, pane can layout it differently by looking into
   properties map of a node. Nodes placed in ButtonBar can have
   OS-specific ordering depending on assigned ButtonData. These
   properties can be specified via keywords namespaced by container's
   fx-type. Example:
   ```clj
   (fx/on-fx-thread
     (fx/create-component
       {:fx/type :stage
        :showing true
        :scene {:fx/type :scene
                :root {:fx/type :stack-pane
                       :children [{:fx/type :rectangle
                                   :width 200
                                   :height 200
                                   :fill :lightgray}
                                  {:fx/type :label
                                   :stack-pane/alignment :bottom-left
                                   :stack-pane/margin 5
                                   :text "bottom-left"}
                                  {:fx/type :label
                                   :stack-pane/alignment :top-right
                                   :stack-pane/margin 5
                                   :text "top-right"}]}}}))
   ```
   Evaluating code above produces this window:

   ![](doc/pane-example.png)

   For a more complete example of available pane keys, see
   [examples/e07_extra_props.clj](examples/e07_extra_props.clj)

### Subscriptions and contexts

Once application becomes complex enough, you can find yourself passing
very big chunks of state everywhere. Consider this example: you develop
a task tracker for an organization. A typical task view on a dashboard
displays a description of that task and an assignee. Required state for
this view is plain and simple, just a simple data like that:
```clj
{:title "Fix NPE on logout during full moon"
 :state :todo
 :assignee {:id 42 :name "Fred"}}
```
Then one day comes a requirement: users of this task tracker should be
able to change assignee from the dashboard. Now, we need a combo-box
with all assignable users to render such a view, and then you find
yourself tediously passing data all over the place just to render one
view. Instead of doing this cljfx allows to have root description of the
application to be passed to every function component. Usually function
components get re-rendered when their arguments change, so passing root
description would trigger re-rendering of everything whenever this
description changes, which is undesired. This is why cljfx introduces
abstraction called **context**, inspired by re-frame's subscriptions.

## More examples

There are various examples available in [examples](examples) folder.

## License

TBD, need to consult my employer first

## TODO

- advanced docs:
  - subs / contexts
  - lifecycles
  - opts
  - contexts
  - lack of local state
  - why no :fx/type on insets (descs of immutable vs mutable values)
  - where to put :fx/key
  - fx/sub and lazy seqs
  - styles etc.

## Food for thought
- make exceptions more informative
- controlled props (mostly in controls, also stage's `:showing`)
- wrap-factory may use some memoizing and advancing
- add tests for various lifecycles and re-calculations
- prop lifecycle
- how to handle dialogs, animations and other possibly missed things?
- update to same desc should be identical (component-vec)
- optional flatten in wrap-many for maps?
- expand on props and composite lifecycle. What's known about them:
  - ctor:
    - scene requires root, root can be replaced afterwards
    - xy-chart requires axis, they can't be replaced afterwards
  - some props do not create instances, they use provided instead
    (dialog pane in dialog)
  - is it possible to inject components/lifecycles into cells? they are
    a bit different (triggered via updateItem), and instances are
    created for us, but otherwise it's just a node and we have props for
    them
  - prop in composite lifecycle may be a map or a function taking
    instance and returning prop!
  - changing media should re-create media player
- big app with everything in it to check if/how it works (generative
  tests maybe?)