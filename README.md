# JSON:API:WebSockets

This document describes a way to support publishing data updates to clients over a WebSocket connection.

In particular, this specification is meant to be used alongside a REST service implementing the [JSON:API](https://jsonapi.org/) specs.

## Overview

The client subscribes and unsubscribes to data updates using HTTP paths as "channel" names, as well as a subscription type.

When the server updates a record, it sends either a partial update or full copy of the record to the client, or sends no data at all.

The messages passed between client and server are JSON serialized, using ordered arrays instead of named properties.

---

## Client Message Structure

Messages sent from the client to the server look like this:

```json
[
	"001",
	"SUBSCRIBE",
	[
		[ "/articles/123", "FULL" ],
		[ "/authors/456", "DIFF" ]
	]
]
```

### Index 0: Message Identifier

Each message sent to the client must specify a valid [request identifier](#request-identifier).

Since message passing is asynchronous, the message identifier is included on the server response to let the client connect responses to requests.

#### Index 1: Request Type

The string must be a valid [request type](#request-type).

The client makes requests to the server, such as subscribing or unsubscribing to paths, or listing the current subscriptions.

#### Index 2: Request Payload

A payload, as defined by the [request type](#request-type), such as a list of paths to subscribe to.

---

## Server Response Structure

Messages sent from the server to the client in response to a request look like this:

```json
[
	"0",
	"200",
	"Subscribed to all paths",
	[ "100", "101", "102" ]
]
```

#### Index 0: Message Identifier

This array element *must* be the [request identifier](#request-identifier) provided by that client request.

#### Index 1: Status Code

The HTTP status code of the response, expressed as a string value.

#### Index 2: Response Title

A short, human-readable summary of the problem that **SHOULD NOT** change from occurrence to occurrence of the problem, except for purposes of localization.

#### Index 3: Response body

The payload response for the request.

---

## Server Update Structure

Resource updates sent from the server to the client look like this:

```json
[
	null,
	"101",
	"DIFF",
	{
		"data": {
			"id": "456",
			"type": "author",
			"attributes": {
				"books": 3
			}
		}
	}
]
```

#### Index 0: Unused

In a response message, this value would be an identifier. An update message is not a response, so it does not not start with an identifier.

This value must be exactly `null`.

#### Index 1: Subscription Identifier

If the update message is associated with a subscription, this value would be the [subscription identifier](#subscription-identifier), otherwise it must be exactly `null`.

#### Index 2: Update Type

The string value of a valid [update type](#update-types).

#### Index 3: Body

The body is determined by the [update type](#update-types).

---

## Request Type

The request type is a string and must be one of:

* `SUBSCRIBE`
* `UNSUBSCRIBE`
* `LIST`

### SUBSCRIBE

Direct the server to inform the client of changes to resources described by one or more paths.

The request payload is an ordered array containing one or more [subscriptions](#subscription), which are arrays containing two properties:

* **Index 0:** The API path to subscribe to.
* **Index 1:** The *subscription type*.

A successfull response from the server would be a matched-ordered array of [identifiers](#identifiers) which correspond to those subscriptions.

For example, given this request payload:

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

In this example, the identifier `001` would correspond to the subscription `[ "/articles/123" "FULL" ]`, would be used by the client to unsubscribe, and would be included in the updates from the server.

### UNSUBSCRIBE

Direct the server to stop informing the client of changes to resources described by one or more paths.

The payload is an array of [subscription identifiers](#subscription-identifier).

For example:

```json
[ "001", "002" ]
```

### LIST

Request the server to send a list of all subscribed paths.

This request does not have a payload.

---

## Subscriptions

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

---

## Update Types

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

#### `DELETE`

Sent by the server to indicate that a resource or relationship should be removed.

The body must be a valid [JSON:API resource identifier object][json-api-resource-identifier-object].

#### `PING`

The server does not send an object, but will send an empty body on any resource change.

---

## Request Identifier

An [identifier string](#identifier-string) which *should* be unique across all requests for the active connection.

---

## Subscription Identifier

An [identifier string](#identifier-string) which *must* be unique for the [subscription](#subscription), for the active connection.

For example, if the client mistakenly requests the same subscription twice, the subscription identifier for both would be the same.

---

## Identifier String

In the context of this document, a valid "identifier" is a string containing only the values:

* U+0061 to U+007A, "a-z"
* U+0041 to U+005A, "A-Z"
* U+0030 to U+0039, "0-9"

The identifier must be unique in the context of the connection session, but do not need to be globally unique.

[json-api-resource]: https://jsonapi.org/format/#document-resource-objects

[json-api-resource-identifier-object]: https://jsonapi.org/format/#document-resource-identifier-objects
