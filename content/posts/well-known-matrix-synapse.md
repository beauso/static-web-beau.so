---
title: "Allowing Matrix (synapse) federation without exposing 8448 via /.well-known/ delegation and nginx"
date: 2022-06-14T00:48:54+01:00
draft: false
---
## problem
I have a matrix server called `example.com`, but it's actually hosted on `matrix.example.com`. I want to keep the cosmetic name of `example.com` and have all federation traffic route through 443 instead of 8488.

## what is delegation?

Delegation is just a fancy term in the Matrix world for having a Matrix server name of `example.com` while having the actual endpoint other servers connect to being different, for example:   

  - `different.example.com:443`


There are two main reasons why this is useful:
- Most people want their _second-level domain_ as the cosmetic server name for their homeserver, e.g. `@beau:example.com`, even though the server itself is actually hosted on `matrix.example.com`.
- Most people running a homeserver will expose other services via a reverse-proxy and therefore will already have 443 open. Opening another port in this case is unnecessary. By default, Matrix will look to federate on port 8448 with other servers, so we need to fix this by using the `/.well-known/` method.
- You want to hide your public IP and proxy traffic to your homeserver via Cloudflare so need to listen on 443.

## wtf is well-known if I don't know what it is?
_"well-known"_ locations are paths outlined in [RFC 8615](https://www.rfc-editor.org/rfc/rfc8615). They are 'reserved' conventional locations that services can query to find information, if present:
- ACME (Automatic Certificate Management Environment) challenges to verify ownership of a domain via HTTP. Simply a check to see if a file at `http://<YOUR_DOMAIN>/.well-known/acme-challenge/<TOKEN>.` contains the correct token to prove ownership of a domain before requests can be made to the CA to obtain a TLS cert. 

- Service discovery: before making a request (e.g: on what hostname and port should I make this request?) 

- App relations - [digital asset links](https://developers.google.com/digital-asset-links/v1/getting-started) allow websites to open content in associated applications rather than the web browser.

In this case (service discovery), for federation between Matrix homeservers; it allows the requesting server to find on what location:port federation should take place. This is outlined in the [server-server](https://spec.matrix.org/v1.2/server-server-api/) API specification.

## delegating to 443 on your actual matrix server instead of 8488 on the second-level domain

### 1. verify failure on 8488
By default, Matrix servers will check your server name, resolve the A/AAAA record to an IP address and make an HTTPS request to that IP on port 8488.

You can use the [matrix federation tester](https://federationtester.matrix.org) and you'll probably get a result like:

`Get "https://103.31.4.5:8448/_matrix/key/v2/server": context deadline exceeded (Client.Timeout exceeded while awaiting headers)`

### 2. make a JSON file on your web server containing the endpoint of your actual matrix server on 443

This is the key/value Matrix homeserver will inspect. I'm using [SWAG](https://hub.docker.com/r/linuxserver/swag), so I've created this file under the root directory `/config/www/synapse/delegation`: 
```json
{
    "m.server": "matrix.example.com:443"
}
```

### 3. add nginx location block to serve the above JSON from the .well-known/ URI 

The Matrix delegation `.well-known/` (remember, the RFC states that each is registered to a 'reserved' conventional location) is at `.well-known/matrix/server`.

So, we'll create a location block in nginx to make this discoverable. You need to specify `default_type` if you did not append the file-type `.json` to your delegation JSON.

```nginx
    location = /.well-known/matrix/server {
        alias /config/www/synapse/delegation;
        default_type application/json;
    }
```

You should then (after restarting nginx), be able to reach this at `https://example.com/.well-known/matrix/server`.

### 4. verify federation is working on 443 now
![Federation Successful](/images/well-known-matrix-synapse/federation-success.png)

You should also see a note that the the test _found_ the .well-known file : 
> _server name/.well-known result contains explicit port number: no SRV lookup done_

If you haven't yet proxied traffic through Cloudflare, feel free to do so and retry the federation tester.
