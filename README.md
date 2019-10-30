# JSON:API:WebSockets

This document describes a way to support publishing data updates to clients over a WebSocket connection.

In particular, this specification is meant to be used alongside a REST service implementing the [JSON:API](https://jsonapi.org/) specs.

## Overview

The client subscribes and unsubscribes to data updates using HTTP paths as "channel" names, as well as a subscription type.

When the server updates a record, it sends either a partial update or full copy of the record to the client, or sends no data at all.

The syntax used by the client is terse, yielding small WebSocket payload frame sizes, yet is expressive enough to be readable by a human.

## Example

After connecting and authenticating, the client subscribes to one or more channels by sending a JSON serialized string (described later in this document):

```json
[
	"0",
	"subscribe",
	[
		[ "/articles/123", "FULL" ],
		[ "/authors/456", "DIFF" ],
		[ "/comments?include=author", "PING" ]
	]
]
```

The server responds asynchronously with a status code and optional payload:

```json
[
	"0",
	200,
	"Subscribed to all paths",
	[ "a", "b", "c" ]
]
```

The client unsubscribes to one or more channels in the same way:

```json
[
	"1",
	"unsubscribe",
	[ "a", "c" ]
]
```

The server responds asynchronously in a similar way:

```json
[
	"0",
	200,
	"Unsubscribed from all paths"
]
```

The client can request a list of all subscribed paths:

```json
[
	"2",
	"list"
]
```

And the server responds asynchronously in a similar way:

```json
[
	"2",
	200,
	"List of subscribed paths",
	[
		[ "c", "/authors/456", "DIFF" ]
	]
]
```

When a resource is updated, the server sends updates to all subscribed clients:

```json
[
	null,
	"/authors/456",
	"DIFF",
	{
		"id": "456",
		"type": "author",
		"attributes": {
			"books": 3
		}
	}
]
```

---

## Core Concepts

### Identifiers

In the context of this document, a valid "identifier" is a string containing only the values:

* U+0061 to U+007A, "a-z"
* U+0041 to U+005A, "A-Z"
* U+0030 to U+0039, "0-9"

The identifier must be unique in the context of the connection session, but do not need to be globally unique.

### Subscriptions

When a client subscribes to a path, it is requesting that the server send updates when resources change for that path.

Any resource update which would cause a GET request to the endpoint path to return different data *should* be published by the server to the client.

Consider this example request and response:

```
GET /articles/123?include=comments
```

```json
{
	"data": {
		"id": "123",
		"type": "article",
		"attributes": {
			"title": "My News Article",
			"content": "Some long text."
		},
		"relationships": {
			"comments": {
				"data": [
					{
						"id": "456",
						"type": "comment"
					}
				]
			}
		}
	},
	"included": [
		{
			"id": "456",
			"type": "comment",
			"attributes": {
				"message": "Great article A++ would read again."
			}
		}
	]
}
```

If the client subscribed to the path `/articles/123?include=comments` the following updates would be expected to be published by the server to the client:

* One or more properties on the article are changed, added, or removed.
* One or more properties on the comment are changed, added, or removed.
* The `relationships` property of the article is changed, adding or removing a related comment resource.

### Subscription Types

Subscriptions can be one of these types:

#### `FULL`

The object sent to the client is the full resource of the updated object.

For example, if the client is subscribed to `/articles/123?include=comments` and a comment title is changed, the object sent to the client would be the full comment object.

#### `DIFF`

The object sent to the client is a sparsely populated JSON:API "resource" object, containing only the changed properties.

The object must conform to the JSON:API specifications for a request body containing a single resource. E.g. there must be a root `data` object, which contains an `id` and `type`, and so on.

In a `DIFF` subscription type, the resource sent by the server *must* be the final changes after all server-side updates are applied, such that the client does a simple apply-merge to get the final data.

If the resource has an `attributes` property which is an array, the published resource *must* contain the full array object.

For example, suppose the client is subscribed to `/articles/123` and the server receives an update for that articles title:

```json
{
	"data": {
		"id": "123",
		"type": "article",
		"attributes": {
			"title": "My Updated Title"
		}
	}
}
```

It is a common design pattern that the update to the title will also change properties such as a last-updated date value. In this case, the server should publish a resource containing the updated title as well as the last-updated date value.

```json
{
	"data": {
		"id": "123",
		"type": "article",
		"attributes": {
			"title": "My Updated Title"
		},
		"meta": {
			"updated": "2019-10-30T12:34:56+0000"
		}
	}
}
```

#### `PING`

The server does not send an object, but will send an empty body on any resource change.

---

## Client Message Structure

The message sent by the client to the server is an ordered array, with the elements of the array as follows:

* **Index 0:** Message Identifier
* **Index 1:** Action Name
* **Index 2:** Action Payload

#### Index 0: Message Identifier

Each message sent to the client must start with a valid *identifier*.

When the server responds to the request, it will use that same identifier to let the client know which request the response is for.

#### Index 1: Action Name

The action that the client desires the server to take.

Supported actions are:

* `subscribe` - Direct the server to inform the client of changes to resources described by one or more paths.
* `unsubscribe` - Direct the server to stop informing the client of changes to resources described by one or more paths.
* `list` - Request the server to send a list of all subscribed paths.

#### Index 2: Action Payload

Each action may include a payload of data, such as a list of paths to subscribe to.

##### Action Payload: `subscribe`

The payload for the `subscribe` action is an ordered array containing one or more subscription identifiers. These are arrays containing two properties:

* **Index 0:** The API path to subscribe to.
* **Index 1:** The *subscription type*.

The servers response is a matched-ordered array of *identifiers* which correspond to those subscriptions.

For example, given this *action payload*:

```json
[
	[ "/articles/123", "FULL" ],
	[ "/authors/456", "DIFF" ]
]
```

The server response payload would be an array containing two identifiers. For example:

```json
[ "001", "002" ]
```

Each element of the array is an identifier corresponding to the same index of the action payload.

In this example, the identifier `001` would correspond to the subscription `[ "/articles/123" "FULL" ]` and would be used to unsubscribe.

##### Action Payload: `unsubscribe`

The payload is an array of subscription *identifiers*.

For example:

```json
[ "001", "002" ]
```

##### Action Payload: `list`

The `list` action does not have a payload.












In pseudo-code you might have something like:

```js
const identifier = '001'
socket.publish
```









the server REST is JSON API
	- one big change: no includes?
the websocket connection can be a different domain (where to document? unspecified)

websocket requests from client are basically subscribe/unsubscribe and list/

```json
[
	0, // request id
	"subscribe", // action
	// any changes to whatever would have come back
	[ "FULL", "/some/path" ],
	// only the complete PATCH as the server finalizes it
	// aka if you patch field X and the final database write is
	// field X and Y, this would include both
	[ "DIFF", "/another/path" ],
	// no data will be sent, only an alert that the data
	// has been changed and you will need to re-fetch it
	[ "PING", "/articles?include=author" ]
]
// =>
[
	0, // request id, echoed back
	200, // HTTP-like status TODO needs discussion, it's based on the JSON-API specs
	"OK", // very short human readable message
	"Human readable message", // longer human readable message is optional (put 0 here if not used)
	{ "foo": "bar" } // optional response body
]
```

```json
[
	1,
	"unsubscribe",
	[
		"/some/path",
		"/another/path"
	]
]
// =>
[ 1, 200, "OK", "Human readable message", { "foo": "bar" } ]
```

```json
[ 2, "list" ]
// =>
[
	2, // the echoed back id
	[ "/articles?include=author" ] // the list of active subscriptions
]
```
