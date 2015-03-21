<article class="post">
For more than a decade the Web has used XMLHttpRequest (XHR) to achieve
asynchronous requests in JavaScript. While very useful, XHR is not a very nice 
API. It suffers from lack of separation of concerns. The input, output and state
are all managed by interacting with one object, and state is tracked using 
events. Also, the event-based model doesn’t play well with JavaScript’s recent 
focus on Promise- and generator-based asynchronous programming.

The [Fetch API][1] intends to fix most of these problems. It does this by
introducing the same primitives to JS that are used in the HTTP protocol. In 
addition, it introduces a utility function`fetch()` that succinctly captures
the intention of retrieving a resource from the network.

The [Fetch specification][2], which defines the API, nails down the semantics
of a user agent fetching a resource. This, combined with ServiceWorkers, is an 
attempt to:

1.  Improve the offline experience.
2.  Expose the building blocks of the Web to the platform as part of the 
    [extensible web movement][3].

As of this writing, the Fetch API is available in Firefox 39 (currently Nightly
) and Chrome 42 (currently dev). Github has a[Fetch polyfill][4].

## Feature detection

Fetch API support can be detected by checking for `Headers`,`Request`, 
`Response` or `fetch` on the `window` or `worker` scope.

## Simple fetching

The most useful, high-level part of the Fetch API is the `fetch()` function. In
its simplest form it takes a URL and returns a promise that resolves to the 
response. The response is captured as a`Response` object.

| fetch("/data.json").then(function(res) {
      // res instanceof Response ==
true.
      if (res.ok) {
        res.json().then(function(data) {
          console.log(data.entries);
        });
      } else {
        console.log("Looks like the response wasn't perfect, got
status", res.status
);
      }
    }, function(e) {
      console.log("Fetch failed!", e);
    }); |
||

Submitting some parameters, it would look like this:

| fetch("http://www.example.org/submit.php", {
      method: "POST",
      headers: {
        "Content-Type": "application/x-www-form-urlencoded"
      },
      body: "firstName=Nikhil&favColor=blue&password=
easytoguess
"
    }).then(function(res) {
      if (res.ok) {
        alert("Perfect! Your settings are saved.");
      } else if (res.status
== 401
) {
        alert("Oops! You are
not authorized.
");
      }
    }, function(e) {
      alert("Error submitting form!");
    }); |
||

The `fetch()` function’s arguments are the same as those passed to the  
`Request()` constructor, so you may directly pass arbitrarily complex requests
to`fetch()` as discussed below.

## Headers

Fetch introduces 3 interfaces. These are `Headers`, `Request` and  
`Response`. They map directly to the underlying HTTP concepts, but have  
certain visibility filters in place for privacy and security reasons, such as
  
supporting CORS rules and ensuring cookies aren’t readable by third parties.

The [Headers interface][5] is a simple multi-map of names to values:

| var content = "Hello World";
    var reqHeaders = new Headers();
    reqHeaders.append("Content-Type", "text/
plain
"
    reqHeaders.append("Content-Length",
content.length.toString
());
    reqHeaders.append("X-Custom-Header", "
ProcessThisImmediately
"); |
||

The same can be achieved by passing an array of arrays or a JS object literal
  
to the constructor:

| reqHeaders = new Headers({
      "Content-Type": "text/plain",
      "Content-Length": content.length.
toString
(),
      "X-Custom-Header": "
ProcessThisImmediately
",
    }); |
||

The contents can be queried and retrieved:

| console.log(reqHeaders.has("Content-Type")); // true
    console.log(
reqHeaders.has("Set-Cookie")); // false
    reqHeaders.set("
Content-Type", "text/html
");
    reqHeaders.append("X-
Custom-Header", "AnotherValue
");
     
    console.log(reqHeaders.get("Content-Length")); // 11
    console.log(
reqHeaders.getAll("X-Custom-Header")); // ["ProcessThisImmediately", "
AnotherValue
"]
     
    reqHeaders.delete("X-Custom-Header");
    console.log(reqHeaders.getAll("X-
Custom-Header
")); // [] |
||

Some of these operations are only useful in ServiceWorkers, but they provide  
a much nicer API to Headers.

Since Headers can be sent in requests, or received in responses, and have
various limitations about what information can and should be mutable,`Headers`
objects have a**guard** property. This is not exposed to the Web, but it
affects which mutation operations are allowed on the Headers object.  
Possible values are:

*   “none”: default.
*   “request”: guard for a Headers object obtained from a Request (
    `Request.headers`).
*   “request-no-cors”: guard for a Headers object obtained from a Request
    created
     
    with mode “no-cors”.
*   “response”: naturally, for Headers obtained from Response (
    `Response.headers`).
*   “immutable”: Mostly used for ServiceWorkers, renders a Headers object
      
    read-only.

The details of how each guard affects the behaviors of the Headers object are
  
in the [specification][2]. For example, you may not append or set a “request
” guarded Headers’ “Content-Length” header. Similarly, inserting “Set-Cookie” 
into a Response header is not allowed so that ServiceWorkers may not set cookies
via synthesized Responses.

All of the Headers methods throw TypeError if `name` is not a 
[valid HTTP Header name][6]. The mutation operations will throw TypeError if
there is an immutable guard. Otherwise they fail silently. For example:

| var res = Response.error();
    try {
      res.headers.set("Origin", "http://mybank.com");
    } catch(e) {
      console.log("Cannot pretend to be a bank!");
    } |
||

## Request

The Request interface defines a request to fetch a resource over HTTP. URL,
method and headers are expected, but the Request also allows specifying a body, 
a request mode, credentials and cache hints.

The simplest Request is of course, just a URL, as you may do to GET a resource
.

| var req = new Request("/index.html");
    console.log(req.method); // "GET"
    console.log(req.url); // "http://
example.com/index.html
" |
||

You may also pass a Request to the `Request()` constructor to create a copy.  
(This is not the same as calling the `clone()` method, which is covered in  
the “Reading bodies” section.).

| var copy = new Request(req);
    console.log(copy.method); // "GET"
    console.log(copy.url); // "http://
example.com/index.html
" |
||

Again, this form is probably only useful in ServiceWorkers.

The non-URL attributes of the `Request` can only be set by passing initial  
values as a second argument to the constructor. This argument is a dictionary
.

| var uploadReq = new Request("/uploadImage", {
      method: "POST",
      headers: {
        "Content-Type": "image/png",
      },
      body: "image data"
    }); |
||

The Request’s mode is used to determine if cross-origin requests lead to
valid responses, and which properties on the response are readable. Legal mode 
values are`"same-origin"`, `"no-cors"` (default) and `"cors"`.

The `"same-origin"` mode is simple, if a request is made to another origin with
this mode set, the result is simply an error. You could use this to ensure that
  
a request is always being made to your origin.

| var arbitraryUrl = document.getElementById("url-input").value;
    fetch(
arbitraryUrl, { mode: "same-origin" }).then(function(res
) {
      console.
log("Response succeeded?", res.ok
);
    }, function
(e
) {
      console.
log("Please enter a same-origin URL!
");
    }); |
||

The `"no-cors"` mode captures what the web platform does by default for scripts
you import from CDNs, images hosted on other domains, and so on. First, it 
prevents the method from being anything other than “HEAD”, “GET” or “POST”. 
Second, if any ServiceWorkers intercept these requests, they may not add or 
override any headers except for[these][7]. Third, JavaScript may not access any
properties of the resulting Response. This ensures that ServiceWorkers do not 
affect the semantics of the Web and prevents security and privacy issues that 
could arise from leaking data across domains.

`"cors"` mode is what you’ll usually use to make known cross-origin requests
to access various APIs offered by other vendors. These are expected to adhere to
  
the [CORS protocol][8]. Only a [limited set][9] of headers is exposed in the
Response, but the body is readable. For example, you could get a list of Flickr’
s[most interesting][10] photos today like this:

| var u = new URLSearchParams();
    u.append('method', 'flickr.interestingness.
getList
');
    u.append('api_key', '<insert api key
here>
');
    u.append('format', 'json');
    u.append('nojsoncallback', '1');
     
    var apiCall = fetch('https://api.flickr.com/services/rest?' + u);
     
    apiCall.then(function(response) {
      return response.json().then(function
(json
) {
        // photo is a list of photos.
        return json.photos.photo;
      });
    }).then(function(photos) {
      photos.forEach(function(photo) {
        console.log(photo.title);
      });
    }); |
||

You may not read out the “Date” header since Flickr does not allow it via  
`Access-Control-Expose-Headers`.

| response.headers.get("Date"); // null |
||

The `credentials` enumeration determines if cookies for the other domain are  
sent to cross-origin requests. This is similar to XHR’s `withCredentials`  
flag, but tri-valued as `"omit"` (default), `"same-origin"` and `"include"`.

The Request object will also give the ability to offer caching hints to the
user-agent. This is currently undergoing some[security review][11]. Firefox
exposes the attribute, but it has no effect.

Requests have two read-only attributes that are relevant to ServiceWorkers  
intercepting them. There is the string `referrer`, which is set by the UA to be
  
the referrer of the Request. This may be an empty string. The other is  
`context` which is a rather [large enumeration][12] defining what sort of
resource is being fetched. This could be “image” if the request is from an
![][13] tag in the controlled document, “worker” if it is an attempt to
load a worker script, and so on. When used with the`fetch()` function, it is
“fetch
”.

## Response

`Response` instances are returned by calls to `fetch()`. They can also be
created by JS, but this is only useful in ServiceWorkers.

We have already seen some attributes of Response when we looked at `fetch()`.
The most obvious candidates are`status`, an integer (default value 200) and 
`statusText` (default value “OK”), which correspond to the HTTP status code
and reason. The`ok` attribute is just a shorthand for checking that `status` is
in the range 200-299 inclusive.

`headers` is the Response’s Headers object, with guard “response”. The `url`
attribute reflects the URL of the corresponding request.

Response also has a `type`, which is “basic”, “cors”, “default”, “error” or  
“opaque”.

*   `"basic"`: normal, same origin response, with all headers exposed except
      
    “Set-Cookie” and “Set-Cookie2″.
*   `"cors"`: response was received from a valid cross-origin request. 
    [Certain headers and the body][9] may be accessed.
*   `"error"`: network error. No useful information describing the error is
    available. The Response’s status is 0, headers are empty and immutable. This is 
    the type for a Response obtained from
   `Response.error()`.
*   `"opaque"`: response for “no-cors” request to cross-origin resource. 
    [Severely  
    restricted][14] 

The “error” type results in the `fetch()` Promise rejecting with TypeError.

There are certain attributes that are useful only in a ServiceWorker scope. The
  
idiomatic way to return a Response to an intercepted request in ServiceWorkers
is:

| addEventListener('fetch', function(event) {
      event.respondWith(new
Response("Response body
", {
        headers: { "Content-Type
" : "text/plain
" }
      });
    }); |
||

As you can see, Response has a two argument constructor, where both arguments
are optional. The first argument is a body initializer, and the second is a 
dictionary to set the`status`, `statusText` and `headers`.

The static method `Response.error()` simply returns an error response.
Similarly,`Response.redirect(url, status)` returns a Response resulting in  
a redirect to `url`.

## Dealing with bodies

Both Requests and Responses may contain body data. We’ve been glossing over
it because of the various data types body may contain, but we will cover it in 
detail now.

A body is an instance of any of the following types.

In addition, Request and Response both offer the following methods to extract
their body. These all return a Promise that is eventually resolved with the 
actual content.

*   `arrayBuffer()`
*   `blob()`
*   `json()`
*   `text()`
*   `formData()`

This is a significant improvement over XHR in terms of ease of use of non-text
data!

Request bodies can be set by passing `body` parameters:

| var form = new FormData(document.getElementById('login-form'));
    fetch("/
login
", {
      method
: "POST
",
      body:
form
    }) |
||

Responses take the first argument as the body.

| var res = new Response(new File(["chunk", "chunk"], "archive.zip",

||

Both Request and Response (and by extension the `fetch()` function), will try
to intelligently[determine the content type][15]. Request will also
automatically set a “Content-Type” header if none is set in the dictionary.

### Streams and cloning

It is important to realise that Request and Response bodies can only be read
once! Both interfaces have a boolean attribute`bodyUsed` to determine if it is
safe to read or not.

| var res = new Response("one time use");
    console.log(res.bodyUsed); //
false
    res.text().then(function(v) {
      console.log(res.bodyUsed); // true
    });
    console.log(res.bodyUsed); // true
     
    res.text().catch(function(e) {
      console.log("Tried to read already
consumed Response
");
    }); |
||

This decision allows easing the transition to an eventual [stream-based][16]
Fetch API. The intention is to let applications consume data as it arrives, 
allowing for JavaScript to deal with larger files like videos, and perform 
things like compression and editing on the fly.

Often, you’ll want access to the body multiple times. For example, you can
use the upcoming[Cache API][17] to store Requests and Responses for offline use
, and Cache requires bodies to be available for reading.

So how do you read out the body multiple times within such constraints? The API
provides a`clone()` method on the two interfaces. This will return a clone of
the object, with a ‘new’ body.`clone()` MUST be called before the body of the
corresponding object has been used. That is,`clone()` first, read later.

| addEventListener('fetch', function(evt) {
      var sheep = new Response("
Dolly
");
      console.log(sheep.bodyUsed
); // false
      var clone = sheep.clone();
      console.log(clone.bodyUsed); // false
     
      clone.text();
      console.log(sheep.bodyUsed); // false
      console.log(clone.bodyUsed
); // true
     
      evt.respondWith(cache.add(sheep.clone()).then(function(e) {
        return
sheep;
      });
    }); |
||

## Future improvements

Along with the transition to streams, Fetch will eventually have the ability to
abort running`fetch()`es and some way to report the progress of a fetch. These
are provided by XHR, but are a little tricky to fit in the Promise-based nature 
of the Fetch API.

You can contribute to the evolution of this API by participating in discussions
on the[WHATWG mailing list][18] and in the issues in the [Fetch][19] and 
[ServiceWorker][20] specifications.

For a better web!

*The author would like to thank Andrea Marchesini, Anne van Kesteren and Ben  
Kelly for helping with the specification and implementation.*</article>

 [1]: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
 [2]: https://fetch.spec.whatwg.org
 [3]: https://extensiblewebmanifesto.org/
 [4]: https://github.com/github/fetch
 [5]: https://fetch.spec.whatwg.org/#headers-class
 [6]: https://fetch.spec.whatwg.org/#concept-header-name
 [7]: https://fetch.spec.whatwg.org/#simple-header
 [8]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS
 [9]: https://fetch.spec.whatwg.org/#concept-filtered-response-cors
 [10]: https://www.flickr.com/services/api/flickr.interestingness.getList.html
 [11]: https://github.com/slightlyoff/ServiceWorker/issues/585
 [12]: https://fetch.spec.whatwg.org/#requestcredentials
 [13]: img/
 [14]: https://fetch.spec.whatwg.org/#concept-filtered-response-opaque
 [15]: https://fetch.spec.whatwg.org/#concept-bodyinit-extract
 [16]: https://streams.spec.whatwg.org/

 [17]: http://slightlyoff.github.io/ServiceWorker/spec/service_worker/index.html#cache-objects
 [18]: https://whatwg.org/mailing-list

 [19]: https://www.w3.org/Bugs/Public/buglist.cgi?product=WHATWG&component=Fetch&resolution=---
 [20]: https://github.com/slightlyoff/ServiceWorker/issues