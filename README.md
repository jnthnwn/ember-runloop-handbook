# Ember JS Runloop Handbook

by [Eoin Kelly](https://twitter.com/eoinkelly)

![Creative Commons License](https://i.creativecommons.org/l/by-sa/4.0/88x31.png)

## Current status: Such incomplete, much TODO.

`TODO: remove this when complete`
Very much a work in progress. Currently still compiling my notes and researching
topics.

## Contributing

You should. :-). If you spot any of the (inevitable) errors, omissions, things
which are unclear you would be doing me a great favour by opening an issue.


# Section: Introduction

```
    do this last

    intro the 2 sections
        why the runloop
        what it is
        how to use it
    (address the "do I need to know it" question)
    my goals for reader after reading this:
        don't be scared of the runloop
        understand how to wield it skillfully

```

## What will we do?

We are about to take a deep dive into the Ember JS runloop. Why? Because eventually you need to.
Together we will answer these questions:

2. What problems does the runloop address.
1. What even is this "runloop" thing?
3. How can I use it.

Hopefully by the end of this we will have the required background to understand the
[Official Ember Run-loop guide](http://emberjs.com/guides/understanding-ember/run-loop/) and the [Ember
API docs](http://emberjs.com/api/) on the topic.

# section: why a runloop?

```
    background (the environment ember lives in)
        JS event loop
        all about events:
            where they come from
            what they are
        examples of how vanialla JS handles them
        show timeline of how vanilla JS responds to events
        show how ember is just a fancy example of handling events
        discuss the ways in which the vanilla approach doesn't scale well if you are doing lots of work
        end with a clear statement of the problems that the runloop solves
```

## Background

When does Javascript run?

While parsing your HTML the browser executes each `<script>` that it finds as it
finds them (there are some exceptions to this e.g. `defer`). This "setup phase" happens well
before the user sees any content or gets a chance to do anything so how does it
help?

The browser has a built-in set of _events_ that it watches for all the time:
These include

* User moved their mouse
* The DOM has been completely built
* User clicked on something
* All assets have been loaded on the page (`window.load`
* User typed a key in a form input

How does the browser know when to bother JS about an event? Well JS is lazy but
well perpared!

During its setup phase, JS prepared its work space (or `mise en place` if you
prefer) - it created the objects it will need later to respond to orders
(events) from the browser and told the browser in detail what events it cares
about e.g.

> Hey browser, you need to wake me up whenever the user scrolls the page or
> clicks on an element with the `#do-stuff` id.

The description above makes it look like the browser the one giving all the
orders but the browser is a team player and has a few things it can do to help
JS do its job.

1. Timers. JS can use the browser like an alarm clock

> Hey browser, there is a chunk of work I need to do in 5 seconds. Can you
> make a new "event" for me that will fire when that time has elapsed so you
> can wake me up to do that work.

2. Talking to other systems. If JS needs to send or receive data to other
computers it asks the browser to do it and the browser promises to wake JS up
again and report back how it got on

> JS: Hey browser, I want to get whatever data is at http://foo.com/things.json
> please.
> Browser: Sure thing but it might take a while (networks can be slow), I'll
> try to get that data and wake you up again when it is done. What do you
> want me to do when it comes back?
> JS: I have two chunks of work ready to go (one for a successful and one
> for a failure) so just wake me up to run them when you finish
> Browser: cool.

Developers call this _talking to other systems_ stuff Web APIs e.g. XHR requests, Web workers
etc.

### What does "wake up JS" mean?

There is no one Javascript "intellegence" to wake up - there is only chunks of
work. During the setup phase the browser considers a whole script a chunk of
work and will run it from top to bottom but for the rest of the time, the
_chunk of work_ is a Javascript function. JS functions are neatly packaged units
of work that can be passed around and stored so are perfect for this job.

The browser does not wake up JS and ask it to figure out what to do in response
- it needs JS to tell it exactly what chunk of work JS wants to run


JS can use these services of the browser both during its setup phase and while
responding to another event e.g. part of JS response to a "click" event on a
certain element might be to retrieve some data from the network and also
schedule a timer to do some future work.

The general pattern of how javascript does work is
1. In the short _setup phase_ the browser runs each script it finds on the page
from start to finish. JS uses this as time to do some preparation for its real
job.
2. In response to events. Many events come from the user but JS can also
schedule events for itself by using the many services (web APIs) that the
browser provides.

Most of a JS applications life is spent in section 2 above so we can visualise
our app as a "thing which is mostly sleeping but when woken by the browser
responds quickly to the event that work it before going back to sleep again"

```
TODO:
    talk about event bubbling and capturing but not in heaps of detail
        they just need to understand that listening on the bubbling is more
        usual but that the capturing is possible so they will understand how
        they could make JS run before ember runs its response
```

### A little more on event listening

We need to know a little more detail on how Javascript tells the browser what
events it cares about.

The browser takes the string of HTML it got from the server and uses it to
create  a _tree like_ structure in memory. We call this "tree
structure" the Document Object Model or DOM for short. It is commonly called a
"tree structure" but is usually visualied as an "upside down tree" or the root
system of a tree.

[diagram of simple DOM tree here]

The interface that the browser presents to JS makes it look like events come
"from" particular nodes in this tree.

Lets discuss

1. How JS can register its interest in hearing about certain events from
certain nodes
2. How the browser figures out what JS chunks of work to call when an event
actually arrives

Timeline of response to browser event
![graph](https://docs.google.com/drawings/d/10HAJdly4R_31NE0n7Lt8XcLr_TlYwfsal-SZl7pINsM/pub?w=498&amp;h=749)

## An example with code

Lets look again at our sample set of events and see how JS might schedule work

* User moved their mouse
* The DOM has been completely built
* User clicked on something

```html
<!DOCTYPE HTML>
<html>
  <head>
    <meta charset="utf-8">
    <title>Plain old Javascript way</title>
    <script>
        // give the browser a function ("chunk of work") to run when certain
        // event happens

    </script>
  </head>
  <body>
    <button id="do-thing">Do the thing!</button>
  </body>
</html>
```

same

```html
<!DOCTYPE HTML>
<html>
  <head>
    <meta charset="utf-8">
    <title>jQuery way</title>
    <script src="node_modules/jquery/dist/jquery.js"></script>
    <script>
        // Tell the browser (via jQuery) that we have a function (chunk of work)
        // we want it to run when DOM is fully parsed and ready i.e.
        // DOMContentLoaded event happens
        $(document).ready(
            function () { // <-- the chunk of work
                // I am run by the browser when the DOM is ready. It is a common
                // pattern for me to register JS for other browser events e.g.
                // "click" because the DOM might not have been complete before
                // this event fired.


                // TODO: in vanilla js do I have to wait for DOMContentLoaded
                // before I can register listeners?
            }
        );
    </script>
  </head>
  <body>
    <button id="do-thing">Do the thing!</button>
  </body>
</html>
```


## Enter the Ember!

### Things we already know

We have refreshed how Javascript works and we know that your Ember app is JS
(wow!) so we already know quite a bit about how Ember works:

* Apart from when the code is first found, all Ember framework and application
  code is run in response to "events" from the browser.
* The `DOMContentLoaded` event is significant in the life of an Ember app. It tells
  it that it now has a full DOM to play with so much of the "setup work"
  (registering for event handlers etc.) happens in response to this event.
* Your Ember app can schedule its own events by asking the browser to do some work
  on its behalf (e.g. AJAX requests) or simply by asking the browser to be its
  alarm clock (e.g. `setTimeout`)

## How ember listens for events

The Ember docs have a list of [events Ember listens for by
default](http://emberjs.com/api/classes/Ember.View.html#toc_event-names). These
are 28 the entry points into our code. Anytime Ember does anything it is in response to an event.

Ember registers handlers for these events in the same way you would if you were
doing it with jQuery.

* It listens to events on only one element in the DOM: `<body>` (or `rootElement` if you specified one)
* It listens on the normal "bubbling" phase
* The handler that it registers decides what the Ember response to the event should be.

Implications:

* It is possible and likely to run event handlers _before_ ember runs. Any event handlers you add manually to elements lower down in the DOM tree will be run
  **before** Ember sees them. Hint: this is a bad idea.


TODO: be consistent in use of "listener", "handler" etc.


## Where does the framework end an my app begin

How does your Ember _application_ relate to the Ember _framework_?
The framework listens for events and runs your _application_ code to provide a
_meaningful_ response.

The machinery for responding to events is part of Ember itself but it does not
have a meaningful response without application code.

For example if the
user is on `/#/blog/posts` and clicks a link to go to `/#/authors/shelly` the
Ember _framework_ will recieve the click event but it won't be able to do anything
meaningful with it without your _application_ code like

* The Router in your applicaiton tells the framework what Route objects are part
* of the URL
* The Route objects themselves
* lots of other stuff

## Digging deeper into Ember event response

So the pattern of how ember works is have periods of intense activity in
response to some event and then go back to being idle until the next event
happens and the browser runs the Ember event handler again.

    [graph of timeline with spaced out chunks of activity]

Lets dig a little deeper into these periods of intense activity.

    [graph of a single chunk from graph above zoomed in to see more detail]

We already know that the first code to get run in reponse to an event is the
handler function that Ember registered with the browser. What happens after
that? Lets consider an imaginary example of how a simpler, no runloop Ember
might respond:

    [explain here the kinds of work that ember does and make it clear that there are
    natural phases to it]
    [walk through a simple example showing how it would look if ember just did
    owrk as it needed to]

We can see from this example that the work can be grouped into just a few
categories:

1. Bindings need to be synced
2. ???
3. Update the DOM (rendering)
4. Manipulate the new DOM (after rendering)

and we can also see that our _do work as you need it_ approach means that these
types of work get interleaved. In fact there are some significant downsides to this:

1. It is Inefficent. Every time we changed the DOM in the example above the browser
did a layout and paint - these are expensive operations which can create a laggy
UI.
2. A pain to work with if we want to do stuff with the newly changed DOM. Since
the rendering happened a little bit at a time it is difficult to know when we
can safely work with the new state of the DOM .e.g. if we wanted to animate an
element into view

How might we solve this? How does Ember solve this?

Instead of just doing work as it finds it, Ember schedules the work on an
internal set of work queues (one for each phase).

When all the work needed to
respond to the current event has been scheduled it runs the queues in a sensible
order. Jobs on the queue can add other jobs to the queu.

This is the Ember Runloop.

1. Open the queues for business and start accepting jobs
2. Close the queues and run all jobs on them. The [Runloop Guide] has an
excellent visualisaiton of the algorithm works but in brief,
    * Ember will make sure that all previous queues are empty before moving on to the next one.
    * If some job


Things the runloop is not

* At first glance it may seem that the Runloop has two distinct phases
    1. schedule work
    2. perform the work
  but this is subtly incorrect. Functions that have been scheduled on a Runloop queue
  can themselves schedule function on **any** queue in the same runloop. It is
  true that once the runloop starts executing the queues that code outside those
  queues cannot schedule new jobs. In a sense the initial set of jobs that are
  scheduled are a "starter set" of work and ember commits to performing and also
  performing any jobs that result from those jobs - Ember is a pretty great
  person to have working for you! :-)

A more correct view is

1. Open Runloop and accept any job given by any code.
2. Close the "open access".
3. Start executing any jobs. Accept any new jobs that are scheduled from jobs
already on the runloop.
4. Work thorugh each queue according to the runloop algorithm (TODO: more here)


* It is not a "singleton" gateway to all DOM access

There is no concept of "the" runloop, there can be multiple instances of "a
runloop"
TODO: be consistent in terminology of things on the runloop: jobs vs functions
vs callbacks vs handlers

  It does not have two distinct phases
A consequence of this two phase "schedule and execute" approach is that all DOM

TODO: it is not really "two phase" because while flushing the queue existing
jobs can add work to another queue



A consequence of this approach is that all DOM rendering happens on one queue.
It is not correct to say that the Runloop is the "gatekeeper" to all DOM access
in Ember, rather that "coordinated DOM access" is a pleasant (and deliberate!)
side-effect of this approach.

This is an important and subtle distinction between the Ember Runloop and
Angulars "dirty checking" algorithm.



A discussion of the full timeline (including ember):

```
    we should use a cut down version of this here and the full one later
-- some custom JS before the ember.js script e.g. jquery, your own stuff
-- the ember.js script
    executed as soon as the browser finds it
    creates a bunch of objects in memory
    adds handlers to a number of browser events. in particular adds a callback
    that will "boot" the ember app when DOMContentLoaded is fired by the browser
-- Ember is sitting in memory, waiting, doing nothing
-- Your app script is found and executed by browser
    it registers a bunch of new objects in memory and configures some of the
    existing ember ones. Ember can now do useful work when it boots
-- some other JS is found and executed

-- ... relatively speaking a long time passes ...

-- DOMContentLoaded is fired!
    ember boots - it creates a bunch of new objects in memory and draws stuff to
    the element in the DOM that you gave it as rootElement

-- JS goes to sleep waiting for the next event

-- Some browser event happens e.g. click
    ember has registered a handler for many browser events so it responds
    part of its response involves running code from your application objects
    e.g. route, controller. There are many things Ember can do in response e.g.
    send data to a server, draw new things on the page but whatever they are
    they are registered ahead of time and run now.

-- JS goes to sleep waiting for the next event

Questions:
How can I schedule JS to run *before* Ember?
    A: you probably shouldn't
    Add listener directly to an element that is not `<body>`
    Add listener to the capturing phase

Without knowing the internals of how Ember handles events it is difficult to
definitely get in front of it

We want to play nicely with Ember
```



Timeline of response to browser event
![graph](https://docs.google.com/drawings/d/10HAJdly4R_31NE0n7Lt8XcLr_TlYwfsal-SZl7pINsM/pub?w=498&amp;h=749)

# section: what is the runloop?

```
    the ember solution: run-loop
        demo how to instrument the runloop in a simple app
    go ghrough some detailed examples of event handling w. runloop added
        explain the bandaid on "non runloop aware code" that is autoruns
    explain granularity of the api

    discuss how often runloops happen, how long they last
    trade-offs of the runloop
        - extra complexity to understand
        + better performance

    briefly discuss some alternate solutions
        (mostly cover how they are/are not solving the same problems, not so
        much *how* they work
        angular dirty checking
            what problems does it solve (compare to the set the runloop solves)
            trade-offs compared to runloop
        react
            what problems does it solve (compare to the set the runloop solves)
            trade-offs compared to runloop
```


# Section: How do I use the runloop?

```
    the API
        give a high-level overview, help people mentally categorise it
        what are the main categories of methods
            which ones create new runloops
            which ones assume an existing one
            which ones perform a runloop cycle synchronously
            which ones do that at some time in the future

    when do I _need_ to know about runloop API
        1. non ember js - show w. timeline how using vanilla JS outside runloop causes problem in ember

    How is runloop behaviour different in testing?
        explain why auto-run is turned off

    How to use the runloop API
        this is well covered in the guide, refer mostly to it
```

# Appendices

## Sources

The primary documentation for the Ember runloop is [Official Ember Run-loop
guide](http://emberjs.com/guides/understanding-ember/run-loop/) and the [Ember
API docs](http://emberjs.com/api/)

These are other sources I studied in compiling this research:

* [Ember source code](https://github.com/emberjs/ember.js)
* Books
    * [Developing an Ember Edge](http://bleedingedgepress.com/our-books/developing-an-ember-edge/)
    * [Ember.js in Action](http://www.manning.com/skeie/)

