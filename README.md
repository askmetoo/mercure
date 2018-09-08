# Mercure, Server-Sent Live Updates
*Protocol and Reference Implementation*

Mercure is a protocol allowing to push data updates to web browsers and other HTTP clients in a fast, reliable and battery-efficient way.
It is especially useful to publish real-time updates of resources served through web APIs, to reactive web and mobile apps.

In addition to the full specification, a reference, production-grade implementation of **a Mercure server** (the hub) is provided in this repository. It is written in Go (golang) and is a free software licensed under the AGPL license.
It also includes a library that can be used in any Go application to implement the Mercure protocol directly (without a hub).

Mercure in a few words:

* native browser support, no lib nor SDK required (built on top of [server-sent events](https://www.smashingmagazine.com/2018/02/sse-websockets-data-flow-http2/))
* compatible with all existing servers, even those who don't support persistent connections (serverless, PHP, FastCGI...)
* builtin connection re-establishment and state reconciliation
* [JWT](https://jwt.io/)-based authorization mechanism (securely dispatch an update to some selected subscribers)
* performant, leverages [HTTP/2 multiplexing](https://developers.google.com/web/fundamentals/performance/http2/#request_and_response_multiplexing)
* designed with [hypermedia in mind](https://en.wikipedia.org/wiki/HATEOAS), also supports [GraphQL](https://graphql.org/)
* auto-discoverable through [web linking](https://tools.ietf.org/html/rfc5988)
* message encryption support
* can work with old browsers (IE7+) using an `EventSource` polyfill
* [connection-less push](https://html.spec.whatwg.org/multipage/server-sent-events.html#eventsource-push) in controlled environments (e.g. browsers on mobile handsets tied to specific carriers)

Example implementation of a client in JavaScript:

```javascript
// The subscriber subscribes to updates for the https://example.com/foo topic
// and to any topic matching https://example.com/books/{name}
const params = new URLSearchParams([
    ['topic', 'https://example.com/foo'],
    ['topic', 'https://example.com/books/{name}'],
]);
const eventSource = new EventSource(`https://hub.example.com?${params}`);

// The callback will be called every time an update is published
eventSource.on message = function ({data}) {
    console.log(data);
};
```

To dispatch an update (in any JS environment, including Node with the fetch polyfill):

```javascript
// ...
```

Example Use Cases:

**Live Availability**

* a Progressive Web App retrieves the availability status of a product from a REST API and displays it: only one is still
  available
* 3 minutes later, the last product is bought by another customer
* the PWA's view instantly show that this product isn't available anymore

**Collaborative Editing**

* a webapp allows several users to edit the same document concurently
* changes made are immediately broadcasted to all connected users

**Mercure gets you covered!**

## Protocol Specification

### Terminology

* Topic: An HTTP [RFC7230](https://tools.ietf.org/html/rfc7230) (or HTTPS [RFC2818](https://tools.ietf.org/html/rfc2818)) topic URL. The unit to which one can subscribe to changes.
* Publisher: An owner of a topic. Notifies the hub when the topic feed has been updated. As in almost all pubsub systems, the publisher is unaware of the subscribers, if any. Other pubsub systems might call the publisher the "source". Typically a website or a web API.
* Subscriber: A client application that subscribes to real-time updates of topics (typically a Progressive Web App or a Mobile
  App).
* Hub: A server that handles subscription requests and distributes the content to subscribers when the corresponding topics
  have been updated (a Hub implementation is provided in this repository). Any hub MAY implement its own policies on who can use it.

### Discovery

The publisher SHOULD advertises the URL of one or more hubs to the subscriber, allowing it to receive live updates when topics are updated.
If more than one hub URL is specified, it is expected that the publisher notifies each hub, so the subscriber may subscribe to one or more of them.

The publisher SHOULD include at least one Link Header [RFC5988](https://tools.ietf.org/html/rfc5988) with `rel=mercure-hub` (a hub link header). The target URL of these links MUST be a hub implementing the Mercure protocol.

Note: this relation type has not been [registered](https://tools.ietf.org/html/rfc5988#section-6.2.1) yet. During the meantime, the relation type `https://git.io/mercure` can be used instead.

The publisher MAY provide the following target attributes in the Link headers:

* `last-event-id`: the globally unique identifier of the last event dispatched by the publisher at the time of the generation of this resource. If provided, it must be passed to the hub through a query parameter called `Last-Event-ID` and will be used to ensure that possible updates having been made during between the resource generation time and the connection to the hub are not lost (see the connection re-establishment section). If this attribute is provided, the publisher MUST always set the `id` parameter when sending updates to the hub.
* `content-type`: the content type of the updates that will pushed by the hub. If omited, the subscriber MUST assume that the content type will be the same than the one of the original resource. Setting the `content-type` attribute is especially useful to hint that partial updates will be pushed, using formats such as JSON Patch ([RFC6902](https://tools.ietf.org/html/rfc6902)) or JSON Merge Patch ([RFC7386](https://tools.ietf.org/html/rfc7386)).
* `key-set=<JWKS>`: the key(s) to decrypt updates encode in the JWKS (JSON Web Key Set) format (see the Encryption section).

All these attributes are not optional.

The publisher MAY also include one Link Header [RFC5988] with `rel=self` (the self link header). It SHOULD contain the canonical URL for the topic to which subscribers are expected to use for subscriptions. If the Link with `rel=self` is ommitted, the current URL of the resource MUST be used as fallback.

Minimal example:

    GET /books/foo.jsonld HTTP/1.1
    Host: example.com

    HTTP/1.1 200 Ok
    Content-type: application/ld+json
    Link: <https://hub.example.com/updates/>; rel="https://git.io/mercure"

    {"@id": "/books/foo.jsonld", "foo": "bar"}

Links embedded in HTML or XML documents (as defined in the WebSub recommendation) MAY also be supported by subscribers.

Note: the discovery mechanism described in this section [is strongly inspired from the one specified in the WebSub recommendation](https://www.w3.org/TR/websub/#discovery).

### Subscriptions

The subscriber subscribes to an URL exposed by a hub to receive updates of one or many topics.
To subscribe to updates, the client opens an HTTPS connection following the [Server-Sent Events specification](https://html.spec.whatwg.org/multipage/server-sent-events.html) to the hub's subscription URL advertised
by the Publisher.
The connection SHOULD use HTTP/2 to leverage mutliplexing and other advanced features of this protocol.

The subscriber specifies the list of topics to get updates for by using one or several query parameters named `topic`.
The value of these query parameters MUST be [URI templates (RFC6570)](https://tools.ietf.org/html/rfc6570).

Note: an URL is also a valid URI template.

The protocol doesn't specify the maximum number of `topic` parameters that can be sent, but the hub MAY apply an arbitrary limit.

The [`EventSource` JavaScript interface](https://html.spec.whatwg.org/multipage/server-sent-events.html#the-eventsource-interface)
MAY be used to establish the connection.
Any other appropriate mechanism including but not limited to [readable streams](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API/Using_readable_streams) and [XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/Using_XMLHttpRequest) (used by popular polyfills) MAY
also be used.

The hub sends updates concerning all subscribed resources matching the provided URI templates.
The hub MUST send these updates as [`text/event-stream` compliant events](https://html.spec.whatwg.org/multipage/server-sent-events.html#sse-processing-model).

The `data` property MUST contain the new version of the topic. It can be the full resource, or a partial update
by using formats such as [JSON Patch](https://tools.ietf.org/html/rfc6902) or
[JSON Merge Patch](https://tools.ietf.org/html/rfc7386)).

All other properties defined in the Server-Sent Events specification MAY be used and SHOULD be supported by hubs.

The resource SHOULD be represented in a format with hypermedia capabilities such as [JSON-LD](https://www.w3.org/TR/json-ld/), [Atom](https://tools.ietf.org/html/rfc4287), [XML](https://www.w3.org/XML/) or [HTML](https://html.spec.whatwg.org/).

[Web Linking](https://tools.ietf.org/html/rfc5988) SHOULD be used to indicate the IRI of the resource sent in the event.
When using Atom, XML or HTML as serialization format for the resource, the document SHOULD contain a `link` element with a `self` relation containing the IRI of the resource.
When using JSON-LD, the document SHOULD contain an `@id` property containing the IRI of the resource.

Example:

```javascript
// The subscriber subscribes to updates for the https://example.com/foo topic
// and to any topic matching https://example.com/books/{name}
const params = new URLSearchParams([
    ['topic', 'https://example.com/foo'],
    ['topic', 'https://example.com/books/{name}'],
]);
const eventSource = new EventSource(`https://hub.example.com?${params}`);

// The callback will be called every time an update is published
eventSource.on message = function ({data}) {
    console.log(data);
};
```

### Hub

The hub receives updates from the publisher on a dedicated HTTPS endpoint.
The connection MUST use an encryption layer, such as TLS. HTTPS certificate can be obtained for free using [Let's Encrypt](https://letsencrypt.org/).

When it receives an update, the hub dispatches it to subsribers using the established server-sent events connections.

An application CAN send events directly to the subscribers, without using an external hub server, if it is able to do so.
In this case, it MAY not implement the endpoint to publish updates.

The endpoint to publish updates is an HTTPS URL accessed using the `POST` method.
The request must be encoded using the `application/x-www-form-urlencoded` format and contains the following data:

* `topic`: IRIs of the updated topic. If this key is present several times, the first occurence is considered to be the canonical URL of the topic, and other ones are considered to be alternate URLs. The hub must dispatch this update to subscribers subscribed to both canonical or alternate URLs.
* `data`: the content of the new version of this topic
* `target` (optional): target audience of this event, see the Authorization section for further information.
* `id` (optional): the topic's revision identifier, it will be used as the SSE's `id` property, if omited the hub MUST generate a valid UUID.
* `type` (optional): the SSE's `event` property (a specific event type)
* `retry` (optional): the SSE's `retry` property (the reconnection time)

The request MUST also contain an `Authorization` HTTP header containing the string `Bearer ` followed by a valid [JWS (RFC7515)](https://tools.ietf.org/html/rfc7515) in compact serialization that the hub will check to ensure that the publisher is authorized to publish
the update.

## Authorization

If a topic is not public, the update request sent by the publisher to the hub MUST also contain a list of keys named `target[]`.
Theirs values are `string`. They can be, for instance a user ID, or a list of group IDs.

To receive updates for private topics, the subscriber must send a cookie called `mercureAuthorization` when connecting
to the hub.

The cookie SHOULD be set by the publisher during the discovery. The cookie SHOULD have the `Secure`, `HttpOnly`.
It MAY have the `SameSite` flag if appropriate. When skipping the discovery mechanism, the client
MAY set the cookie itself.

Consequently if the subscriber is a web browser, both the publisher and the hub have to share the
same second level domain to use the autorization feature. The `Domain` flag MAY be used to allow the publisher
and the host to use different subdomains.

By the `EventSource` specification, connections can only be estabilished using the `GET` HTTP method,
and it is not possible to set custom HTTP headers (such as the `Authorization` one).

However, cookies are supported, and can be included even in crossdomain requests if [the CORS credentials are set](https://html.spec.whatwg.org/multipage/server-sent-events.html#dom-eventsourceinit-withcredentials):

The value of this cookie MUST be a JWS in compact serialization. It MUST have a claim named `mercureTargets` that contains
an array of strings: the list of targets the user is authorized to receive updates for.
For instance, valid targets can be a username or a list of group identifiers.
The JWS SHOULD be short lived, especially if the subscriber is a web browser.

If one or more targets are specified, the update MUST NOT be sent to the subscriber by the hub, unless the `mercureTargets`
claim of the subscriber contains at least one target specified for the topic by the publisher. 

When using the authorization mechanism, the connection between the subscriber and the hub MUST use an encryption layer (HTTPS
is required).

### Re-Connection and State Reconciliation

To allow re-establisment in case of connection lost, events dispatched by the hub SHOULD include an `id` property.
The value contained in this `id` property SHOULD be a globally unique identifier.
To do so, [UUID](https://tools.ietf.org/html/rfc4122) MAY be used.

According to the server-sent events specification, in case of connection lost the subscriber will try to automatically reconnect. During the reconnection the subscriber MUST send the last received event id in a [`Last-Event-ID`](https://html.spec.whatwg.org/multipage/iana.html#last-event-id) HTTP header.

The server-sent events specification doesn't allow to set this HTTP header during the first connection (before a re-connection occurs).
In order to fetch any update dispatched between the initial resource generation by the publisher and the connection to he hub,
the subscriber MUST send the event id provided during the discovery in the `last-event-id` link's attribute in a query parameter named `Last-Event-ID` when connecting to the hub.

If both the `Last-Event-ID` HTTP header and the query parameter are present, the HTTP header MUST take precedence.

If the `Last-Event-ID` header or query parameter exists, the hub SHOULD send to the subscriber all events published since the one having this identifier.

The hub MAY discard some messages for operational reasons. The subscriber MUST NOT assume that no update will be lost, and MUST re-fetch the original topic to ensure this (for instance, after a long deconnection time).

The hub MAY also specify the reconnection time using the `retry` key, as specified in the server-sent events format.

### Encryption

Using HTTPS doesn't prevent the hub to access to the update's content.
Depending of the intended privacy of informations contained in the updates, it MAY be necessary to prevent eavesdropping by the hub.

To make sure that the message content can not be read by the hub, the publisher MAY encode the message before sending it to the hub.
The publisher SHOULD use [JSON Web Encryption, (JWE, RFC7516)](https://tools.ietf.org/html/rfc7516) to encrypt the update content.
The publisher MAY provide the relevant encryption key(s) in the `key-set` attribute of the Link HTTP header during the discovery. The `key-set` attribute SHOULD contain a key encoded using the [JSON Web Key Set (JWKS, RFC7517](https://tools.ietf.org/html/rfc7517) format.
Any other out-of-band mechanism MAY be used instead to share the key between the publisher and the subscriber.

Updates encyption is considered a best practice to prevent mass surveillance.
This is especially relevant if the hub is managed by an external provider.

## The Hub Implementation

Environment variables:

* `PUBLISHER_JWT_KEY`: must contain the secret key to valid publishers' JWT
* `SUBSCRIBER_JWT_KEY`: must contain the secret key to valid subscribers' JWT
* `CORS_ALLOWED_ORIGINS`: must contain a comma separated list of hosts allowed to subscribe
* `ALLOW_ANONYMOUS`:  set to `1` to allow subscribers with no valid JWT to connect
* `DEBUG`: set to `1` to enable the debug mode (prints recovery stack traces)
* `DEMO`: set to `1` to enable the demo mode (automatically enabled when `DEBUG=1`)
* `DB_PATH`: the path of the [bbolt](https://github.com/etcd-io/bbolt) database (default to `updates.db` in the current directory)

## Resources

* [JavaScript library to decrypt JWE using the WebCrypto API](https://github.com/square/js-jose)
* [`EventSource` polyfill for old browsers](https://github.com/Yaffle/EventSource)
* [`EventSource` implementation for Node](https://github.com/EventSource/eventsource)
* [`Server-sent events` client (and server) for Go](https://github.com/donovanhide/eventsource)

## Contributing

Clone the project:

    $ git clone https://github.com/dunglas/mercure
    
Install Gin for Live Reloading:

    $ go get github.com/codegangsta/gin

Install the dependencies:

    $ cd mercure
    $ go get

Run the server:

    $ gin run main.go

Go to `http://localhost:3000` and enjoy!
