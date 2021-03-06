# FamilySearch PHP Lite SDK

[![Packagist](https://img.shields.io/packagist/v/familysearch/fs-php-lite.svg)](https://packagist.org/packages/familysearch/fs-php-lite)
[![Build Status](https://travis-ci.org/FamilySearch/fs-php-lite.svg?branch=master)](https://travis-ci.org/FamilySearch/fs-php-lite)

Lite PHP SDK for the [FamilySearch API](https://familysearch.org/developers/).

__Warning__: this SDK requires hard-coding the API endpoint URLs. That is
considered bad practice when using the API. In most cases, FamilySearch does not
consider URL changes as breaking changes. Read more about 
[dealing with change](https://familysearch.org/developers/docs/guides/evolution).

There is a sample app in the `/examples` directory that is deployed to 
http://fs-php-lite-sdk.herokuapp.com/examples/.

## Usage

```php

include_once('FamilySearch.php');

// Create the SDK instance
$fs = new FamilySearch([
  'environment' => 'production',
  'appKey' => 'ahfud9Adjfia',
  'redirectUri' => 'https://example.com/fs-redirect',
  
  // Tell it to automatically save and load the access token from $_SESSION. 
  'sessions' => true, // This defaults to true
  'sessionVariable' => 'FS_ACCESS_TOKEN',
  
  // Necessary for when the developer wants to store the accessToken somewhere
  // besides $_SESSION
  'accessToken' => '',
  
  // How many times should a throttled response be retried? Defaults to 5
  'maxThrottledRetries' => 5,
  
  // Activate pending modifications
  'pendingModifications' => ['consolidate-redundant-resources', 'current-person-401'],
  
  // Modify the default user agent by appending this value
  'userAgent' => 'myApp/1.2.3',
  
  // Enable optional serialization and deserialization with objects via gedcomx-php
  'objects' => true
]);

// OAuth step 1: Redirect
$fs->oauthRedirect();

// OAuth step 2: Exchange the code for an access token.
//
// This will automatically retrieve the code from $_GET and exchange it for
// an access token. The access token is contained in the response object if the
// request was successful. The token doesn't need to be saved to a variable if
// sessions are enabled because the SDK will automatically save it.
$response = $fs->oauthResponse();

// Get the current user
$response = $fs->get('/platform/users/current');

// All response objects have the following properties
$response->statusCode;     // Integer
$response->statusText;     // String
$response->headers;        // Array
$response->effectiveUrl;   // String
$response->body;           // String
$response->requestMethod;  // String
$response->requestHeaders; // Array
$response->requestBody;    // String
$response->redirected;     // Boolean; defaults to false
$response->throttled;      // Boolean; defaults to false
$response->curl;           // A reference to the curl resource for the request

// If the response included JSON in the body then it will be parsed into an
// associative array and be available via the `data` property.
$response->data; 

// If a request is forwarded then the response will contain the original URL
$response->originalUrl;

// If a request is throttled then the response will tell how many times it was
// throttled until it finally succeeded.
$response->retries;

// You can POST too. The body may be an array or a string.
$response = $fs->post('/platform/tree/persons/PPPP-PPP', [
  'body' => $personData
]);

// The SDK defaults the Accept and Content-Type headers to application/x-fs-v1+json
// for all /platform/ URLs. But that doesn't work for some endpoints that require
// the atom data format so you'll need to set the headers yourself.
$response = $fs->get('/platform/tree/persons/PPPP-PPP/matches?collection=records', [
  'headers' => [
    'Accept' => 'application/x-gedcomx-atom+json'  
  ]
]);

// You can also pass the query parameters to the HTTP methods if you don't want
// to construct the URL yourself.
$response = $fs->get('/platform/tree/persons/PPPP-PPP/matches', [
  'query' => [
    'collection' => 'records'
  ],
  'headers' => [
    'Accept' => 'application/x-gedcomx-atom+json'  
  ]
]);

// Supported HTTP methods are `get()`, `post()`, `head()`, and `delete()`. They
// all call the core `request()` method which has the same signature.
$response = $fs->request('/platform/tree/persons/PPPP-PPP', [
  'method' => 'POST',
  'body' => $personData
]);
```

## Serialization with gedcomx-php

When the `objects` configuration option is set to true, the 
[gedcomx-php](https://github.com/FamilySearch/gedcomx-php) library can be used
for serialization from objects for requests and deserialization into objects
for responses.

```php
$fs = new FamilySearch({
    'objects' => true
});

$response = $fs->post('/platform/tree/persons', [
    'body' => new \Gedcomx\Extensions\FamilySearch\FamilySearchPlatform([
        'persons' => [ $personData ]
    ])
]);

$persons = $response->gedcomx->getPersons();
```

When a response body is present, it will be deserialized as either an 
[Atom Feed](http://familysearch.github.io/gedcomx-php/class-Gedcomx.Atom.Feed.html)
or a [FamilySearchPlatform](http://familysearch.github.io/gedcomx-php/class-Gedcomx.Extensions.FamilySearch.FamilySearchPlatform.html)
object.

gedcomx-php must be installed and included separately. gedcomx-php version 3.1.2
or later is required.