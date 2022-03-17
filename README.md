Webauthz
========

# Introduction

Webauthz is a token-based authorization protocol that is a simpler
replacement for OAuth 2.0.

Webauthz contains only one sequence, or "flow", which is comparable
to the OAuth 2.0 "authorization code" flow. Webauthz adds features
not found in OAuth 2.0, such as a authorization server discovery and
path matching.

The benefit of this work is that Webauthz implementations are simpler
to understand and easier to maintain than OAuth 2.0, and application
developers are less likely to make mistakes that will create security
vulnerabilities.

Webauthz works in typical scenarios, such as allowing an application to
access a network printer, or the user's photos or documents, or interact
with the user's social media accounts, or allowing a SaaS extension to
access APIs that it needs to perform its functions while the user is not
logged in, or in allowing a mail client to access the user's mailbox
on a remote server using some authentication system other than passwords.

More abstractly, you can use Webauthz where a website or application
needs to access a user's resource on another server in a limited way,
without obtaining complete control over it, and
is able to either redirect the user or open a window to to allow the
user to approve access to the resource.

Unlike OAuth 2.0, you don't need to make long-term choices such as which
flow to use, or consider whether you need to implement alternative
flows for different clients. When using Webauthz, there is only one
implementation so you can pick up the Webauthz libraries and start
integrating them immediately, and things will just work.

# Overview

The Webauthz protocol assumes the existence of the following parties:

* Application (a website, desktop, or mobile application requesting access to a resource)
* Resource Server (a website or other resource being accessed by the application)
* Authorization Server (a website that authenticates the user and manages access to a resource)
* Resource Owner (the user or system authorizing the access)

The resource server and authorization server could be combined into a single
entity, or they could be separate. It is possible for an authorization server
to work with multiple resource servers. Applications only need to know the
URL of the resource they will access, and they will discover the authorization
server via the Webauthz protocol.

This is the setup sequence which can be done just once, or as needed,
between an application and a resource server:

* [Discovery](#discovery)
* [Registration](#registration)

Here is the authorization sequence for obtaining access to a resource,
which can be repeated as needed:

* [Request](#request)
* [Prompt](#prompt)
* [Grant](#grant)
* [Exchange](#exchange)

When these steps are complete, the application obtains an access token
that it can use to make requests to the resource server while that token
is valid:

* [Access](#access)

# Specification

## Discovery

The discovery process informs the application about where it should make
Webauthz authorization requests.

Discovery can start with a system administrator configuring a web application
with a Webauthz Discovery URI, or it can be dynamic where the Webauthz Discovery
URI is provided in response to a resource [access](#access) request, along with the HTTP status
code of 401 Unauthorized or 403 Forbidden.

To discover the authorization server configuration, the application
needs to know only one URI:

* Webauthz Discovery URI

If the URI scheme is `https`, the client can retrieve the
configuration settings with an HTTPS GET request. An example of
a Webauthz Discovery URI is "https://resource.example.com/webauthz.json".

To make the request with `curl`:

```
curl \
  -H 'Accept: application/json' \
  '<webauthz_discovery_uri>'
```

This allows a resource server to host more than one Webauthz
configuration, if needed for different applications.

A successful discovery response looks like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "webauthz_register_uri": "https://resource.example.com/webauthz/register",
  "webauthz_request_uri": "https://resource.example.com/webauthz/request",
  "webauthz_exchange_uri": "https://resource.example.com/webauthz/exchange"
}
```

The response object SHALL include values for the following keys:

* `webauthz_register_uri` indicates where a client can register to receive a
  client ID and client token (see [Registration](#registration))
* `webauthz_request_uri` indicates where a client can initiate access requests
  to eventually obtain access tokens for resources (see [Request](#request))
* `webauthz_exchange_uri` indicates where a client can exchange grant tokens
  for access tokens (see [Exchange](#exchange))

The values are determined by the authorization server administrator.

The client application SHOULD store the discovery information, indexed by
the exact value of `webauthz_discovery_uri`. This is because any change in
the value of `webauthz_discovery_uri` could result in a different document.

The application SHOULD
check to see if the discovery settings have changed each time it receives a
401 or 403 status code from the resource server. Existing HTTP mechanisms
such as the `HEAD` request, entity tags, and conditional requests are sufficient
to ensure the application has the latest version.

## Registration

Before an application can request access to a resource, it must
first register with the authorization server.

The request MUST include a `client_name` to label the client. The value
of `client_name` is arbitrary and does not need to be unique. It SHOULD
identify the client application's software name.

A client application SHOULD NOT include its version number in the `client_name`
property because the Webauthz protocol does not include a way to update this
value later. See [registration update](#registration-update) for more information.

The request MUST include a `client_origin`. The value of `client_origin` is a URL.

A client origin is fixed to a specific scheme, fully-qualified domain name, and port.
For example, a client registering with a `client_origin` value of
'https://example.com' must be able to respond to requests for resources in this
origin or it will not be able to complete the access request procedure.

If the Register Client URI scheme is `https`, the application sends an HTTPS POST
request. An example of a Webauthz Registration URI is
"https://resource.example.com/webauthz/register".

To make the request with `curl`:

```
curl \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -X POST \
  --data '{"client_name": "<client_name>", "client_origin": "<client_origin>"}' \
  '<webauthz_register_uri>'
```

If a request does not include `client_name` or `client_origin`,
an authorization server MUST deny the request with
`400 Bad Request`.

The authorization server should generate a random `client_id` and
`client_token`, compute the `client_token_digest` from the `client_token`,
and store the `client_id` and the `client_token_digest` together with
the `client_name` and other values provided by the client.

If the authorization server does not allow open client registration,
the authorization server SHALL deny the request with `401 Unauthorized`.

Where an authorization server
allows client registration, it responds to a valid registration request
with `client_id`, `client_token`, and `client_token_max_seconds`.

Where an authorization server allows clients to automatically refresh their client
tokens (instead of creating a new registration), it adds `refresh_token` and
`refresh_token_max_seconds` to the registration response. The authorization server
may also add `client_token_min_seconds` to the response to indicate the earliest
time that the `client_token` may be refreshed.

A successful registration response looks like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "client_id": "<client_id>",
  "client_token": "<client_token>",
  "client_token_max_seconds": <client_token_max_seconds>,
  "client_token_min_seconds": <client_token_min_seconds>,
  "refresh_token": "<refresh_token>",
  "refresh_token_max_seconds": <refresh_token_max_seconds>
}
```

The response object SHALL include the following keys:

* `client_id` is a unique client identifier assigned to this client
* `client_token` is a bearer token that authorizes the client to use the
  authorization server APIs
* `client_token_max_seconds` is the number of seconds the client token is
  valid; null or undefined means the client token does not expire, or should
  be used indefinitely until access is denied
* `client_token_min_seconds` is the number of seconds that a client must wait
  before attempting to refresh the client token; null or undefined means the
  client should wait until the client token expires before refreshing it; when
  specified, this value MUST be less than or equal to `client_token_max_seconds`
* `refresh_token` is the bearer token that authorizes requests to refresh the
  client token using the exchange API; a null or undefined value means the client token
  cannot be refreshed
* `refresh_token_max_seconds` is the number of seconds the refresh token is
  valid; null or undefined means the client token does not expire, or should
  be used indefinitely until access is denied

Where the registration response includes a `refresh_token`,
clients SHOULD immediately compute and store the client token expiration date from
the `client_token_max_seconds` value. The application SHOULD NOT
use the client token after `client_token_max_seconds` have passed. Clients can refresh
their expired client tokens using the [client token exchange](#client-token-exchange) API.

Client applications will not need to send the `client_id` to the authorization server
in the Webauthz protocol because all issued tokens already identify the client.
However, when storing tokens or storing context for pending access requests, client
applications should group all that information using the authorization server's origin
and the specified client id because if the registration is ever repeated in the future
and the client application is registered as a new client, the data belonging to
different client identifiers should not be mixed.

Client applications SHOULD also include the authorization server's origin,
computed from the value of `webauthz_register_uri`, as an additional attribute
that is always used together with the `client_id`, `client_token`, and other
registration information. For example, for a `webauthz_register_uri` value of
`https://example.com/webauthz/register`, a client application would store the
origin and client id as a pair (`https://example.com`, `<client_id>`) such
that if another authorization server generates an identical `client_id` value,
they would not be mixed up in storage. This is important for both avoiding
accidental disclosure of tokens to the wrong party, and also for avoiding
confusion about why access is denied (when the wrong tokens are sent to an
authorization server).

In contrast to the discovery URI, where any change in the URI could mean that it
points to a different document, a change in the path or query parameters of a
registration URI does NOT indicate a different authorization server. The origin
of a registration URI indicates the authorization server.

A client application MUST NOT send its `client_token` or corresponding
`refresh_token` obtained from a given origin to any other origin.

An authorization server MUST host all its Webauthz APIs and extensions
on the same origin in order for clients to recognize that they can authenticate
to these APIs with the same credentials.

## Request

The first step of the routine authorization sequence is to request access
to a resource as described in this section.

To start the access request, the application sends a request to
the authorization server's Webauthz Request URI. If the Webauthz Request URI
scheme is `https`, the application sends an HTTPS POST request.
An example of a Webauthz Request URI is
"https://resource.example.com/webauthz/request".

The request must include the
Webauthz challenge parameters `realm` and `scope`.

Where the client application needs the authorization server to redirect users
after the access prompt, the request MUST include `grant_redirect_uri` with
an origin matching the client's registered `client_origin`. A client application
SHOULD customize the path and query parameters of `grant_redirect_uri` so it
will display an appropriate page when the user is redirected back to the client
application at that location.

The client application SHOULD include a CSRF token in the `grant_redirect_uri`.
When user is redirected to that location, the client application SHOULD check that the
CSRF token matches the expected value.

The request format MAY
be extended by an authorization server to accept additional parameters not
defined in this specification. Any such additional parameters MUST be
optional to preserve interoperability between client applications and authorization
servers.

To make the request with `curl`:

```
curl \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <client_token>' \
  -X POST \
  --data '{"realm": "<realm>", "scope": "<scope>", "grant_redirect_uri": "<grant_redirect_uri>"}' \
  '<webauthz_request_uri>'
```

The `webauthz_request_uri` was or obtained via the [discovery](#discovery) step.

The `realm` and `scope` parameters MUST be included if they were
present in `WWW-Authenticate` or `Proxy-Authenticate` header value in the resource server response.

An authorization server may serve multiple resources, so the `realm` parameter
provides a namespace for the `scope` parameter to avoid a collision in scope names
across various resources.

Where the `grant_redirect_uri` is specified, the authorization server SHALL check
that the value is a URI whose origin matches the client's registered `client_origin`.

If the `grant_redirect_uri` does not match, the authorization server SHALL deny the request with
`403 Forbidden`.

The authorization server MUST store the matching `grant_redirect_uri` value
with the request so it can use it to redirect the user after the access prompt.

The authorization server should generate a redirect URL to which the
application can redirect the user to grant or deny the access.
The authorization server MUST generate and include a request identifier
in the URL so it can recognize the user when the user is redirected.

The authorization server responds to the client with a request identifier and redirect URL.
If the redirect URL will expire, or contains a unique identifier referencing an
access request record that will expire, the response MUST include a `redirect_max_seconds`
property indicating when this happens. This enables applications to present the
redirect link to users for a limited time and then replace it with a link or button
to start a new request, to avoid sending the
user to the authorization server with an invalid link.

Alternatively, client applications can show users a message that accessing the
resource requires authorization and a link or button to continue, and only start
the access request when the user taps the link or button, redirecting the user
immediately to the `access_redirect_uri` without implementing any logic about when
that link expires.

A successful response with a redirect looks like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "state": "<state>",
  "state_max_seconds": <state_max_seconds>,
  "redirect": "<access_request_uri>",
  "redirect_max_seconds": <redirect_max_seconds>
}
```

The response object SHALL include the following keys:

* `state` is a request identifier generated by the authorization server; this could be
  a lookup token or a self-encoded token; the client application will also receive this
  value in the `grant_redirect_uri` later
* `redirect` is a URL to which the client must redirect the user to continue
  the access request (typically a site where the user will authenticate and approve
  the access request)

The response object MAY include the following keys:

* `state_max_seconds` is an integer indicating the maximum number of seconds that the `state`
  value must be stored by the client application
* `redirect_max_seconds` is an integer indicating the number of seconds that the
  `redirect` link is valid

The `state_max_seconds` allows the application to safely delete stale access requests.

The `redirect_max_seconds` key allows the authorization server to issue time-limited
tokens in that URL. If this key is undefined, or is present with a `null` value,
the redirect link does not expire. A negative or zero value is invalid. Some clients
may automatically redirect the user when the `redirect` link is received. Other clients
may display a button for the user to continue when they are ready. In this case,
the clients can use the `redirect_max_seconds` value to inform the user of how much
time they have to click on the link. Clients could automatically disable the link when
it expires, or react to an expired link by automatically starting a new request and then
automatically redirecting the user (since the user indicated they are ready to continue).

The `redirect_max_seconds` value indicates the maximum time that is allowed for
redirecting a user to the authorization server. The `state_max_seconds` value indicates
the maximum time that is allowed for the user to complete the authorization prompt
at the authorization server and be redirected back to the client application.
Therefore, the `state_max_seconds` value MUST be equal to or greater than the
`redirect_max_seconds` value.

The application SHALL store the `state` value and associate it with the access request.

Where a client application has multiple registrations with the authorization server,
the client application MUST associate the appropriate `client_id` with the request `state`
in order to use the correct `client_token` later when exchanging the `grant_token` for
an `access_token`.

Where a client application is a multi-user application and the access request
is NOT for a resource that is intended to be shared among all users,
the client application SHALL associate the request with the currently authenticated
user.

If the Access Request URI scheme is `https`, the application redirects the user
to the authorization server by providing the user's browser with a
link which it will use to make an HTTPS GET request to the authorization
server. 

To make the request, the application either redirects the user
with the `Location` header in an HTTPS response, or directs the user to
click on a link. In either case, the link is to `<access_request_uri>`.


To make the request with `curl`:

```
curl '<access_request_uri>'
```

When the link is accessed by the user's browser, the authorization server response
should be the user interface of the authorization server
that will show the request to the user and [prompt](#prompt) the user to grant or
deny the access. If the Webauthz Request URI scheme is `https`, the
server response would typically be HTML, like this:

```
HTTP/1.1 200 OK
Content-Type: text/html

<html>
... prompt user here ...
</html>
```

## Prompt

The second step of the authorization routine is to prompt the user
to grant or deny the access. The implementation of this step is
specific to the authorization server, but typically it would display
the request and then offer two choices to the user. It might look
like this:

```
Webauthz Resource Request

<client_name> (<client_origin>)
is requesting access to
<resource>.

<permissions>

<optional_parameters>

[Grant] [Deny]
```

The `<client_name>` placeholder SHOULD be a human-readable format
identifying the application making the request. 

The `<client_origin>` placeholder SHOULD be the registered client origin.

The `<resource>` placeholder SHOULD be a human-readable format,
possibly including the `realm` attribute from the `WWW-Authenticate`
or `Proxy-Authenticate` header.

The `<permissions>` placeholder SHOULD be a human-readable
format explaining what permissions the user will be granting the
application, possibly including human-readable names for the `scope`
attribute from the `WWW-Authenticate` 
or `Proxy-Authenticate` header.

The `<optional_parameters>` placeholder SHOULD be a human-readable
format for whatever optional parameters were specified by the
application in its access request, which are recognized by the
authorization server.

The user may then tap the `[Grant]` button to continue, or tap
the `[Deny]` button to cancel the request.

The labels of the buttons and the corresponding action between the
user's browser and the resource server are not specified. However,
a simple implementation would be to make the buttons perform an
HTTP POST to the resource server with the selection.

## Grant

The third step of the authorization routine is to redirect the user
back to the application. If the user granted the requested
access, the authorization server will generate a `grant_token` and
include it in the Grant Redirect URI as a query parameter.
If the user denied
the requested access, the redirect will NOT include a `grant_token`.

If the `grant_redirect_uri` scheme is `https`, the authorization server
redirects the user to the application by providing the user's browser
with a link which it will use to make an HTTPS GET request to the
application server. An example of a Grant Redirect URI is
"https://app.example.com/webauthz/grant".

To make the request, the authorization server either redirects the user
with the `Location` header in an HTTPS response, or directs the user to
click on a link to continue.

The access denied link looks like this:

```
<grant_redirect_uri>?state=<state>
```

The access granted link looks like this:

```
<grant_redirect_uri>?state=<state>&grant_token=<grant_token>
```

The `state` parameter is the same value that was returned earlier from the request API.
The `grant_redirect_uri` was specified by the client application.

Where the `grant_redirect_uri` already includes query parameters, the authorization
server SHALL merge the `state` and `grant_token` query parameters into the query string.

The `grant_token` was generated by the authorization
server in response to the user tapping the `[Grant]` button.

When the link is accessed by the user's browser, the application
response should be the user interface of the application that will
either display either a message informing the user to wait while the
request is being finalized (if a `grant_token` is present in the query
parameters and the application proceeds to the exchange step next) or
that the request failed (if the `grant_token` is not present in the query
parameters).

If the application is a website and the Grant Redirect URI scheme is 
`https`, the application response would typically be HTML, like this:

```
HTTP/1.1 200 OK
Content-Type: text/html

<html>
... inform user here ...
</html>
```

If the application is a mobile or desktop application, the operating system
may open it directly based on the
Grant Redirect URI, or it may load a website that causes the application
to open and passes the query parameters to it.

Where a client application is a multi-user application and the access request
is NOT for a resource that is intended to be shared among all users,
the client application MUST ensure that the currently authenticated user is
the same user that initiated the access request.

The client application SHALL check that the value of the `state` query parameter
matches the stored `state` value in the request record.
If the received value does not match the stored value,
the client application MUST stop processing the access request. When this error happens,
client applications MAY start a new access request.

An application MUST validate the `state` value in the query matches the
stored `state` value from the [request](#request) step. If they do not
match, the client application SHALL stop processing the request. The client
application SHOULD inform the user of the error.

The client SHOULD use its own query or path parameter in the `grant_redirect_uri`
or the `state` value to identify the context of the original access request,
so the application can repeat the resource
access that requires authorization after it obtains the access token
in the exchange step.

## Exchange

The exchange API is used in the following ways:

* to exchange a `grant_token` for an `access_token`
* to refresh an `access_token` using a `refresh_token`
* to refresh a `client_token` using a `refresh_token`
* to exchange a `permit_token` for an `access_token`

If the Webauthz Exchange URI scheme is `https`, the application sends
an HTTPS POST request. An example of an Webauthz Exchange URI is
"https://resource.example.com/webauthz/exchange".

Requests to the exchange API must be authenticated using either a
`client_token` (used when exchanging a `grant_token` for an `access_token`)
or a `refresh_token` (used when refreshing an `access_token` or `client_token`).

A multi-user client application MUST store all information received from
this response in a per-user storage area unless the access is intended
to be shared among multiple users. See
[multi-user applications](#multi-user-applications) for more information.

### Grant token exchange

The fourth step of the authorization routine is for the application
to exchange the `grant_token` for an `access_token`. This is
done because the `grant_token` may be stored in the user's browser
history or in other non-trusted areas due to being passed between
the browser and a mobile or desktop application on the user's system,
and also because the `access_token` may be very large and we want to
avoid relying on clients and servers to handle very large URLs.

The exchange request MUST include an `Authorization` header with the
`client_token`.

The exchange request MUST include the `grant_token` to reference the 
permission(s) granted by the resource owner.

To make the request with `curl`:

```
curl \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <client_token>' \
  -X POST \
  --data '{"grant_token": "<grant_token>"}' \
  '<webauthz_exchange_uri>'
```

Alternatively, the request body may be empty and the parameters
may be appended to the query string portion of the `webauthz_exchange_uri`:

```
curl \
  -H 'Accept: application/json' \
  -H 'Authorization: Bearer <client_token>' \
  -X POST \
  --data '' \
  '<webauthz_exchange_uri>?grant_token=<grant_token>'
```

Note that query string parameter values MUST be URL-encoded.

The authorization server validates the `client_token`, then
validates the `grant_token`.

The validation MUST include that the `client_token` is valid
for a registered client, and that the `grant_token` was issued to
the same client.

If all the checks pass, the authorization server generates the
`access_token` and responds to the application. 

The authorization server MAY include a `refresh_token` attribute
in the response. When a `refresh_token` is provided, it indicates
the application MAY renew the access token when it expires.

A successful exchange response looks like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "access_token": "<access_token>",
  "access_token_max_seconds": <access_token_max_seconds>,
  "access_token_min_seconds": <access_token_min_seconds>,
  "refresh_token": "<refresh_token>",
  "refresh_token_max_seconds": <refresh_token_max_seconds>,
  "permit_token": "<permit_token>",
  "permit_token_max_seconds": <permit_token_max_seconds>
}
```

A successful response object SHALL include the following keys:

* `access_token` is the bearer token that authorizes requests for the resource
* `access_token_max_seconds` is the number of seconds the access token is
  valid; null or undefined means the access token does not expire, or should
  be used indefinitely until access is denied
* `access_token_min_seconds` is the number of seconds that a client must wait
  before attempting to refresh the access token; null or undefined means the
  client should wait until the access token expires before refreshing it; when
  specified, this value MUST be less than or equal to `access_token_max_seconds`
* `refresh_token` is the bearer token that authorizes requests to refresh the
  access token using the exchange API; null or undefined means the access token
  cannot be refreshed
* `refresh_token_max_seconds` is the number of seconds the refresh token is
  valid; null or undefined means the access token does not expire, or should
  be used indefinitely until access is denied
* `permit_token` is a token that can be exchanged for a new set of `access_token`
  and `refresh_token` using the exchange API; null or undefined means the
  client would need to make a new authorization request with the user
* `permit_token_max_seconds` is the number of seconds the permit token is
  valid; null or undefined means the permit token does not expire, or should
  be used indefinitely until access is denied

The application SHOULD store the `access_token` somewhere
safe for subsequent use.

A multi-user client application MUST store all information received from
this response in a per-user storage area unless the access is intended
to be shared among multiple users. See
[multi-user applications](#multi-user-applications) for more information.

The format of `access_token_max_seconds`, `access_token_min_seconds`,
`refresh_token_max_seconds`, and `permit_token_max_seconds`
is a non-negative integer, specifying the number of seconds that
token is valid since the moment it was issued. Applications SHOULD
convert this to their preferred time zone and store the computed
timestamp, or they should store the current timestamp when they received
the token along with the max seconds value and compute the expiration
period later.

The application SHOULD NOT
make resource access requests with the access token after
`access_token_max_seconds` have passed. Clients can refresh
their expired access tokens using the [access token exchange](#access-token-exchange) API.

Where the application received a `refresh_token`, the application SHOULD
attempt to refresh the access token after it expires and use the new
`access_token` for subsequent requests. Where the `access_token_min_seconds`
was also provided in the response, the application MAY attempt to refresh
the access token before it expires and after `access_token_min_seconds` have
elapsed.
See the [access token exchange](#access-token-exchange) section
for more details.

If any of the checks fail, the authorization server generates
a response with the `401 Unauthorized` or `403 Forbidden`,
depending on the failure. A missing or expired `client_token`
would result in `401 Unauthorized`, indicating to the application
that it needs to repeat the [registration](#registration) step
and then start over with a new access [request](#request),
whereas an invalid or expired `grant_token`
would result in `403 Forbidden`, indicating
to the application that the exchange is denied, and this is due
either to a bug in the application code, or to an active attack
in progress, and the request should not be repeated.

When the client receives a `403 Forbidden` response, the client SHOULD
inform the user and start a new access request for the resource.

### Access token exchange

To refresh an access token, the application sends an exchange
request with the `access_token`.

The exchange request MUST include an `Authorization` header with the
`refresh_token` that was issued for the given `access_token`.

```
curl \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <refresh_token>' \
  -X POST \
  --data '{"access_token": "<access_token>"}' \
  '<webauthz_exchange_uri>'
```

Alternatively, the request body may be empty and the parameters
may be appended to the query string portion of the `webauthz_exchange_uri`:

```
curl \
  -H 'Accept: application/json' \
  -H 'Authorization: Bearer <refresh_token>' \
  -X POST \
  --data '' \
  '<webauthz_exchange_uri>?access_token=<access_token>'
```

The authorization server validates the `refresh_token`, then
validates the `access_token`. 

The validation MUST include that the `refresh_token` is valid
for a registered client, and that the `access_token` was issued to
the same client, that the `refresh_token` and the `access_token`
point to the same underlying permission, and that the `access_token`
was created on or after the given `refresh_token`.

An authorization server MUST reject an exchange request containing an expired
refresh token.

An authorization server MUST reject an exchange request containing an access token that was created before the given refresh token.

An authorization server MUST reject an access token exchange request
if the underlying permission has expired or been revoked.

An authorization server MAY reject an exchange request with
`429 Too Many Requests` if the `access_token_min_seconds` has not yet elapsed
or the authorization server does not support early refresh.
The authorization server SHOULD include the `Retry-After` header in the
response to indicate how long the client application should wait before
attempting to refresh that access token. The `Retry-After` value SHOULD be
computed as the difference between the expected earliest refresh time
(`access_token_min_seconds` after the access token creation) and the current
time. If the authorization server does not support early refresh,
it should use an implied value of `access_token_max_seconds` after the access
token creation as the expected earliest refresh time.

If all the checks pass, the authorization server generates a new
`access_token` and responds to the application.

A successful response looks like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "access_token": "<access_token>",
  "access_token_max_seconds": <access_token_max_seconds>,
  "access_token_min_seconds": <access_token_min_seconds>,
  "refresh_token": "<refresh_token>",
  "refresh_token_max_seconds": <refresh_token_max_seconds>
}
```

The application SHOULD replace its old access token with the new
access token and compute the new expiration date to schedule the next
refresh.

A multi-user client application MUST store all information received from
this response in a per-user storage area unless the access is intended
to be shared among multiple users. See
[multi-user applications](#multi-user-applications) for more information.

The format of `access_token_max_seconds`, `access_token_min_seconds`,
and `refresh_token_max_seconds`
is a non-negative integer, specifying the number of seconds that
token is valid since the moment it was issued. Applications SHOULD
convert this to their preferred time zone and store the computed
timestamp, or they should store the current timestamp when they received
the token along with the max seconds value and compute the expiration
period later.

If the `refresh_token` will expire before the next `access_token` refresh,
and where the underlying permission will be valid beyond that time, the
authorization server SHOULD generate and include a new `refresh_token` and
`refresh_token_max_seconds` with the response.

Where the `refresh_token` is included in the exchange response,
the application MUST replace its stored `refresh_token` with the
new `refresh_token`, and use the new `refresh_token` in subsequent
client token refresh exchange requests.

If any of the checks fail, the authorization server generates
an error response. A missing or expired `client_token`
would result in `401 Unauthorized`, indicating to the application
that it needs to repeat the [registration](#registration) step
and then start over with a new access [request](#request),
whereas an invalid or expired `access_token`
would result in `403 Forbidden`, indicating
to the application that the exchange is denied, and this is due
either to a bug in the application code, or to an active attack
in progress, and the request should not be repeated. An attempt to
refresh an access token too early would result in `429 Too Many Requests`
with a `Retry-After` header.

When the client receives a `403 Forbidden` response, the client SHOULD
inform the user and start a new access request for the resource.

### Client token exchange

To refresh a client token, the application sends an exchange
request with the `client_token`.

The exchange request MUST include an `Authorization` header with the
`refresh_token` that was issued for the given `client_token`.

```
curl \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <refresh_token>' \
  -X POST \
  --data '{"client_token": "<client_token>"}' \
  '<webauthz_exchange_uri>'
```

Alternatively, the request body may be empty and the parameters
may be appended to the query string portion of the `webauthz_exchange_uri`:

```
curl \
  -H 'Accept: application/json' \
  -H 'Authorization: Bearer <refresh_token>' \
  -X POST \
  --data '' \
  '<webauthz_exchange_uri>?client_token=<client_token>'
```

The authorization server validates the `refresh_token`, then
validates the `client_token`. 

The validation MUST include that the `refresh_token` is valid
for a registered client, and that the `client_token` was issued to
the same client.

An authorization server MUST reject an exchange request containing an expired
refresh token.

An authorization server MUST reject an exchange request containing a client token that was created before the given refresh token.

An authorization server MUST reject a client token exchange request
if the client registration has expired or been revoked.

An authorization server MAY reject an exchange request with
`429 Too Many Requests` if the `client_token_min_seconds` has not yet elapsed
or the authorization server does not support early refresh.
The authorization server SHOULD include the `Retry-After` header in the
response to indicate how long the client application should wait before
attempting to refresh that client token. The `Retry-After` value SHOULD be
computed as the difference between the expected earliest refresh time
(`client_token_min_seconds` after the client token creation) and the current
time. If the authorization server does not support early refresh,
it should use an implied value of `client_token_max_seconds` after the client
token creation as the expected earliest refresh time.

If all the checks pass, the authorization server generates a new
`client_token` and responds to the application. 

A successful response looks like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "client_token": "<client_token>",
  "client_token_max_seconds": <client_token_max_seconds>,
  "client_token_min_seconds": <client_token_min_seconds>,
  "refresh_token": "<refresh_token>",
  "refresh_token_max_seconds": <refresh_token_max_seconds>
}
```

The response object SHALL include the following keys:

* `client_token` is a bearer token that authorizes the client to use the
  authorization server APIs
* `client_token_max_seconds` is the number of seconds the client token is
  valid; null or undefined means the client token does not expire, or should
  be used indefinitely until access is denied
* `client_token_min_seconds` is the number of seconds that a client must wait
  before attempting to refresh the client token; null or undefined means the
  client should wait until the client token expires before refreshing it; when
  specified, this value MUST be less than or equal to `client_token_max_seconds`
* `refresh_token` is the bearer token that authorizes requests to refresh the
  client token using the exchange API; null or undefined means the client token
  cannot be refreshed
* `refresh_token_max_seconds` is the number of seconds the refresh token is
  valid; null or undefined means the client token does not expire, or should
  be used indefinitely until access is denied

The application SHOULD immediately replace its old client token with the new
client token and compute the new expiration date to schedule the next
refresh.

A multi-user client application MUST store all information received from
this response in a per-user storage area unless the access is intended
to be shared among multiple users. See
[multi-user applications](#multi-user-applications) for more information.

The format of `client_token_max_seconds`, `client_token_min_seconds`,
and `refresh_token_max_seconds`
is a non-negative integer, specifying the number of seconds that
token is valid since the moment it was issued. Applications SHOULD
convert this to their preferred time zone and store the computed
timestamp, or they should store the current timestamp when they received
the token along with the max seconds value and compute the expiration
period later.

If the `refresh_token` will expire before the next `client_token` refresh,
and where the client registration will be valid beyond that time, the
authorization server SHOULD generate and include a new `refresh_token` and
`refresh_token_max_seconds` with the response.

Where the `refresh_token` is included in the exchange response,
the application MUST replace its stored `refresh_token` with the
new `refresh_token`, and use the new `refresh_token` in subsequent
client token refresh exchange requests.

If any of the checks fail, the authorization server generates
a response with the `401 Unauthorized` or `403 Forbidden`,
depending on the failure. A missing or expired `client_token`
would result in `401 Unauthorized`, indicating to the application
that it needs to repeat the [registration](#registration) step
and then start over with a new access [request](#request),
whereas an invalid or expired `refresh_token`
would result in `403 Forbidden`, indicating
to the application that the exchange is denied, and this is due
either to a bug in the application code, or to an active attack
in progress, and the request should not be repeated.

When the client receives a `403 Forbidden` response, the client SHOULD
inform the administrator and start a new client registration request.

### Permit token exchange

To obtain a new set of `access_token` and corresponding `refresh_token` after the `refresh_token` has expired, the application sends an exchange
request with the `permit_token`.

The exchange request MUST include an `Authorization` header with the
`client_token`.

```
curl \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <client_token>' \
  -X POST \
  --data '{"permit_token": "<permit_token>"}' \
  '<webauthz_exchange_uri>'
```

Alternatively, the request body may be empty and the parameters
may be appended to the query string portion of the `webauthz_exchange_uri`:

```
curl \
  -H 'Accept: application/json' \
  -H 'Authorization: Bearer <client_token>' \
  -X POST \
  --data '' \
  '<webauthz_exchange_uri>?permit_token=<permit_token>'
```

The authorization server validates the `client_token`, then
validates the `permit_token`. 

The validation MUST include that the `client_token` is valid
for a registered client, and that the `permit_token` was issued to
the same client.

An authorization server MUST reject an exchange request containing an expired
client token or an expired permit token.

An authorization server MUST reject a permit token exchange request
if the client registration has expired or been revoked, or if the underlying permission referenced by the permit token has expired or been revoked.

If all the checks pass, the authorization server generates a new
`access_token` and `refresh_token` for the permission referenced by the `permit_token`, generates a new `permit_token` for the permission, revokes the `permit_token` that was used in the exchange request, and responds to the application. 

A successful response looks like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "access_token": "<access_token>",
  "access_token_max_seconds": <access_token_max_seconds>,
  "access_token_min_seconds": <access_token_min_seconds>,
  "refresh_token": "<refresh_token>",
  "refresh_token_max_seconds": <refresh_token_max_seconds>,
  "permit_token": "<permit_token>",
  "permit_token_max_seconds": <permit_token_max_seconds>
}
```

The response is the same as the original response from the [grant token exchange](#grant-token-exchange). The client application SHOULD store the new `access_token`, `refresh_token`, and `permit_token` somewhere safe for future use.

A multi-user client application MUST store all information received from
this response in a per-user storage area unless the access is intended
to be shared among multiple users. See
[multi-user applications](#multi-user-applications) for more information.

If any of the checks fail, the authorization server generates
a response with the `401 Unauthorized` or `403 Forbidden`,
depending on the failure. A missing or expired `client_token`
would result in `401 Unauthorized`, indicating to the application
that it needs to repeat the [registration](#registration) step
and then start over with a new access [request](#request),
whereas an invalid or expired `permit_token`
would result in `403 Forbidden`, indicating
to the application that the exchange is denied, and this is due
either to a bug in the application code, or to an active attack
in progress, and the request should not be repeated.

When the client receives a `403 Forbidden` response, the client SHOULD
inform the administrator and start a new client registration request.

## Access

The application may then interact with the resource server
using the `access_token`.

The request and response formats for this
are not specified here. However, it would typically be HTTPS API
requests with the `access_token` provided in a request header.
For example:

```
curl \
  -H 'Authorization: Bearer <access_token>' \
  '<resource_uri>'
```

If an access request is missing the `Authorization` header, or has an
invalid `Authorization` header, or presents an invalid or expired
`access_token`, the resource server MUST deny the request and reply
with a status code of `401 Unauthorized`.

If the resource server is a proxy, it would look for the
`Proxy-Authorization` header and reply with
a status code of `407 Proxy Authentication Required`.

If an access request contains an `Authorization` or `Proxy-Authorization`
header with a valid
`access_token` but the token does not have permission to access the
requested resource or perform the requested operation on that resource,
the resource server MUST deny the request and reply with a status code
of `403 Forbidden`.

The `401 Unauthorized` response and the `403 Forbidden` response
MUST contain a `WWW-Authenticate`
header. The `WWW-Authenticate` header format is defined by
[RFC 2617](https://tools.ietf.org/html/rfc2617) for HTTP/1.1 `Basic`
and `Digest` authentication schemes,
is updated by [RFC 7235](https://tools.ietf.org/html/rfc7235),
and is extended by
[RFC 6750](https://tools.ietf.org/html/rfc6750) with the `Bearer`
scheme. We use the `Bearer` scheme here, and we further extend 
it with two more auth-param attributes, `webauthz_discovery_uri`
and `path`.

The `407 Proxy Authentication Required` response
MUST contain a `Proxy-Authenticate`
header. The `Proxy-Authenticate` header format is defined by
[RFC 2617](https://tools.ietf.org/html/rfc2617) for HTTP/1.1 `Basic`
and `Digest` authentication schemes,
is updated by [RFC 7235](https://tools.ietf.org/html/rfc7235),
but is not mentioned by
[RFC 6750](https://tools.ietf.org/html/rfc6750) for use with the `Bearer`
scheme. We use the `Bearer` scheme here, and we further extend 
it with two more auth-param attributes, `webauthz_discovery_uri`
and `path`.

The `webauthz_discovery_uri` attribute MUST be included to indicate the
settings for the authorization server that can be used to obtain access
to the resource via the Webauthz protocol.

The `error`, `error_description`, and `error_uri` attributes MAY be
included as described in [RFC 6750](https://tools.ietf.org/html/rfc6750).

Here is an example response from an application:

```
401 Unauthorized
WWW-Authenticate: Bearer realm=Example,
  scope=read-contacts%20edit-contacts,
  webauthz_discovery_uri=https%3A%2F%2Fresource.example.com%2Fwebauthz.json,
  path=%2Fcustomer
```

Here is an example response from a proxy:

```
407 Proxy Authentication Required
Proxy-Authenticate: Bearer realm=Proxy,
  scope=access-backend-app,
  webauthz_discovery_uri=https%3A%2F%2Flogin.example.com%2Fwebauthz.json,
  path=%2F
```

The presence of the `webauthz_discovery_uri` indicates the resource server
implements Webauthz and the client SHOULD follow this specification to
obtain an access token and retry the request with the access token.

To make implementation easier, the attribute values MUST be URI-encoded.
The URI-encoded values MAY be quoted or unquoted. For example, both
`%2Fcustomer` and `"%2Fcustomer"` would be valid. Where a value is quoted,
applications MUST remove quotes before URI-decoding the value.

When the `webauthz_discovery_uri` auth-param
attribute is present, the application SHOULD
immediately conduct [discovery](#discovery) to ensure that it
has the most recent document referenced by the `webauthz_discovery_uri`,
and then check if it has already completed [registration](#registration)
with the referenced authorization server, and complete it as necessary,
and then redirect the user to make an access [request](#request) to
the authorization server.

When the application later receives an `access_token` as a result of
completing the process, the application SHOULD automatically retry its
original resource access request.

The `path` attribute MUST be included to inform the application about
URIs where it may pre-emptively include the access token in requests,
namely in all requests to the same origin and matching path, which
could be the exact path, or with a trailing "/", or any deeper path.

NOTE: applications MUST take note of whether the authentication challenge
was received via the `WWW-Authenticate` header or the `Proxy-Authenticate`
header, so they can include the eventual access token in the corresponding
request header, `Authorization` or `Proxy-Authorization`.

# Configuration

To participate in the protocol, each of the parties must determine some
configuration settings that are shared with the other parties.

## Application

The application determines (automatically or via configuration) the following URIs,
which it will share with the authorization server during registration and access
requests:

* Client Origin URI (at registration)
* Grant Redirect URI (for each access request)

After the user grants or denies the access request, the authorization server
will redirect the user to the Grant Redirect URI, with query parameters that
vary depending on the user's selection.

The Grant Redirect URI origin must match the registered Client Origin URI.

## Authorization Server

To enable applications to request access to a resource, the authorization
server must define the following URIs:

* Register Client URI
* Webauthz Request URI

The Register Client URI is used by applications to obtain their client
credentials in the [registration](#registration) step, which are later used
during the [exchange](#exchange) step.

When an application needs to request access to a resource, it redirects
the user to the Webauthz Request URI with some query parameters during the
[request](#request) step.

## Resource Server

To enable applications to automataically discover the authorization server,
the resource server must define the following URIs:

* Discovery URI

This URI should be returned to applications when a request to access a
resource is denied, and the URI should resolve to the discovery document
that directs the application how it can interact with the authorization server.

## Resource Owner

To complete the resource owner's part of the protocol, which is to grant
or deny access requests, the resource owner must already have a
relationship with the resource server and authorization server.

The configuration for this is outside the scope of this protocol, but typically
if the resource owner is a user, then the user must have a means to
authenticate to the authorization server in order to grant or deny
a request.

# Token usage

A token presented to the server must be sufficient to both
identify and authenticate the client.

## Client token usage

Generated by the authorization server in response to an application
registration request.

Webauthz clients are authenticated via a client token which is
opaque to clients. To avoid accidentally disclosing this token to
attackers, the client token MUST NOT be used in any context outside
the the `Authorization` header of client
requests to the authorization server.

Specifically:

* A client token MUST NOT be written to log files in production
* A client token MUST NOT be sent to resource servers
* Hash of a client token MUST NOT be used in any publicly-visible location

An application may use a client token until it expires.

To avoid an interruption in service, a client application MUST
refresh a client token before it expires. Where a
`client_token_min_seconds` value was provided with the client token, a
client application MUST wait at least that amount of time before attempting
to refresh the client token. If a `client_token_min_seconds` value was not
provided with the token, a client SHOULD wait at least 80% of the
`client_token_max_seconds` value before attempting to refresh the client token.

For example, if
`client_token_max_seconds` is 2592000 (30 days), and `client_token_min_seconds` is 2073600 (24.5 days)
or not provided, a client application may
send the client token refresh request after at least 24.5 days have elapsed.

## Grant token usage

Generated by the authorization server in response to an application
resource request.

The grant tokens are used when the user is redirected from
the authorization server back to the application. The
redirect URL includes the grant token.

Grant tokens are used because the access tokens may be too large
to include in a URL. They are essentially a reference to an approved
access request, which only the same application that made the
request can use to later obtain the access token. Although the
grant tokens may be exposed in the user's browser history or through
an inter-process communication mechanism on the user's operating system,
the grant tokens alone are NOT sufficient to obtain access tokens.

To exchange a grant token for an access token, an application must
provide its client token during that request. So if a grant token
is intercepted by some other party that does not have access to the
application, it cannot be used to obtain the corresponding access
token.

Grant tokens cannot be refreshed. If a grant token expires or the
authorization server denies exchanging a grant token for an access
token for any other reason, an application would need to start the
process again with a new request.

## Access token usage

Generated by the authorization server in response to an application
token exchange request. The application must provide a valid
grant token as the input to the exchange.

An application may use an access token until it expires.

To avoid an interruption or delay in access, a client application SHOULD
attempt to refresh an access token before it expires. Where a
`access_token_min_seconds` value was provided with the access token, a
client application MUST wait at least that amount of time before attempting
to refresh the access token. If a `access_token_min_seconds` value was not
provided with the token, a client SHOULD wait at least 80% of the
`access_token_max_seconds` value before attempting to refresh the access token.

For example, if
`access_token_max_seconds` is 4500 (75 minutes), and `access_token_min_seconds`
is 3600 (60 minutes) or not provided, a client application may
send the access token refresh request after at least 60 minutes have elapsed.

## Refresh token usage

Generated by the authorization server in response to an application
client registration or exchange request. The refresh token is used
by the client application to automatically refresh its access token or client token.
This allows the access token or client token validity period to be limited in order
to mitigate exhaustive search attacks.

An application may use a refresh token until it expires.

An authorization server will automatically include new refresh tokens
in exchange responses as needed. Clients cannot explicitly request a
refresh of a refresh token.

## Permit token usage

Generated by the authorization server in response to an exchange request
with a grant token. The permit token is used by the client application to
recover from expired access and refresh tokens without prompting the user
for permission again, as long as the underlying permission has not expired
or been revoked.

This allows client applications to minimize the volume of refresh traffic
to an authorization server. Client applications can refresh access tokens
for a resource while it is being used, and when they detect the usage of
that resource is becoming infrequent they can choose to not refresh access
tokens for that resource without affecting their user experience. When a
client application needs to access that resource again after the access and
refresh tokens have expired, it can use a permit token if one was provided.

An application may use a permit token until it expires. A permit token can only be used once.

An authorization server will automatically include a new permit token
in the permit token exchange response.

# Token format

The format of the token is unspecified because it is opaque
to clients. An authorization server may use any suitable format.

However, we suggest two exemplary formats:

* pointer to data
* encoded data

An authorization server could, for example, use the encoded data
format for grant tokens, access tokens, and client tokens, while
using the pointer to data format for refresh tokens and permit tokens.

## Pointer to data

A lookup token could be any value that the recipient can then
lookup in a database to retrieve the information associated with
the token.

For example, a lookup token could be a UUID, a base64-encoded random
value, or a hash of the associated data. This section describes a
custom format that an authorization server could use for its lookup
tokens.

This format is comprised of two fields, the `client_id` and
the `token_value`, separated by a single character which could
be a colon `:`, period `.`, tilde `~`, or other character.

Both `client_id` and `token_value` SHOULD be URL-safe, but
applications SHOULD escape all dynamic values used in URLs
for safety anyway. The URL-safeness of the `client_id` and
`token_value` only means the encoded version will look
substantially similar to the non-encoded version.

If a colon `:` is used as the separator, it will be URL-encoded
to `%3A`. If a period `.` or tilde `~` are used as the separator,
they do not change in the URL-encoded version.

> An alternative would be to use only the `token_value` as
> the token, and not use the `client_id` as a prefix, but we do
> not recommend this alternative approach. Using the `client_id`
> as a prefix means the token value can be scoped to each client,
> which means attackers trying an exhaustive search attack on the
> token space have to do it separately for each client.

When the authorization server generates the `token_value` for
a client token, a grant token, or an access token,
it computes and stores
the `token_value_digest` using a cryptographic hash function
like SHA-384. The server does NOT store the original `token_value`
in its database.

Where a `token_value` is a base64url-encoded random value,
the trailing equal sign `=` SHOULD be omitted, and the `token_value`
SHOULD be base64-decoded before applying the hash function to
compute the `token_value_digest`.

When the server receives the client token, grant token, or access
token, it parses it to obtain the
`client_id` and `token_value`, then computes the `token_value_digest`
using a cryptographic hash function like SHA-384, and looks up
the appropriate record using the `client_id` and `token_value_digest`.
This prevents anyone with read-only access
to the authorization server's database from being able to
impersonate arbitrary applications.

The data retrieved from storage should include the time the token
was issued, the time it expires (optional), and other attributes.

Authorization servers using tokens of this type MUST NOT store the
original token value so the original token cannot be retrieved from
the authorization server's database and used to masquerade as existing
clients.

## Encoded data

This format could be a JSON Web Token (JWT) as described in
[rfc7519](https://tools.ietf.org/html/rfc7519). See
[jsonwebtoken.io](https://www.jsonwebtoken.io/) for examples.

This is the same method used by OAuth 2.0 implementations.

# Discussion

## Registration modes

### Closed registration

Authorization servers may deny client registrations. This should be the
default mode for authorization servers that are not fully configured
(especially the data storage mechanism and access to storage).

### Open registration

Authorization servers may allow open registration, where any client may
register and use the Webauthz request and exchange APIs. This is
the recommended mode for authorization servers that don't have any
specific need to deny open registration.

### Gated registration

Authorization servers may deny open registration and allow only
pre-authorized clients to register. This mode requires interaction by
the client application administrator (for web applications) or user
(for mobile and desktop applications) to request approval for their
client application.

A pre-authorization for client applications may be confusing to users
who are trying to access a resource because they need to first sign in or
register for access to the authorization server, then configure their
client application with a one-time
registration token, before continuing with their original request.

To avoid such confusion, client applications SHOULD make a clear distinction
in their user interface between 1) a redirect to the authorization server for
pre-authorization to register the client application, and 2) a redirect to
the authorization server for request access to a resource.

Furthermore, web application clients SHOULD NOT redirect their non-administrative
users to the authorization server for pre-authorization. Instead, when a web client
application encounters a non-open authorization server, the client application
SHOULD notify the user that it cannot request access to the resource at that time,
then notify the administrator of the web application that action is required to
register with the authorization server.

## Registration update

In Webauthz, a client application may register with an authorization server
using its `client_name` and `client_origin`, but does not have a way to
update these values after registration. To change these values, a client
application must register again with the new values and obtain its permissions
again from the user.

The Webauthz protocol does not
include a capability to edit these values after registration because when
a user approves a permission for a client application, the user sees
the client application's `client_name` and `client_origin` in the authorization
prompt. If these are allowed to change, a user reviewing their authorizations
later might not recognize a client application to which a permission was
granted.

This could be mitigated by having the authorization server keep track
of the changes and show them to the user. However, this represents additional
development effort for all authorization servers implementing the Webauthz
protocol, whereas simply registering as a new client with the updated
`client_name` or `client_origin` values and then obtaining a new permission
from the user is easy and doesn't require additional development.

Webauthz can be extended to allow clients to register and update their
version information separate from their name, and also to allow clients to
update their `client_name` or `client_origin` values with appropriate
controls or mitigations in place for issues that could arise from such updates.

The following extensions allow edits to the client registration:

* [edit name](client_edit_name/README.md)
* [edit origin](client_edit_origin/README.md)
* [edit version](client_edit_version/README.md)

Alternatively, where `client_origin` must be updated, an application may
establish its own internal redirect from the previous
origin to the new origin to avoid a new registration or any re-verification.

## Storage management

### Token data storage

Authorization servers may delete grant tokens using the lookup format after
their expiration date or after they are successfully exchanged for access tokens.

Authorization servers may delete refresh tokens using the lookup format after
their expiration date.

Authorization servers using the lookup format for access tokens MUST keep
expired access tokens until the corresponding refresh tokens expire so that clients
can exchange expired access tokens.

Authorization servers using the lookup format for client tokens MUST keep
expired client tokens until the corresponding refresh tokens expire so that clients
can exchange expired client tokens.

### Secret key and private key storage

Authorization servers SHOULD delete secret keys used to
generate self-encoded tokens after all tokens generated using those keys
have expired and the corresponding refresh tokens have expired.

Authorization servers SHOULD delete public keys used to
verify self-encoded tokens after all tokens generated using the corresponding
private keys have expired and the corresponding refresh tokens have expired.

Authorization servers SHOULD delete private keys used to
generate self-encoded tokens when those private keys are no
longer needed to generate new tokens.

### Client data

Authorization servers may delete old registrations to limit the
size of the stored client data. Possible schemes include:

* delete all registrations earlier than specified date (some time ago)
* delete all registrations older than a specified number of the most recent registrations
* set an expiration date on new registrations, and routinely deleted expired registrations

When an authorization deletes a client registration, the authorization
server MUST also delete, expire, or revoke all client tokens, access tokens,
refresh tokens, and related permissions granted to the client. 

## Cryptography

For applications, Webauthz does not require using any cryptography
outside of TLS.

For authorization servers, it's possible to start quickly
with a simple token implementation. If you already know and love JWT,
you could start with that instead, but if you don't know what JWT is or
why you'd want to use it, start with simple tokens. It's a decision you
can change later without affecting applications at all, or with only a 
minor impact as they need to request access again to get the new token.

## Secrets

Applications MUST safeguard the folowing secrets:

* Client token and corresponding refresh token
* Access token and corresponding refresh token and permit token, if provided

> The `client_id` and `grant_token` are NOT secret and may be exposed to users
> and browsers (including browser plugins) in the URL during the access request
> process. However, exposure of these values could help an attacker so they
> should be treated as confidential.

Resource servers MUST safeguard the following secrets:

* One or more secrets to verify and decrypt self-encoded access tokens (where applicable)
* API credentials for authorization server or database server to verify lookup access tokens

Authorization servers MUST safeguard the following secrets:

* One or more secrets to generate self-encoded client tokens (where applicable)
* One or more secrets to generate self-encoded access tokens (where applicable)

Authorization servers MUST safeguard the following confidential information:

* Client registration information
* Hashes for lookup tokens (where applicable)
* Granted permissions

## Token management, origins, and paths

Applications MUST keep track of where access tokens should be used, to avoid
accidentally disclosing an access token to an unrelated resource.
This means all access tokens MUST be associated with the origin (scheme, host, and port)
of the resource with which they are used.

Within an origin, access tokens SHOULD be associated to a path, so the application
can pre-emptively send the access token with any request to that path or a sub-path.

An application MUST also track whether a Webauthz challenge was received
via a `401 Unauthorized` response or a `407 Proxy Authentication Required`
response, and include the access token in the corresponding request header,
either `Authorization` or `Proxy-Authorization`. It is possible that a single
request will require both types of access tokens, so the lookup and token management
for resource and proxy authorization should be performed independently.

For example:

1. An application
   makes a request to "https://example1.com/customer/profile" and the server responds
   with a `WWW-Authenticate` or `Proxy-Authenticate` header
   that includes a `realm` attribute with the value
   "Customers", and a `path` attribute with the value "/customer".
   When the application obtains an access token, it is associated
   with the origin "https://example1.com" and the path "/customer". The same
   token will also be sent with access requests for any deeper paths such as
   "/customer/profile/contacts".
2. The application makes a request to "https://example1.com/directory",
   which has the same origin but a different path that does not match "/profile",
   so the application does NOT include
   the access token. The the server responds
   with a `WWW-Authenticate` or `Proxy-Authenticate` header
   that includes a `realm` attribute with the value "Employees".
   The application requests an access token for this,
   and the access token is associated with the origin "https://example1.com" and the
   path "/directory". So the application has two different
   tokens now: when it needs to make another request to "https://example1.com/customer" or
   any matching path, it uses the first token, and whe it needs to make another request to 
   "https://example1.com/directory", it uses the second token.
3. The application makes a request to "https://example1.com/customer/profile/contacts", and
   it automatically includes the first token because the origin "https://example1.com"
   is the same, and the path "/customer/profile/contacts" matches the path associated to the
   access token of "/customer" and is valid for any deeper paths.
   Let's say that accessing the contacts requires an additional permission over what was
   originally granted to the application, so the resource server responds
   with a `WWW-Authenticate` or `Proxy-Authenticate` header
   that includes a new `scope` attribute with a value that
   wasn't originally in the `scope` attribute in step 1.
   The application obtains an access token that is associated with
   the same origin "https://example1.com", the same path "/customer", and now has
   the added scope for contacts associated with it. The application replaces the original token
   with the new token and all subsequent requests to "https://example1.com/customer"
   are made with the new token.
4. The application makes a request to "https://example2.com/api", and does not include
   any access token because the origin "https://example2.com" does not match any
   access token that it has.


When granting access to additional scopes for a resource to which an application
already has an access token,
the authorization server should issue a new access token with both the original
and the new scopes, so that applications only need one token for an origin and path,
and do not need to guess which token to use for which request. An authorization server
could make the permission for different scopes to expire at different times, while
the token itself remains valid, such that an admin permission lasts only 15 minutes before
the user needs to approve it again, while a standard permission might last for
1 hour or more, referenced by the same access token. Issuing a new access token
for the new set of permission provides simplicity for the application, which always
replaces prior access tokens with new access tokens for the same origin and path.
To enforce that the old token is not used, an authorization server could revoke the
original access token when issuing a replacement access token.

## Refresh access tokens

To limit the amount of time that attackers have to mount an attack on an
access token, the authorization server may issue access tokens with a relatively
short validity period. The client may refresh the access tokens using the
[Exchange](#exchange) API to continue accesing the resource.

When using lookup tokens,
the authorization server can revoke the access tokens granted to any
application by deleting tokens or setting their expiration time to
zero.

When using self-encoded tokens, immediate revocation is not possible
since the tokens are verified by a cryptographic key and not by
looking up anything in a database or remote server. This is because
people who use self-encoded tokens do it with the intent to avoid such
lookups on every request.

Authorization servers using short duration self-encoded access tokens may
enable long-term use with refresh tokens. This works by
allowing self-encoded tokens to be used for
a limited time without a database lookup, and then a more occasional
use of the refresh token (with a database lookup) to request a new
access token to be issued. The use of the refresh token provides the
opportunity to revoke access by refusing to issue a new access token.

The [exchange](#exchange) API
allows the client to exchange an old access token for a new access token
by providing the old access token. The authorization server may limit the
number of times a client refreshes a particular access token, or it may
keep track of a maximum refresh date beyond which a particular access token
may not be refreshed. These limits may be stored in the access token
(if it is self-encoded) or in the corresponding record (when using database
lookups) or in a separate underlying permission.

An authorization server informs client applications of the access token
duration using the `access_token_max_seconds` attribute
in the [exchange](#exchange) response.

Where an exchange response includes `refresh_token` and `refresh_token_max_seconds`,
the application may use the refresh token to refresh the access token. A single
refresh token may be used multiple times until a new one is returned in an exchange
response. A refresh token corresponds to a specific access token or its underlying
permission and is not valid for refreshing any other access tokens.

Applications SHOULD avoid making a resource
request with the `access_token` after the expiration date computed with `access_token_max_seconds`
until they have refreshed the access token.

An application MAY pre-emptively refresh the access token before the
`access_token_max_seconds` have passed.

An application may refresh an expired access token any time before the refresh token
expires. If the refresh token expires and the application has not yet obtained a
replacement refresh token, the application must request the access again from the user.

The authorization
server MAY refuse to issue a new access token until the `access_token`
expires, or within a window of time
around the access token expiration date to allow for clock difference and a little
planning to avoid service interruptions.

To obtain a new access token, an application makes an [exchange](#exchange)
request to the authorization server with the `access_token`
instead of the `grant_token`. The [exchange](#exchange) request
MUST include an `Authorization` header with the refresh token.

This means a client application could delegate the access to a subcomponent by sharing
the access token and refresh token, while preventing the subcomponent from requesting
new permissions by not sharing the client token with the subcomponent.

The new access token MAY have different scopes or permissions than the prior access token.

## Access control in resource server

Beyond the obvious attributes identifying the application, realm, and scope
of the token, the authorization server may include other attributes that will
be used for access control.

Scopes required to access a specific resource may be
suggested by the resource server in its 401 Unauthorized response, and may
be edited by the application prior to requesting access. An application MAY
remove or insert scopes from the list. For this reason, the authorization
server MUST display to the user the human-readable meaning of each requested
scope. The authorization server SHOULD allow the user to grant or deny each
scope individually.

One approach to protecting users against applications that attempt to
manipulate the scope list is to use the scopes for broad permissions like
"contacts" and "calendar", and to allow the user to further restrict these
to specific contact groups, calendar names, etc. during the prompt.

When the application attempts to use the token to access
a specific resource, the scope might allow access to the resource type
but the other attributes might control which specific records the
application can read or write.

For example, the authorization server may include a unique identifier for the
user who granted the access. The scope might be "contacts", and the user id
might be "sparky". When the application attempts to use the token to access
a *specific* contact, the user id "sparky" determines *which* contacts -- the ones
for which sparky has read or write access. When approving the access request,
the user might also limit which contacts are accessible to the application.
The user might choose a specific set of contacts, or addressbook,
like "work" or "school", so the token might get a scope of "contacts" and
an additional attribute "contacts_lists=work" to allow just one set of
contacts, or "contact_lists=work,school" to allow both sets. When the resource server
checks an access request for a contact list, it will see the access token has
the scope "contacts", and user "sparky", and then it will check the "contact_lists"
attribute to see which contacts were approved by "sparky" for the application
to access.

This is all specific to the resource server and does not affect the
protocol.

## Multi-user applications

A multi-user client application MUST store all information received from
the exchange API in a per-user storage area unless the access is intended
to be shared among multiple users.

A `client_token` and corresponding
`refresh_token` represent the client application itself and MAY be shared
among all users of a web application, while a desktop or mobile application
typically runs as a specific user and SHOULD store the `client_token` and
corresponding `refresh_token` in the user's storage area as opposed to a
shared system-wide storage on the local system.

An `access_token` and corresponding
`refresh_token` MUST be stored in a per-user storage area unless the
access is intended to be shared among multiple users. A client application
SHOULD clearly inform users when such sharing will take place.

Client applications MUST search for tokens in a per-user storage area
for the current user unless the token is intended for sharing and is stored
in a shared storage area.

To illustrate the problem that could happen if client applications do NOT
store all tokens separately for each user, consider the following sequence
in which Bob accidentally gets access to a resource belonging to Alice:

1. Alice attempts to use resource R with the client application
2. The client application redirects Alice to the authorization server
3. Alice signs in at the authorization server and grants permission for R
4. The client application stores the access token in a shared storage location (INSECURE STEP)
5. Bob attempts to use resource R with the client application
6. The client application finds the access token and uses it (SECURITY FAULT)

However, when a client application stores access tokens separately for each
user, the security issue is resolved and Bob cannot access a resource
belonging to Alice:

1. Alice attempts to use resource R with the client application
2. The client application redirects Alice to the authorization server
3. Alice signs in at the authorization server and grants permission for R
4. The client application stores the access token in Alice's token storage location (OK)
5. Bob attempts to use resource R with the client application
6. The client application redirects Alice to the authorization server
7. Bob signs in at the authorization server, but doesn't have access to R
8. The access request fails and Bob does not get access to R (OK)

The application user id does NOT need to match the authorization server user
id. The simple rule is that tokens obtained by the current user MUST only be
usable by the current user, unless the application has a specific need.
For example, an administrator obtaining a token for the application
that is intended to be used for auto-configuration with a remote system, or for
system-wide or background activities and is
not linked to a specific user on the resource server.

The Webauthz protocol has two steps at the application (initiate a request, 
and later exchange a grant token for an access token) compared to only one step
at the authorization server (approve or deny the access). So it's possible
that a user "A" might initiate a request, redirect to the authorization server,
but switch to user "B" before returning to the application. In this case,
it might not be obvious which user should be assigned the token -- is it the
user who made the request, or the user who received the access token?

Applications SHOULD ensure that the same user is present in both steps,
to avoid confusion. This can be done by storing the current user id with
the request information when initiating the request. Later, when the user
returns from the authorization server, the application can check that it's
the same user that initiated the request.

## Temporary elevated privileges

There are situations where a user may want to approve additional temporary
permissions for an application and then downgrade back to the original
permissions.

You can use refresh tokens to force a refresh of an access token
and take advantage of the refresh as an opportunity to downgrade privileges.
In this method,
the authorization server issues an access token with the elevated privileges
identified as scopes, sets a short validity period for that access token, and
makes a note that when the token is renewed later, to issue the renewed token
without the elevated privileges.

A similar scheme can be used to switch out access tokens according to
a schedule. For example, if an application has different permissions during
work hours compared to non-work hours.

> An alternative method would be to implement time-limited scopes at the
> resource server, where the resource server includes some logic about when
> a specific permission expires. However, this alternative method is not recommended
> because it requires additional development on the resource server, whereas
> the method presented above only requires additional implementation in the
> authorization server and is usable with any Webauthz-compatible resource server.

## Client application types

A Webauthz client application could be a web, mobile, or desktop application.

The `grant_redirect_uri` value is used to redirect users from the authorization
server's access request prompt back to the client application.

For web applications, this is a URL that points back to the same web
application. The client origin indicates the scheme, host, and port on which
the web application is listening for requests.

For mobile and desktop applications, this is a URL that points to a locally
installed application. The client origin indicates the scheme, host,
and port which the user agent will recognize and open the corresponding
application. Some user agents allow multiple applications to be registered
for the same origin and prompt the user to select one. Others will defer to
the operating system which would implement the same capability.

For example, a client origin `https://example.com` could be a web appplication.
It could also be a mobile or desktop application that is registered to open
when a user visits any page under `https://example.com`, and the full URL
would be provided to the application when it opens. This means millions of
people could be running the example application on their mobile or desktop
devices, and the same origin is shared by millions of different client applications.

Because the authorization server cannot distinguish between these client types
based on the client origin value, and because multiple client applications running
on behalf of different users could be using the same client origin value,
an authorization server MUST NOT limit the use of a client origin to a single client
and MUST NOT assume that clients using the same client origin belong to the same
user or administrator. That is, the client id is the ONLY value suitable for
distinguishing different clients, and the client token is the ONLY value suitable
for authenticating a client.

# Differences from OAuth 2.0

## Discovery mechanism

The OAuth 2.0 specification leaves it to application developers to
know where to request access tokens for the resource. This is
reasonable, because in practice, application developers do write
code to access a specific API, and the documentation would indicate
where to request access tokens, where to register clients,
etc. But we can do it a little better.

Webauthz includes a discovery mechanism so that applications don't
need to be pre-configured with the authorization server information.

Webauthz discovery allows a completely automated round-trip between
accessing the resource and being denied, to requesting access, and
then to receiving the access token and re-trying the resource access.

It also means that authorization server operators can make changes
to their server configuration, or have the flexibility to support
different user interfaces for different resource prompts, without
breaking any existing Webauthz clients or having to notify them in
advance of any changes.

The Webauthz discovery document is also a natural extension point
for configuring new authorization server features or announcing
their availability to compatible clients, while providing a
backward-compatible request URI that will either continue working
or notify users that the calling application needs an upgrade
and then returning them to the application without a grant token.

## Only one flow

The authorization code flow in OAuth 2.0 is the inspiration for
Webauthz, so if you're familiar with it, you'll recognize these
five steps:

1. Registration. The client (website or application) registers with the
   authorization server to obtain a `client_id` and `client_token`

2. Request. The client redirects the user to the authorization server
   with an access request for a given `realm` and `scope`

3. Prompt. The authorization server authenticates the user and then
   prompts the user to allow or deny the request

4. Grant. The authorization server redirects the user to the client with a temporary
   `grant_token` ("authorization code" in OAuth 2.0)

5. Exchange. The client exchanges the temporary `grant_token` for an
   `access_token` that can be used to make requests to the resource server;
   this request is authenticated using the `client_token` to ensure only the
   original client gets the `access_token` and `refresh_token`

After the five steps are complete, the client has an authorization token that
it can use to make requests to the resource server.

Notice that in Webauthz, the registration step is considered part of one-time
setup (although it can be repeated as needed, if the registration expires or was
revoked or lost), so the routine authorization sequence is comprised of the four
steps after registration.

Webauthz does not have flows equivalent to any of the other OAuth 2.0 flows.

## Client token instead of client secret

In OAuth 2.0, the `client_secret` is only issued to web applications,
and it's expected to be a long-term secret that specific registered
applications have, and to be stored on their servers.

Then, because mobile apps need to be protected from MITM attacks,
[RFC 7636](https://tools.ietf.org/html/rfc7636) defines a temporar
value called a "code verifier" and its
SHA-256 digest value, the "code challenge".

However, this is over-complicating things. If we treat the `client_id`
and `client_token` to be ephemeral instead of a permanent website-only
setting like "client secret" from OAuth 2.0, then
[RFC 7636](https://tools.ietf.org/html/rfc7636) isn't needed
at all -- mobile apps can simply send their `client_token` along with
their final request for the `access_token`, and since a MITM attacker
cannot get access to the `client_token` (for the same reasons they wouldn't
be able to access the "code verifier" in RFC 7636), the MITM attack would
fail.

## Request step requires back-end request

In OAuth 2.0, an application redirects the user to the authorization
server to request access. In Webauthz, an application first makes a
back-end request to the authorization server with the access request
parameters, obtains a redirect URL from the authorization server, then
redirects the user to that URL to request access.

This backend-request-then-redirect mechanism provides three important
features:

First, since the Webauthz protocol
relies on HTTPS and does not require applications to implement any
other cryptography, the mechanism secures the `client_state`, `realm`,
and `scope` parameters against tampering by the user or any attacker
who may be positioned to view and edit this URL before the user is
redirected.

Second, if the authorization server implements any optional
access request parameters, the backend request has more flexiblity
to accomodate larger amounts of data, or accept structured data, that
is limited by URLs.

Third, it enables the authorization server to prevent
application enumeration by requiring a valid and current access request URL
with the randomly generated request identifier before it provides any
application information to the user. While application privacy is not
needed in every deployment, and applies only to non-users of the application,
the ones that need it will be satisfied by this feature.

## Request step does not specify state parameter

In OAuth 2.0, an application must generate a `state` value and include
it as a query parameter when redirecting the user to the authorization
server. This both identifies the request to the client and acts as a
CSRF token.

In Webauthz, the authorization server generates the `state` value and
includes it in the response of the request API. The authorization server
includes the same `state` value in as a query parameter when redirecting
the user to the client application's `grant_redirect_uri`. The client
application MUST check that the query parameter is the expected value.

In Webauthz, a client application SHOULD include its own CSRF token in the
`grant_redirect_uri`. When user is redirected to that location, the
client application SHOULD check that the CSRF token matches the expected
value. However, since the `state` value can be used for both purposes,
this is not strictly required.

Moving the generation of the `state` value to the authorization server
relieves the client application of the responsibility to generate unique
random values in the protocol, simplifying the implementation of Webauthz
client libraries and applications.

## Exchange step always requires client authentication

In OAuth 2.0, the access token request may be "authenticated"
by merely including a `client_id` parameter, unless the client
has previously registered and obtained credentials to use with
the authorization server.

In Webauthz, client registration is mandatory and cannot
be skipped, and the exchange step requires client authentication
via the client token (for exchanging grant tokens) or a refresh
token (for exchanging access tokens or client tokens).

This means possession of a grant token is not sufficient to obtain
an access token -- the application's client token is also required
to make the exchange. This makes harmless the exposure of the grant
token in the user's browser history or through
an inter-process communication mechanism on the user's operating system.

## Path matching for pre-emptively sending access tokens

The inclusion of the `path` attribute in the `WWW-Authenticate` or
`Proxy-Authenticate` response header,
and its use by the application to determine where the access token may be sent
pre-emptively, is a small improvement over [RFC 2617](https://tools.ietf.org/html/rfc2617),
which assumes that "all paths at or deeper than [the same request URI]
are within the protection space specified by the Basic realm value of the
current challenge", and over OAuth 2.0, which leaves it up to each implementation
to know where the access token should and shouldn't be used.

The improvement is that if the application's first
request is to a deeper path such as "/customer/profile" but the resulting
access token is actually valid for anything under "/customer", the application
will later be doing additional access requests for "/customer/contacts",
"/customer/storage", "/customer/account", etc. when these could all be
covered by the same access token, but the application doesn't know it.

One possible workaround would be to force applications to make their first request
to a "top-level" path such as "/customer", so they can then pre-emptively
include the access token for any deeper paths.

However, the same problem has already
been solved for use of cookies in [RFC 6265](https://tools.ietf.org/html/rfc6265),
and we adopt a similar mechanism in Webauthz with the `path` attribute and request
matching.

Unlike cookies, which ignore the port number, in Webauthz the entire origin
is considered as the namespace for the path.

# Testing

A test environment is available to see how Webauthz works with
one resource server that has a built-in authorization server, and one application.

You can run everything on your local system. Here are the steps:

1. clone [test-resource-node-js](https://github.com/webauthz/test-resource-node-js), then inside run `npm install` and `npm start`
2. visit the url shown in the output (probably "http://localhost:29001")
3. pick a name for yourself and "login", then create a resource like "hello world"
4. clone [test-app-node-js](https://github.com/webauthz/test-app-node-js), then inside run `npm install` and `npm start`
5. visit the url shown in the output (probably "http://localhost:29002")
6. pick a name for yourself and "login", then paste the URL of the resource you created in step 3 and tap "access"
7. you will be redirected to the resource website to approve the request, then back to the application to access the resource

Usernames are not required for Webauthz but they are included in the sample code
to demonstrate that tokens need to be organized per user.

To see this in action,
you can logout of the application and login as another user (just pick another name),
then paste the same URL again from step 3 and tap "access", and you will see that
since the new  user didn't have an access token, you will be redirected to the
authorization service to approve the request.

You can also try logging out of the resource server and login as another user,
create a second resource, and then paste that URL into the application. You will get
a "403 Forbidden" error because the application already has an access token for
the resource but the access token is bound to the first account on the resource,
which doesn't have permission to access the document created by the second account.

In a production enviornment, the application might offer some options on how to
resolve this, since it only needs to request a new access token for that resource,
assuming the user has permission for it.

# Copyright

Copyright (C) 2021 Jonathan Buhacoff

This work is licensed under the Creative Commons Attribution-ShareAlike 4.0 International License. To view a copy of this license, visit http://creativecommons.org/licenses/by-sa/4.0/ or send a letter to Creative Commons, PO Box 1866, Mountain View, CA 94042, USA.
