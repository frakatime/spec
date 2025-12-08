# Frakatime Specification
Frakatime is designed for only **one user**, not multiple users per server. This *might* be changed in the future.

## Current Version
`v2`

## REST API
### Prefix
All methods should be available at `/api/vCURRENT_VERSION` (E.g. `/api/v2/health`).

### Authentication
A username and password should be provided by the server by default. For example, `ft-server-node` uses the `USERNAME` and `PASSWORD` env variables. All requests require authentication and should have the `Authorization` header. It should be formatted like this:
```
Authorization: Basic Base64Encoded(USERNAME PASSWORD)
```
For example,
```
Authorization: Basic dXNlciBwYXNzd2Q=
```
`dXNlciBwYXNzd2Q=` is a base64 encoded string that when decoded results in `user passwd`. `user` is the username and `passwd` is the password, separated by a space.

###  `GET /health`
This endpoint does not require authentication.\
Simple health check endpoint, should return `200` and
```json
{
    "status": "ok"
}
```

### `GET /time`
Get the entire database represented as a JSON object with all services, sub-services, and their respective times.\
Return `200` and the JSON object, for example:
```json
{
    "services": [
        {
            "service": "app1",
            "time": "1765169654",
            "children": [
                {
                    "service": "category1",
                    "time": "500000",
                    "children": []
                },
                {
                    "service": "category2",
                    "time": "1000000",
                    "children": [
                        {
                            "service": "subcategory1",
                            "time": "600000",
                            "children": [
                                {
                                    "service": "item1",
                                    "time": "400000",
                                    "children": []
                                }
                            ]
                        }
                    ]
                }
            ]
        },
        {
            "service": "app2",
            "time": "5000000",
            "children": []
        }
    ]
}
```

### `GET /time/:service`
Get the time you've spent on a service as a UNIX timestamp string.\
If the service doesn't exist, return `404` and
```json
{
    "status": "not found",
    "error": "service does not exist"
}
```
Otherwise, return `200` and
```json
{
    "time": "UNIX Timestamp"
}
```

### `GET /time/:service/*path`
Get the time you've spent on a sub-service as a UNIX timestamp string. The path can be nested infinitely using `/` as a separator. For example, `/api/v2/time/app1/category2/subcategory1/item1` would get the time spent on `item1`.\
If the service or any part of the path doesn't exist, return `404` and
```json
{
    "status": "not found",
    "error": "service does not exist"
}
```
Otherwise, return `200` and
```json
{
    "time": "UNIX Timestamp"
}
```

### `GET /tree/:service`
Get the entire tree of a service and all its sub-services with their respective times.\
If the service doesn't exist, return `404` and
```json
{
    "status": "not found",
    "error": "service does not exist"
}
```
Otherwise, return `200` and the JSON object, for example:
```json
{
    "service": "app1",
    "time": "1765169654",
    "children": [
        {
            "service": "category1",
            "time": "500000",
            "children": []
        },
        {
            "service": "category2",
            "time": "1000000",
            "children": [
                {
                    "service": "subcategory1",
                    "time": "600000",
                    "children": [
                        {
                            "service": "item1",
                            "time": "400000",
                            "children": []
                        }
                    ]
                }
            ]
        }
    ]
}
```

### `POST /service/:service`
Create a service.\
If the service already exists, return `409` and
```json
{
    "status": "conflict",
    "error": "service already exists"
}
```
Otherwise, return `201` and
```json
{
    "status": "created"
}
```

### `POST /service/:service/*path`
Create a sub-service at the specified path. The path can be nested infinitely. For example, `/api/v2/service/app1/category2/subcategory1` would create a sub-service called `subcategory1` under `app1/category2`.\
If any part of the parent path doesn't exist, return `404` and
```json
{
    "status": "not found",
    "error": "parent service does not exist"
}
```
If the sub-service already exists at that path, return `409` and
```json
{
    "status": "conflict",
    "error": "service already exists"
}
```
Otherwise, return `201` and
```json
{
    "status": "created"
}
```

### `PUT /service/:service`
Update the name of a service.\
Request body:
```json
{
    "new_name": ""
}
```
If the service doesn't exist, return `404` and
```json
{
    "status": "not found",
    "error": "service does not exist"
}
```
Otherwise, return `200` and
```json
{
    "status": "ok"
}
```

### `PUT /service/:service/*path`
Update the name of a sub-service at the specified path.\
Request body:
```json
{
    "new_name": ""
}
```
If the service or any part of the path doesn't exist, return `404` and
```json
{
    "status": "not found",
    "error": "service does not exist"
}
```
Otherwise, return `200` and
```json
{
    "status": "ok"
}
```

### `DELETE /service/:service`
Delete a service and all of its sub-services.\
If the service doesn't exist, return `404` and
```json
{
    "status": "not found",
    "error": "service does not exist"
}
```
Otherwise, return `200` and
```json
{
    "status": "ok"
}
```

### `DELETE /service/:service/*path`
Delete a sub-service at the specified path and all of its children.\
If the service or any part of the path doesn't exist, return `404` and
```json
{
    "status": "not found",
    "error": "service does not exist"
}
```
Otherwise, return `200` and
```json
{
    "status": "ok"
}
```

## Time Tracking
The client reports time tracking to the server via a WebSocket endpoint, `/api/v2/ws/track`.\
Upon connection, the client should send the following message:
```
USERNAME:PASSWORD|SERVICE/path/to/subservice
```
The service path uses `/` as a separator for nested sub-services. For example, `app1/category2/subcategory1/item1` would track time for `item1`.

The server checks for proper authentication, then checks if the service and full path exists.
- If authentication fails, the server sends `unauthorized` and closes the connection.
- If the service or any part of the path doesn't exist, the server sends `non-existant service` and closes the connection.
- If successful, the server sends `ok`.

After receiving `ok`, the client sends `+` every second. The server internally adds a second to the tracked time for that specific sub-service **and** all parent services in the hierarchy on every message containing `+`. For example, tracking `app1/category2/subcategory1/item1` would increment time for `item1`, `subcategory1`, `category2`, and `app1`. The client should close the WebSocket connection whenever the user stops using the service.

## Errors
For incorrect authentication attempts, return `401` and
```json
{
    "status": "unauthorized"
}
```
For bad requests, return `400` and
```json
{
    "status": "bad request",
    "error": ""
}
```
You can optionally tell the client what they did wrong, or not.
For invalid endpoints, return `404` and
```json
{
    "status": "not found"
}
```
For any internal errors, return `500` and
```json
{
    "status": "internal server error",
    "error": "stack trace"
}
```
The `error` part is optional.
For any endpoints not implemented, return `501` and
```json
{
    "status": "not implemented"
}
```
