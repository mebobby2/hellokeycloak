# Keycloak

## Tips
* By delegating authentication of the user to Keycloak, the application does not have to know how to authenticate the user. This is especially relevant when the authentication mechanisms change. For example, two-factor authentication can be enabled without having to make changes to the application. This also means the application does not have access to the user's credentials.

## Notes
### JSON Web Signature
By default, Keycloak uses JSON Web Signature (JWS) as the token format. These types of tokens are often referred to as non-opaque tokens, meaning the contents of the token are directly visible to the application.

The token also includes a digital signature, making it possible to verify that the token was indeed issued by Keycloak. In essence, this means that the client can both verify the token and read the contents without a request to Keycloak, resulting in less demand on the Keycloak server and lower latency when processing requests.

### Which OAuth Flow
* If the application is accessing the resource on behalf of itself (the application is the resource owner), use the Client Credentials flow.
* If the application is running on a device without a browser or is input-constrained, use the Device flow. This could, for example, be a smart TV where it would be difficult for the user to enter the username and password.
* If none of the preceding conditions are applicable, use the Authorization Code flow.

Legacy flows that should not be used:
* Implicit flow: This was a simplified flow for native applications and client-side applications, which is now considered insecure and should not be used.
* Resource Owner Password Credentials flow: In this flow, the application collects the user's credentials directly and exchanges them for an access token. It may
be tempting to use this grant type for native applications, when a browser is not available, or simply because you want the login form to be directly integrated with your application. You should not be tempted, though. It is inherently insecure as you are exposing the user's credentials directly to the application, and you will also run into other problems in the long run, when you want your users to use stronger authentication than only a password, for example.

### OAuth Client Types
Within an OAuth 2.0 flow there are two client types, which are confidential and public clients. Confidential clients are applications such as a server-side web application that are able to safely store credentials that they can use to authenticate with the authorization server. Public clients, on the other hand, are client-side applications that are not able to safely store credentials. As public clients are not able to authenticate with the authorization server, there are two safeguards in place:

* The authorization server will only send the authorization code to an application hosted on a pre-configured URL, in the form of a previously registered redirect URI.
* Proof Key for Code Exchange (PKCE, RFC 7636), which is an extension to OAuth 2.0, prevents anyone that intercepts an authorization code from exchanging it for an access token.

### OAuth Refresh Token
As access tokens are passed around from the application to services, they typically have a short lifetime. To allow applications to obtain new access tokens without going through the complete flow, a refresh token is used. A refresh token should be kept securely by the application and can be used by the application to obtain new access tokens.

### Tokens
* Bearer Tokens (RFC 6750): OAuth 2.0 does not describe the type of access token, or how it should be used. Bearer tokens are by far the most commonly used type of access tokens, and they are typically sent to resource servers through the HTTP Authorization header. They can also be sent in the form-encoded body, or as a query parameter. An important thing to note here is that sending bearer tokens as a query parameter has inherent security weaknesses and should be avoided.
* Token Introspection (RFC 7662): In OAuth 2.0, the contents of access tokens are opaque to applications, which means the content of the access token is not readable by the application. The token introspection endpoint allows the client to obtain information about the access token without understanding its format.
* Token Revocation (RFC 7009): OAuth 2.0 considers how access tokens are issued to applications, but not how they are revoked. This is covered by the token revocation endpoint.

### Is Token Introspection Needed for JWTs
JWTs are typically validated locally on the resource server.

It's a technical detail that IdentityServer can also validate JWTs at the introspection endpoint. That could be used e.g. when the resource server does not have an appropriate JWT library (and you don't want to store reference tokens on the IS side).

https://stackoverflow.com/questions/46802954/is-introspection-endpoint-needed-for-jwt-token

### OpenID Connect
While OAuth 2.0 is a protocol for authorization, it does not cover authentication. OpenID Connect builds on top of OAuth 2.0 to add an authentication layer.

In essence, in an OpenID Connect protocol flow, the Relying Party requests the identity of the end user from the OpenID Provider. As it builds on top of OAuth 2.0 at the same time as the identity of the user is requested, it can also obtain an access token.

OpenID Connect utilizes the Authorization Code grant type from OAuth 2.0. The main difference is that the client includes scope=openid in the initial request, which makes it an authentication request, rather than an authorization request.

While OAuth 2.0 calls the different flows grant types, OpenID Connect refers to them as flows. There are two flows in OpenID Connect that you should care about:
* Authorization Code flow: This uses the same flow as the OAuth 2.0 Authorization Code grant type and returns an authorization code like OAuth 2.0 that can be exchanged for an ID token, an access token, and a refresh token
* Hybrid flow: In the Hybrid flow, the ID token is returned from the initial request alongside an authorization code

Just like OAuth 2.0, OpenID Connect also defines the Implicit flow. However, we recommend that you do not use the Implicit flow at all.

OpenID Connect does not define equivalents to the Client Credential flow and the Device flow. This makes sense as neither of these flows requires authenticating users, instead just granting access to a service.

OpenID Connect has two additional concepts on top of OAuth 2.0. It clearly specifies the format of the ID token by leveraging the JWT specification, which, unlike the access token in OAuth 2.0, is not opaque. It has a well-specified format, and the values (called claims) within the token can be directly read by the client. This allows the clients to discover information about the authenticated user in a standard way. In addition, it defines a userinfo endpoint, which can be invoked with an access token and returns the same standard claims as found in the ID token. In the next chapter, we will cover the userinfo endpoint in more detail, including how you can control what information is returned for a user.

### Why OAuth Uses JWT
Using a standard format, which is relatively easily consumable, makes it easier to integrate with Keycloak. As JWT is based on JSON, it can also easily be parsed and understood in any programming language.

In addition, as the resource servers are now able to directly read the value of the access token, they do not always have to make a request to the OAuth 2.0 token introspection endpoint, or the OpenID Connect UserInfo endpoint. This potentially eliminates two additional requests to Keycloak for a request to the resource server, reducing latency as well as significantly reducing the number of requests to Keycloak.

OpenID Connect Discovery endpoint advertises an endpoint where the JSON Web Key Set (JWKS) can be retrieved, as well as what signing and encryption mechanisms from the JWA specification are supported.

When a resource server receives an access token, it is able to verify the token in the following ways:
* Retrieving the JWKS URL from the OpenID Connect Discovery endpoint.
* Downloading the public signing keys for the OpenID Provider from the JWKS URL endpoint. These are typically cached/stored at the Resource Server.
* Verifying the signature of the token using the public signing keys from the OpenID Provider.

### How should you deal with logout?
The simplest approach is simply to rely on relatively short application sessions and token expiration. As Keycloak will keep the user logged in, it is possible to effectively use short application sessions without requiring users to frequently re-authenticate.

In other cases, or where logout has to be instant, you should leverage OIDC Back-Channel logout.

Do not use the OpenID Connect Front-Channel Logout as it renders a hidden iframe for each application. This suffers from browsers blocking third-party content, which means that when the OpenID Provider opens the logout endpoint in an iframe there is no access to any application-level cookies, leaving the application unable to access the current authentication session.

## Source code
https://github.com/PacktPublishing/Keycloak-Identity-and-Access-Management-for-Modern-Applications

## Upto
Page 80

Invoking the UserInfo endpoint
