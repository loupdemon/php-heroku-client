# PHP Heroku Client
[![Latest Version](https://img.shields.io/github/release/TransitScreen/php-heroku-client.svg)](https://github.com/TransitScreen/php-heroku-client/releases)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg)](LICENSE)
[![Build Status](https://img.shields.io/github/workflow/status/TransitScreen/php-heroku-client/CI/master)](https://github.com/TransitScreen/php-heroku-client/actions/workflows/ci.yml)
[![Maintainability Rating](https://sonarcloud.io/api/project_badges/measure?project=TransitScreen_php-heroku-client&metric=sqale_rating)](https://sonarcloud.io/summary/overall?id=TransitScreen_php-heroku-client)
[![Code Coverage](https://img.shields.io/sonar/coverage/TransitScreen_php-heroku-client/master?server=https%3A%2F%2Fsonarcloud.io)](https://sonarcloud.io/summary/overall?id=TransitScreen_php-heroku-client)
[![Downloads](https://img.shields.io/packagist/dt/php-heroku-client/php-heroku-client.svg)](https://packagist.org/packages/php-heroku-client/php-heroku-client)

A PHP client for the Heroku Platform API, similar to [platform-api](https://github.com/heroku/platform-api) for Ruby and [node-heroku-client](https://github.com/heroku/node-heroku-client) for Node.js. With it you can create and alter Heroku apps, install or remove add-ons, scale resources up and down, and use any other capabilities documented by the [Platform API Reference](https://devcenter.heroku.com/articles/platform-api-reference).

## Features
- Reads `HEROKU_API_KEY` for zero-config use (deprecated)
- Returns JSON-decoded Heroku API responses
- Exposes response headers (necessary for some API functionality)
- Uses a built-in cURL-based HTTP client or one that you provide
- Accepts cURL options and custom request headers
- Throws informative exceptions for authentication, JSON, and HTTP errors
- Designed around the [PSR-7](http://www.php-fig.org/psr/psr-7/) (Request/Response) and [PSR-18](https://www.php-fig.org/psr/psr-18/) (HTTP client) standards

## Requirements
All versions are dependent on cURL, unless providing an HTTP client without cURL dependencies (such as [Socket Client](http://docs.php-http.org/en/latest/clients/socket-client.html)). In addition:
- v4.x
  - PHP 7.3+ or 8.x
  - Incompatible with Guzzle _less than_ v7.x (due to guzzlehttp/psr7:^2.0 dependency)
- v3.x
  - PHP 7.3+ or 8.x
  - Incompatible with Guzzle v7.x (due to http-interop/http-factory-guzzle dependency)
- v2.x
  - PHP 7.1 or 7.2
  - Incompatible with Guzzle v7.x (due to http-interop/http-factory-guzzle dependency)
- v1.x
  - PHP 5.6 or 7.0
  - Incompatible with Guzzle v7.x (due to http-interop/http-factory-guzzle dependency)

So modern projects will either use v3 or v4 of this library, depending on what version of Guzzle they use. Note that Guzzle is not required by this library; it's just that it may have dependency conflicts if you do also use it in your project.

## Installation
```
$ composer require php-heroku-client/php-heroku-client
```

## Quick start
Instantiate the client:
```php
require_once __DIR__ . '/vendor/autoload.php';

use HerokuClient\Client as HerokuClient;

$heroku = new HerokuClient([
    'apiKey' => 'my-api-key',
]);
```
Find out how many web dynos are currently running:
```php
$currentDynos = $heroku->get('apps/my-heroku-app-name/formation/web')->quantity;
```
Scale up:
```php
// patch(), post(), and put() accept an array or object as a body.
$heroku->patch(
    'apps/my-heroku-app-name/formation/web',
    ['quantity' => $currentDynos + 1]
);
```
Find out how many more calls we can make in the near future:
```php
// Underlying PSR-7 Request and Response objects are available for header inspection and general debugging.
$remainingCalls = $heroku->getLastHttpResponse()->getHeaderLine('RateLimit-Remaining');
```

## Configuration
The client can be configured at instantiation with these settings, all of which are optional and have sane defaults:
```php
new HerokuClient([
    'apiKey' => 'my-api-key',                 // If not set, the client finds HEROKU_API_KEY (deprecated) or fails
    'baseUrl' => 'http://custom.base.url/',   // Defaults to https://api.heroku.com/
    'httpClient' => $myFavoriteHttpClient,    // Any PSR-18 compatible HTTP client
    'curlOptions' => [                        // Options can be set when using the default HTTP client
        CURLOPT_TIMEOUT => 10,
        CURLOPT_USERAGENT => 'My Agent',
        ...
    ],
]);
```

## Making calls
Make calls using the client's `get()`, `delete()`, `head()`, `patch()`, `post()`, and `put()` methods. The first argument is always the required `string` path and the last is always an optional `array` of custom headers. `patch()`, `post()`, and `put()` take an `array` or `object` body as the second argument. These methods return the `\stdClass` object that results from JSON-decoding the Heroku API response.
- See the [Quick Start](#quick-start) for examples with and without a body.
- See the full [Platform API Reference](https://devcenter.heroku.com/articles/platform-api-reference) for a list of all endpoints and their responses.

## Using headers
The Heroku API uses headers as a separate channel for [range](https://devcenter.heroku.com/articles/platform-api-reference#ranges), [rate limit](https://devcenter.heroku.com/articles/platform-api-reference#rate-limits), and [caching](https://devcenter.heroku.com/articles/platform-api-reference#caching) information. You can read and send headers like this:
```php
$page1 = $heroku->get('apps');

// 206 Partial Content means there are more records to get.
if ($heroku->getLastHttpResponse()->getStatusCode() == 206) {
    $nextRange = $heroku->getLastHttpResponse()->getHeaderLine('Next-Range');
    $page2 = $heroku->get('apps', ['Range' => $nextRange]);
}
```

## Using other Request/Response features
Underlying HTTP requests and responses are exposed via the `getLastHttpRequest()` and `getLastHttpResponse()` methods. These return instances of [Guzzle's implementations](http://docs.guzzlephp.org/en/latest/psr7.html) of the [PSR-7 Request and Response interfaces](http://www.php-fig.org/psr/psr-7/). Note that the Response body is a stream, so it has [special handling considerations](http://docs.guzzlephp.org/en/latest/psr7.html#streams). Response bodies are rewound by this client so that you can access them again immediately with a call to `getBody()->getContents()` on the Response. The properties exposed via the `getLast...()` methods are nulled initially and whenever you call one of the entry point methods (`get`/`delete`/`head`/`patch`/`post`/`put`), then set again as soon as their corresponding objects are generated. So for certain failures (such as a hard network error in the HTTP client) `getLastHttpRequest()` would return the attempted Request object while `getLastHttpResponse()` would return `null`.

## Reacting to problems
You may wish to recognize and react to specific error conditions. In this example we use the API's [data integrity mechanism](https://devcenter.heroku.com/articles/platform-api-reference#data-integrity) to require that the requested data hasn't changed since an earlier call. If it has, we will receive a `412 Precondition Failed` response. We handle that case specially, then catch more general situations:
```php
use HerokuClient\Exception\BadHttpStatusException;

try {
    $heroku->get('some/path', ['If-Match' => $eTagFromEarlierCall]);
} catch (BadHttpStatusException $exception) {
    if ($heroku->getLastHttpResponse()->getStatusCode() == 412) {
        // React to the fact that our requested data has changed.
    } else {
        // React to all other bad HTTP status codes.
    }
} catch (\Exception $exception) {
    // React to all other problems.
}
```

## Exceptions thrown
- `BadHttpStatusException`
- `JsonDecodingException`
- `JsonEncodingException`
- `MissingApiKeyException`

In addition to exceptions thrown directly from this API client, [standardized exceptions](https://www.php-fig.org/psr/psr-18/#clientexceptioninterface) may bubble up from the HTTP client.

## Contributing
Pull Requests are welcome. Please see our [Contribution Guidelines](CONTRIBUTING.md).
