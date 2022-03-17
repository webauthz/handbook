Client Edit
===========

# Introduction

This is an extension to the Webauthz specification.

In Webauthz, a client application may register with an authorization server
using its `client_name`, but does not have a way to
update this value after registration. To change its `client_name`, a client
application must register again with the new `client_name` and obtain its permissions
again from the user.

There are situations where a client application's information changes
and needs to be updated at the authorization server without requiring its
users to re-authorize all permissions.

The capability to edit the `client_name` after registration was intentionally omitted
from the Webauthz protocol. This extension adds the capability to edit
the `client_name` after registration.

# Overview

A new endpoint is added to the discovery document to inform clients that
`client_name` may be edited after registration.

The client edit name process has the following steps:

* [Edit Client Name](#edit-client-name)

When the edit is successfully completed, the authorization server
updates the client's registration to indicate the new `client_name` value.

The authorization server MUST keep a history of the `client_name` values
so that users inspecting permissions granted to client applications can see
the client's name as it appeared when they granted the permission in addition
to its current name.

# Specification

## Discovery

The client edit name extension is declared in the discovery document
with `client_edit_name_uri`. This informs clients about the location of
the client edit name API.

The discovery response should look like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "webauthz_register_uri": "https://resource.example.com/webauthz/register",
  "webauthz_request_uri": "https://resource.example.com/webauthz/request",
  "webauthz_exchange_uri": "https://resource.example.com/webauthz/exchange",
  "client_edit_name_uri": "https://resource.example.com/webauthz/client-edit-name"
}
```

In addition to the keys specified by Webauthz (`webauthz_register_uri`,
`webauthz_request_uri`, and `webauthz_exchange_uri`),
the response object SHALL include values for the following keys:

* `client_edit_name_uri` indicates where a client can edit the `client_name` value (see [Edit Client Name](#edit-client-name))

The values are determined by the authorization server administrator.

## Edit Client Name

To use this API, a client MUST already be registered with the authorization server.

A client application edits the `client_name` value by by sending a request
to the `client_edit_name_uri` specified in the discovery response.

To make the request with `curl`:

```
curl \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <client_token>' \
  -X POST \
  --data '{"client_name": "<client_name>"}' \
  '<client_edit_name_uri>'
```

If the authorization header is missing or the client token is invalid,
the authorization server SHALL deny the request with
`401 Unauthorized`.

If the `client_name` parameter is missing or is not a valid client name, the
authorization server SHALL deny the request with `400 Bad Request`.

The authorization server MAY require special client permissions to use this API.
If the client is not permitted to edit the `client_name` value, the
authorization server SHALL deny the request with `403 Forbidden`.

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

* Client Edit Name URI

The Client Edit Name URI is used by applications to edit the `client_name`
value after registration (see [Edit Client Name](#edit-client-name)).

# Discussion

## Client application types

This extension can be used by mobile, desktop, and web applications.
