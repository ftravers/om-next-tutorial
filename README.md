<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Introduction</a></li>
<li><a href="#sec-2">2. App Database</a>
<ul>
<li><a href="#sec-2-1">2.1. Explanation</a></li>
</ul>
</li>
<li><a href="#sec-3">3. Wire into a web page</a>
<ul>
<li><a href="#sec-3-1">3.1. Existing Documentation Deviation</a></li>
<li><a href="#sec-3-2">3.2. Fix up web page</a></li>
</ul>
</li>
<li><a href="#sec-4">4. Questions</a></li>
</ul>
</div>
</div>

# Introduction<a id="sec-1" name="sec-1"></a>

This will be an evolving tutorial, representing what I know, and
presumably at the end of the document, where I'm stuck.

There are at least two good spots to study om-next,
[the
quick-start](https://github.com/omcljs/om/wiki/Quick-Start-(om.next)) on the wiki, and
[Tony Kay's devcards tutorial](https://github.com/awkay/om-tutorial).
Even after reading/doing those sources, I was still too confused to
build anything.  This document will outline where I'm still confused.

# App Database<a id="sec-2" name="sec-2"></a>

Om next has a concept of a properly formed local database format.
This should be a clojure map.  Storing simple data like the current
users name should look like this:

    {:current/user {:user/name "Fenton"}}

I'm not sure how important it is to namespace like: `:current/user`
versus `:current-user`, but I think it is a best practice to help keep
your application data more organized.

The absolutely essential part is you have a map with a key to another
map.

Now lets test working with this client database, create a `*.cljc`
file with the following contents and fire up your **CLOJURE** (not
ClojureScript) REPL.  **NOTE:** feel free to use whatever namespace you
want to, `omn1,core2` is tangential information to the lesson.

    (ns omn1.core2
      (:require [om.next :as om]))
    (def app-state (atom {:current/user {:user/name "Fenton"}}))

Here `app-state` is your client database.

Now in your REPL execute the following command:

    omn1.core2> (om/db->tree [:current/user] @app-state @app-state)
    #:current{:user #:user{:name "Fenton"}}

The `#: ... {` syntax is just the namespacing representation.  This is
equivalent to:

    {:current/user {:user/name "Fenton"}}

## Explanation<a id="sec-2-1" name="sec-2-1"></a>

We have seen how to store **singleton'ish** information.  We use a map.
We have also seen how to extract our information, the key here is the
array with the keyword in it:

    [:current/user]

This is the first introduction to what we call the **Query Syntax**.
Here is a reference to [query syntax](https://awkay.github.io/om-tutorial/#!/om_tutorial.D_Queries).

So lets add some more interesting data.  Lets add a list of things.

    (def app-state (atom {:current/user {:user/name "Fenton"}
                          :my-cars [{:make "Toyota"
                                     :model "Tacoma"
                                     :year "2013"}
                                    {:make "BMW"
                                     :model "325xi"
                                     :year "2001"}]}))

Access it the same old way:

    omn1.core2> (om/db->tree [:my-cars] @app-state @app-state)
    {:my-cars
     [{:make "Toyota", :model "Tacoma", :year "2013"}
      {:make "BMW", :model "325xi", :year "2001"}]}
    omn1.core2>

# Wire into a web page<a id="sec-3" name="sec-3"></a>

Now lets wire this up into a web page.  I'm going to separate this
web-app into two files.  One `*.cljs` file and one `*.cljc` file.
I'll put the parts that have to be in clojurescrip into the `*.cljs`
file while parts that can run either server or client side (java or
javascript) into the `*.cljc` file.

The source tree will look like:

    ╭─fenton@ss9 ~/projects/omn1/src  ‹master*› 
    ╰─➤  tree
    |-- cljc
    |   `-- omn1
    |       `-- data.cljc
    `-- cljs
        `-- omn1
            `-- webpage.cljs
    4 directories, 2 files

File: `webpage.cljs`

    (ns omn1.webpage
      (:require
       [om.next :as om :refer-macros [defui]]
       [om.dom :as dom]
       [goog.dom :as gdom]
       [omn1.data :as dat]))
    
    (defui Blah
      static om/IQuery
      (query [this] [:current/user])
      Object
      (render
       [this]
       (let [data (om/props this)]
         (dom/div nil (str "App Data: " data)))))
    
    (om/add-root! dat/reconciler Blah (gdom/getElement "app"))

File: `data.cljc`

    (ns omn1.data
      (:require [om.next :as om :refer-macros [defui]]))
    
    (def app-state (atom {:current/user {:user/name "Fenton"}
                          :my-cars [{:make "Toyota"
                                     :model "Tacoma"
                                     :year "2013"}
                                    {:make "BMW"
                                     :model "325xi"
                                     :year "2001"}]}))
    
    (defn reader
      [{:keys [query state]} _ _]
      (let [st @state]
        {:value (om/db->tree query st st)}))
    
    (def parser (om/parser {:read reader}))
    
    (def reconciler
      (om/reconciler {:state app-state :parser parser}))

Okay there is a lot of stuff here, but it should be familiar from the
other two sources of information.  This doc is just filling out parts
I didn't understand or I felt needed a more pedantic. 

## Existing Documentation Deviation<a id="sec-3-1" name="sec-3-1"></a>

In the examples in the quick start, the reader function calls
`db->tree` in a different way, notice the second argument difference: 

    (om/db->tree key (get st k) st)

Whereas we are passing in the full state as the second parameter.  I
found the [quick start](https://github.com/omcljs/om/wiki/Thinking-With-Links%21#the-application-state) way didn't work for me when I had a simple
**property** query, i.e. our singleton query for current user.

## Fix up web page<a id="sec-3-2" name="sec-3-2"></a>

Actually the web page is quite messy.  We do manage to get the data to
the page, but we dont really display it very well.

When we have tabular data with rows, we create UI to handle each
individual row.

# Questions<a id="sec-4" name="sec-4"></a>

My webpage only has the output: `{:user {}}` with the following code.

    (ns omn1.core
      (:require
       [om.next :as om :refer-macros [defui]]
       [om.dom :as dom :refer [div]]
       [goog.dom :as gdom]))
    
    (defui MyComponent
      static om/IQuery
      (query [this] [:user])
      Object
      (render
       [this]
       (let [data (om/props this)]
         (div nil (str data)))))
    
    (def app-state (atom {:user {:name "Fenton"}}))
    
    (defn reader [{q :query st :state} _ _]
      (.log js/console (str "q: " q))
      {:value (om/db->tree q @app-state @app-state)})
    
    (def parser (om/parser {:read reader}))
    
    (def reconciler
      (om/reconciler
       {:state app-state
        :parser parser}))
    
    (om/add-root! reconciler MyComponent (gdom/getElement "app"))

When I check the browser console, I notice that my query is nil.  Why
doesn't it get passed into my reader function?