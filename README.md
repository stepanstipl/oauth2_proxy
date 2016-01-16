oauth2_proxy
=================

<small>(This project was renamed from Google Auth Proxy - May 2015)</small>

A reverse proxy and static file server that provides authentication using Providers (Google, Github, and others)
to validate accounts by email, domain or group.

[![Build Status](https://secure.travis-ci.org/bitly/oauth2_proxy.png?branch=master)](http://travis-ci.org/bitly/oauth2_proxy)


![Sign In Page](https://cloud.githubusercontent.com/assets/45028/4970624/7feb7dd8-6886-11e4-93e0-c9904af44ea8.png)

## Architecture

![OAuth2 Proxy Architecture](https://cloud.githubusercontent.com/assets/45028/8027702/bd040b7a-0d6a-11e5-85b9-f8d953d04f39.png)

## Installation

1. Download [Prebuilt Binary](https://github.com/bitly/oauth2_proxy/releases) (current release is `v2.0.1`) or build with `$ go get github.com/bitly/oauth2_proxy` which will put the binary in `$GOROOT/bin`
2. Select a Provider and Register an OAuth Application with a Provider
3. Configure OAuth2 Proxy using config file, command line options, or environment variables
4. Configure SSL or Deploy behind a SSL endpoint (example provided for Nginx)

## OAuth Provider Configuration

You will need to register an OAuth application with a Provider (Google, Github or another provider), and configure it with Redirect URI(s) for the domain you intend to run `oauth2_proxy` on.

Valid providers are :

* [Google](#google-auth-provider) *default*
* [GitHub](#github-auth-provider)
* [LinkedIn](#linkedin-auth-provider)
* [MyUSA](#myusa-auth-provider)

The provider can be selected using the `provider` configuration value.

### Google Auth Provider

For Google, the registration steps are:

1. Create a new project: https://console.developers.google.com/project
2. Under "APIs & Auth", choose "Credentials"
3. Now, choose "Create new Client ID"
   * The Application Type should be **Web application** and click **Configure Consent Screen**
   * Fill out the appropriate details on the Consent Screen page and hit **Save**
   * On the next screen, leaving **Web Application** checked, enter your domain in the Authorized Javascript Origins `https://internal.yourcompany.com`
   * Enter the correct Authorized Redirect URL `https://internal.yourcompany.com/oauth2/callback`
     * NOTE: `oauth2_proxy` will _only_ callback on the path `/oauth2/callback`
4. Take note of the **Client ID** and **Client Secret**

It's recommended to refresh sessions on a short interval (1h) with `cookie-refresh` setting which validates that the account is still authorized.

#### Restrict auth to specific Google groups on your domain. (optional)

1. Create a service account: https://developers.google.com/identity/protocols/OAuth2ServiceAccount and make sure to download the json file.
2. Make note of the Client ID for a future step.
3. Under "APIs & Auth", choose APIs.
4. Click on Admin SDK and then Enable API.
5. Follow the steps on https://developers.google.com/admin-sdk/directory/v1/guides/delegation#delegate_domain-wide_authority_to_your_service_account and give the client id from step 2 the following oauth scopes:
```
https://www.googleapis.com/auth/admin.directory.group.readonly
https://www.googleapis.com/auth/admin.directory.user.readonly
```
6. Follow the steps on https://support.google.com/a/answer/60757 to enable Admin API access.
7. Create or choose an existing administrative email address on the Gmail domain to assign to the ```google-admin-email``` flag. This email will be impersonated by this client to make calls to the Admin SDK. See the note on the link from step 5 for the reason why.
8. Create or choose an existing email group and set that email to the ```google-group``` flag. You can pass multiple instances of this flag with different groups
and the user will be checked against all the provided groups.
9. Lock down the permissions on the json file downloaded from step 1 so only oauth2_proxy is able to read the file and set the path to the file in the ```google-service-account-json``` flag.
10. Restart oauth2_proxy.

Note: The user is checked against the group members list on initial authentication and every time the token is refreshed ( about once an hour ).

### GitHub Auth Provider

1. Create a new project: https://github.com/settings/developers
2. Under `Authorization callback URL` enter the correct url ie `https://internal.yourcompany.com/oauth2/callback`

The GitHub auth provider supports two additional parameters to restrict authentication to Organization or Team level access. Restricting by org and team is normally accompanied with `--email-domain=*`

    -github-org="": restrict logins to members of this organisation
    -github-team="": restrict logins to members of this team


### LinkedIn Auth Provider

For LinkedIn, the registration steps are:

1. Create a new project: https://www.linkedin.com/secure/developer
2. In the OAuth User Agreement section:
   * In default scope, select r_basicprofile and r_emailaddress.
   * In "OAuth 2.0 Redirect URLs", enter `https://internal.yourcompany.com/oauth2/callback`
3. Fill in the remaining required fields and Save.
4. Take note of the **Consumer Key / API Key** and **Consumer Secret / Secret Key**

### MyUSA Auth Provider

The [MyUSA](https://alpha.my.usa.gov) authentication service ([GitHub](https://github.com/18F/myusa))

## Email Authentication

To authorize by email domain use `--email-domain=yourcompany.com`. To authorize individual email addresses use `--authenticated-emails-file=/path/to/file` with one email per line. To authorize all email addresse use `--email-domain=*`.

## Configuration

`oauth2_proxy` can be configured via [config file](#config-file), [command line options](#command-line-options) or [environment variables](#environment-variables).

### Config File

An example [oauth2_proxy.cfg](contrib/oauth2_proxy.cfg.example) config file is in the contrib directory. It can be used by specifying `-config=/etc/oauth2_proxy.cfg`

### Command Line Options

```
Usage of oauth2_proxy:
  -approval_prompt="force": Oauth approval_prompt
  -authenticated-emails-file="": authenticate against emails via file (one per line)
  -client-id="": the OAuth Client ID: ie: "123456.apps.googleusercontent.com"
  -client-secret="": the OAuth Client Secret
  -config="": path to config file
  -cookie-domain="": an optional cookie domain to force cookies to (ie: .yourcompany.com)*
  -cookie-expire=168h0m0s: expire timeframe for cookie
  -cookie-httponly=true: set HttpOnly cookie flag
  -cookie-key="_oauth2_proxy": the name of the cookie that the oauth_proxy creates
  -cookie-refresh=0: refresh the cookie after this duration; 0 to disable
  -cookie-secret="": the seed string for secure cookies
  -cookie-secure=true: set secure (HTTPS) cookie flag
  -custom-templates-dir="": path to custom html templates
  -display-htpasswd-form=true: display username / password login form if an htpasswd file is provided
  -email-domain=: authenticate emails with the specified domain (may be given multiple times). Use * to authenticate any email
  -github-org="": restrict logins to members of this organisation
  -github-team="": restrict logins to members of this team
  -google-group="": restrict logins to members of this google group
  -google-admin-email="": the google admin to impersonate for api calls
  -google-service-account-json="": the path to the service account json credentials

  -htpasswd-file="": additionally authenticate against a htpasswd file. Entries must be created with "htpasswd -s" for SHA encryption
  -http-address="127.0.0.1:4180": [http://]<addr>:<port> or unix://<path> to listen on for HTTP clients
  -https-address=":443": <addr>:<port> to listen on for HTTPS clients
  -login-url="": Authentication endpoint
  -pass-access-token=false: pass OAuth access_token to upstream via X-Forwarded-Access-Token header
  -pass-basic-auth=true: pass HTTP Basic Auth, X-Forwarded-User and X-Forwarded-Email information to upstream
  -basic-auth-password="": the password to set when passing the HTTP Basic Auth header
  -pass-host-header=true: pass the request Host Header to upstream
  -profile-url="": Profile access endpoint
  -provider="google": OAuth provider
  -proxy-prefix="/oauth2": the url root path that this proxy should be nested under (e.g. /<oauth2>/sign_in)
  -redeem-url="": Token redemption endpoint
  -redirect-url="": the OAuth Redirect URL. ie: "https://internalapp.yourcompany.com/oauth2/callback"
  -request-logging=true: Log requests to stdout
  -scope="": Oauth scope specification
  -skip-auth-regex=: bypass authentication for requests path's that match (may be given multiple times)
  -tls-cert="": path to certificate file
  -tls-key="": path to private key file
  -upstream=: the http url(s) of the upstream endpoint or file:// paths for static files. Routing is based on the path
  -validate-url="": Access token validation endpoint
  -version=false: print version string
```

See below for provider specific options

### Upstreams Configuration

`oauth2_proxy` supports having multiple upstreams, and has the option to pass requests on to HTTP(S) servers or serve static files from the file system. HTTP and HTTPS upstreams are configured by providing a URL such as `http://127.0.0.1:8080/` for the upstream parameter, that will forward all authenticated requests to be forwarded to the upstream server. If you instead provide `http://127.0.0.1:8080/some/path/` then it will only be requests that start with `/some/path/` which are forwarded to the upstream.

Static file paths are configured as a file:// URL. `file:///var/www/static/` will serve the files from that directory at `http://[oauth2_proxy url]/var/www/static/`, which may not be what you want. You can provide the path to where the files should be available by adding a fragment to the configured URL. The value of the fragment will then be used to specify which path the files are available at. `file:///var/www/static/#/static/` will ie. make `/var/www/static/` available at `http://[oauth2_proxy url]/static/`.

Multiple upstreams can either be configured by supplying a comma separated list to the `-upstream` parameter, supplying the parameter multiple times or provinding a list in the [config file](#config-file). When multiple upstreams are used routing to them will be based on the path they are set up with.

### Environment variables

The environment variables `OAUTH2_PROXY_CLIENT_ID`, `OAUTH2_PROXY_CLIENT_SECRET`, `OAUTH2_PROXY_COOKIE_SECRET`, `OAUTH2_PROXY_COOKIE_DOMAIN` and `OAUTH2_PROXY_COOKIE_EXPIRE` can be used in place of the corresponding command-line arguments.

## SSL Configuration

There are two recommended configurations.

1) Configure SSL Terminiation with OAuth2 Proxy by providing a `--tls-cert=/path/to/cert.pem` and `--tls-key=/path/to/cert.key`.

The command line to run `oauth2_proxy` in this configuration would look like this:

```bash
./oauth2_proxy \
   --email-domain="yourcompany.com"  \
   --upstream=http://127.0.0.1:8080/ \
   --tls-cert=/path/to/cert.pem \
   --tls-key=/path/to/cert.key \
   --cookie-secret=... \
   --cookie-secure=true \
   --provider=... \
   --client-id=... \
   --client-secret=...
```


2) Configure SSL Termination with [Nginx](http://nginx.org/) (example config below), Amazon ELB, Google Cloud Platform Load Balancing, or ....

Because `oauth2_proxy` listens on `127.0.0.1:4180` by default, to listen on all interfaces (needed when using an
external load balancer like Amazon ELB or Google Platform Load Balancing) use `--http-address="0.0.0.0:4180"` or
`--http-address="http://:4180"`.

Nginx will listen on port `443` and handle SSL connections while proxying to `oauth2_proxy` on port `4180`.
`oauth2_proxy` will then authenticate requests for an upstream application. The external endpoint for this example
would be `https://internal.yourcompany.com/`.

An example Nginx config follows. Note the use of `Strict-Transport-Security` header to pin requests to SSL
via [HSTS](http://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security):

```
server {
    listen 443 default ssl;
    server_name internal.yourcompany.com;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/cert.key;
    add_header Strict-Transport-Security max-age=2592000;

    location / {
        proxy_pass http://127.0.0.1:4180;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_connect_timeout 1;
        proxy_send_timeout 30;
        proxy_read_timeout 30;
    }
}
```

The command line to run `oauth2_proxy` in this configuration would look like this:

```bash
./oauth2_proxy \
   --email-domain="yourcompany.com"  \
   --upstream=http://127.0.0.1:8080/ \
   --cookie-secret=... \
   --cookie-secure=true \
   --provider=... \
   --client-id=... \
   --client-secret=...
```


## Endpoint Documentation

OAuth2 Proxy responds directly to the following endpoints. All other endpoints will be proxied upstream when authenticated. The `/oauth2` prefix can be changed with the `--proxy-prefix` config variable.

* /robots.txt - returns a 200 OK response that disallows all User-agents from all paths; see [robotstxt.org](http://www.robotstxt.org/) for more info
* /ping - returns an 200 OK response
* /oauth2/sign_in - the login page, which also doubles as a sign out page (it clears cookies)
* /oauth2/start - a URL that will redirect to start the OAuth cycle
* /oauth2/callback - the URL used at the end of the OAuth cycle. The oauth app will be configured with this as the callback url.
* /oauth2/auth - only returns a 202 Accepted response or a 401 Unauthorized response; for use with the [Nginx `auth_request` directive](http://nginx.org/en/docs/http/ngx_http_auth_request_module.html)

## Logging Format

OAuth2 Proxy logs requests to stdout in a format similar to Apache Combined Log.

```
<REMOTE_ADDRESS> - <user@domain.com> [19/Mar/2015:17:20:19 -0400] <HOST_HEADER> GET <UPSTREAM_HOST> "/path/" HTTP/1.1 "<USER_AGENT>" <RESPONSE_CODE> <RESPONSE_BYTES> <REQUEST_DURATION>
```


## Adding a new Provider

Follow the examples in the [`providers` package](providers/) to define a new
`Provider` instance. Add a new `case` to
[`providers.New()`](providers/providers.go) to allow `oauth2_proxy` to use the
new `Provider`.
