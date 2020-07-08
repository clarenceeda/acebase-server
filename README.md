# AceBase realtime database server

This repository is to setup an http endpoint for a local AceBase database instance. See [AceBase](https://www.npmjs.com/package/acebase) for more information about AceBase databases and usage.

## Getting started

Install the *acebase-server* npm package: ```npm install acebase-server``` ([github](https://github.com/appy-one/acebase-server), [npm](https://www.npmjs.com/package/acebase-server))

## Creating an AceBase server

To launch an AceBase webserver using default settings
- requires users to sign in (denies anonymous access)
- runs without SSL (http)

```javascript
const { AceBaseServer } = require('acebase-server');
const dbname = 'mydb';
const server = new AceBaseServer(dbname, { host: 'localhost', port: 5757 });
server.on("ready", () => {
    console.log("SERVER ready");
});
```

**NOTE**: The above code will create an admin user with a generated password, which will be displayed once in the console output. Only randomly salted SHA512 hashes of user passwords are stored in the auth database, so it'll become _fairly hard_ to sign into the admin account if you don't copy/paste the password somewhere. If you don't want to use a generated password, see _Enable authentication_ below for info about how to supply a default password.

### Enable SSL (https)

To run the server on a secure https connection, you can simply provide details about the location of your certificate files:

```javascript
const settings = {
    host: 'mydb.example.com',
    port: 443,
    https: {
        certPath: './mycertificate.pem',
        keyPath: './myprivatekey.pem'
    }
};
const server = new AceBaseServer(dbname, settings);
```

You can either pass ```certPath``` and ```keyPath```, or ```pfxPath``` and ```passphrase``` - depending on the type of certificate files you have.

### Enable authentication

Authentication settings determine who is allowed access to your server. By enabling authentication, the server will allow users to sign in (and signup) and authorization rules to be defined for data being read/written to the database.

```javascript
const settings = {
    host: 'mydb.example.com',
    port: 80,
    authentication: {
        enabled: true,
        allowUserSignup: false,
        defaultAccessRule: 'auth',
        defaultAdminPassword: '75sdDSFg37w5'
    }
}
```

The following ```authentication``` settings are available:
- ```enabled```: whether to enable user authentication (default: ```true```)
- ```allowUserSignup```: whether users can sign up for an account themselves, or if the admin has to (default: ```false```)
- ```defaultAccessRule```: default authorization rule for read/write operations. Either ```deny```, ```allow``` or ```auth```. The first time the server is launched with authentication enabled, it will create a ```rules.json``` file that contains this access rule for any path. ```deny``` will deny all users (except admin) read/write access, ```allow``` will grant access to anyone, ```auth``` will grant access only to authenticated users (default: ```auth```)
- ```defaultAdminPassword```: supply a default password for the admin account that will be used when it is created. If you do not supply this, a generated password will be used instead and displayed only once in the console output.

### Setup authorization rules

If you enabled authentication, you can also define access rules for your data. Using rules, you can allow or deny specific (or anonymous) users read and/or write access to your data. These rules are identical to those used by [Firebase](https://firebase.google.com/docs/database/security/) (Note: ".read" and ".write" only, ".validate" might be implemented in the future, ".schema" implemented as alternative) and are saved in a file called _rules.json_ in your database directory. The default rules written to the file are determined by the ```defaultAccessRule``` authentication setting at the first server launch with ```authentication``` enabled.

The default _rules.json_ file content is based on the value of the ```defaultAccessRule``` setting, possible values are:
 * ```"auth"```: Only allow authenticated users read/write access to the database
 * ```"allow"```: Allow anyone (including anonymous users) read/write access to the database
 * ```"deny"```: Deny anyone (except admin user) read/write access to the database

When ```defaultAccessRule: "auth"``` is used, it will generate the following _rules.json_ file:
```json
{
    "rules": {
        ".read": "auth !== null",
        ".write": "auth !== null"
    }
}
```

When ```"allow"``` or ```"deny"``` is used, the ```".read"``` and ```".write"``` properties will be set to ```true``` or ```false``` respectively.

If you want to further restrict what data users can access and/or write to (RECOMMENDED!), you could edit the file as such, granting users read/write access to their own user node:
```json
{
    "rules": {
        "users": {
            "$uid": {
                ".read": "auth.uid === $uid",
                ".write": "auth.uid === $uid"
            }
        }
    }
}
```

NOTE: Just like Firebase, access is denied by default when nu rule is found for the target path. If an access rule is found, it will be used for any child path. Eg: access for child/descending paths can not be overridden.

### Schema validation

AceBase server now supports TypeScript based schema definitions and validation. Once you've defined a schema for a path, all data being written must adhere to the set schema. Data to be stored/updated will be validated against the schema and denied or allowed accordingly.

To ensure all users have a ```name``` (string), ```email``` (string) and ```language``` (either Dutch, English, German, French or Spanish), optionally a ```birthdate``` (Date) and ```address``` (custom object definition), add the following to your _rules.json_ file:
```json
{
    "rules": {
        "users": {
            "$uid": {
                ".read": "auth.uid === $uid",
                ".write": "auth.uid === $uid",
                ".schema": {
                    "name": "string",
                    "email": "string",
                    "language": "'nl'|'en'|'de'|'fr'|'es'",
                    "birthdate?": "Date",
                    "address?": {
                        "street": "string",
                        "city": "string",
                        "country": "string",
                        "geo?": {
                            "lat": "number",
                            "lon": "number"
                        }
                    }
                }
            }
        }
    }
}
```

You can also decide to split the schema up into multiple levels:
```json
{
    "rules": {
        "users": {
            "$uid": {
                ".read": "auth.uid === $uid",
                ".write": "auth.uid === $uid",

                ".schema": {
                    "name": "string",
                    "email": "string",
                    "language": "'nl'|'en'|'de'|'fr'|'es'",
                    "birthdate?": "Date",
                    "address?": "Object"
                },

                "address": {
                    ".schema": {
                        "street": "string",
                        "city": "string",
                        "country": "string",
                        "geo?": "Object"
                    },

                    "geo": {
                        ".schema": {
                            "lat": "number",
                            "lon": "number"
                        }
                    }
                }
            }
        }
    }
}
```

And, if you prefer, schema definitions can be defined as strings instead:
```json
(...)
    "address": {
        ".schema": "{ street: string, city: string, country: string, geo?: { lat: number, lon: number } }"
    }
(...)
```

## Sending user e-mails [NEW]

If your app allows users to sign up, you'll need a way to verify their e-mail addresses, send welcome and password reset e-mails etc. AceBase Server does not send the e-mails itself (yet), but provides a way to handle this yourself:

```javascript
const server = new AceBaseServer(('mydb', { host: 'localhost', port: 5757, authentication: { enabled: true, allowUserSignup: true }, email: { send: sendRequestedEmail } });

function sendRequestedEmail(request) {
    switch (request.type) {
        case 'user_signup': {
            console.log(`Should send an e-mail to ${request.user.email} to verify their e-mail address with code ${request.verifyCode}`);
            break;
        }
        case 'user_reset_password': {
            console.log(`Should send an e-mail to ${request.user.email} to reset their password with code ${request.resetCode}`);
            break;
        }
    }
}
```

Any links you would add to the e-mails you send out should point to your own website/app, where you would handle account verification / password resets through an ```AceBaseClient``` connected to your server:

```javascript
const resetCode = 'weydgaed7gjsdhfjadbsfadsfasq3w7dtuqwebd'; // eg from your_reset_url?code=weydga...
const newPassword = 'MyNewPassword'; // from user input on your page
client.auth.resetPassword(resetCode, newPassword)
.then(() => {
    // User can now sign in with their new password
})
.catch(err => {
    // Something went wrong
})
```

OR, if you have direct access to your AceBaseServer instance from your website:

```javascript
server.resetPassword(req.ip, resetCode, newPassword)
.then(() => { ... });
```

To verify a user's email address:
```javascript
const verificationCode = 'weydgaed7gjsdhfjadbsfadsfasq3w7dtuqwebd'; // eg from your_verify_url?code=weydga...
client.auth.verifyEmailAddress(verificationCode)
.then(() => { ... })
```

OR, directly on your server:

```javascript
server.verifyEmailAddress(req.ip, verificationCode)
.then(() => { ... });
```

## Using third party login providers [NEW]

You can enable users to sign into your app through a third party login provider, such as Facebook, Google, Twitter etc. To enable this, follow these steps for each provider:

* First, get ```client_id``` and ```client_secret``` API keys from the auth providers' developer environment. This will allow your app to use with the provider's auth API.

* Most OAuth providers restrict authentication callback uri's to a predefined set of uris, so make sure you add AceBase's callback URL in the provider's API settings: ```"https://your.acebase.server/oauth2/dbname/signin"``` (replace hostname and dbname to your server Url)

* Then, add your API keys to your ```AceBaseServer``` config with ```configAuthProvider```:

```javascript
server.configAuthProvider('facebook', { client_id: '[your fb app_id]', client_secret: '[your fb app_secret]', scopes: [/* Any additional scopes, such as 'user_birthday' */] });
```

* Now, you can kick off Facebook authentication in your app:
```javascript
// In your login.js:
const callbackUrl = 'http://your.app.url/authenticated';
client.auth.startAuthProviderSignIn('facebook', callbackUrl)
.then(redirectUrl => {
    window.location = redirectUrl; // Send user to auth provider's login screen
});
```

* Once the user authenticated your request, they are redirected back to your website (your callbackUrl):
```javascript
// In your authenticated.js: (executes from /authenticated?result=awefi873r4gqw...)
const callbackResult = window.location.search.match(/[?&]result=(.*?)(?:&|$)/)[1]; // Or some other way you'd get the ?result from the url
client.auth.finishAuthProviderSignIn(callbackResult)
.then(result => {
    console.log(`User ${result.user.email} signed in with ${result.provider.name}`);
})
```

The ```result``` object will also contain the provider's ```access_token``` and ```refresh_token``` in case you want to make custom calls to the provider's API. If you want to keep the provider's access_token active, you will have to call ```client.auth.refreshAuthProviderToken``` before it expires:

```javascript
const keepAlive = (provider) => {
    // Schedule token refresh 1 minute before it expires
    const refreshMs = (provider.expires_in - 60) * 1000;
    setTimeout(() => {
        client.auth.refreshAuthProviderToken(provider.refresh_token)
        .then(result => {
            keepAlive(result.provider); // Schedule again
        })
    }, refreshMs);
};

client.auth.finishAuthProviderSignIn(callbackResult)
.then(result => {
    // schedule a token refresh:
    keepAlive(result.provider);
});
```

Currently implemented auth providers are:
* Google
* Facebook*
* Spotify

*NOTE: Facebook access tokens are short-lived by default, but will be exchanged for a long-lived (60 day) access token upon refresh. If you need to keep the Facebook access token active, execute ```refreshAuthProviderToken``` immediately after ```finishAuthProviderSignIn```, and keep refreshing every time the user starts your app. Once a Facebook access token has expired, it cannot be refreshed and the user will have to sign in again.

## Extending the server API [NEW]

You can add your own custom API functions to the server with the ```server.extend(method, path, handler)``` method:

```javascript
const server = new AceBaseServer(...);
await server.ready();

const _quotes = [...];
server.extend('get', 'quotes/random', (req, res) => {
    let index = Math.round(Math.random() * _quotes.length);
    res.send(quotes[index]);
});
server.extend('get', 'quotes/specific', (req, res) => {
    let index = req.query.index;
    res.send(quotes[index]);
});
server.extend('post', 'quotes/add', (req, res) => {
    let quote = {
        text: req.body.text,
        author: req.body.author
    };
    quotes.push(quote);
    res.send('thanks!');
});
```

You can then call your API methods on the client side as follows:
```javascript
// Get random quote:
client.callExtension('get', 'quotes/random')
.then(quote => {
    console.log(`Got random quote: `, quote);
});

// Get specific quote:
client.callExtension('get', 'quotes/specific', { index: 15 })
.then(quote => {
    console.log(`Got quotes[15]: `, quote);
});

// Add a quote
client.callExtension('post', 'quotes/add', { 
    text: 'The greatest glory in living lies not in never falling, but in rising every time we fall', 
    author: 'Nelson Mandela' })
.then(result => {
    // "thanks!"
});
```

## Connecting to a server

See *acebase-client* on [npm](https://www.npmjs.com/package/acebase-client) or [github](https://github.com/appy-one/acebase-client)

## Upgrade notices

v0.9.7: user accounts and server logs are now stored in the target database, instead of in a seperate auth database. If you upgraded from version 0.9.6 or lower and want to keep using the existing auth database, start the server with ```seperateDb: true``` in the options.