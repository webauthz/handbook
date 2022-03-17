Client Edit
===========

# Introduction

This is an extension to the Webauthz specification.

In Webauthz, a client application may register with an authorization server
using its `client_origin`, but does not have a way to
update this value after registration. To change its `client_origin`, a client
application must register again with the new `client_origin` and obtain its permissions
again from the user.

There are situations where a client application's information changes
and needs to be updated at the authorization server without requiring its
users to re-authorize all permissions.

The capability to edit the `client_origin` after registration was intentionally omitted
from the Webauthz protocol. This extension adds the capability to edit
the `client_origin` after registration.

# Overview

A new endpoint is added to the discovery document to inform clients that
`client_origin` may be edited after registration.

The client edit origin process has the following steps:

* [Edit Client Origin](#edit-client-origin)

When the edit is successfully completed, the authorization server
updates the client's registration to indicate the new `client_origin` value.

The authorization server MUST keep a history of the `client_origin` values
so that users inspecting permissions granted to client applications can see
the client's origin as it appeared when they granted the permission in addition
to its current origin.

# Specification

## Discovery

The client edit origin extension is declared in the discovery document
with `client_edit_origin_uri`. This informs clients about the location of
the client edit origin API.

The discovery response should look like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "webauthz_register_uri": "https://resource.example.com/webauthz/register",
  "webauthz_request_uri": "https://resource.example.com/webauthz/request",
  "webauthz_exchange_uri": "https://resource.example.com/webauthz/exchange",
  "client_edit_origin_uri": "https://resource.example.com/webauthz/client-edit-origin"
}
```

In addition to the keys specified by Webauthz (`webauthz_register_uri`,
`webauthz_request_uri`, and `webauthz_exchange_uri`),
the response object SHALL include values for the following keys:

* `client_edit_origin_uri` indicates where a client can edit the `client_origin` value (see [Edit Client Origin](#edit-client-origin))

The values are determined by the authorization server administrator.

## Edit Client Origin

To use this API, a client MUST already be registered with the authorization server.

A client application edits the `client_origin` value by by sending a request
to the `client_edit_origin_uri` specified in the discovery response.

To make the request with `curl`:

```
curl \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <client_token>' \
  -X POST \
  --data '{"client_origin": "<client_origin>"}' \
  '<client_edit_origin_uri>'
```

If the authorization header is missing or the client token is invalid,
the authorization server SHALL deny the request with
`401 Unauthorized`.

If the `client_origin` parameter is missing or is not a valid client origin, the
authorization server SHALL deny the request with `400 Bad Request`.

The authorization server MAY require special client permissions to use this API.
If the client is not permitted to edit the `client_origin` value, the
authorization server SHALL deny the request with `403 Forbidden`.

An authorization server MAY require a client application to verify
its new `client_origin` value before making the change. This could be done using
the `verify-client-origin` extension. When a client receives a `403 Forbidden`
response from an authorization server that also has the `verify-client-origin`
extension, a client application SHOULD attempt to verify its new `client_origin`
value using the `verify-client-origin` extension and then repeat the
edit request after verification is complete.

A successful response should look like this:

```
HTTP/1.1 204 No Content
```

# Configuration

To participate in the protocol, each of the parties must determine some
configuration settings that are shared with the other parties.

## Application

There are no special configuration requirements for the application.

## Authorization Server

To enable applications to request access to a resource, the authorization
server must define the following URIs:

* Client Edit Origin URI

The Client Edit Origin URI is used by applications to edit the `client_origin`
value after registration (see [Edit Client Origin](#edit-client-origin)).

# Discussion

## Client application types

This extension can be used by mobile, desktop, and web applications.
