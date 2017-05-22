---
layout: post
title:  "JWT authentication for re-frame using Auth0"
date:   2017-05-22 22:00:00 +0200
categories: clojurescript re-frame
tags: clojurescript re-frame auth0 jwt authentication react
---

As a new clojurescript user looking to get started with a personal project, authentication was
a problem I encountered quickly.
Finding out how to use the Auth0 service with my application proved surprisingly time consuming,
so here is a simple example that will enable you to hit the ground running.
It is tailored for the [re-frame][re-frame] framework, but you can still salvage some code
if you want to use something else.

**Jump [straight to the code][github-project]**

# Requirements
Before we start, I'll assume that you are somewhat familiar with:
- [ClojureScript][ClojureScript] or [Clojure][Clojure]
- [Leiningen][Leiningen]
- [re-frame][re-frame]
- The basics of [*JWT*][JWT] authentication

# Setting up the clojurescript project

Let's create a new re-frame application, we'll use the vanilla template, but you
can add other profiles if you want.

`lein new re-frame reframe-auth0`
```
reframe-auth0/
├── project.clj
├── README.md
├── resources
│   └── public
│       └── index.html
└── src
    ├── clj
    │   └── reframe_auth0
    │       └── core.clj
    └── cljs
        └── reframe_auth0
            ├── config.cljs
            ├── core.cljs
            ├── db.cljs
            ├── events.cljs
            ├── subs.cljs
            └── views.cljs
```


Open the `project.clj` file, and add the latest cljsjs jar of the [auth0-lock][auth0-lock] library :
[![Clojars Project](https://img.shields.io/clojars/v/cljsjs/auth0-lock.svg)](https://clojars.org/cljsjs/auth0-lock)

# Configuring the auth0 account

If you don't already have an account on [Auth0][auth0], create one.

You'll then need to [add a new client](https://manage.auth0.com/#/clients) for the application we're creating.

Take note of the `domain` and `client id` properties, we'll need them later.

In `Allowed Callback URLs` and `Allowed Origins`, enter `http://localhost:3449/`, and save the changes.


# Setting up the login screen

Go to the `config.cljs` file, and add a definition for the auth0 credentials.

Add the auth0 **client id** and **domain**
```clojure
(def auth0
  {:client-id    "abcd1234"
   :domain       "xyz.auth0.com"})
```

In the directory with all the `.cljs` files, we'll create an `auth0.cljs` file.
Make sure it starts with the following namespace declaration.
```clojure
(ns reframe-auth0.auth0
  (:require [re-frame.core :as re-frame]
            [reframe-auth0.config :as config]
            [cljsjs.auth0-lock]))
```

Declare an [Auth0 lock][auth0-lock] configured with the relevant properties.
```clojure
(def lock
  "The auth0 lock instance used to login and make requests to Auth0"
  (let [client-id (:client-id config/auth0)
        domain (:domain config/auth0)
        options (clj->js {})]
    (js/Auth0Lock. client-id domain options)))
```

Here, we'll use the default options (empty map). The configuration options are described [here][lock-options].

Let's add a simple authentication callback for now
```clojure
(defn on-authenticated
  "Function called by auth0 lock on authentication"
  [auth-result-js]
  (js/alert (str "Auth0 authentication result: "
                 (js->clj auth-result-js))))

(.on lock "authenticated" on-authenticated)
```

# Login button

Let's update the `views.cljs` file to add a simple login button.
```clojure
(ns reframe-auth0.views
  (:require [re-frame.core :as re-frame]
            [reframe-auth0.auth0 :as auth0]))

(defn button [text on-click]
  [:button
   {:type     "button"
    :on-click on-click}
   text])

(def login-button
  (button "Log in" #(.show auth0/lock)))

(defn main-panel []
  (let [name (re-frame/subscribe [:name])]
    (fn []
      [:div
       [:div "Hello from " @name]
       login-button]
      )))
```

# First test

Run your application with `lein figwheel`, and go to [http://localhost:3449/](http://localhost:3449/).
When you login, the result returned by auth0 should appear in a browser alert box.

# Retrieving the user profile details

When authenticating, the auth0 lock gives you an `authResult` containing the properties:
`accessToken`, `idToken`, `idTokenPayload`, `state`, `refreshToken`. (see [doc][auth0-api])

The `idToken` is sufficient to secure your API calls, but by default it does not contain
informations like the user name or email.
You could create a token with additional informations in it, but in general you want to keep it
small since it will be sent with every API request.

However, you can retrieve the user profile from Auth0 using the `accessToken`.

Let's edit the code. We'll convert the `authResult` to a clojure map and extract the `accessToken`,
the we'll make a call to auth0 using the `getUserInfo` function provided by the lock.

```clojure
(defn handle-profile-response [error profile] *
  "Handle the response for Auth0 profile request"
  (js/alert (str "Auth0 user profile: "
                 (js->clj profile))))

(defn on-authenticated
  "Function called by auth0 lock on authentication"
  [auth-result-js]
  (js/alert (str "Auth0 authentication result: "
                 (js->clj auth-result-js)))

  (let [auth-result-clj (js->clj auth-result-js :keywordize-keys true)
        access-token (:accessToken auth-result-clj)]

    (.getUserInfo lock access-token handle-profile-response)))

```

If you try it now, you will get one alert box with the `authResult`, and then another one
with the content of the user profile.

# Storing the access token and user details.

In typical re-frame fashion, we'll store the information in the central storage atom.
Let's store everything we got this far in the following data structure :
```clojure
{
  :user {
    :auth-result xxxx
    :profile xxxx
  }
}
```

Let's add the required events and subscriptions.

It is best practice to put events and subscription in dedicated files, but for something this simple
I'm tempted to put them in the `auth0.cljs` file, to have everything available at a glance.

This will require us to make sure that the events are registered before loading other parts of the application, so let's
require the auth0 namespace in `core.cljs` first:
```clojure
(ns reframe-auth0.core
    (:require [reagent.core :as reagent]
              [re-frame.core :as re-frame]
              [reframe-auth0.events]
              [reframe-auth0.subs]
              [reframe-auth0.auth0]
              [reframe-auth0.views :as views]
              [reframe-auth0.config :as config]))
```

Then add to the `auth0.cljs` file:
```clojure
;;; events
(re-frame/reg-event-db
  ::set-auth-result
  (fn [db [_ auth-result]]
    (assoc-in db [:user :auth-result] auth-result)))

(re-frame/reg-event-db
  ::set-user-profile
  (fn [db [_ profile]]
    (assoc-in db [:user :profile] profile)))
```

We can then edit the handler:
```clojure
(defn handle-profile-response [error profile] *
  "Handle the response for Auth0 profile request"
  (let [profile-clj (js->clj profile :keywordize-keys true)]
    (re-frame/dispatch [::set-user-profile profile-clj])))

(defn on-authenticated
  "Function called by auth0 lock on authentication"
  [auth-result-js]
  (let [auth-result-clj (js->clj auth-result-js :keywordize-keys true)
        access-token (:accessToken auth-result-clj)]
    (re-frame/dispatch [::set-auth-result auth-result-clj])
    (.getUserInfo lock access-token handle-profile-response)))

```
# Personalizing the view
Let's personalize the default re-frame page by changing it to
`Hello 'username' from re-frame`

We'll also add a logout button that removes all the user data we stored.

We'll first create a subscription to get the user name in `auth0.cljs`
```clojure
;;; subscriptions

(re-frame/reg-sub
  ::user-name
  (fn [db]
    (get-in db [:user :profile :name])))
```

Also we'll register a logout event
```clojure
(re-frame/reg-event-db
  ::logout
  (fn [db [_ profile]]
    (dissoc db :user)))
```

Then we'll update the `views.cljs`.
```clojure
(def logout-button
  (button "Log out" #(re-frame/dispatch [::auth0/logout])))

(defn main-panel []
  (let [name (re-frame/subscribe [:name])
        user-name (re-frame/subscribe [::auth0/user-name])]
    (fn []
      (if @user-name
        [:div
         [:div "Hello " @user-name " from " @name]
         logout-button]
        [:div
         [:div "Hello from " @name]
         login-button]))))
```

You can now login, logout, and see the user name on the main page.

**Full project on [github][github-project]**

# What's Next
1. Create a simple backend and make a secure API call using the JWT.
2. Persist the information to local storage.
3. Tokens expire. Implement validity checks and a renewal mechanism.
4. Let's see if we can create a [reframe template][reframe template]

[github-project]: https://github.com/randomlurker/reframe-auth0

[Leiningen]: https://leiningen.org/
[Clojure]: https://clojure.org
[ClojureScript]: https://clojurescript.org/

[re-frame]: https://github.com/Day8/re-frame
[reframe template]:https://github.com/Day8/re-frame-template

[JWT]: https://jwt.io/introduction/

[auth0]: https://auth0.com/
[auth0-lock]: https://auth0.com/docs/libraries/lock/v10
[lock-options]: https://auth0.com/docs/libraries/lock/v10/customization
[auth0-api]: https://auth0.com/docs/libraries/lock/v10/api#on-
