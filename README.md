# whatwg

Notes to keep track of the things I learn while working on and learning about web standards

# Working on Fetch domintro boxes

> Learning about WebIDL [union-types](https://heycam.github.io/webidl/#idl-union), the
> [es-union](https://heycam.github.io/webidl/#es-union) algorithm, and
> [stringifiers](https://url.spec.whatwg.org/#URL-stringification-behavior).

While working on the fetch standard's domintro boxes (https://github.com/whatwg/fetch/issues/543) I
ran down a few rabbit holes. While reading the description of the
[`Request`](https://fetch.spec.whatwg.org/#requests) object I had noticed it mentioned that a `Request`
is the input to the fetch API. I had obviously used a regular string as the first (and sometimes only)
parameter to the fetch API before as have many people, so upon reading the spec I was curious as to how
this conversion was taking place. I had also vaguely recalled seeing a `URL` object being used as the input
parameter for the fetch API, which added to the curiousity. A member of the WHATWG organization pointed out
over IRC that step 2 of the [fetch method](https://fetch.spec.whatwg.org/#fetch-method) section of the spec indicated
that whatever we passed in as `input` always went through the
[`Request` constructor](https://fetch.spec.whatwg.org/#dom-request) to sort of sanitize the input. This means
that the following calls to fetch:

 - `fetch('https://domfarolino.com')`
 - `fetch(new URL('https://domfarolino.com'))`
 - `fetch(new Request('https://domfarolino.com'))`

Are the same as:

 - `fetch(new Request('https://domfarolino.com'))`
 - `fetch(new Request(new URL('https://domfarolino.com')))`
 - `fetch(new Request(new Request('https://domfarolino.com')))`

At this point my confusion about being able to pass in string, `URL`, and `Request` objects was still with me
but had shifted focus to the `Request` constructor as opposed to the fetch API. What specifically in spec-land
allows us to handle this? When looking at this constructor, I noticed that step 5 handles the case where `input`
is a string, while step 6 handles the case where `input` is an object of type `Request`. So here I wondered how,
if we accept string and `Request` objects, are we able to accept something like a `URL` object? The short answer
is that WebIDL stringifies everything that gets passed into a method taking a `DOMString`/`USVString`. The `URL`
object happens to have a custom [stringifier](https://url.spec.whatwg.org/#URL-stringification-behavior) which
returns the `URL`'s `href` attribute upon string coercian. This is nice because it spits out a type that the `Request`
constructor is designed to take.

I was then curious as to what stopped *`Request`* objects being passed into the `Request` constructor from undergoing
the same stringification as `URL` objects, since the string type is what we first look for. To understand this we have
to look at the WebIDL [es-union](https://heycam.github.io/webidl/#es-union) algorithm. In short, this algorithm defines
the steps to run when we convert an ECMAScript value to one of several IDL types single targeted type. These types are
specified in WebIDL as a [union type](https://heycam.github.io/webidl/#idl-union). The `Request` class IDL in the fetch
spec defines its `input` parameter as an object of type [RequestInfo](https://fetch.spec.whatwg.org/#requestinfo), which
is a union type of `Request or USVString`. The reason `Request` objects are not stringified like `URL` objects are is due
to step 4, substep 1 of the es-union algorithm. In short, this algorithm will favor interface types that ECMAScript object
implements before trying to stringify.

----

# `for=/`
I wondered what the purpose of `for=/` was in `<a for=/>referrer policy</a>`. After a quick read of the bikeshed
documentation I learned that it was to link to a `<dfn>` which doesn't have a `for` attribute. We use it in cases
where there exist ambiguous `<dfn>` tags. In this particular case, both the
<a href=https://w3c.github.io/webappsec-referrer-policy/>referrer policy</a> and fetch specifications define the term
**referrer policy**. The referrer policy spec defines [it](https://w3c.github.io/webappsec-referrer-policy/#referrer-policy)
as a regular definition with no `for` attribute representing an enum, however fetch defines it as a concept associated with
"`request`" objects. Therefore later in the spec when we are referring to a value that must be of type `referrer policy`
(as in, a value that exists in the enum defined by the referrer policy spec) we must link our term to the correct definition
since there is a little ambiguity. We don't want to refer to the concept associated with (`for`) "`request`" objects, so we
explicitly tell bikeshed we want to be linked to the `<dfn>` that doesn't have a `for` attribute. Bam.

# Utility of fetch `{mode: "no-cors"}` and `{redirect: "manual"}`

While reading the fetch standard, I was wondering what the point of Requests with things like `{mode: "no-cors"}`,
and `{redirect: "manual"}` were. Upon looking both things up independently, I learned that most had little or no
utility in general web application code, but existed because:

 - Web browsers make use of them internally for other requests (since the Fetch spec defines a general request object)
 - They could be of use when dealing with a ServiceWorker

I was a tad vexed to find that they both gave me similar findings (people saying typically you don't want to use them
unless doing something tricky/niche with a ServiceWorker), which made me curious as to why this was.
[This](https://stackoverflow.com/questions/42716082/fetch-api-whats-the-use-of-redirect-manual/42717388#42717388)
stackoverflow answer was particularly useful and helped me remember that yes, other standards (like HTML) use the
Fetch specification to make internal requests that are not directly exposed. For example, in a blog post, Jake Archibald
points out that `<img>` makes requests whose mode is `no-cors`, whereas `<img crossorigin>` makes requests whose mode is
`cors`. This is useful because a lot of CDN content is not familiar with the CORS protocol, so the `<img>` element is happy
with an opaque response so long as it can display it to the user without exposing it to application code. Prior to
ServiceWorkers, we wouldn't have much of a reason to kick off a request for an image via `fetch` that returns an opaque
response, however with ServiceWorkers, this becomes commonplace.
[This](https://github.com/domfarolino/pwa-meetup/blob/05-Full-Offline/public/sw.js#L3) example shows this behavior in action,
where we want to retrieve an image (or some x-origin) assets right when a ServiceWorker is registered so we can serve them
(opaquely) later in response to DOM requests for the same assets.

Furthermore, the GitHub issue in the above stackoverflow answer points out why a redirect mode of "manual" can be useful
in web application code in a few corner cases where ServiceWorkers need to handle redirects in a way that is consistent
with the rest of the platform.
