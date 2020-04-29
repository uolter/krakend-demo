# Krakend Demo App

## Introduction

This repository shows how to configure the **Application Gateway** [Krakend.io](https://www.krakend.io/) to serve static content, rest api, and protected api.

## Requirements.

To run the demo application you need:

1. docker version 18.09.9 or grater
2. docker compose version 1.23.2 or grater
3. an https://auth0.com/ account

## Run the application

Start the application:

```
$ sudo docker-compose up
```

* The **Application Gateway** responds on port 8080

* A simple **rest api** application build in golang - which is the backend of the application gateway - responds on port 8081

## Gateway configurations:

All configurations are placed in **./krakend/krakend.json** file


## Testing

### Static public content

*krakend.json*
```
...
{
  "endpoint": "/news",
  "method": "GET",
  "extra_config": {},
  "output_encoding": "no-op",
  "concurrent_calls": 1,
  "backend": [
    {
      "url_pattern": "/",
      "encoding": "no-op",
      "sd": "static",
      "extra_config": {},
      "method": "GET",
      "host": [
        "http://kraken-demo_rest_api_1:8081/"
      ],
      "disable_host_sanitize": false
    }
  ]
},
...
```

*curl*
```
$ curl --request GET \
  --url http://localhost:8080/news

Welcome home!
```

### Json content

*krakend.json*
```
{
  "endpoint": "/events",
  "method": "GET",
  "output_encoding": "json",
  "concurrent_calls": 1,
  "backend": [
    {
      "url_pattern": "/events",
      "encoding": "json",
      "sd": "static",
      "method": "GET",
      "host": [
      "http://kraken-demo_rest_api_1:8081"
      ],
      "disable_host_sanitize": false,
      "extra_config": {}
    }
  ]
}
```

*curl*
```
$ curl --request GET \
  --url http://localhost:8080/events

{
  "events": [
    {
      "description": "Simple krakend application",
      "id": "1",
      "title": "Introduction to Krakend"
    }
  ]
}
```

### Protect your api with auth0 jwt token.

It is possible to procted private content with jwt token.

This example uses the free identity service [Auth0](https://auth0.com/)

1. Sign in in Auth0.
2. Register your APIs
3. Name it *krakend-demo* or any other name you like.
4. The *identifier* must be http://kraken-demo_rest_api_1:8081 or the hostname registerd within docker-compose
5. Go in to the *Machine to Machine Applications* tab and inside the Authorized applicatio get your **Client id**, **Client secret** and **Domain**
6. Get the jwt bearer token:

```
$ curl --request POST \
  --url https://<auth0 domain>/oauth/token \
  --header 'content-type: application/json' \
  --data '{"client_id":"<your client id>","client_secret":"<your >","audience":"http://kraken-demo_rest_api_1:8081","grant_type":"client_credentials"}'

{
  "access_token": "<jwt token>",
  "token_type": "Bearer"
}
```

7. Configure the endpoint

*krakend.json*
```
....
{
  "endpoint": "/event",
  "method": "POST",
  "output_encoding": "json",
  "concurrent_calls": 1,
  "backend": [
    {
      "url_pattern": "/event",
      "encoding": "json",
      "host": [
        "http://kraken-demo_rest_api_1:8081"
      ],
      "disable_host_sanitize": false
    }
  ],
  "extra_config": {
    "github.com/devopsfaith/krakend-jose/validator": {
      "alg": "RS256",
      "audience": ["http://kraken-demo_rest_api_1:8081"],
      "jwk-url": "https://dev-zf2hwbcb.eu.auth0.com/.well-known/jwks.json",
      "disable_jwk_security": true
    }
  }
}
...
```

*curl*
```
$ curl --request POST \
  --url http://localhost:8080/event \
  --header 'content-type: application/json' \
  --data '{
	"id": "2",
	"title": "Golang Meetup",
	"description": "Wonderful meetup"
}'

HTTP/1.1 401 Unauthorized
```

8. Add the Bearer authorization

*curl*
```
curl --request POST \
  --url http://localhost:8080/event \
  --header 'authorization: Bearer <your jwt token>' \
  --header 'content-type: application/json' \
  --data '{
	"id": "2",
	"title": "Golang Meetup",
	"description": "Wonderful meetup"
}'

200 OK
{
  "description": "Wonderful meetup",
  "id": "2",
  "title": "Golang Meetup"
}
```

## Query string parameter(s) and Respons with collection

When the content in the response does not encapsulate an object (json inside brackets, e.g: { "status":"OK" } but a collection (e.g: [ "a", "b" ]) you can use the attribute **is_collection** within the backend block.

Moreover you can specify with query string parameter(s) to forward to the backend with the **querystring_params** attribute.
eg:

*krakend.json*
```
"endpoint": "/events",
  "querystring_params": [
    "flatten"
  ],
  "method": "GET",
  "output_encoding": "json",
  "concurrent_calls": 1,
  "backend": [
    {
      "url_pattern": "/events",
      "encoding": "json",
      "host": [
        "http://kraken-demo_rest_api_1:8081"
      ],
      "disable_host_sanitize": false,
      "extra_config": {},
      "is_collection": true,
      "target": ""
    }
  ]
},
```

*curl*
```
$ curl --request GET \
  --url 'http://localhost:8080/events?flatten=true'

{
  "collection": [
    {
      "description": "Simple krakend application",
      "id": "1",
      "title": "Introduction to Krakend"
    }
  ]
}
```
