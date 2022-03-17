Client Edit
===========

# Introduction

This is an extension to the Webauthz specification.

In Webauthz, a client application may register with an authorization server
using its `client_name`, but does not have a way to
update this value after registration. For this reason, clients are advised
to NOT include their version information in the `client_name`.

However, in situations where the client applications need to report their
version information to the authorization server, there needs to be a way to
do this without including the version information in the `client_name`,
and without requiring clients to obtain their permissions again from the user.

This extension adds the capability to edit a `client_version` value
after registration.

# Overview

A new endpoint is added to the discovery document to inform clients that
`client_version` may be edited after registration.

The client edit version process has the following steps:

* [Edit Client Version](#edit-client-version)

When the edit is successfully completed, the authorization server
updates the client's registration to indicate the new `client_version` value.

The authorization server MUST keep a history of the `client_version` values
so that users inspecting permissions granted to client applications can see
the client's version as it appeared when they granted the permission in addition
to its current version.

# Specification

## Discovery

The client edit version extension is declared in the discovery document
with `client_edit_version_uri`. This informs clients about the location of
the client edit version API.

The discovery response should look like this:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "webauthz_register_uri": "https://resource.example.com/webauthz/register",
  "webauthz_request_uri": "https://resource.example.com/webauthz/request",
  "webauthz_exchange_uri": "https://resource.example.com/webauthz/exchange",
  "client_edit_version_uri": "https://resource.example.com/webauthz/client-edit-version"
}
```

In addition to the keys specified by Webauthz (`webauthz_register_uri`,
`webauthz_request_uri`, and `webauthz_exchange_uri`),
the response object SHALL include values for the following keys:

* `client_edit_version_uri` indicates where a client can edit the `client_version` value (see [Edit Client Version](#edit-client-version))

The values are determined by the authorization server administrator.

## Edit Client Version

To use this API, a client MUST already be registered with the authorization server.

A client application edits the `client_version` value by by sending a request
to the `client_edit_version_uri` specified in the discovery response.

To make the request with `curl`:

```
curl \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <client_token>' \
  -X POST \
  --data '{"client_version": "<client_version>"}' \
  '<client_edit_version_uri>'
```

If the authorization header is missing or the client token is invalid,
the authorization server SHALL deny the request with
`401 Unauthorized`.

If the `client_version` parameter is missing or is not a valid client version, the
authorization server SHALL deny the request with `400 Bad Request`.

The authorization server MAY require special client permissions to use this API.
If the client is not permitted to edit the `client_version` value, the
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

* Client Edit Version URI

The Client Edit Version URI is used by applications to edit the `client_version`
value after registration (see [Edit Client Version](#edit-client-version)).

# Discussion

## Client application types

This extension can be used by mobile, desktop, and web applications.
