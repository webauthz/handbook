# v0.3

* Replaced client_domain with client_origin in registration request
* Added verify-client-origin extension
* Added verify-client-domain extension
* Restored use of refresh token in exchange request authorization header
* Removed client_state from access request
* Added state to access request response
* Removed `status` query parameter from grant_redirect_uri when access is denied

# v0.2

* Added client_token_max_seconds to registration response
* Added redirect and redirect_max_seconds to registration response
* Replaced refresh token with access token in access token exchange
* Added client token refresh to exchange API with refresh token
* Replaced grant_redirect_uri with client_domain in registration request
* Added grant_redirect_uri to access request

# v0.1

* Initial specification, based on OAuth 2.0 "authorization code" flow
