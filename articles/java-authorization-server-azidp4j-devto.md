---
title: Writing Java library to build OAuth 2.0 Authorization Server / OpenID Connect Identity Provider
published: false
description: Framework free Java library to build OAuth 2.0 Authorization Server / OIDC Identity Provider
tags: oauth2, openidconnect, oidc, java
# cover_image: https://direct_url_to_image.jpg
# Use a ratio of 100:42 for best results.
# published_at: 2022-12-22 08:53 +0000
---

## AzIdP4J

I'm writing [AzIdP4J](https://github.com/inabajunmr/azidp4j) that the library to build Authorization Server and Identity Provider in Java.
I talk about what I think while developing.

https://github.com/inabajunmr/azidp4j

The library works as the following example.

```java
// convert HTTP Request query parameters to Map
var authorizationRequestQueryParameterMap =
        Map.of(
                "scope", "openid item:read",
                "response_type", "code",
                "client_id", "xyz-client",
                "redirect_uri", "https://client.example.com/callback",
                "state", "abc",
                "nonce", "xyz");
var authorizationRequest =
        new AuthorizationRequest(
                "inabajun", // authenticated user subject. Application needs to implements user authentication. If no user is authenticated, specify null.
                Instant.now().getEpochSecond(),
                Set.of("openid", "item:read"), // User consented scopes. Application needs to implements user consent.
                authorizationRequestQueryParameterMap);
var response = azIdP.authorize(authzReq);
// Library returns what should do next so implements application's behavior.
switch (response.next) {
    case redirect -> {
        // Redirect to response.redirect.redirectTo
    }
    case additionalPage -> {
        // Implements additional process like login or consent or...
    }
    case errorPage -> {
        // Error happend but can't redirect to redirct_uri
    }
}
```

## When you want authorization server / identity provider

When we want authorization server / identity provider, there are the following patterns.

* Develop them by yourself
* Develop them with libraries
* Using services that work as authorization server / identity provider with your application
* Using standalone authorization server / identity provider

In the case of scratch development, we need to write many sources. We need to understand the specifications. We need to continue learning about new specifications and trends...

I often feel to develop user experience myself but delegating processes about protocols like OAuth 2.0 / OIDC to a library. So I develop the library to build OAuth 2.0 Authorization Server / OIDC Identity Provider in Java. (Because Spring Security OAuth was EOL already.)

## Policies

I think the most important thing about a library is the interface.
I decide on the following policies to consider interfaces.

* Don't depend on a specific framework
* Easy to issue access token / ID token
* Applications can implement user experiments by themselves
* No datastore implementations

### Don't depend on a specific framework

OAuth 2.0 / OpenID Connect depends on HTTP as a specification. So if a library can be used without any additional implementations, the library needs to accept HTTP requests directly.

But there are many Java web application frameworks and I want to implement a library that can be used for any framework.

So AzIdP4J has more abstract interfaces than accepting HTTP requests directly.

For example, token request / token response is expressed as following examples.

* [TokenRequest](https://github.com/inabajunmr/azidp4j/blob/main/azidp4j/src/main/java/org/azidp4j/token/request/TokenRequest.java)
* [TokenResponse](https://github.com/inabajunmr/azidp4j/blob/main/azidp4j/src/main/java/org/azidp4j/token/response/TokenResponse.java)

[Spring based example implementation](https://github.com/inabajunmr/azidp4j/blob/main/azidp4j-spring-security-sample/src/main/java/org/azidp4j/springsecuritysample/handler/TokenEndpointHandler.java#L50) is like this.

```java
@PostMapping(value = "token", consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE)
public ResponseEntity<Map> tokenEndpoint(@RequestParam MultiValueMap<String, Object> body) {

    // client authentication by the application...
    // convert HTTP request query parameters to Map
    var request = new TokenRequest(authenticatedClientId, body.toSingleValueMap();
    // Token Request
    var response =
            azIdP.issueToken(request);

    // convert AzIdP4J resposne to HTTP response
    return ResponseEntity.status(response.status).body(response.body);
}
```

On the other hand, AzIdP4J doesn't support functions within the protocol that be highly dependent on the framework. For example, I think many frameworks have functions to authenticate a client and authorize by bearer token so AzIdP4J doesn't support them.

### Easy to issue access token / ID token

I decided to make it easy to implement issuing each token by Authorization Code Flow. If supporting many functions makes interfaces complicated, AzIdP4J doesn't support them.

For example, if AzIdP4J supports the UserInfo endpoint, AzIdP4J needs to know Users but it makes interfaces complicated so I don't support it yet.

### Applications can implement user experiments by themselves

Behavior after authorization request doesn't close in protocols. When the authorization request has prompt=login, the application needs to show a login page but how to provide authentication user experiences are out of the scope of these specifications. The application needs to provide the application's authentication flow. AzIdP4J wants not to affect user experiences like user authentication, consent, etc...

So AzIdP4J provides the following interface.

```Java
// authzReq has authenticated user subject but can expresses not authenticated.
var response = azIdP.authorize(authzReq);
// Library returns what should do next so implements application's behavior.
switch (response.next) {
    case redirect -> {
        // Redirect to response.redirect.redirectTo
    }
    case additionalPage -> {
        // Implements additional process like login or consent or...
    }
    case errorPage -> {
        // Error happend but can't redirect to redirct_uri
    }
```

https://github.com/inabajunmr/azidp4j/blob/main/docs/endpoint-implementations.md#authorization-request

### No datastore implementations

I like [ory/fosite](https://github.com/ory/fosite) interfaces so AzIdP4J imitates them. AzIdP4J needs to manage tokens so it provides only interfaces and an Application needs to implement them.

* [Token Stores Configuration](https://github.com/inabajunmr/azidp4j/blob/main/docs/config.md#token-stores-configuration)

## Things not to support

When I implement protocols, I become to wish to implement any specifications but I decided on things that don't support to control my motivation.

I decided first milestones.

* Make AzIdP4J work at least
* Writing documents with concrete examples

For the milestone, I removed the following things at the first milestone.

* Doesn't support all of the core specification
    * For example, Request Object
    * Doesn't support specifications that I don't understand concrete demands.

It's difficult what specifications should be supported.

### Make AzIdP4J work at least

First of all, I implement accepting authorization requests and issuing access tokens and ID tokens via the token endpoint. I often test [OICD Conformance Test](https://openid.net/certification/testing/) against my implementation While implementing. This makes progress visible so It's good to continue my motivation. The conformance test reveals some specifications that I can't understand correctly also.

### Writing documents with concrete examples

I understood some of the specifications through conformance tests, writing sources and so one. But I can't understand whether the library's interfaces are good or bad. I want feedback about the library so I wrote documents with concrete examples. I'll be glad if you provide feedback.

https://github.com/inabajunmr/azidp4j/tree/main/docs

## How to process development

For continuous checking of the policy "Don't depend on a specific framework", I implement 2 types of sample identity frameworks.

* [with Spring Boot and Spring Security](https://github.com/inabajunmr/azidp4j/tree/main/azidp4j-spring-security-sample)
* [with com.sun.net.httpserver.HTTPServer](https://github.com/inabajunmr/azidp4j/tree/main/azidp4j-httpserver-sample)

The approach helped my decision on interfaces. For example, when I considered whether the application with the library can implement client authentication or not, multiple implementations help my consideration.

The library development with client implementations is better than just with test cases. I sometimes appear cases that I can't discover from the latter.

## Conclusion

I'm writing [AzIdP4J](https://github.com/inabajunmr/azidp4j) that the library to build Authorization Server and Identity Provider.
I talk about what I think while developing.

https://github.com/inabajunmr/azidp4j

The library works as the following example.

```java
// convert HTTP Request query parameters to Map
var authorizationRequestQueryParameterMap =
        Map.of(
                "scope", "openid item:read",
                "response_type", "code",
                "client_id", "xyz-client",
                "redirect_uri", "https://client.example.com/callback",
                "state", "abc",
                "nonce", "xyz");
var authorizationRequest =
        new AuthorizationRequest(
                "inabajun", // authenticated user subject. Application needs to implements user authentication. If no user is authenticated, specify null.
                Instant.now().getEpochSecond(),
                Set.of("openid", "item:read"), // User consented scopes. Application needs to implements user consent.
                authorizationRequestQueryParameterMap);
var response = azIdP.authorize(authzReq);
// Library returns what should do next so implements application's behavior.
switch (response.next) {
    case redirect -> {
        // Redirect to response.redirect.redirectTo
    }
    case additionalPage -> {
        // Implements additional process like login or consent or...
    }
    case errorPage -> {
        // Error happend but can't redirect to redirct_uri
    }
}
```

I hope your feedbacks. Thank you.