# Microfocus Access Manager JWT Verifier for Node.js

[![npm version](https://img.shields.io/npm/v/@okta/jwt-verifier.svg?style=flat-square)](https://www.npmjs.com/package/@okta/jwt-verifier)
[![build status](https://img.shields.io/travis/okta/okta-oidc-js/master.svg?style=flat-square)](https://travis-ci.org/okta/okta-oidc-js)

This library verifies access tokens (issued by [Microfocus Access Manager](https://www.microfocus.com/en-us/products/netiq-access-manager/overview) by fetching the public keys from the JWKS endpoint of the authorization server. If the access token is valid it will be converted to a JSON object and returned to your code.

This library does not yet verify id tokens.

For any access token to be valid, the following are asserted:

- Signature is valid (the token was signed by a private key which has a corresponding public key in the JWKS response from the authorization server).
- Access token is not expired (requires local system time to be in sync with Okta, checks the `exp` claim of the access token).
- The `aud` claim matches any expected `aud` claim passed to `verifyAccessToken()`.
- The `iss` claim matches the issuer the verifier is constructed with.
- Any custom claim assertions that have been configured.

> This library is for Node.js applications and will not compile into a front-end application.

## Upgrading

For information on how to upgrade between versions of the library, see UPGRADING.md

## How to use

```bash
npm install --save @pointblue/jwt-verifier
```

Create a verifier instance, bound to the issuer (authorization server URL):

```javascript
const PointblueJwtVerifier = require("@pointblue//jwt-verifier");

const pointblueJwtVerifier = new PointblueJwtVerifier({
  issuer: "https://{yourDomain}/nidp/oauth/nam",
});
```

With a verifier, you can now verify access tokens:

```javascript
pointblueJwtVerifier
  .verifyAccessToken(accessTokenString, expectedAud)
  .then((jwt) => {
    // the token is valid (per definition of 'valid' above)
    console.log(jwt.claims);
  })
  .catch((err) => {
    // a validation failed, inspect the error
  });
```

The expected audience passed to `verifyAccessToken()` is required, and can be either a string (direct match) or an array strings (the actual `aud` claim in the token must match one of the strings).

```javascript
// Passing a string for expectedAud
pointblueJwtVerifier
  .verifyAccessToken(accessTokenString, "api://default")
  .then((jwt) => console.log("token is valid"))
  .catch((err) => console.warn("token failed validation"));

pointblueJwtVerifier
  .verifyAccessToken(accessTokenString, ["api://special", "api://default"])
  .then((jwt) => console.log("token is valid"))
  .catch((err) => console.warn("token failed validation"));
```

## Custom Claims Assertions

For basic use cases, you can ask the verifier to assert a custom set of claims. For example, if you need to assert that this JWT was issued for a given client id:

```javascript
const verifier = new PointblueJwtVerifier({
  issuer: 'https://{yourDomain}/nidp/oauth/nam',
  clientId: '{clientId}'
  assertClaims: {
    cid: '{clientId}'
  }
});
```

Validation fails and an error is returned if an access token does not have the configured claim.

For more complex use cases, you can ask the verifier to assert that a claim includes one or more values. This is useful for array type claims as well as claims that have space-separated values in a string.

You use the form: `<claim name>.includes` in the `assertClaims` object with an array of values to validate.

For example, if you want to assert that an array claim named `groups` includes (at least) `Everyone` and `Another`, you'd write code like this:

```javascript
const verifier = new PointblueJwtVerifier({
  issuer: ISSUER,
  clientId: CLIENT_ID,
  assertClaims: {
    "groups.includes": ["Everyone", "Another"],
  },
});
```

If you want to assert that a space-separated string claim name `scp` includes (at least) `promos:write` and `promos:delete`, you'd write code like this:

```javascript
const verifier = new PointblueJwtVerifier({
  issuer: ISSUER,
  clientId: CLIENT_ID,
  assertClaims: {
    "scp.includes": ["promos:write", "promos:delete"],
  },
});
```

The values you want to assert are included are always represented as an array (the right side of the expression). The claim that you're checking against (the left side of the expression) can have either an array (like `groups`) or a space-separated list in a string (like `scp`) as its value type.

NOTE: Currently, `.includes` is the only supported claim operator.

## Caching & Rate Limiting

- By default, found keys are cached by key ID for one hour. This can be configured with the `cacheMaxAge` option for cache entries.
- If a key ID is not found in the cache, the JWKs endpoint will be requested. To prevent a DoS if many not-found keys are requested, a rate limit of 10 JWKs requests per minute is enforced. This is configurable with the `jwksRequestsPerMinute` option.

Here is a configuration example that shows the default values:

```javascript
// All values are default files
const pointblueJwtVerifier = new PointblueJwtVerifier({
  issuer: "https://{yourDomain}/nidp/oauth/nam",
  clientId: "{clientId}",
  cacheMaxAge: 60 * 60 * 1000, // 1 hour
  jwksRequestsPerMinute: 10,
});
```
