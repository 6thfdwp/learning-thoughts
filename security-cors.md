## What  
CORS (Cross-Origin Resource Sharing) is a security relaxation for the default browser security mechanism [same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy), which it does not allow requests to a different host other than the one JS is running at.


## Why
As in `What`, it is to allow requests to go to a different host for some legit scenarios as commonly seen in the SPA setup.   
It is also good to know why same-origin needed, because otherwise some malicious page (hackerdomain.com) can be sent to user, and from there issues a request to yourdomain.com. And if yourdomain uses session cookike to store some sensitive data (e.g authed token), hackerdomain.com could just use it along with the request and get the response or perform some actions that should not be publicly available.    


## How it works
We need to configure the server to respond with certain headers `Access-Control-*`, so browser knows it can continue the request and consume the response.   
▪ `Access-Control-Allow-Origin`   
▪ `Access-Control-Allow-Methods`   
▪ `Access-Control-Allow-Headers`   
▪ `Access-Control-Allow-Max-Age`   

The general flow looks like:  

Browser checks if the request is a **'simple'** request, a request is simple when:    
▪ The request method is either of `GET`, `POST` or `HEAD`   
▪ [CORS safe header](https://fetch.spec.whatwg.org/#cors-safelisted-request-header) or no custom header   
▪ Content-Type header is either of `application/x-www-form-urlencoded`, `multipart/form-data` or `text/plain`

If it is a simple request, it will be directly sent as normal and `Allow-Control-*` headers will be checked when response comes back.   
Otherwise the browser will send an **preflight** request with `OPTIONS` http method. This is used to determine what the server is supporting, if the response of the `OPTIONS` showing not allowed, the following request won't be sent at all. 

An example
```sh
curl -i -X OPTIONS https://api.domain.com \
-H 'Access-Control-Request-Method: POST' \
-H 'Access-Control-Request-Headers: Content-Type, Accept, Authorization' \
-H 'Origin: https://api.domain.com'
```
It basically says: "I want to make a POSTrequest to https://api.domain.com" with `Content-Type`, `Accept` and custom `Authorization` headers, is this allowed?"   
If the server responded with `Access-Control-*` headers covering those requested in `OPTIONs`, the actual POST request will be sent.
