# Storage Policy API.

Mike O'Neill, April 2019

Contributors:
* Mike O'Neill <[michael.oneill@baycloud.com](mailto:michael.oneill@baycloud.com)>


## Introduction
How applications deploy user agent storage has major privacy implications for privacy, for example
see the ease with which mechanisms like cookies, designed to support session state persistence, 
have been used to enable invisible long term surveillance. 

This has long been recognised by privacy regulators and legislators, 
and laws have been introduced requiring web sites to manage storage in privacy protective ways, 
for example by requiring prior user consent before storage items, such as cookies, 
are used (unless they are for an exempted purpose).

Unfortunately sites are usually built from a variety of components, such as embedded elements and external scripts,
often supplied by third-party service providers, and these can employ user agent storage outside the control, and sometimes
even the awareness, of site managers.

This can make it difficult for them to make enforceable commitments to the site's users, 
inevitably making the commitments they do make seem inaccurate or even dishonest, leading to a loss of trust in sites, brands and publishers.

This document defines a mechanism via which a site can control user agent storage by setting overall limits on its use, 
whether it is associated with the top level site's origin or an embedded "third-party" origin.

Web applications can already ensure that all data associated with a controlled origin is removed when no longer required 
by using the Clear Site Data API. 
By including a `Clear-Site-Data` header in a response all or particular categories of stored data can be immediately removed,
including cookies, local storage, indexed database storage, JavaScript execution contexts, the client-side cache etc.

This document describes a mechanism whereby the data can be automatically deleted after a specified period after the user 
has stopped interacting with a site,
without requiring a further request to the controlling origin.

This would allow applications to store data, such as cookies or local storage,
for short periods to support required functionality such as  "session state",
but ensure this data is not retained beyond the minimum period required to persist a session. 

The cookie API already contains an expiry mechanism, i.e. the `expires` or `max-age` attribute,
though this does not stop cookies from being perpetually regenerated. 
This API not only extends a similar mechanism to all origin controlled storage, 
i.e. all the storage types controlled by the `Clear-Site-Data` API,
but also ensures the expiration timer is held off  while the user is interacting with the site. 

## Storage-Policy response header
Sites can limit access to default limits storage by returning a `Storage-Policy` 
response header of the following form:

`Storage-Policy: max-age 3600;inactivity-timeout 1800;allow-cookies consent preference;allow-origins example.com `

The header's value contains a **storage policy descriptor**, 
consisting of one or more **storage policy attributes** separated by semicolons `;`.


Where the **storage policy attributes** have the following meaning

| attribute        | default    | description  |
| :-------------: |:-------------:| :----- |
| max-age      | 86,400 (24 hours)| Number of seconds after header received before all top-level origin storage is cleared, irrespective of user activity. |
| inactivity-timeout      | 3600 ( 1 hour)| Number of seconds of no user activity before all top-level origin storage is cleared. The timer is restarted (unless expired) on detection of user activity. |
| allow-cookies | 'none' |   space separated list of names of cookies that are not to be deleted by this mechanism, or 'none' if all cookies are to be deleted.|
| allow-origins | 'none' |   space separated list of origins that can receive and place cookies, and whose browsing contexts can access other storage, or 'none' if no subresource may access storage. |

If a `Storage-Policy` header contains an attribute specifying a **less restrictive** storage policy than the default, 
i.e. a higher value for the `inactivity-timeout` or `max-age` attributes, or an `allow-origins` other than `'none'`,
the user agent MUST alert the user in some way, such as with a permission prompt
similar to the prompt resulting from a call by a nested browsing context to the `Storage Access` 
API in webkit's Intelligent Tracking Prevention (ITP) feature.

If a site determines that a user has already given their consent to storage access, 
then it may be able to avoid a separate prompt resulting from a less restrictive `Storage-Policy` 
by using an API to set a verifiable and universal consent signal, such as `DNT` or its replacement.

If a top level browsing context has a `Storage-Policy` header in its initiating response
then subresources (other than those indicated by an `allow-origins` attribute),
will have any `Set-Cookie` headers in the response ignored, 
any request to the origin will not include any `Cookies` headers, and any resulting
browsing context will have no access to terminal storage. 

Storage associated with such a subresource will not be deleted, 
but cookies will not be communicated or set, and any associated nested browsing context will not have access to any storage.

When either of the timers completes all top-level origin storage, other than cookies indicated by a `allow-cookies` list, is deleted.
This would have the same effect as if the user agent had received a `Clear-Site-Data: ,"cache", "cookies", "storage", "executionContexts"`
in the response to the next HTTP request to the origin after the user has stopped interacting for the `inactivity-timeout` period.


## JavaScript API
### Determine current duration limits.
Script can execute the following JavaScript function to determine the storage durations associated with the script origin.

```
var storagePolicyObject = navigator.storagePolicy.Get()
```

The returned object would contain the duration currently in effect:
```
{
          maxAge: 86400,
          inactivityTimeout: 3600
}
```
### Change current duration limits
Script can change the current storage policy duration, 
but only if the browsing context was initialised **without** an associated `Storage-Policy` header.
If there was a `Storage-Policy` header the user agent MUST ignore the call.
```
navigator.storagePolicy.Update(dictionary);
```
where `dictionary` is an object indicating the required duration for specified category types:
```
{
          maxAge: 3600,
          inactivityTimeout: 1800
}

```



## Prior Art
*  The Clear-Site-Data API defines a mechanism to remove data from local storage, 
giving web developers the ability to clear out a user’s local cache of data via the Clear-Site-Data HTTP response header. 
"[Clear Site Data](https://www.w3.org/TR/clear-site-data/)"

*   Mike West has proposed a mechanism which allows HTTP servers to maintain stateful sessions with HTTP user agents, 
addressing some of the security and privacy considerations of HTTP Cookies. 
"[HTTP State Tokens](https://mikewest.github.io/http-state-tokens/draft-west-http-state-tokens.html)" 

*   The Tracking Protection Working Group discussed limits on all browser storage (identifiers) to a default short duration when DNT:1 was set
e.g. 
"[public-tracking@w3.org Mail Archives](https://lists.w3.org/Archives/Public/public-tracking/2013Jun/0262.html)" 

*   Safari's ITP 2.1 limits the duration of first-party cookies created by script to a default 7 days duration. 
"[Intelligent Tracking Prevention 2.1](https://webkit.org/blog/8613/intelligent-tracking-prevention-2-1/)"



  
