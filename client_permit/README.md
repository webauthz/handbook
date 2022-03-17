Client Permit
=============

# Introduction

This is an extension to the Webauthz specification.

In Webauthz, a client application may register with an authorization server
to use its APIs to request access to a resource. The Webauthz protocol is
written to allow discovery of the authorization server and, using an "open"
registration style, to allow any client to register and start using the APIs.

However, there are situations in which an authorization server needs to limit
access to selected clients instead of allowing an "open" client registration.
This capability could be implemented as a custom flow by each authorization
server that needs it, but Webauthz already has all the necessary pieces.
This extension adds the capability to limit access to authorized clients
as an optional feature.

# Overview

A new endpoint is added to the discovery document to inform clients that
a special permit is required to use the authorization server.

The client permit process has the following steps:

* [Get Client Permit](#get-client-permit)

When client permit is successfully completed, the authorization server
updates the client's registration to indicate it is permitted to use the
other Webauthz APIs. This verification might be a precondition for the client to
start a new access request, to use the exchange API, or to use  other features
of the authorization server that are not in the scope of this specification.

# Specification

## Discovery

The client permit extension is declared in the discovery document
with `client_permit_uri`. This informs clients about the location of
the client permit API.

The discovery response should look like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "webauthz_register_uri": "https://resource.example.com/webauthz/register",
  "webauthz_request_uri": "https://resource.example.com/webauthz/request",
  "webauthz_exchange_uri": "https://resource.example.com/webauthz/exchange",
  "client_permit_uri": "https://resource.example.com/webauthz/client-permit"
}
```

In addition to the keys specified by Webauthz (`webauthz_register_uri`,
`webauthz_request_uri`, and `webauthz_exchange_uri`),
the response object SHALL include values for the following keys:

* `client_permit_uri` indicates where a client can request a client permit (see [Get Client Permit](#get-client-permit))

The values are determined by the authorization server administrator.

## Get Client Permit

A client application starts the client permit process by sending a request
to the `client_permit_uri` specified in the discovery response.

The `client_permit_uri` should be treated by the client application as a
resource of the authorization server. Where the client has an access token
matching that location, the client SHALL include the `Authorization` header
with the `<access_token>`. If the client does not have an access token
matching that location, the client SHALL send the request without the
`Authorization` header.

To make the request with `curl`, without an `Authorization` header:

```
curl \
  -H 'Accept: application/json' \
  '<client_permit_uri>'
```

When the authorization server receives this request, the authorization
server SHALL respond with a `401 Unauthorized` response in accordance
with the Webauthz protocol.

Here is an example response from an authorization server:

```
401 Unauthorized
WWW-Authenticate: Bearer realm=Webauthz,
  scope=client-permit,
  webauthz_discovery_uri=https%3A%2F%2Fresource.example.com%2Fwebauthz.json,
  path=%2Fwebauthz%2Fclient-permit
```

When the client appplication receives this response, the client application
SHALL follow the Webauthz protocol. This includes:

* Discovery (possibly using a cached discovery document if it was recently downloaded)
* Registration (possibly skipped if the client is already registered)
* Request (using the realm, scope, and path provided in the client permit response)
* Prompt (using the redirect URL provided by the request API)
* Grant
* Exchange (to obtain an access token for the client permit API)

When the process is complete, the client application will have an access token
for the client permit API. The client application can then repeat the original
request, this time with the access token.

To make the request with `curl`, with an `Authorization` header:

```
curl \
  -H 'Accept: application/json' \
  -H 'Authorization: Bearer <access_token>' \
  '<client_permit_uri>'
```

When the authorization server receives this request, the authorization server
SHALL validate the `access_token`. If the `access_token` is missing, invalid, or
expired, the authorization server SHALL respond with `401 Unauthorized` as
described above.

The authorization server SHALL determine the `client_id` from the `access_token`
and respond with the client permit information.

A successful response should look like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "client_permit_max_seconds": <client_permit_max_seconds>
}
```

The response object SHALL include the following keys:

* `client_permit_max_seconds` is the number of seconds the client permit is
  valid; null or undefined means the client permit does not expire, or should
  be used indefinitely until access is denied

Where the client permit response includes a `client_permit_max_seconds` value,
clients SHOULD immediately compute and store the client permit expiration date.
To avoid an interruption in service, clients SHOULD notify their administrators
to renew the client permit before the client permit expiration date.

To renew a client permit, clients applications repeat this process by
accessing the `client_permit_uri` without an `Authorization` header.

# Configuration

To participate in the protocol, each of the parties must determine some
configuration settings that are shared with the other parties.

## Application

There are no special configuration requirements for the application.

However, since the client permit process is interactive, client applications
MUST have a way to notify their administrators that a client permit is required
or that it must be renewed. This is out of scope for this specification,
but for example, mobile and desktop apps could show a notification,
while web applications could send an email notification.

## Authorization Server

To enable applications to request access to a resource, the authorization
server must define the following URIs:

* Client Permit URI

The Client Permit URI is used by applications to start a client permit
request in the [Get Client Permit](#get-client-permit) step.

An authorization server must also implement the appropriate user interface
for the interactive client permit request and approval.

# Discussion

## Scope of permit

A client permit is only valid for the authorization server that issued it,
but may cover multiple resource servers that use the same authorization server.

## Client application types

Client permit can be used by mobile, desktop, and web applications.
