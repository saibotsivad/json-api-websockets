# Example

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
	"c",
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
