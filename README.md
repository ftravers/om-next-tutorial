<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Source Code</a></li>
<li><a href="#sec-2">2. In the beginning</a>
<ul>
<li><a href="#sec-2-1">2.1. Overview</a></li>
<li><a href="#sec-2-2">2.2. App State</a></li>
<li><a href="#sec-2-3">2.3. Component</a></li>
<li><a href="#sec-2-4">2.4. 1st Component</a></li>
<li><a href="#sec-2-5">2.5. The query</a></li>
<li><a href="#sec-2-6">2.6. Reader</a></li>
<li><a href="#sec-2-7">2.7. Other Wiring: Parser, Reconciler, add-root!</a></li>
<li><a href="#sec-2-8">2.8. Summary</a></li>
</ul>
</li>
<li><a href="#sec-3">3. Writing/Transacting/Mutating App State</a></li>
<li><a href="#sec-4">4. Datomic remotes</a></li>
<li><a href="#sec-5">5. Om-next login</a></li>
<li><a href="#sec-6">6. THE REMAINDER IS WORK IN PROGRESS - IGNORE</a></li>
<li><a href="#sec-7">7. Om Next plus Datomic Tutorial</a>
<ul>
<li><a href="#sec-7-1">7.1. Om Next Data Model</a></li>
<li><a href="#sec-7-2">7.2. Datomic Modeling</a></li>
<li><a href="#sec-7-3">7.3. Om Next UI</a></li>
<li><a href="#sec-7-4">7.4. Compare Queries</a></li>
</ul>
</li>
<li><a href="#sec-8">8. Datomic Again</a>
<ul>
<li><a href="#sec-8-1">8.1. Om Next Reader Return Value</a></li>
</ul>
</li>
</ul>
</div>
</div>

# Source Code<a id="sec-1" name="sec-1"></a>

The source code for this project can be found at the github repo:

    https://github.com/ftravers/omn1

Go ahead and clone that now.

Throughout this tutorial you'll see sections like:

GIT BRANCH: some-name

To see the code corresponding to that section, just checkout the
branch by that name.

# In the beginning<a id="sec-2" name="sec-2"></a>

## Overview<a id="sec-2-1" name="sec-2-1"></a>

Lets start with the absolute bare bones setup.  We'll display one
value from an om-next client data store.

## App State<a id="sec-2-2" name="sec-2-2"></a>

When I say data store, that is a glamorous word for a clojurescript
map wrapped in an atom.

    (def app-state (atom {:name "Bob"}))

## Component<a id="sec-2-3" name="sec-2-3"></a>

The word *component* is often used to describe a chunk of logically
related html code.  For example a table is often written as two
components, one for the table heading and column headers, and another
component that is repeatedly used for each row.

## 1st Component<a id="sec-2-4" name="sec-2-4"></a>

GIT BRANCH: simple-reading

Lets write the 'html' component that will display this greeting.

    (defui Greeter
      static om/IQuery
      (query [_] [:name])
    
      Object
      (render
       [this]
       (div nil "Hello " (:name (om/props this)))))

## The query<a id="sec-2-5" name="sec-2-5"></a>

In this component we *query* for `[:name]`.  If you've checked out the
code and run it, you will see the logging in this component outputs:

    {:name "Bob"}

So from the components perspective, it asks for `[:name]` and it gets
back: `{:name "Bob"}` according to our `app-state`:

    (def app-state (atom {:name "Bob"}))

## Reader<a id="sec-2-6" name="sec-2-6"></a>

In om-next we write *reader* functions.  These are the functions that
actually get the data for us.  Our reader function looks like this
mess:

    (defmethod reader :default
      [{st :state} key _]
      {:value (key (om/db->tree [key] @st @st))})

Lets examine the output of the log statements:

    [DEFAULT READER KEY]: :name

So `key` has the value `:name`.  So far this looks reasonable.

    [STATE]: {:name "Bob"}

So `@st` is nothing other than our app-state.

`om/db->tree` hydrates a query, the first argument, from the state
provided in the second/third arguments.  I'm actually a bit unclear
why this function takes state twice, maybe someone will enlighten ;).

    [OM DB->TREE]: {:name "Bob"}

Finally, the response is:

    [RESPONDING WITH]: 
    {:value "Bob"}

This may be a bit surprising.  The reader returns `{:value "Bob"}`,
but the component gets `{:name "Bob"}`.  I'm not sure why this is
setup this way, but this is readers work.  They will replace the
keyword `:value` with the keyword in the `key` second parameter of the
function, which in our case is `:name`.

## Other Wiring: Parser, Reconciler, add-root!<a id="sec-2-7" name="sec-2-7"></a>

The rest of the code is boilerplate required to wire things up.  The
`parser` wraps up the `reader` function.  The `reconciler` combines
the `parser` with the `app-state`.  Finally `om/add-root!` combines
the root component with the reconciler and mounts it into the `app`
html element in `index.html`.

## Summary<a id="sec-2-8" name="sec-2-8"></a>

This wraps up a basic *reading* component in om-next.  Other tutorials
out there cover this stuff already, and arguably in better detail, but
I wanted to lay out a foundation for further discussions.

# Writing/Transacting/Mutating App State<a id="sec-3" name="sec-3"></a>

Lets modify our value for name and see that get reflected in the
webpage.

First we write a mutate function:

    (defmethod mutate 'new-name
      [{state :state} ky params]
      {:value {:keys (keys params)}
       :action #(swap! state merge params)})

And we can call it from the REPL like so:

    (om.next/transact! reconciler '[(new-name {:name "joe"})])

Mutate functions should return a map with two keys `:value` and
`:action`.  `:value` should be a map of the keys that are going to be
updated by this transaction.  This helps om-next refresh those parts
of the DOM that are connected to the values contained in those keys.

`:action` should return a function that moves the app-state atom from
its current value to a new value.

So now we can modify in our app-state atom as om-next wants us to.

GIT BRANCH: simple-mutate

Let's now look out how we'd integrate an external server into our
setup.

# Datomic remotes<a id="sec-4" name="sec-4"></a>

To setup a remote we create a function like so:

    (defn make-remote-req
      [qry cb]
      (cb {:name "Fred"}))
    
    (defmethod reader :default
      [{st :state} key _]
      (log "default reader" key)
      {:value (key (om/db->tree [key] @st @st))
       :remote true                         ; This is added
       })
    
    (def reconciler
      (om/reconciler
       {:state app-state
        :parser parser
        :send make-remote-req               ; This is added
        }))

So here we define a function that is our remote call.  It just calls
the callback `cb` with the data to be merged into the `@app-state`.

The other two parts are the wiring.

GIT BRANCH: remote-integration

Now that we have the basics of reading/writing to the local app-state,
and have simulated reading from the remote state, let's try to create
a more realistic example.  The login use-case.

# Om-next login<a id="sec-5" name="sec-5"></a>

The way this login will work is that there will be username and
password field along with a submit button.  This will submit both to
the server and the server will respond with a user valid/invalid
response.  First we'll simulate this without a backend.

GIT BRANCH: login-step1

# THE REMAINDER IS WORK IN PROGRESS - IGNORE<a id="sec-6" name="sec-6"></a>

# Om Next plus Datomic Tutorial<a id="sec-7" name="sec-7"></a>

This tutorial will simulate a data flow between om-next and datomic.
A user will enter a car make, like BMW or Toyota, and a list of models
will be sent from the backend to the front end.

## Om Next Data Model<a id="sec-7-1" name="sec-7-1"></a>

On the front end we'll model this with the following map structure:

    {:current/car
     {:car/make "Toyota"
      :make/models [{:model "Tacoma"}
                    {:model "Tercel"}]}}

## Datomic Modeling<a id="sec-7-2" name="sec-7-2"></a>

To support this on the datomic side we'll have data stored like:

    [{:car/make "Toyota"
      :make/models [{:model "Tacoma"}
                    {:model "Tercel"}]}
     {:car/make "BMW"
      :make/models [{:model "325xi"}
                    {:model "x5"}]}]

Our pull pattern in our query will look like:

    [{:make/models [:model]}]

## Om Next UI<a id="sec-7-3" name="sec-7-3"></a>

The queries of our Om Next components are:

    (defui CarModel
      (query [this] [:model])
    (defui CarRoot
      (query [this] [:current/car {:make/models (om/get-query CarModel)}])

## Compare Queries<a id="sec-7-4" name="sec-7-4"></a>

Om Next Query

    [:current/car {:make/models [:model]}]

Datomic Pull Pattern

    [{:make/models [:model]}]

# Datomic Again<a id="sec-8" name="sec-8"></a>

So our query on the datomic takes in a make as a variable and returns
the associated models.

We could say the input would be two values.  The first is the where
clause.  Get the entity who's `:car/make` attribute has the value
`Toyota`.  We can express this with a datomic where clause that looks
like:

    [?e :car/make "Toyota"]

In this case, the found entity is stored in the variable `?e`.  

Next we have to say, with this found entity, what data of it do we
want back?  Remember the shape of the data in Datomic looks like:

    [{:car/make "Toyota"
      :make/models [{:model "Tacoma"}
                    {:model "Tercel"}]}
     {:car/make "BMW"
      :make/models [{:model "325xi"}
                    {:model "x5"}]}]

So we can say, well, we want the associate model names.  A pull
pattern that looks like this:

    [{:make/models [:model]}]

or more completely:

    '[(pull ?e [{:make/models [:model]}]) ...]

which basically reads: from the found entity `?e` find the reference
attribute `:make/models`.  Follow that reference, and from the found
children entities, get the values for the `:model` attribute.

When we run this in datomic we predictably get the pull pattern
*filled* out:

    #:make{:models [{:model "Tacoma"} {:model "Tercel"}]}

## Om Next Reader Return Value<a id="sec-8-1" name="sec-8-1"></a>

If I query for a key, then the returned value from the reader function
should be a map with a key
