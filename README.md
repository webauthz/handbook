Webauthz
========

# Introduction

Webauthz is a token-based authorization protocol that is a simpler
replacement for OAuth 2.0.

Webauthz contains only one sequence, or "flow", which is comparable
to the OAuth 2.0 "authorization code" flow. Webauthz adds
a discovery mechanism and a path matching mechanism that are not
found in OAuth 2.0.

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
needs to access a resource the user controls in a limited way,
without obtaining complete control over it, and
is able to either redirect the user or open a window to to allow the
user to approve access to the resource.

Unlike OAuth 2.0, you don't need to make long-term choices such as which
flow to use, or worry about whether you need to implement alternative
flows for different clients. When using Webauthz, there is only one
implementation so you can pick up the Webauthz libraries and start
integrating them immediately, and things will just work.

# Overview

The Webauthz protocol assumes the existence of the following parties:

* Application (a website, desktop, or mobile application requesting access to a resource)
* Resource Server (a website or other resource being accessed by the application)
* Authorization Server (a website that authenticates the user and manages access to a resource)
* Resource Owner (the user or system authorizing the access)

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

If the URI scheme is `https`, the client can retrieve the two
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

The HTTPS response should look something like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "webauthz_register_uri": "https://resource.example.com/webauthz/register",
  "webauthz_request_uri": "https://resource.example.com/webauthz/request",
  "webauthz_exchange_uri": "https://resource.example.com/webauthz/exchange",
}
```

The application should store this information. The application SHOULD
check to see if the discovery settings have changed each time it receives a
401 or 403 status code from the resource server. Existing HTTP mechanisms
such as the `HEAD` request, entity tags, and conditional requests are sufficient
to ensure the application has the latest version.

## Registration

Before an application can request access to a resource, it must
first register with the authorization server.

If the Register Client URI scheme is `https`, the application sends an HTTPS POST
request. An example of a Webauthz Registration URI is
"https://resource.example.com/webauthz/register".

To make the request with `curl`:

```
curl \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -X POST \
  --data '{"client_name": "<client_name>", "grant_redirect_uri": "<grant_redirect_uri>"}' \
  '<webauthz_register_uri>'
```

The authorization server should generate a random `client_id` and
`client_token`, compute the `client_token_digest` from the `client_token`,
and store the `client_id` and the `client_token_digest` together with
the `client_name` and `grant_redirect_uri` provided by the client,
and then respond to the client with these two generated values.

The HTTPS response should look something like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "client_id": "<client_id>",
  "client_token": "<client_token>"
}
```

## Registration update

This section describes a feature that is not a part of the routine
authorization sequence.

Sometimes an application's `client_name`, `grant_redirect_uri`, or other
registration detail may change, and the application needs to update the
authorization server with the new information.

The authorization server SHOULD allow applications to submit an
authenticated registration request to update registration details. The
authenticated registration request is the same as the original described
above, with the addition of an `Authorization` header.

To make the request with `curl`:

```
curl \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <client_token>' \
  -X POST \
  --data '{"client_name": "<client_name>", "grant_redirect_uri": "<grant_redirect_uri>"}' \
  '<webauthz_register_uri>'
```

The authorization
server SHOULD keep track of changes to `client_name`, so when a resource
owner reviews existing authorizations, if they don't recognize an
application they can see that they previously approved it under a different
name. The authorization server SHOULD keep track of changes to other registration
details in the same way. The authorization server SHOULD allow changes to
the path or query parameters of `grant_redirect_uri`.

The authorization server MUST NOT allow applications to update the origin
component of the `grant_redirect_uri`.

If an application needs to change
its scheme, domain, or port number, it can submit a new registration request
to receive a new set of `client_id` and `<client_token>`. Alternatively, an
application may establish its own redirect from the previous
origin to the new origin to avoid a new registration.

The HTTPS response should look something like this:

```
HTTP/1.1 204 No Content
```


## Request

The first step of the routine authorization sequence is to request access
to a resource as described in this section.

To start the access request, the application sends a request to
the authorization server's Webauthz Request URI. If the Webauthz Request URI
scheme is `https`, the application sends an HTTPS POST request.
An example of a Webauthz Request URI is
"https://resource.example.com/webauthz/request".

The request must include the application's generated `client_state` and the
Webauthz challenge parameters `realm` and `scope`. The request format MAY
be extended by an authorization server to accept additional parameters not
defined in this specification. Any such additional parameters MUST be
optional to preserve interoperability between applications and authorization
servers.

To make the request with `curl`:

```
curl \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <client_token>' \
  -X POST \
  --data '{"client_state": "<client_state>", "realm": "<realm>", "scope": "<scope>"}' \
  '<webauthz_request_uri>'
```

The `webauthz_request_uri` was or obtained via the [discovery](#discovery) step.
The `client_id` is implicit because of the use of the client-specific `client_token`
in the `Authorization` header. The `client_state` is a value generated by
the application to help it associate the subsequent grant, if it occurs, with the
original access request. An application MUST generate a unique `client_state` value
for each access request.
The `realm` and `scope` parameters MUST be included if they were
present in `WWW-Authenticate` or `Proxy-Authenticate` header value in the resource server response.

An authorization server may serve multiple resources, so the `realm` parameter
provides a namespace for the `scope` parameter to avoid a collision in scope names
across various resources.

The authorization server should generate a redirect URL to which the
application can redirect the user to grant or deny the access.
The authorization server MUST generate and include a request identifier
in the URL so it can recognize the user when the user is redirected.

The authorization server responds to the client with the redirect URL.

The HTTPS response should look something like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "redirect": "<access_request_uri>"
}
```

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
server response would typically be HTML, something like this:

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
specific to the resource server, but typically it would display
the request and then offer two choices to the user. It might look
something like this:

```
Webauthz Resource Request

<client_name> (<origin_from_grant_redirect_uri>)
is requesting access to
<resource>.

<permissions>

<optional_parameters>

[Grant] [Deny]
```

The `<client_name>` placeholder SHOULD be a human-readable format
identifying the application making the request. 

The `<origin_from_grant_redirect_uri>` placeholder SHOULD be the
domain name part of the `grant_redirect_uri` the application registered
with the authorization server.

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
the requested access, the redirect will NOT include a `grant_token`,
but MAY include `status=denied`.

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
<grant_redirect_uri>?client_id=<client_id>&client_state=<client_state>&status=denied
```

The access granted link looks like this:

```
<grant_redirect_uri>?client_id=<client_id>&client_state=<client_state>&grant_token=<grant_token>
```

The `grant_redirect_uri` was obtained during the
registration step. The `client_id` and `client_state` were obtained
from the request. The `grant_token` was generated by the authorization
server in response to the user tapping the `[Grant]` button.

When the link is accessed by the user's browser, the application
response should be the user interface of the application that will
either display either a message informing the user to wait while the
request is being finalized (if a `grant_token` is present in the query
parameters and the application proceeds to the exchange step next) or
that the request failed (if the `grant_token` is not present in the query
parameters).

If the application is a website and the Grant Redirect URI scheme is 
`https`, the application response would typically be HTML, something like this:

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

An application MUST validate the `client_id` and `client_state` values
to avoid the exchange step for an access request that it did not originate.
The `client_id` must identify the application's registered client,
for which it has a `client_token`. The `client_state` must identify the
context of the access request, so the application can repeat the resource
access that requires the access token after it obtains the access token
in the exchange step.

## Exchange

The fourth step of the authorization routine is for the application
to exchange the `grant_token` for an `access_token`. This is
done because the `grant_token` may be stored in the user's browser
history or in other non-trusted areas due to being passed between
the browser and a mobile or desktop application on the user's system,
and also because the `access_token` may be very large and we want to
avoid relying on clients and servers to handle very large URLs.

If the Webauthz Exchange URI scheme is `https`, the application sends
an HTTPS POST request. An example of an Webauthz Exchange URI is
"https://resource.example.com/webauthz/exchange".

The exchange request MUST include an `Authorization` header with the
application's `client_token` so that no other party
may exchange a grant token that is intended only for that application.

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

A successful HTTPS response should look something like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "access_token": "<access_token>",
  "access_token_max_seconds": <access_token_max_seconds>,
  "refresh_token": "<refresh_token>",
  "refresh_token_max_seconds": <refresh_token_max_seconds>
}
```

The application SHOULD store the `access_token` somewhere
safe for subsequent use.

Where the response includes the `refresh_token`, the application
SHOULD store the `refresh_token` somewhere safe for subsequent
use and associate it to the `access_token`, because it is only
valid to refresh that specific `access_token`.

Where the response includes the `access_token_max_seconds` attribute,
the application SHOULD store its value directly or compute and store the
corresponding future timestamp. The application SHOULD NOT
make resource access requests with the access token after
`access_token_max_seconds` have passed. Where the exchange response includes
a `refresh_token`, the application SHOULD attempt to refresh the
access token some time before it expires to avoid an extra request
to the resource where its access is denied with the expired
`access_token`.

Where the response includes the `refresh_token_max_seconds` attribute,
the application SHOULD store it directly or compute and store the
corresponding future timestamp. The application SHOULD NOT
make refresh exchange requests with the refresh token after
`refresh_token_max_seconds` have passed. When the access token and
refresh token have both expired, the application SHOULD make
a new [request](#request) for further access.

The format of `access_token_max_seconds` and `refresh_token_max_seconds`
is a non-negative integer, specifying the number of seconds that
token is valid since the moment it was issued. Applications SHOULD
convert this to their preferred time zone and store the computed
timestamp, or they should store the current timestamp when they received
the token along with the max seconds value and compute the expiration
period later.

To refresh an access token, the application repeats the exchange
request but uses the `refresh_token` instead of the `grant_token`:

```
curl \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <client_token>' \
  -X POST \
  --data '{"refresh_token": "<refresh_token>"}' \
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
  '<webauthz_exchange_uri>?refresh_token=<refresh_token>'
```

The authorization server validates the `client_token`, then
validates the `refresh_token`.

The validation MUST include that the `client_token` is valid
for a registered client, and that the `refresh_token` was issued to
the same client.

If all the checks pass, the authorization server generates the
`access_token` and responds to the application. 

The response format for a refresh exchange is the same as
for the original exchange, and may look something like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "access_token": "<access_token>",
  "access_token_max_seconds": <access_token_max_seconds>,
  "refresh_token": "<refresh_token>",
  "refresh_token_max_seconds": <refresh_token_max_seconds>
}
```

Where the `refresh_token` is included in the exchange response,
the application MUST replace its stored `refresh_token` with the
new `refresh_token`, and use the new `refresh_token` in subsequent
refresh exchange requests.

The authorization server SHOULD only
accept refresh exchange requests with the most recent `refresh_token`
value.

If any of the checks fail, the authorization server generates
a response with the `401 Unauthorized` or `403 Forbidden`,
depending on the failure. A missing or expired `client_token`
would result in `401 Unauthorized`, indicating to the application
that it needs to repeat the [registration](#registration) step
and then start over with a new access [request](#request),
whereas an invalid or expired `grant_token` or `refresh_token`,
would result in `403 Forbidden`, indicating
to the application that the exchange is denied, and this is due
either to a bug in the application code, or to an active attack
in progress, and the request should not be repeated.

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

The application determines (automatically or via configuration) the following URI,
which it will share with the authorization server during registration:

* Grant Redirect URI

After the user grants or denies the access request, the authorization server
will redirect the user to the Grant Redirect URI, with query parameters that
vary depending on the user's selection.

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

## Client token

Generated by the authorization server in response to an application
registration request.

## Grant token

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

## Access token

Generated by the authorization server in response to an application
token exchange request. The application must provide a valid
grant token as the input to the exchange.

## Refresh token

When using simple tokens,
the resource server can revoke the access tokens granted to any
application by deleting tokens or setting their expiration time to
zero.

When using self-encoded tokens, revocation is not directly possible
since the tokens are verified by a cryptographic key and not by
looking up anything in a database or remote server. This is because
people who use self-encoded tokens do it with the intent to avoid such
lookups on every request. Refresh tokens are used in OAuth 2.0 as
a workaround, essentially allowing self-encoded tokens to be used for
a limited time without a database lookup, and then a more occasional
use of the refresh token (with a database lookup) to request a new
access token to be issued. The use of the refresh token provides the
opportunity to revoke access by refusing to issue a new access token.

However, there is another reason to implement a refresh functionality,
which is applicable to both token types, and that is simply
to limit the amount of time that attackers have to
mount an attack on the token.

An implementation may limit the validity period of a token
and require the application to obtain a replacement token at the end of
that validity period. This is indicated by the `refresh_token` attribute
provided to the application in the [exchange](#exchange) response.

Where an exchange response includes `refresh_token`, the application MAY
repeat the exchange step to obtain a new access token. The authorization
server MAY refuse to issue a new access token until the `access_token`
expires, or within a window of time
around the access token expiration date to allow for clock difference and a little
planning to avoid service interruptions.

The authorization server MAY include a `access_token_max_seconds` value
in the exchange response to indicate to the application when the
access token becomes invalid. Applications SHOULD avoid making a resource
request with the `access_token` after that date.

An application MAY pre-emptively refresh the access token before the
`access_token_max_seconds` have passed.

The authorization server MAY include a `refresh_token_max_seconds` value
in the exchange response to indicate to the application when the
refresh token becomes invalid. Applications SHOULD avoid making a resource
request with the `refresh_token` after that date.

To obtain a new access token, an application makes an [exchange](#exchange)
request to the authorization server with the `refresh_token`
instead of the `grant_token`. The [exchange](#exchange) request
MUST include an `Authorization` header with the application's client token.

The authorization server response to an exchange request MAY
include a new refresh token to use in addition to a new access token.
The new access token MAY have different scopes than the prior access token.

We use a separate `refresh_token` instead of the original `grant_token`
for the renewal capability because of the possibility that the
`grant_token` is visible to outside parties during the redirect.

# Token format

The format of the token is unspecified because it is opaque
to clients.

However, we suggest two exemplary formats:

* pointer to data
* encoded data

## Pointer to data

This format is comprised of two fields, the `client_id` and
the `token_value`, separated by a single character which could
be a colon `:`, period `.`, tilde `~` or other character.

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

## Encoded data

This format could be a JSON Web Token (JWT) as described in
[rfc7519](https://tools.ietf.org/html/rfc7519). See
[jsonwebtoken.io](https://www.jsonwebtoken.io/) for examples.

This is the same method used by OAuth 2.0 implementations.


# Discussion

## Registration

Authorization servers may delete old registrations to limit the
size of the client data. Possible schemes include:

* delete all registrations earlier than specified date (some time ago)
* delete all registrations older than a specified number of the most recent registrations
* set an expiration date on new registrations, and routinely deleted expired registrations

Some authorization servers may require a pre-approved relationship
to establish access, which is possible while complying with this
specification. The applications can simply register in advance
via a registration resource, and the authorization server operator
may mark individual requests as approved.

## Cryptography

For applications, Webauthz does not require using any cryptography
outside of TLS.

For authorization servers, it's possible to start quickly
with a simple token implementation. If you already know and love JWT,
you could start with that instead, but if you don't know what JWT is or
why you'd want to use it, start with simple tokens. It's a decision you
can change later without affecting applications at all, or with only a 
minor impact as they need to request access again to get the new token.

## Token management, origins, and paths

Applications MUST keep track of where access tokens should be used, to avoid
accidentally disclosing an access token to an unrelated resource.
This means all tokens MUST be associated with the origin (scheme, host, and port)
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

## Access control in application

Because an authorization server is likely to associate a specific
user with a token, applications that serve multiple users MUST store their
access tokens separately for each user. For example, a mobile or desktop
application requesting access to a resource would store any obtained access
tokens in a per-user directory, NOT a system-wide directory, so that if
the OS user changes, then requests will use tokens granted to the current user
and not any other user. A web application would associate all stored tokens
with the user who obtained them, and would ensure that it includes the user id
as a criteria when searching tokens.

The application user id does NOT need to match the authorization server user
id. The simple rule is that tokens obtained by the current user must only be
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
to avoid confusion.

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

# Differences from OAuth 2.0

## Discovery

The OAuth 2.0 specification leaves it to application developers to
know where to request access tokens for the resource. This is
reasoanble, because in practice, application developers do write
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

1. Registration. The client (website or application) registers with the authorization server
   to obtain a `client_id` and `client_token`

2. Request. The client redirects the user to the authorization server with a request
   that includes the `client_id`

3. Prompt. The authorization server prompts the user to allow or deny the request

4. Grant. The authorization server redirects the user to the client with a temporary
   `grant_token` ("authorization code" in OAuth 2.0)

5. Exchange. The client exchanges the temporary `grant_token` for an `access_token`
   that can be used to make requests to the resource server;
   this also requires the `client_id` and `client_token` to 
   ensure only the original client gets the `access_token`

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


## Exchange step always requires client authentication

In OAuth 2.0, the access token request may be "authenticated"
by merely including a `client_id` parameter, unless the client
has previously registered and obtained credentials to use with
the authorization server.

In Webauthz, client registration is mandatory and cannot
be skipped, and the exchange step requires client authentication.

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
