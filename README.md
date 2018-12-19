# webhooktunnel
[![Task Status](https://github.taskcluster.net/v1/repository/taskcluster/webhooktunnel/master/badge.svg)](https://github.taskcluster.net/v1/repository/taskcluster/webhooktunnel/master/latest)

Webhooktunnel is a service that allows its clients to publicly expose specific HTTP services without publicly exposing the entire host.
"Clients" connect to the webhooktunnel service with a specific ID, authenticating with a signed JWT, and upgrade the connectino to a websocket.
"Viewers" then connect to the service using a path containing that ID.
The viewer's connection is then proxied to the client via its websocket.
The critical characteristic is that all connections are made *to* the webhooktunnel service, so the clients need not be publicly addressible.

# Configuration

Configuration is via environment variables:

* `HOSTNAME` gives the hostname of the webhookproxy service itself. This is used to construct URLs for use by viewers.
* `ENV` defines the runtime environment; use `ENV=production` for production deployments.
* `SYSLOG_ADDR` (optional) defines a syslog server to which log messages will be sent in production
* `TASKCLUSTER_PROXY_SECRET_A` and `TASKCLUSTER_PROXY_SECRET_B` define two secrets, either of which may be used to sign valid JWTs.
  Either secret is accepted, supporting downtime-free rotation of secrets.
* `TLS_KEY` and `TLS_CERTIFICATE` (both optional) define a TLS certificate that is used for the main HTTP service.
  If not given, the service will default to plain HTTP.
  This is not recommended for production usage!
* `PORT` gives the port on which the HTTP server should run, defaulting to 443 (or if not using TLS, 80).

In non-production mode, the service logs its activities to stdout in a human-readable format.

# API

The server operates in two modes: "domain-hosted" and "single-hosted".
In domain-hosted mode, the id is included as a prefix to the hostname, e.g., for id `abcd` and `HOSTNAME=whp.example.com`, the hostname would be `abcd.whp.example.com`.
This requires a "star" TLS certificate.
If `TLS_KEY` or `TLS_CERTIFICATE` are not available, this mode is disabled.

In single-hosted mode, the id is included in the URL path, as shown below.

## Client Connections

To establish a new client connection, make a GET request to the service's hostname with path `/` and the usual websocket upgrade headers, as well as:

 * `Authorization` containing `Bearer <jwt>`; see below
 * `x-webhooktunnel-id` containing the client ID

The connection will be upgraded to a websocket connection.
The response will contain the haeader `x-webhooktunnel-client-url` giving the URL viewers can use to reach this client.
Clients can pass this URL, or URLs derived from it, to viewers.
Although clients should not make assumptions about the form of this URL, at present it will either look like `https://<id>.<hostname>` (for domain-hosted mode) or `https://<hostname>/<id>` (for single-hosted mode).

### Authorization

The token included in the `Authorization` header must be a [JWT](https://jwt.io/).
It must use method `HS256` and be signed with either of the secrets in the service configuation.
It must be valid at the current time, and must not be valid for more than 31 days (specifically, the `nbf` and `exp` claims must be less than 31 days apart).
Its `tid` claim must match the client ID exactly.

### Multiplexed Websockets

The protocol used within the websocket connection between a client and the webhooktunnel service is beyond the scope of this document.
It is implemented by the `github.com/taskcluster/webhooktunnel/wsmux` package, and the expectation is that clients will use this package directly.

## Viewer Connections

Viewers are given a client URL based on that provided to the cient as described above.
All access to such URLs will be tunneled to the client as described above.
If the client is not available, the viewer will get a 404 error response.

No special considerations are required to access viewer URLs: the intent is that any HTTP client can do so.
Both "normal" HTTP requests and websocket connections are supported.

Note that viewer connections do not require any kind of authentication.
That is entirely up to the client.

# CLI

The `whclient` command implements a client that will connect to a webhooktunnel service and proy all connections to a specific local port.
It takes Taskcluster credentials and uses them to generate JWTs using the Auth service.
