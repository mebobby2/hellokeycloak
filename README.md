# Keycloak

## Tips
* By delegating authentication of the user to Keycloak, the application does not have to know how to authenticate the user. This is especially relevant when the authentication mechanisms change. For example, two-factor authentication can be enabled without having to make changes to the application. This also means the application does not have access to the user's credentials.

## Notes
### JSON Web Signature
By default, Keycloak uses JSON Web Signature (JWS) as the token format. These types of tokens are often referred to as non-opaque tokens, meaning the contents of the token are directly visible to the application.

The token also includes a digital signature, making it possible to verify that the token was indeed issued by Keycloak. In essence, this means that the client can both verify the token and read the contents without a request to Keycloak, resulting in less demand on the Keycloak server and lower latency when processing requests.
