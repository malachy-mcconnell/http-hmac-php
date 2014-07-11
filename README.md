# HMAC Request Signer

[![Build Status](https://travis-ci.org/acquia/hmac-request.svg)](https://travis-ci.org/acquia/hmac-request)
[![Latest Stable Version](https://poser.pugx.org/acquia/hmac-request/v/stable.svg)](https://packagist.org/packages/acquia/hmac-request)
[![License](https://poser.pugx.org/acquia/hmac-request/license.svg)](https://packagist.org/packages/acquia/hmac-request)

### Why?

Securing RESTful web APIs is challenging. This library is intended to simplify
implementing an authentication system to your API that balances security and
simplicity using code that can be reused in client-side libraries as well.

[HMAC authentication](http://en.wikipedia.org/wiki/Hash-based_message_authentication_code)
is a shared-secret cryptography method where signatures are generated on the
client side and validated by the server in order to authenticate the request. It
is used by popular web services such as [AWS](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html)
and in protocols such as [OAuth 1.0a](http://oauth.net/core/1.0a/) to sign and
authenticate API requests.

#### Why not HTTP basic authentication?

Basic authentication is the simplest way to add authentication to a REST API,
however it is generally considered the least secure authentication method since
the hashed password must be sent on every API request.

#### Why not OAuth 1.0a?

OAuth 1.0a is a widely adopted protocol that also uses an HMAC-based algorithm
to sign and authenticate API requests. The main security advantage that OAuth
1.0a has over bare HMAC authentication systems is the "request token" workflow
that enables browsers to initiate authentication requests on behalf of a server
without ever being passed the shared secret.

The downside of this technique is the overall complexity it adds by requiring
the application making requests to implement the OAuth 1.0a protocol as well. If
passing the shared secret in a browser is not a concern for the app, then bare
HMAC authentication systems can provide equivalent security with less
complexity.

#### Why not OAuth 2.0?

This is best explained by Eran Hammer's [OAuth 2.0 and the Road to Hell](http://hueniverse.com/2012/07/26/oauth-2-0-and-the-road-to-hell/)
blog post explaining why he resigned as lead author and editor of the spec.

### What?

HMAC Request Signer is a PHP library that signs HTTP requests using a HMAC
digest. The algorithm is heavily inspired by [Amazon Web Service's](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html)
implementation and in part is derived from the HMAC authentication system
developed for [Acquia Search](https://www.acquia.com/products-services/acquia-network/cloud-services/acquia-search).

The goal of this project is to generate a digest from common HTTP request
objects, e.g. [Guzzle](http://api.guzzlephp.org/class-Guzzle.Http.Message.Request.html)
and [Symfony](http://api.symfony.com/2.0/Symfony/Component/HttpFoundation/Request.html),
so that it can be reused in both server-side and client-side applications.

The following pseudocode illustrates the construction of the Authorization
header and signature.

```
Authorization = "Provider" + " " + ID + ":" + Signature;

Signature = Base64( HMAC-SHA1 )( SecretKey, Message ) );

Message =
    HTTP-Verb + "\n" +
	MD5( Request-Body ) + "\n" +
	Content-Type + "\n" +
	Date + "\n" +
    Custom-Headers + "\n"
	Resource
;
```

#### Authorization Header

The value of the `Authorization` header contains the following parts:

* `Provider`: The provider, for example "Acquia"
* `ID`: The API key's unique identifier
* `Signature`: The base64 encoded HMAC digest as described below

#### Signature

The signature is a base64 encoded binary HMAC digest generated from the
following parts:

* `SecretKey`: The API key's shared secret
* `Message`: The string being signed as described below

#### Message

The message is a concatenated string  generated from the following parts:

* `HTTP-Verb`: The uppercase HTTP request method e.g. "GET", "POST"
* `Request-Body`: The raw body of the HTTP request
* `Content-Type`: The lowercase value of the "Content-type" header
  * Note that in the message, the body is hashed using the MD5 algorithm
* `Date`: The value of the "Date" header
  * Can be configured to read the timestamp from a custom header, e.g. `x-acquia-timestamp`
* `Custom-Headers`: A canonicalized concatenation of specified custom headers
  * Each header is in `x-custom-header: value` format
  * Header names are lowercase
  * Headers with multiple multiple values are separated by a comma and space, e.g. `value1, value2`
  * Each header is separated by a newline in the concatenated string
* `Resource`: The HTTP request path + query string, e.g. `/resource?key=value`

## Installation

HMAC Request can be installed with [Composer](http://getcomposer.org)
by adding it as a dependency to your project's composer.json file.

```json
{
    "require": {
        "acquia/hmac-request": "*"
    }
}
```

Please refer to [Composer's documentation](https://github.com/composer/composer/blob/master/doc/00-intro.md#introduction)
for more detailed installation and usage instructions.

## Usage

Sign an API request sent via Guzzle.

```php

use Acquia\Hmac\Guzzle3\HmacAuthPlugin;
use Acquia\Hmac\RequestSigner;
use Guzzle\Http\Client;

$requestSigner = new RequestSigner();
$plugin = new HmacAuthPlugin($requestSigner, 'apiKeyId', 'secretKey');

$client = new Client('http://example.com');
$client->addSubscriber($plugin);

$client->get('/resource')->send();

```

Authenticate the request in a Symfony-powered app e.g. [Silex](https://github.com/silexphp/Silex).

```php

use Acquia\Hmac\RequestAuthenticator;
use Acquia\Hmac\RequestSigner;
use Acquia\Hmac\Request\Symfony as RequestWrapper;

// $request is a \Symfony\Component\HttpFoundation\Request object.
$requestWrapper = new RequestWrapper($request);

// $keyLoader implements \Acquia\Hmac\KeyLoaderInterface

$authenticator = new RequestAuthenticator(new RequestSigner(), '+15 minutes');
$key = $authenticator->authenticate($requestWrapper, $keyLoader);

```

## Contributing and Development

Submit changes using GitHub's standard [pull request](https://help.github.com/articles/using-pull-requests) workflow.

All code should adhere to the following standards:

* [PSR-1](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-1-basic-coding-standard.md)
* [PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md)
* [PSR-4](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md)

It is recommend to use the [PHP Coding Standards Fixer](https://github.com/fabpot/PHP-CS-Fixer)
tool to ensure that code adheres to the coding standards mentioned above.

Refer to [PHP Project Starter's documentation](https://github.com/cpliakas/php-project-starter#using-apache-ant)
for the Apache Ant targets supported by this project.

## Attribution

The techniques in this library were adapted from the work done by Peter Wolanin,
Jacob Singh, and Nick Veenhof for the [Acquia Search](https://www.acquia.com/products-services/acquia-network/cloud-services/acquia-search)
project.
