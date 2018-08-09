# EmpowerID OpenID Connect Authenticator for Tomcat

This is an authenticator implementation for [Apache Tomcat 8.5](http://tomcat.apache.org/) that allows web-applications to use [OpenID Connect](http://openid.net/connect/) to log users in.

## Table of Contents

* [Introduction](#introduction)
* [Operation](#operation)
* [Installation](#installation)
* [Configuration](#configuration)
* [The Web-Application](#the-web-application)
  * [The Login Page](#the-login-page)
  * [The Login Error Page](#the-login-error-page)
  * [Authorization Info](#authorization-info)

## Introduction

Tomcat comes with a number of [built-in authenticators](http://tomcat.apache.org/tomcat-8.5-doc/config/valve.html#Authentication) for the standard authentication mechanisms defined in section 13.6 of the [Java Servlet 3.1 specification](http://download.oracle.com/otndocs/jcp/servlet-3_1-fr-spec/index.html), such as _HTTP Basic_, _HTTP Digest_, and _Form Based_. These standard authenticators are implemented as [Valves](http://tomcat.apache.org/tomcat-8.5-doc/config/valve.html) and are deployed in the web-application's context automatically depending on the application deployment descriptor's `login-config` element (section 14.4.18 of the Servlet specification). The _OpenID Connect Authenticator_ extends the standard form-based authenticator and adds ability to use an _OpenID Provider_ (OP) to log users in instead of or in addition to the login form hosted by the web-application itself. The OPs that implement the standard and can be used with authenticator include [Auth0](https://auth0.com/), [Google Identity Platform](https://developers.google.com/identity/protocols/OpenIDConnect) (including G Suite), [Amazon Cognito](https://aws.amazon.com/cognito/), [Microsoft Azure AD](https://azure.microsoft.com/en-us/services/active-directory/), [Yahoo!](https://developer.yahoo.com/oauth2/guide/openid_connect/), [empowerID](http://www.empowerid.com/) and others.

The web-application is developed for form-based authentication so that the `auth-method` element in the deployment descriptor's `login-config` is `FORM` and the `form-login-config` element defines the login and error pages included with the web-application. The same web-application can be deployed with or without the _OpenID Connect Authenticator_. The login page, while remaining compatible with standard form-based authentication, is the only place in the whole application that is aware that it can be deployed with the _OpenID Connect Authenticator_ and contains logic that either redirects to the login page hosted by the OP, offers the user links or buttons to go to one of the configured OP login pages or use a login form and standard form-based authentication, or some combination of the above. The authenticator valve and its configuration are specified in the web-application's context, so the same application binary can be deployed with different authenticators and authenticator configurations in different runtime environments. The fact that the application is developed as if for the standard form-based authentication makes this authenticator implementation suitable for adding OpenID Connect authentication capability to legacy web-applications.

**This autnenticator is intended for traditional Java Servlet web-applications with server-side page rendering and use of HTTP sessions. It is not intended for RESTful applications. The same way as the standard authenticator implementations included with Tomcat, this authenticator utilizes server-side HTTP sessions defined in the Servlet specification to maintain the authenticated user information.**

## Operation

The authenticator implements OpenID Connect's [Authorization Code Flow](http://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth). Once Tomcat sees an unauthenticated request to a protected web-application resource, it saves the request details in the HTTP session and forwards the request to the login page configured in the application deployment descriptor's `form-login-config` element. Normally, the login page is a JSP that displays a login form and submits the login information (the username and password) to the special application URI `/j_security_check` (see section 13.6.3.1 of the Java Servlet 3.1 specification). As an extension to the standard form-based process, the _OpenID Connect Authenticator_ provides the login page with special request attributes that include URLs of the login pages for every OP configured in the application context. The login page may contain logic that decides how to present the login options to the user. For example, if the application uses only one OP for the authentication and the local form-based login is disabled, the login page may immediately redirect the user's browser to the OP's login page. If more than one OP can be used by the application to log users in, it may display the login page that presents all the login options: links to the configured OPs as well as the local login form (or none). If the login page detects that it is not deployed with the _OpenID Connect Authenticator_ (the special request attributes containing the OP configurations are missing), it can simply display the login form as any other application designed for the form-based authentication would.

If the login page submits standard login form username and password (as `j_username` and `j_password` form fields) to the `/j_security_check` endpoint, the authenticator proceeds with the standard form-based authentication logic. As an alternative, the user ends up on the login page provided by and hosted at the OP. After the login process is complete at the OP, it redirects the user's browser back to the web-application, namely to its `/j_security_check` endpoint with the authorization code (as `code` URL query string parameter). The authenticator detects that the given `/j_security_check` call is indeed a redirect from the OP by the presence of the `code` parameter. If so, the authorization code is used to call the OP's [Token Endpoint](http://openid.net/specs/openid-connect-core-1_0.html#TokenEndpoint) and exchange it for the [ID Token](http://openid.net/specs/openid-connect-core-1_0.html#IDToken), which is a cryptographically signed object containing the authenticated user information. If the authentication at the OP was not successful, the authenticator displays the login error page configured in the deployment descriptor's `form-login-config` element. As an extension to the standard form-based authentication, the error page may receive special request attributes with the error description.

Once the authenticator exchanges the authoization code for the ID Token, it extracts a field from the token (a _claim_) used as the username in the Tomcat's [Realm](http://tomcat.apache.org/tomcat-8.5-doc/realm-howto.html) and looks up the user. If the user is found in the realm, the user becomes the authenticated user and the HTTP session becomes authenticated. Note, that as opposed to RESTful applications, once the user is authenticated, the authenticator will not make any more calls to the OP for the duration of the HTTP session.

## Installation

The binary release of the authenticator can be downloaded from _Boyle Software, Inc._'s _Maven_ repository at:

http://boylesoftware.com/maven/repo-os/org/bsworks/catalina/authenticator/oidc/tomcat8-oidcauth/

The JAR then can be added to the Tomcat's classpath. For example, by placing it in `$CATALINA_BASE/lib` directory (see Tomcat's [Class Loader How-To](http://tomcat.apache.org/tomcat-8.5-doc/class-loader-howto.html) for more info).

## Configuration

The authenticator is added to Tomcat configuration as a [Valve](http://tomcat.apache.org/tomcat-8.5-doc/config/valve.html). Normally, it goes into the web-application's [Context](http://tomcat.apache.org/tomcat-8.5-doc/config/context.html). For example:

```xml
<Valve className="org.bsworks.catalina.authenticator.oidc.OpenIDConnectAuthenticator"
       providers="..." />
```

The authenticator is configured using the following attributes on the valve:

* `providers` _(required)_ - JSON-like array of objects, each describing a configured OpenID Provider (OP) used by the application. The syntax differs from the standard JSON in that it does not use double quotes around the property names and values (to make it XML attribute value friendly). A property value can be surrounded with single quotes if it contains commas, curly braces or whitespace. Each object describing an OP includes the following properties:

  * `issuer` _(required)_ - The OP's unique _Issuer Identifier_ corresponding to the `iss` claim in the [ID Token](http://openid.net/specs/openid-connect-core-1_0.html#IDToken). The issuer identifier is a URL that is used for identifying the OP to the application, validating the ID Token `iss` claim and, unless `documentConfigurationUrl` property described below is included, to load the OP configuration document according to the [OpenID Connect Discovery](http://openid.net/specs/openid-connect-discovery-1_0.html)'s [Obtaining OpenID Provider Configuration Information](http://openid.net/specs/openid-connect-discovery-1_0.html#ProviderConfig) process (by adding `.well-known/openid-configuration` to the issuer identifier to form the OP configuration document URL).

  * `clientId` _(required)_ - The client ID associated with the web-application at the OP.

  * `clientSecret` _(optional)_ - The client secret. Note, that most of the OPs require a client secret to make calls to the OP endpoints. However, some OPs may support public clients so no client secret is utilized.

  * `extraAuthEndpointParams` _(optional)_ - Extra parameters added to the query string of the OP's [Authorization Endpoint](http://openid.net/specs/openid-connect-core-1_0.html#AuthorizationEndpoint) URL. The value is an object with keys being the parameter names and value being the parameter values.

  * `tokenEndpointAuthMethod` _(optional)_ - Explicitly specified authentication method for the OP's [Token Endpoint](http://openid.net/specs/openid-connect-core-1_0.html#TokenEndpoint). Normally the authenticator will use "client_secret_basic" if the OP configuration includes a client secret and "none" if it doesn't. This property, however, allows to override this logic and force a specific authentication method. For the description of the authentication methods see [Client Authentication](http://openid.net/specs/openid-connect-core-1_0.html#ClientAuthentication). Note, that currently "client_secret_jwt" and "private_key_jwt" methods are not supported.

  * `configurationDocumentUrl` _(optional)_ - URL for the OP configuration document. If not specified, the URL is built from the issued ID using the process defined in [OpenID Connect Discovery](http://openid.net/specs/openid-connect-discovery-1_0.html). However, some non-standard OPs may have their configuration document at a different location. This property allows to configure such non-standard OPs.

  * `usernameClaim` _(optional)_ - Claim in the [ID Token](http://openid.net/specs/openid-connect-core-1_0.html#IDToken) used as the username for the realm. The default is `sub`, which is the ID of the user in the OP database. Often, however, claims such as `email` are more convenient to use as the username. To use properties in nested objects in the ID Token, the `usernameClaim` supports dot notaion.

  * `additionalScopes` _(optional)_ - Space-separated list of scopes to add to the `openid` scope for the OP's [Authorization Endpoint](http://openid.net/specs/openid-connect-core-1_0.html#AuthorizationEndpoint). For example, if `email` claim is used for the username, the `email` scope can be added (`email` claim is normally not included in the ID Token unless explicitely requested as part of the `email`, `profile` or similar scope at the Authorization Endpoint).

* `usernameClaim` _(optional)_ - Default username claim to use for OPs that do not override it in their specific descriptors.

* `additionalScopes` _(optional)_ - Default additional scopes to use for OPs that do not override it in their specific descriptors.

* `hostBaseURI` _(optional)_ - Base URI for the web-application used to construct the `redirect_uri` parameter for the OP's Authorization Endpoint. The `redirect_uri` is constructed by appending `<context_path>/j_security_check` to the `hostBaseURI` value. Normally, there is no need to specify this configuration attribute as the authenticator can automatically construct it from the current request URI.

* `noForm` _(optional)_ - If "true", the form-based authentication is disabled and only OpenID Connect authentication is allowed. The default is "false". The value of the flag is made available to the web-application's login page as well, so it can make decision to display the local login form or not.

* `httpConnectTimeout` _(optional)_ - Timeout in milliseconds used for establishing server-to-server HTTP connections with the OP (see [java.net.URLConnection.setConnectTimeout()](https://docs.oracle.com/javase/8/docs/api/java/net/URLConnection.html#setConnectTimeout-int-)). The default is 5000 (5 seconds).

* `httpReadTimeout` _(optional)_ - Timeout in milliseconds used for reading data in server-to-server HTTP connections with the OP (see [java.net.URLConnection.setReadTimeout()](https://docs.oracle.com/javase/8/docs/api/java/net/URLConnection.html#setReadTimeout-int-)). The default is 5000 (5 seconds).

In addition to the attributes described above, all the attributes of the standard form-based authenticator are supported. For more information see [Form Authenticator Valve](http://tomcat.apache.org/tomcat-8.5-doc/config/valve.html#Form_Authenticator_Valve).

Here is an example of the valve configuration with multiple OpenID Providers and use of the email address as the username:

```xml
<Valve className="org.bsworks.catalina.authenticator.oidc.OpenIDConnectAuthenticator"
       providers="[
           {
               issuer: https://<YOUREMPOWERIDSERVER>,
               configurationDocumentUrl: https://<YOUREMPOWERIDSERVER>/oauth/.well-known/openid-configuration,
               clientId: <YOURCLIENTID>,
               clientSecret: <YOURCLIENTSECRET>,
               tokenEndpointAuthMethod: client_secret_post,
               usernameClaim: attrib.email
           }
       ]"
       usernameClaim="email" additionalScopes="email" />
```

_Note that contrary to the previous releases of this authenticator, special configuration of the realm where username and password must be always the same is no longer required. This allows using the same realm for both local form-based authentication and the OP-based authentication. When OP-based authentication is used, the user is looked up in the realm by the username without checking the password._

## The Web-Application

As mentioned earlier, the web-application is created as if for form-based authentication. For example, in the application's _web.xml_ it can have:

```xml
<login-config>
    <auth-method>FORM</auth-method>
    <realm-name>My Application</realm-name>
    <form-login-config>
        <form-login-page>/WEB-INF/jsps/login.jsp</form-login-page>
        <form-error-page>/WEB-INF/jsps/login-error.jsp</form-error-page>
    </form-login-config>
</login-config>
```

### The Login Page

Normally, an application that uses form-based authentication has something like the following in the `login.jsp`:

```html
<h1>Login</h1>

<form method="post" action="j_security_check">
  <ul>
    <li>
      <label for="usernameInput">Username</label>
      <div><input id="usernameInput" type="text" name="j_username"></div>
    </li>
    <li>
      <label for="passwordInput">Password</label>
      <div><input id="passwordInput" type="password" name="j_password"></div>
    </li>
    <li>
      <button type="submit">Submit</button>
    </li>
  </ul>
</form>
```

As an extension, the _OpenID Connect Authenticator_ provides the login page with a request attribute under name `org.bsworks.oidc.authEndpoints` with the list of authorization endpoints for each OP configured on the authenticator's valve. Each endpoint element includes two properties:

* `issuer` - The OP's Issuer Identifier.
* `url` - The URL, to which to direct the user's browser for the login.

Also, `org.bsworks.oidc.noForm` request attribute contains the `noForm` flag from the authenticator's valve configuration. So, a login page that allows login using multiple OPs as well as the local login form may look like the following:

```jsp
<h1>Login</h1>

<%-- offer OpenID Connect providers if authenticator is configured --%>
<c:set var="authEndpoints" value="${requestScope['org.bsworks.oidc.authEndpoints']}"/>
<c:if test="${!empty authEndpoints}">
<h2>Using OpenID Connect</h2>
<ul>
  <c:forEach items="${authEndpoints}" var="ep">
  <li><a href="${ep.url}"><c:out value="${ep.issuer}"/></a></li>
  </c:forEach>
</ul>
</c:if>

<%-- offer local login form if not explicitely disabled --%>
<c:if test="${!requestScope['org.bsworks.oidc.noForm']}">
<h2>Using Form</h2>
<form method="post" action="j_security_check">
  <ul>
    <li>
      <label for="usernameInput">Username</label>
      <div><input id="usernameInput" type="text" name="j_username"></div>
    </li>
    <li>
      <label for="passwordInput">Password</label>
      <div><input id="passwordInput" type="password" name="j_password"></div>
    </li>
    <li>
      <button type="submit">Submit</button>
    </li>
  </ul>
</form>
</c:if>
```

Some applications don't want to show any application-hosted login page at all. That may be the case if the authenticator is configured with a single OP and local form-based login is not allowed. So, the `login.jsp` then simply renders a redirect to the first OP authorization endpoint URL:

```jsp
<c:redirect url="${requestScope['org.bsworks.oidc.authEndpoints'][0].url}"/>
```

Or something sophisticated, such as:

```jsp
<%-- redirect to the OP is the only option --%>
<c:set var="authEndpoints" value="${requestScope['org.bsworks.oidc.authEndpoints']}"/>
<c:if test="${requestScope['org.bsworks.oidc.noForm'] and fn:length(authEndpoints) eq 1}">
  <c:redirect url="${authEndpoints[0].url}"/>
</c:if>

<%-- render the login page --%>
<html lang="en">
  ...
</html>
```

### The Login Error Page

In addition to the standard form-based authenticator use cases, the login error page configured in the application deployment descriptor's `form-login-config` element is used by the _OpenID Connect Authenticator_ when either the OP's Authorization or Token endpoint comes back with an error. In that case, the authenticator provides the error page with a request attribute under `org.bsworks.oidc.error` name. The value is a bean with the following properties:

* `code` - The error code. The standard codes are described in [Authentication Error Response](http://openid.net/specs/openid-connect-core-1_0.html#AuthError) and [Token Error Response](http://openid.net/specs/openid-connect-core-1_0.html#TokenErrorResponse) of the OpenID Connect specification.
* `description` - Optional human-readable description of the error provided by the OP.
* `infoPageURI` - Optional URL of the page that contains more information about the error.

Also, both `org.bsworks.oidc.authEndpoints` and `org.bsworks.oidc.noForm` request attributes are made available the same way as for the login page. This allows having the login and the login error page to be implemented in a single JSP.

### Authorization Info

Every HTTP session successfully authenticated by the _OpenID Connect Authenticator_ includes an authorization descriptor object in `org.bsworks.oidc.authorization` session attribute. The object exposed the following properties to the application:

* `issuer` - The OP's Issuer Identifier.
* `issuedAt` - A `java.util.Date` with the timestamp when the authorization was issued.
* `accessToken` - The Access Token.
* `tokenType` - The Access Token type (normally "Bearer").
* `expiresIn` - An integer number of seconds after the authorization (access token) issue when it expires. In some cases this value is unavailable, in which case it's -1. That means the expiration period is defined by the OP somewhere else.
* `refreshToken` - Optional refresh token, or `null` if none.
* `scope` - Optional token scope. Usually `null`, which means the requested scope.
* `idToken` - The ID Token.

More information can be found in [section 5.1](https://tools.ietf.org/html/rfc6749#section-5.1) of the OAuth 2.0 Authorization Framework specification.

_Note, that exposing some of the information contained in the authorization object in the application pages may pose a security risk._
