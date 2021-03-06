---
{:title "Messaging"
 :sequence 2
 :base-ns 'immutant.messaging
 :description "Simple creation and usage of distributed queues and topics"}
---

If you're coming from Immutant 1.x, you may notice that the messaging
artifact has been renamed (`org.immutant/immutant-messaging` is now
`org.immutant/messaging`), and the API has changed a bit. We'll point
out the notable API changes as we go.

## The API

The messaging [API] is backed by
[HornetQ], which is an implementation of [JMS]. JMS provides two
primary destination types: *queues* and *topics*. Queues represent
point-to-point destinations, and topics publish/subscribe.

To use a destination, we need to get a reference to one via the
[[queue]] or [[topic]] functions, depending on the type required. This
will create the destination if it does not already exist. This is a
bit different than the 1.x API, which provided a single `start`
function for this, and determined the type of destination based on
conventions around the provided name. In 2.x, we've removed those
naming conventions.

Once we have a reference to a destination, we can operate on it with
the following functions:

* [[publish]] - sends a message to the destination
* [[receive]] - receives a single message from the destination
* [[listen]] - registers a function to be called each time a message
  arrives at the destination

If the destination is a queue, we can do synchronous messaging
([request-response]):

* [[respond]] - registers a function that receives each request, and
  the returned value will be sent back to the requester
* [[request]] - sends a message to the responder

Finally, to deregister listeners, responders, and destinations, we
provide a single [[stop]] function. This is another difference
from 1.x - the `unlisten` and `stop` functions have been collapsed to
`stop`.

### Some Examples

You should follow the instructions in the [installation] guide to
set up a project using Immutant 2.x, and in addition to
`org.immutant/messaging` add `[cheshire "5.3.1"]` to the project
dependencies (we'll be encoding some messages as JSON in our examples
below, so we'll go ahead and add
[cheshire](https://github.com/dakrone/cheshire) while we're at it).
Then, fire up a REPL, and require the `immutant.messaging` namespace
to follow along:

```clojure
(require '[immutant.messaging :refer :all])
```

First, let's create a queue:

```clojure
(queue "my-queue")
```

That will create the queue in the HornetQ broker for us. We'll need a
reference to that queue to operate on it. Let's go ahead and store
that reference in a var:

```clojure
(def q (queue "my-queue"))
```

We can call `queue` any number of times - if the queue already exists,
we're just grabbing a reference to it.

Now, let's register a listener on our queue. Let's just print every
message we get:

```clojure
(def listener (listen q println))
```

We can publish to that queue, and see that the listener gets called:

```clojure
(publish q {:hi :there})
```

You'll notice that we're publishing a map there - we can publish
pretty much any data structure as a message. By default, that message
will be encoded using [edn]. We also support other encodings, namely:
`:fressian`, `:json`, and `:none`. We can choose a different encoding
by passing an :encoding option to `publish`:

```clojure
(publish q {:hi :there} :encoding :json)
```

If you want to use `:fressian` or `:json`, you'll need to add
`org.clojure/data.fressian` or `cheshire` to your dependencies to
enable them, respectively.

We passed our options to `publish` as keyword arguments, but they can
also be passed as a map:

```clojure
(publish q {:hi :there} {:encoding :json})
```

This holds true for any of the messaging functions that take options.

We're also passing the destination reference to `publish` instead of the
destination name. That's a departure from 1.x, where you could just pass the
destination name. Since we no longer have conventions about how queues and
topics should be named, we can no longer determine the type of the
destination from the name alone.

We can deregister the listener by either passing it to `stop` or
calling `.close` on it:

```clojure
(stop listener)
;; identical to
(.close listener)
```

Now let's take a look at synchronous messaging. Let's create a new
queue for this (you'll want to use a dedicated queue for each
responder) and register a responder that just increments the request:

```clojure
(def sync-q (queue "sync"))

(def responder (respond sync-q inc))
```

Then, we make a request, which returns a [Future] that we can
dereference:

```clojure
@(request sync-q 1)
```

The responder is just a fancy listener, and can be deregistered the
same way as a listener.

## Remote contexts

To connect to a remote HornetQ instance, you'll need to create a
remote context (via the [[context]] function), and use it when getting
a reference to the destination:

```clojure
(with-open [context (context :host "some-host" :port 5445)]
  (publish
    (queue "foo" :context context)
    "a message"))
```

A few things to note about the above example:

* We're using `with-open` because you need to close any contexts you make
* We're passing the context to `queue`, which causes `queue` to
  just return a reference to the remote queue, *without* asking
  HornetQ to create it. You'll need to make sure it already exists.
* We don't need to pass the context to `publish`, since it will
  reuse the context that was used to create the destination
  reference.

## Reusing contexts

By default, Immutant creates a new context object for each `publish`,
`request` or `receive` call. Creating a context isn't free, and incurs
some performance overhead. If you plan on calling any of those
functions in a tight loop, you can gain some performance by creating
the context yourself (via the [[context]] function):

```clojure
(with-open [context (context)]
  (let [q (queue "foo")]
    (dotimes [n 10000]
      (publish q n :context context))))
```

## Context modes

When creating a context, you can pass a `:mode` option that controls
how messages will be acknowledged and delivered.

Immutant provides three modes:

* `:auto-ack` - when this mode is active, receipt of a message is
  automatically acknowledged when a `receive` call completes. This
  mode doesn't affect publication - `publish` calls will complete
  immediately. This is the default mode for contexts.

* `:client-ack` - when this mode is active, you are responsible for
  acknowledging the message manually by calling `.acknowledge` on the
  Message object. This means you need to get the raw message (by
  passing `:decode? false` to `receive`). This mode doesn't affect
  publication - `publish` calls will complete immediately.

* `:transacted` - when this mode is active, you are responsible for
  committing or rolling back (by calling `.commit` or `.rollback` on
  the context, respectively) any actions performed on the
  context. This applies to publishes *and* receives.

:auto-ack is the default for contexts, :transacted for listeners.

If a context is created with `:xa? true`, the `:mode` option is
ignored. See the [Transactions Guide] for more details.

## Context modes for listeners

The [[listen]] and [[respond]] functions take a `:mode` option, which
is used *instead* of the mode of any `:context` option (listeners need
a context per thread, so create and manage their own contexts). The
`:mode` option to these functions takes the same modes as above, with
the following behavioral differences (note that `:transacted` is the
default for listeners, *not* `:auto-ack`):

* `:auto-ack` - when active, the receipt of the message will
  automatically be acknowledged when the handler function completes.

* `:client-ack` - when active, you are responsible for
  acknowledging the message manually by calling `.acknowledge` on the
  Message object. This means you need to get the raw message (by
  passing `:decode? false` to `listen`).

* `:transacted` - when active, `.commit` is called on the context
  automatically if the handler function completes successfully. If it
  throws an exception, `.rollback` is called on the context. Any
  messaging operations that take a context will use the context that
  is active for the listener itself (if not passed one
  explicitly). This means that any messaging operations within the
  handler function are also transacted. This is the default mode for
  listeners.

If you need to use distributed transactions (XA) within a listener
function, you are responsible for demarcating the transaction. See the
[Transactions Guide] for more details.

## HornetQ configuration

When used outside of WildFly, we configure [HornetQ] via a pair of xml
files. If you need to adjust any of the HornetQ
[configuration options], you can provide a copy of one (or both) of
those files (`hornetq-configuration.xml` and `hornetq-jms.xml`, which
should be based off of the [default versions]) on your application's
classpath and your copies will be used instead of the default
ones. When making changes to these files, be careful about changing
existing settings, as Immutant relies on some of them.

We've also exposed a few HornetQ settings as system properties, namely:

| Property             | Description                                                 | Default           |
|----------------------|-------------------------------------------------------------|-------------------|
| `hornetq.data.dir`   | The base directory for HornetQ to store its data files      | `./hornetq-data/` |
| `hornetq.netty.port` | The port that HornetQ will listen on for remote connections | `5445`            |
| `hornetq.netty.host` | The host that HornetQ will listen on for remote connections | `localhost`       |

Note that any custom xml or system properties will be ignored when
running inside WildFly - you'll need to make adjustments to the
WildFly configuration to achieve similar effects.

## More to come

That was just a brief introduction to the messaging API. There are
features we've yet to cover (durable topic subscriptions,
transactional sessions)...

[HornetQ]: http://hornetq.jboss.org/
[API]: immutant.messaging.html
[JMS]: https://en.wikipedia.org/wiki/Java_Message_Service
[installation]: guide-installation.html
[request-response]: https://en.wikipedia.org/wiki/Request-response
[Future]: http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Future.html
[edn]: https://github.com/edn-format/edn
[default versions]: https://github.com/projectodd/wunderboss/blob/{{wunderboss-tag}}/modules/messaging/src/main/resources/
[configuration options]: https://docs.jboss.org/hornetq/2.4.0.Final/docs/user-manual/html_single/#server.configuration
[Transactions Guide]: guide-transactions.html
