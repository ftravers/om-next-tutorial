<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Introduction</a></li>
<li><a href="#sec-2">2. App Database</a>
<ul>
<li><a href="#sec-2-1">2.1. Explaination</a></li>
</ul>
</li>
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

Now lets test working with this database, create a `*.cljc` file with
the following contents and fire up your **CLOJURE** not clojure-script
REPL.

    (ns omn1.core2
      (:require [om.next :as om]))
    (def app-state (atom {:current/user {:user/name "Fenton"}}))

Now in your REPL execute the following command:

    omn1.core2> (om/db->tree [:current/user] @app-state @app-state)
    #:current{:user #:user{:name "Fenton"}}

The `#: ... {` syntax is just the namespacing representation.  This is
equivalent to:

    {:current/user {:user/name "Fenton"}}

## Explaination<a id="sec-2-1" name="sec-2-1"></a>

We have seen how to store **singleton'ish** information.  We use a map.
We have also seen how to extract our information, the key here is the
array with the keyword in it:

    [:current/user]

This is the first introduction to what we call the **Query Syntax**.
Here is a reference to
[[<https://awkay.github.io/om-tutorial/#!/om_tutorial.D_Queries][query>
syntax].

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