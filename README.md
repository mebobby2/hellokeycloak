# Keycloak
## Run
* Postgres.app
* docker run -e KEYCLOAK_USER=admin -e KEYCLOAK_PASSWORD=admin -e DB_ADDR=docker.for.mac.localhost -e DB_USER=bobbylei -e DB_VENDOR=postgres -e DB_PORT=5432 -e DB_DATABASE=keycloak -p 8180:8080 quay.io/keycloak/keycloak
  * Docker for Mac
* http://localhost:8080/auth/
## Tips
* By delegating authentication of the user to Keycloak, the application does not have to know how to authenticate the user. This is especially relevant when the authentication mechanisms change. For example, two-factor authentication can be enabled without having to make changes to the application. This also means the application does not have access to the user's credentials.
* Client scopes are entities in Keycloak, which are configured at the realm level and they can be linked to clients. The client scopes are referenced by their name when a request is sent to the Keycloak authorization endpoint with a corresponding value of the scope parameter. A scope is the smallest entity that describes a single permission.
* A role is a collection of multiple scopes. It can be assigned to a user or included in another role.
* Clients in Keycloak have two out of the box protocols; openid-connect & saml

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

### Acess Token Limiting Strategies
* Audience: Allows listing the resource providers that should accept an access token.
* Roles: Through controlling what roles a client has access to, it is possible to control
what roles an application can access on behalf of the user.
* Scope: In Keycloak, scopes are created through client scopes, and an application can only have access to a specific list of scopes. Furthermore, when applications require consent, the user must also grant access to the scope.

### Verifying Access Tokens
Two options; using the token introspection endpoint or validating it directly inside your application (only if it's a JWT).

If you take the latter approach, you need to:
1. Retrieve the public signing keys from the JWKS endpoint provided by Keycloak.
2. Verify the signature of the access token.
3. Verify that the access token has not expired.
4. Verify the issuer, audience, and type of the token.
5. Verify any other claims that your application cares about.

### PKCE
The PKCE extension is an extension to OAuth 2.0 that binds the authorization code to the application that sent the authorization request. This prevents abuse of the authorization code if it is intercepted.

### Securing different types of applications

#### Web Apps
Use Authorization Code flow with the Proof Key for Code Exchange (PKCE) extension.

When porting existing applications to use Keycloak, it may be tempting to keep the login pages in the existing application, then exchanging the username and password for tokens, by using the Resource Owner Password Credential grant to obtain tokens. This would be similar to how you would integrate your application with an LDAP server.

However, this is simply something that you should not be tempted to do. Collecting user credentials in an application effectively means that if a single application is compromised, an attacker would likely have access to all applications that the user can access. This includes applications not secured by Keycloak, as users often reuse passwords. You
also do not have the ability to introduce stronger authentication, such as two-factor authentication. Finally, you do not get all the benefits of using Keycloak with this approach, such as single sign-on (SSO) and social login.

As an alternative to keeping the login pages within your existing applications, you may be tempted to embed the Keycloak login page as an iframe within the application. This is also something that you should avoid doing. With the login page embedded into the application, it can be affected by vulnerabilities in an application, potentially allowing an attacker access to the username and password.

You should get used to the fact that an application should redirect the user to a trusted identity provider for authentication, especially in SSO scenarios.

##### Server-Side Web Apps
When securing a server-side web application with Keycloak, you should register a confidential client with Keycloak. As you are using a confidential client, a leaked authorization code can't be leveraged by an attacker.

You must also configure applicable redirect URIs for the client as otherwise, you are creating what is called an open redirect.

The application leverages the Authorization Code flow to obtain an ID token from Keycloak, which it uses to establish an authenticated HTTP session.

##### SPA with Dedicated API (hosted on same domain)
A SPA that has a dedicated REST API on the same domain should be secured with Keycloak in the same way as a server-side web application. As the application has
a dedicated REST API, it should leverage the Authorization Code flow with a confidential client for the highest level of security, and use an HTTP session to secure the API requests from the client side to the dedicated REST API.

##### Securing a SPA with an intermediary API
The most secure way to invoke external REST APIs from a SPA is through an intermediary API hosted on the same domain as the SPA. By doing this, you are able to leverage a confidential client and tokens are not directly accessible in the browser, which reduces the risk of tokens, especially the refresh token, being leaked.

This type of SPA is often referred to as the backend for frontends patterns. Not only does it have increased security, but it also makes your SPA more portable and may make it easier to develop. This is due to the application not having to directly deal with external APIs, but rather a dedicated REST API built specifically to service the frontend SPA.

As the SPA is making the requests through an intermediary REST API on the same domain, you don't need to deal with CORS in this case.

##### Securing SPA with an external API
The simplest way to secure a SPA with Keycloak is by doing the Authorization Code flow directly from the SPA itself with a public client registered in Keycloak. This is a somewhat less secure approach as the tokens, including the refresh token, are exposed directly to the browser. For very critical applications, such as financial applications, this is not an approach you want to use. However, there are a number of techniques that can be leveraged to provide a good level of security for this approach.

The REST API is required to include CORS headers in the response. Otherwise, the browser would block the SPA from reading the response.

#### Native & Mobile Applications
Use the Authorization Code flow with the PKCE extension instead.

Effectively, this means that your native or mobile application must use a browser to authenticate with Keycloak. In this regard, there are three options available depending on the type of application:
* embedded web view
  * not recommended as it is open to vulnerabilities where the credentials may be intercepted.
  * it also does not enable SSO as there are no shared cookies between multiple applications.
* external user agent (default browser)
* an in-app browser tab without the application

In all the options, the Keycloak login pages are opened in a browser to authenticate the user. After the user is authenticated, the authorization code is returned to the application, which can then obtain tokens from Keycloak.

To return the authorization code to the application, there are four different approaches using special redirect URIs defined by OAuth 2.0: Claimed HTTPS scheme, Custom URI scheme, Loopback interface, and A special redirect URI.

When available, the claimed HTTPS scheme is the recommended approach, as it is more secure. In cases when neither a claimed HTTPS scheme nor a custom scheme can be used, for example, in a CLI, the loopback interface option is a good approach.

#### APIs and Services
When an application wants to invoke a REST API protected by Keycloak, it first obtains an access token from Keycloak, then includes the access token in the authorization header in requests it sends to the REST API.

With microservices, using tokens to secure the services is especially useful as it enables propagating the authentication context when a service invokes another service, making it easy to provide full end-to-end authentication of the user.

The application can first obtain an access token using the Client Credential flow (instead of the standard flow - authorization code flow)

#### Summary
You learned the difference between an internal and an external application, where external applications require asking the user for consent to grant access, while internal applications do not. You then learned how different web application architectures are secured with Keycloak, and why it is more secure to have a backend for a SPA that obtains tokens from Keycloak, instead of directly obtaining tokens in the SPA itself.

## Source code
https://github.com/PacktPublishing/Keycloak-Identity-and-Access-Management-for-Modern-Applications

## Upto
Page 169

Understanding authorization
