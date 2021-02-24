## What  
CORS (Cross-Origin Resource Sharing) is a security relaxation (based on HTTP-header) for the default browser security mechanism [same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy), which it does not allow requests to a different host other than the one JS is running within.


## Why
As in `What`, it is to allow requests to go to a different host for some legit scenarios as commonly seen in the SPA setup.   
It is also good to know why same-origin needed, because otherwise some malicious page (hackerdomain.com) can be sent to user, and issues a request to yourdomain.com right from user's browser. If yourdomain uses session cookie to store some sensitive data (e.g authed token), hackerdomain.com could just use it along with the request and get the response or perform some actions that should not be publicly available.    


## How it works
We need to configure the server to respond with certain headers `Access-Control-*`, so browser knows it can continue the request and consume the response.   
- `Access-Control-Allow-Origin`   
    `*` if the request can be sent from any origin to the server
- `Access-Control-Allow-Methods`   
    A comma-separated HTTP headers that are allowed to talk to the server 
- `Access-Control-Allow-Headers`   
    A comma-separated list of the custom headers that are allowed to be sent 
- `Access-Control-Allow-Max-Age`   
    The maximum duration that the response to the preflight request can be cached before another call is made

The general flow looks like:  

Browser checks if the request is a **'simple'** request, a request is simple when:    
▪ The request method is either of `GET`, `POST` or `HEAD`   
▪ [CORS safe header](https://fetch.spec.whatwg.org/#cors-safelisted-request-header) or no custom header   
▪ Content-Type header is either of `application/x-www-form-urlencoded`, `multipart/form-data` or `text/plain`

If it is a simple request, it will be directly sent as normal and `Allow-Control-*` headers will be checked when response comes back.   
Otherwise the browser will send a **preflight** request with `OPTIONS` http method. This is used to determine what the server is supporting, if the response of the `OPTIONS` showing not allowed, the following request won't be sent at all. It's like domain needs to be approved before even sending a request.

An example
```sh
curl -i -X OPTIONS https://api.domain.com \
-H 'Access-Control-Request-Method: POST' \
-H 'Access-Control-Request-Headers: Content-Type, Accept, Authorization' \
-H 'Origin: https://api.domain.com'
```
It basically says: "I want to make a POST request to https://api.domain.com" with `Content-Type`, `Accept` and custom `Authorization` headers, is this allowed?"   
If the server responded with `Access-Control-*` headers covering those requested in `OPTIONS`, the actual POST request will be sent.
