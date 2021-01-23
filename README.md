# Inline Cache

The inline cache methodology emphasises fast first page loads. It acts on the basis that opening HTTP connections is slow, and that it is therefore faster for the resources to be inlined, but that we also want to cache those resources so they don't have to be re-downloaded.

Using a combination of client-side JavaScript, a background [service worker](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API), the [Cache Web API](https://developer.mozilla.org/en-US/docs/Web/API/Cache), cookies, and server-side logic, it is possible to conditionally render resources inline on first page load, then resurface those resources via the cache when needed. This gives us the best of both worlds.

## Real world implementations of inline cache

- [inline-cacher](https://github.com/ChrisBAshton/inline-cacher): prototype NPM module
- [wp-inline-cacher](https://github.com/ChrisBAshton/wp-inline-cacher): WordPress plugin

## How it works

The user requests a web page for the first time:

1. The server detects that there are no cookies in the request, since this is a first-time visitor.
1. The server therefore renders each of the page's resources inline, with some data attributes that will help us identify the resource later.
   For example:
   ```html
   <style data-inline-cache="1.0.0" data-src="https://example.com/style.css">
     body {
       color: red;
     }
   </style>
   ```
1. On the client, the page is rendered, and some JavaScript queries the DOM for inline resources such as the one above.
1. The JavaScript initialises a service worker.
1. The JavaScript stores the contents of each inline resource in the cache, so that it can be retrieved later.
1. The JavaScript stores a manifest of all inlined resources in the cache, so that it can be easily queried later.
1. The JavaScript creates a cookie which captures a list of all of the inline resources that have been cached.

Now the user refreshes the page, clicks on a link, or simply returns to the site at a later date:

1. The cookie is sent as part of the user's request.
1. The server detects the cookie and knows which versions of which resources have been cached locally.
1. For any page resources which have been cached already, the server can render a link to the resource.
   For example:
   ```html
   <link
     rel="stylesheet"
     href="https://example.com/style.css"
     data-inline-cache="1.0.0"
   />
   ```
1. On the client, the page begins rendering, and initialises the service worker.
1. The service worker retrieves the manifest from the cache to retrieve a list of resources and versions that have been cached already.
1. The service worker intercepts all requests. For each in turn, it checks if the resource exists in the manifest. If it does, it fetches it from the cache, otherwise it fetches it over HTTP as normal.

### What happens if the cache is cleared?

If the cache is cleared but the cookie still references cached resources:

1. The server would think the client has it cached, so render a link to the resource rather than render it inline.
1. The service worker would fail to find the resource in the cache, so would fetch the resource over HTTP.
1. This shouldn't ever be worse than normal browser behaviour, assuming the usual cache-control headers are set on the resource.
1. Additional logic could be added to have the client side JavaScript refresh its manifest/cookie contents.

### What happens if the cookie is deleted?

If the cookie is deleted but the cache contains the resources:

1. The server would think the client has nothing cached, so would render the contents inline.
1. The client side JavaScript would cache the inline resources as described.

### What happens if a version of an resource is changed?

If an resource is cached but then gets updated server-side:

1. The resource should have a new version associated with it.
1. The server would see that the cookie references an old version of the resource.
1. The server would render the resource inline.
1. The client side JavaScript would cache the resource, overwriting the previous cache entry for that resource URL.

## Impact

This idea is in its infancy; a lot more testing is required. Early tests suggest there isn't much of a difference in page load times on a fast connection: timings and bytes transferred are similar.

However, on a throttled connection, inline cache can dramatically improve the time to first meaningful paint.

### Paint performance

We can see in the performance tab below, a test site with inline cache disabled:

<img width="1409" alt="throttled without" src="https://user-images.githubusercontent.com/5111927/105545454-fe687900-5cf3-11eb-81d6-f3240696eeb0.png">

It takes just over 4 seconds for the First Paint, and just over 5 seconds for the Largest Contentful Paint.

Now let's compare with inline cache enabled:

<img width="1408" alt="throttled with" src="https://user-images.githubusercontent.com/5111927/105545438-fa3c5b80-5cf3-11eb-8aee-f1b911e4c566.png">

We get our First Paint in around 3 seconds, and our Largest Contentful Paint in under 4 seconds.

So we're potentially seeing a 20-25% improvement in paint times.

### Page load times and transfer stats

From a cold cache, with cleared cookies and disabled cache between visits on a test website:

#### Without inline cache

3.1 MB was transferred on each test, over a total of 47-49 requests.

| Attempt number | Finish | DOMContentLoaded | Load  |
| -------------- | ------ | ---------------- | ----- |
| 1              | 3.59s  | 1.22s            | 2.07s |
| 2              | 4.23s  | 2.10s            | 2.90s |
| 3              | 2.90s  | 919ms            | 1.67s |
| 4              | 2.91s  | 971ms            | 1.68s |

#### With inline cache

3.2 MB was transferred on each test, over a total of 46-47 requests.

| Attempt number | Finish | DOMContentLoaded | Load  |
| -------------- | ------ | ---------------- | ----- |
| 1              | 3.23s  | 1.30s            | 2.07s |
| 2              | 3.40s  | 1.32s            | 2.04s |
| 3              | 3.01s  | 908ms            | 1.72s |
| 4              | 2.94s  | 1.01s            | 1.74s |

In summary, the page load times aren't too different on a fast connection.
