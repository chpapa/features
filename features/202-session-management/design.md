# Background

Sessions of auth gear should be configurable and manageable.


# Objectives

- App Developer/User should be able to manage sessions.
- App Developer should be able to configure session lifetime.
- App Developer should be able to configure session token transport.


# Proposed Design

## Stateful Session

Sessions would be stateful (i.e. persisted into database), instead of being
stateless. Sessions would be stored in a low-latency horizontally scalable
store for scalability reasons.

Each session would be associated with attributes:
- session ID
- app ID
- user ID
- identity ID
- expiration time
- creation time
- last access time
- creation IP
- last IP
- user agent

Session attributes can be manipulated through management APIs and web-hook
handler.

Each session is associated with a name, default to empty. User can update the
session name using SDK. This name is shown in portal UI.

Gateway is responsible to update last access time, last IP, and user agent
of session before processing the request.


## Session Management

APIs would be provided to manage sessions of a user.

Session list API returns a list of valid session of a user. Each list entry
contains the session ID and session attributes. Access token / refresh token of
the session would not be included. Parsed user agent data is also returned.

Revoke sessions API revokes a specific session, or all sessions other than the
requesting session. A `session_delete` event would be generated for all sessions
to be revoked.

Update session API allows updating name of session. Custom attributes can also
be updated using master key.


## Access / Refresh Token

Each session would have an access token, and optionally a refresh token. Both
type of tokens are opaque: token formats are unspecified and developer should
not attempt to interpret the content of tokens.

Access token would always have a lifetime; access token would be treated as
invalid after its expiry.

Refresh token can be used to obtain a new access token. If the old access token
is still valid, the old access token is invalidated; there would be at most one
valid access token for a session at any time.

Refresh token would always have a maximum lifetime.

Optionally, a session idle timeout can be specified: refresh token (or access
token if refresh token is disabled) must be used at least once before the
timeout, otherwise the session would be expired.

A session is invalidated if its identity is deleted, or its user is disabled.
A session is expired if its refresh token is expired, or its access token is
expired (if refresh token is disabled).
If a session is invalidated, expired, revoked, or logout, its associated access
token / refresh token would be treated as invalid.

A session is invalid if it is invalidated, expired, revoked or logged out.

For auth gear, if the endpoint requires authentication and the session is detected as invalid,
the HTTP response includes a header `x-skygear-try-refresh-token: true`.

For gateway routing request to microservice, if refresh token is enabled and the session is detected as invalid, the gateway does not forward the request and return a HTTP response with header `x-skygear-try-refresh-token: true`. The caveat is that even the endpoint does not requires authentication, the access token is refreshed. If the refresh token was expired, the user will be asked to login again prematurely.

The client SDK must try to refresh the access token if it sees such header in the response.

The refresh flow should take the following steps

1. Attempt to refresh.
1. If refresh failed due to expired refresh token, clear everything and abort with the refresh error.
1. If refresh failed due to any other causes, abort with the refresh error.
1. If refresh succeeded, retry the origin request.

Client-side SDK should handle refreshing of expired access token automatically
if refresh token is available; SDKs would not expose concept of access token /
refresh token; however, developers should still handle situation where session
is no longer valid due to various reasons.


## Session Cookie

Optionally, developer can choose to use cookies to transport session tokens, in
order to facilitate server-side rendering.

If developer choose to enable session cookie:
- Refresh token would not be issued. (See [Appendix](#session-cookie-and-refresh-token))
- Access token would be transported in session cookie.
- Access token would not be returned from APIs.
- Gateway would read access token from cookie only, ignoring `Authorization`
  header. If a token is present in both header and cookie, it would be treated
  as if session is not found.
- To prevent unrecoverable failure (i.e. site unaccessible until cookies are cleared),
  the session cookie would be cleared when the session is not found or invalid,
  instead of producing error response.
- Some SDKs (e.g. mobile SDKs) would not be able to consume it.

### Security Consideration

- HTTPS is required (can be disabled for development purpose).
- Session cookie is HTTP-only; web app and web SDK would not have access to the
  session tokens.
- Session cookie is set with `SameSite=lax` by default; generally, cross-domain
  requests would not contain the cookie.
- Session cookie would not be shared across custom domains.
- App developers are responsible to prevent CSRF in their services.
- For untrusted multi-tenant hosting where sub-domain is allocated to each
  tenant, the top-level domain must be included in Public Suffix List.


## Extra Information Collection

Optionally, developer can send extra information to server.

This information would be associated with the session. Server would update
the session with the information provided in header (if present) every request.

Following information is available at the moment:
- Device name

SDKs should provide helper functions to collect extra information if available.


## API Key per Client

Each app can support multiple clients, such as web client and mobile client.
It is common that web client uses cookies, while mobile client uses
Authorization header.

Developer can create a API key for each client. Each API key identifies a
specific client, and different attributes can be configured for the client:
- Name
- Enabled status
- Session token transport (cookie / `Authorization` header)
- Session token properties (e.g. lifetime)

Session token transport must be specified at client creation. Developer should
not change it after creation.


# Future Works
- Audit log should include session attributes for historical records (#340)
- A country value should be provided, deriving from available information (#354)


# Appendix

[Session management APIs](./api.md)

[Session configuration](./config.md)

[Use cases](./use-cases.md)


## Session Cookie and Refresh Token

It does not make much sense to use refresh token along with session cookie.
Considering where the refresh token is stored:

If refresh token is stored in Cookie, it is not useful. In OAuth, refresh token
is used to reduce the time where refresh token is in transit (it should be
stored in a protected storage most of time). If refresh token is stored in
cookie, which is sent on every request, it would defeat its purpose.

If refresh token is stored at client side, separated with access token, it is
also not useful. If the session token is expired, then either the gateway,
gear/services, or client should refresh it:
- The gateway, gear, and services can refresh it only if the refresh token is
  stored in cookie, which as mentioned, is undesirable.
- in SSR scenario, the client would be too late to refresh the token, since
  server had already rendered a session expired page.

Therefore, it is proposed that session cookie must not be used with refresh
token.


## Privacy Policy & GDPR compliance

Auth gear collects and saves the IP and user agent (including device model,
OS, browser, etc.) of logged in user for session management. The combination of
IP and user agent can be used to distinguish a person, therefore it can be
considered a personal identifier. We consider it is our legitimate interest to
collect this information for security purpose. This information would be
deleted when session is invalidated. An API is provided so that user can
review collected information.

Developers should consider informing user about collected information in
Privacy Policy of their app. Also, developers are responsible for evaluating
GDPR compliance for custom attributes saved in session.


## References

[Auth0 Security](https://auth0.com/blog/common-threats-in-web-app-security/#Cross-Site-Request-Forgery--CSRF-)

[IdentityServer Token Config](http://docs.identityserver.io/en/latest/topics/refresh_tokens.html)

[Public Suffix List](https://publicsuffix.org/)
