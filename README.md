# Get KeyCloak Public Key [![Build Status](https://travis-ci.org/aquator/node-get-keycloak-public-key.svg?branch=master)](https://travis-ci.org/aquator/node-get-keycloak-public-key) [![Coverage Status](https://coveralls.io/repos/github/aquator/node-get-keycloak-public-key/badge.svg?branch=master)](https://coveralls.io/github/aquator/node-get-keycloak-public-key?branch=master) [![License: MIT](https://img.shields.io/badge/License-MIT-brightgreen.svg)](https://opensource.org/licenses/MIT) [![Downloads Counter](https://img.shields.io/npm/dt/get-keycloak-public-key.svg)](https://www.npmjs.com/package/get-keycloak-public-key)

Provides access to PEM Public Keys from a [KeyCloak][1] server for [JWT][2] validation.

## Introduction

[KeyCloak][1] has a bunch of libraries, but for [NodeJs][3] the only solution is a [Connect based adapter][4]. In case you want to use [koa][5], or something else, you are toast with your token.

This module provides access to the PEM encoded Public Key used for the token based on the KID value, so you can validate the token with anything you want.

The module has no dependencies, the algorithm used to reconstruct the PEM encoded value from the modulus and the exponent is taken from [tracker1's solution](https://github.com/tracker1/node-rsa-pem-from-mod-exp).

## Features

The idea is to keep this simple and stupid, so nothing fancy is included. It can download the certificates JSON from a KeyCloak server, find the one with matching KID value, and reconstruct the Public Key in PEM format. End of story.

If you need improved behavior like caching of Public Keys, you can easily implement one.

Support custom agent for request, and caching requests.
For enable cache need add cacheExpire param

## Installation

```bash
$ npm install --save get-keycloak-public-key
```

## Usage

```javascript
const KeyCloakCerts = require('get-keycloak-public-key');

const keyCloakCerts = new KeyCloakCerts('https://my-keycloak.com', 'my-realm');

// You can also pass the full URL instead, as a single argument:
// 'https://my-keycloak.com/auth/realms/my-realm/protocol/openid-connect/certs'

const publicKey = keyCloakCerts.fetch('my-kid');
```

If your need using proxy on fetch requests or keep alive agent, you need add param agent
If your want cache request to keycloak, you need add expire param, defaults cache is disabled

```javascript
const https = require('https');
const KeyCloakCerts = require('get-keycloak-public-key');
const expireMs = 86400000; // one day
const keyCloakCerts = new KeyCloakCerts(
  'https://my-keycloak.com',
  'my-realm',
  new https.Agent({ keepAlive: true }),
  expireMs
);

// You can also pass the full URL instead, as a single argument:
// 'https://my-keycloak.com/auth/realms/my-realm/protocol/openid-connect/certs'

const publicKey = keyCloakCerts.fetch('my-kid');
```

## Example

Verifying the token using [koa][5] and [jsonwebtoken][6]:

```javascript
const Koa = require('koa');
const KeyCloakCerts = require('get-keycloak-public-key');
const jwt = require('jsonwebtoken');

const keyCloakCerts = new KeyCloakCerts('https://my-keycloak.com', 'my-realm');
const app = new Koa();
app.use(async ctx => {
  // Check the Authorization header
  if (
    !(
      ctx.request.header.autorization &&
      ctx.request.header.authorization.startsWith('Bearer ')
    )
  ) {
    // Authorization header is missing
    ctx.status = 401;
    return;
  }

  // Get the token from the Authorization header, skip 'Bearer ' prefix
  const token = ctx.request.header.authorization.substr(7);

  // decode the token without verification to have the kid value
  const kid = jwt.decode(token, { complete: true }).header.kid;

  // fetch the PEM Public Key
  const publicKey = await keyCloakCerts.fetch(kid);

  if (publicKey) {
    try {
      // Verify and decode the token
      const decoded = jwt.verify(token, publicKey);
      ctx.body = decoded;
    } catch (error) {
      // Token is not valid
      process.stderr.write(error.toString());
      ctx.status = 401;
    }
  } else {
    // KeyCloak has no Public Key for the specified KID
    ctx.status = 401;
  }
});
app.listen(3000);
```

[1]: http://www.keycloak.org/
[2]: https://jwt.io/
[3]: https://nodejs.org/en/
[4]: https://github.com/keycloak/keycloak-nodejs-connect
[5]: http://koajs.com/
[6]: https://github.com/auth0/node-jsonwebtoken
