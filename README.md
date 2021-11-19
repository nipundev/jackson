# SAML Jackson (not fiction anymore)

SAML service [SAML in a box from BoxyHQ]

You need someone like Jules Winnfield to save you from the vagaries of SAML login.

## Source code visualizer
[CodeSee codebase visualizer](https://app.codesee.io/maps/public/53e91640-23b5-11ec-a724-79d7dd589517)

# npm library
Jackson is available as an npm library (https://www.npmjs.com/package/@boxyhq/saml-jackson) that can be integrated into express.js routes. In theory the library should be usable with other node.js we application frameworks but is currently untested. Please file an issue or submit a PR if you encounter any issues.

Refer to https://github.com/boxyhq/jackson#configuration for the configuration options to pass to the library.

Here's how to use the npm library:
```
// express
const express = require('express');
const router = express.Router();
const cors = require('cors'); // needed if you are calling the token userinfo endpoints from the frontend

// Set the required options
const opts = {
  externalUrl: 'https://my-cool-app.com',
  samlAudience: 'https://my-cool-app.com',
  samlPath: '/sso/oauth/saml',
  db: {
    engine: 'mongo',
    url: 'mongodb://localhost:27017/my-cool-app',
  }  
};

// Please note that the initialization of @boxyhq/saml-jackson is async, you cannot run it at the top level
// Run this in a function where you initialise the express server.
asyn function init() {
  const ret = await require('@boxyhq/saml-jackson')(opts);
  const apiController = ret.apiController;
  const oauthController = ret.oauthController;
}

// express.js middlewares needed to parse json and x-www-form-urlencoded
router.use(express.json());
router.use(express.urlencoded({ extended: true }));

// SAML config API. You should pass this route through your authentication checks, do not expose this on the public interface without proper authentication in place.
router.post('/api/v1/saml/config', async (req, res) => {
  try {
    // apply your authentication flow (or ensure this route has passed through your auth middleware)
    ...

    // only when properly authenticated, call the config function
    res.json(await apiController.config(req.body));
  } catch (err) {
    res.status(500).json({
      error: err.message,
    });
  }
});

// OAuth 2.0 flow
router.get('/oauth/authorize', async (req, res) => {
  try {
    await oauthController.authorize(req, res);
  } catch (err) {
    res.status(500).send(err.message);
  }
});

router.post('/oauth/saml', async (req, res) => {
  try {
    await oauthController.samlResponse(req, res);
  } catch (err) {
    res.status(500).send(err.message);
  }
});

router.post('/oauth/token', cors(), async (req, res) => {
  try {
    await oauthController.token(req, res);
  } catch (err) {
    res.status(500).send(err.message);
  }
});

router.get('/oauth/userinfo', cors(), async (req, res) => {
  try {
    await oauthController.userInfo(req, res);
  } catch (err) {
    res.status(500).send(err.message);
  }
});

// set the router
app.user('/sso', router);

```
You can also refer to our usage of the library internally in the Jackson service here - https://github.com/boxyhq/jackson/blob/main/src/jackson.js

# Deployment
The docker container can be found at https://hub.docker.com/r/boxyhq/jackson/tags. It is preferable to use a specific version instead of the `latest` tag. Jackson uses two ports (configurable if needed, see below) 5000 and 6000. 6000 is the internal port and ideally should not be exposed to a public network.

Example of a docker run:
```
docker run -p 5000:5000 -p 6000:6000 boxyhq/jackson:78e9099d
```

# Database Support
Jackson currently supports SQL databases (Postgres, CockroachDB, MySQL and MariaDB), MongoDB and Redis.

# Configuration
Configuration is done via env vars (and in the case of the npm library via an options object). The following options are supported and will have to be configured during deployment:
- HOST_URL: The URL to bind to, defaults to `localhost`
- HOST_PORT: The port to bind to, defaults to `5000`
- EXTERNAL_URL (npm: externalUrl): The public URL to reach this service, used internally for documenting the SAML configuration instructions. Defaults to `http://{HOST_URL}:{HOST_PORT}` for Jackson service, required for npm library
- INTERNAL_HOST_URL: The URL to bind to expose the internal APIs, defaults to `localhost`. Do not configure this to a public network
- INTERNAL_HOST_PORT: The port to bind to for the internal APIs, defaults to `6000`
- SAML_AUDIENCE (npm: samlAudience): This is just an identitifer to validate the SAML audience, this value will also get configured in the SAML apps created by your customers. Once set do not change this value unless you get your customers to reconfigure their SAML again. Defaults to `https://saml.boxyhq.com` and is case sensitive. This does not have be a real URL
- IDP_ENABLED (npm: idpEnabled): Set to `true` to enable IdP initiated login for SAML. SP initiated login is the only recommended flow but you might have to support IdP login at times. Defaults to `false`
- DB_ENGINE (npm: db.engine): Supported values are `redis`, `sql`, `mongo`, `mem`. Defaults to `sql`
- DB_URL (npm: db.url): The database URL to connect to, for example `postgres://postgres:postgres@localhost:5450/jackson`
- DB_TYPE (npm: db.type): Only needed when DB_ENGINE is `sql`. Supported values are `postgres`, `cockroachdb`, `mysql`, `mariadb`. Defaults to `postgres`
- PRE_LOADED_CONFIG: If you only need a single tenant or a handful of pre-configured tenants then this config will help you read and load SAML configs. It works well with the mem db engine so you don't have to configure any external databases for this to work (though it works with those as well). This is a path (absolute or relative) to a direct that contains files organized in the format described in the next section.

# Pre-loaded SAML Configuration
If PRE_LOADED_CONFIG is set then it should point to a directory with the following structure (example below):-
```
boxyhq.js
boxyhq.xml
anothertenant.js
anothertenant.xml
```
The JS file has the following structure:-
```
module.exports = {
  defaultRedirectUrl: 'http://localhost:3000/login/saml',
  redirectUrl: '["http://localhost:3000/*", "http://localhost:5000/*"]',
  tenant: 'boxyhq.com',
  product: 'demo',
};
```
The XML file (should share the name with the .js file) is the raw XML metadata file you receive from your Identity Provider. Please ensure it is saved in the `utf-8` encoding.

The config and XML above correspond to the `SAML API config` (see below).

# SAML Login flows
There are two kinds of SAML login flows - SP-initiated and IdP-initiated. We highly recommend sticking to the SP-initiated flow since it is more secure but Jackson also support the IdP-initiated flow if you enable it. For in-depth understand of SAML and the two flows please refer to Okta's comprehensive guide - https://developer.okta.com/docs/concepts/saml/.

# Setting up SAML with your customer's Identity Provider
Please follow the instructions here to guide your customer's in setting up SAML correctly for your product(s). You should create a copy of the doc and modify it with your custom settings, we have used the values that work for our demo apps - https://docs.google.com/document/d/1fk---Z9Ln59u-2toGKUkyO3BF6Dh3dscT2u4J2xHANE.

# SAML config API
Once your customer has setup the SAML app on their Identity Provider, they Identity Provider will generate an IdP or SP metadata file. Some Identity Providers only generate an IdP metadata file but it usually works for the SP login flow as well. It is an XML file that contains various attributes Jackson needs in order to validate incoming SAML login requests. This step is the equivalent of setting an OAuth 2.0 app and generating a client ID and client Secret that will be used in the login flow.

You will need to provide a place in the UI for your customers (The account settings page is usually a good place for this) to configure this and then call the API below.

The following API call sets up the configuration in Jackson:
```
curl --location --request POST 'http://localhost:6000/api/v1/saml/config' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'rawMetadata=<IdP/SP metadata XML>' \
--data-urlencode 'defaultRedirectUrl=http://localhost:3000/login/saml' \
--data-urlencode 'redirectUrl=["http://localhost:3000/*"]' \
--data-urlencode 'tenant=boxyhq.com' \
--data-urlencode 'product=demo'
```

- rawMetadata: The XML metadata file your customer gets from their Identity Provider
- defaultRedirectUrl: The redirect URL to use in the IdP login flow. Jackson will call this URL after completing an IdP login flow
- redirectUrl: JSON encoded array containing a list of allowed redicrect URLs. Jackson will disallow any redirects not on this list (or not the default URL above)
- tenant: Jackson supports a multi-tenant architecture, this is a unique identifier you set from your side that relates back to your customer's tenant. This is normally an email, domain, an account id or user id
- product: Jackson support multiple products, this is a uniqie identifier you set from your side that relates back to the product your customer is using

The response returns a JSON with `client_id` and `client_secret` that can be stored against your tenant and product for a more secure OAuth 2.0 flow. If you do not want to store the `client_id` and `client_secret` you can alternatively use `client_id=tentant=<tenantID>&product=<productID>` and any arbitrary value for `client_secret` when setting up the OAuth 2.0 flow.

# OAuth 2.0 Flow
Jackson has been designed to abstract the SAML login flow as a pure OAuth 2.0 flow. This means it's compatible with any standard OAuth 2.0 library out there, both client side and server side. It is important to remember that SAML is configured per customer unlike OAuth 2.0 where you can have a single OAuth app supporting logins for all customers.

Jackson also supports the PKCE authorization flow (https://oauth.net/2/pkce/), so you can protect your SPAs.

If for any reason you need to implement the flow on your own, the steps are outlined below:
## Authorize
The OAuth flow begins with redirecting your user to the `authorize` URL:
```
https://localhost:5000/oauth/authorize
  ?response_type=code&provider=saml
  &client_id=<clientID or tenant and product query params as described in the SAML config API section above>
  &redirect_uri=<redirect URL>
  &state=<randomly generated state id>
```

- response_type=code: This is the only supported type for now but may be extended in the future
- client_id: Use the client_id returned by the SAML config API or use `tentant=<tenantID>&product=<productID>` to use the tenant and product IDs instead
- redirect_uri: This is where the user will be taken back once the authorization flow is complete
- state: Use a randomly generated string as the state, this will be echoed back as a query parameter when taking the user back to the `redirect_uri` above. You should validate the state to prevent XSRF attacks

## Code exchange
After successful authorization the user is redirected back the `redirect_uri`. The query parameters will include the `code` and `state` parameters. You should validate that the state matches the one you sent in the authorize request.

The code can then be exchanged for a token by making the following request:
```
curl --request POST \
  --url 'http://localhost:5000/oauth/token' \
  --header 'content-type: application/x-www-form-urlencoded' \
  --data grant_type=authorization_code \
  --data 'client_id=<clientID or tenant and product query params as described in the SAML config API section above>' \
  --data client_secret=<clientSecret or any arbitrary value if using the tenant and product in the clientID> \
  --data 'redirect_uri=<redirect URL>' \
  --data code=<code from the query parameter above>
```
- grant_type=authorization_code: This is the only supported flow for now, we might extend this in the future
- client_id: Use the client_id returned by the SAML config API or use `tentant=<tenantID>&product=<productID>` to use the tenant and product IDs instead
- client_secret: Use the client_secret returned by the SAML config API or any arbitrary value if using the tenant and product in the clientID
- redirect_uri: This is where the user will be taken back once the authorization flow is complete. Use the same redirect_uri as the previous request

If everything goes well you should receive a JSON response which include the access token. This token is needed for the next step where we fetch the user profile.
```
{
  "access_token": <access token>,
  "token_type": "bearer",
  "expires_in": 300
}
```

## Profile request
The short-lived access token can now be used to request the user's profile. You'll need to make the following request:
```
curl --request GET \
  --url https://localhost:5000/oauth/me \
  --header 'authorization: Bearer <access token>' \
  --header 'content-type: application/json'
```

If everything goes well you should receive a JSON response with the user's profile:
```
{
  "email": "sjackson@coolstartup.com",
  "firstName": "SAML"
  "id": <id from the Identity Provider>,
  "lastName": "Jackson",
}
```

- email: The email address of the user as provided by the Identity Provider
- id: The id of the user as provided by the Identity Provider
- firstName: The first name of the user as provided by the Identity Provider
- lastName: The last name of the user as provided by the Identity Provider
